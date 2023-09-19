In the following steps, we will gradually build a minimal `go-dqlite` application that provides an HTTP endpoint to read and write values in a Dqlite cluster.

>You can find the final code here: https://github.com/canonical/go-dqlite/blob/master/cmd/dqlite-demo/dqlite-demo.go  <!-- wokeignore:rule=master --> 

## Create a minimal starting point

We can now begin building our example application. In your project folder, create a file called `go-dqlite-demo.go` and place the following code in it:

```Go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"

	"github.com/canonical/go-dqlite/app"
)

func main() {
	dir := "/tmp/dqlite-data"
	address := "127.0.0.1:9001" // Unique node address

	// Set up Dqlite application
	app, err := app.New(dir, app.WithAddress(address))
	if err != nil {
		log.Fatal(err)
	}
	log.Println("App created")

	// Create a database 'my-database' or just open it if
	// it already exists.
	db, err := app.Open(context.Background(), "my-database")
	if err != nil {
		log.Fatal(err)
	}
	log.Println("Database created")

	// Execute a SQL command on the database.
	// Creates a table 'my_table'
	if _, err := db.Exec("CREATE TABLE my_table (n INT)"); err != nil {
		log.Fatal(err)
	}
	log.Println("Table created")

	// wait until we received a termination signal
	ch := make(chan os.Signal, 32)
	signal.Notify(ch, unix.SIGPWR)
	signal.Notify(ch, unix.SIGINT)
	signal.Notify(ch, unix.SIGQUIT)
	signal.Notify(ch, unix.SIGTERM)

	<-ch

	db.Close()
	app.Close()
}
```

This code will create a Dqlite app, a database, and a table within that database. 

To run this example, first create the directory `/tmp/dqlite-data` with the following command:

```bash
mkdir -p /tmp/dqlite-data
```

Then, use the following command from within your project folder to run the application:

```bash
go run go-dqlite-demo.go
```

This should produce an output similar to the following:
```bash
ubuntu@dqlite-tutorial:~/go-dqlite-demo$ go run go-dqlite-demo.go 
2023/09/11 14:40:46 App created
2023/09/11 14:40:46 Database created
2023/09/11 14:40:46 Table created
```

>Rerunning this app will result in an error since the table already exists. 

Congratulations! You have created your first Dqlite application in Go.

### Add clustering

Since Dqlite is a distributed database, it makes sense to have multiple nodes (a cluster) spun up. 
A cluster of nodes can simply be created by providing the IP address of the leader node when creating a node (thus, joining a cluster).
In the previous example, we hard-coded the address directly in the code. Each node needs a unique IP address, so we need to make this value configurable.
Let's use [Cobra](https://github.com/spf13/cobra) to turn the previous example into a CLI application to which we can pass flags.

>An introduction to Cobra is out of scope for this tutorial. Visit the [official documentation](https://cobra.dev/#getting-started) for more information on this library.

Install the dependency with:
```bash
go get -u github.com/spf13/cobra/cobra
```

Now, move the code into a Cobra command. Expose the `--db` and `--join` flags to set the unique node address and other nodes in the cluster, respectively. 

```Go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"path/filepath"

	"github.com/canonical/go-dqlite/app"
	"github.com/spf13/cobra"
	"golang.org/x/sys/unix"
)

func main() {
	var db string
	var join *[]string
	var dir string

	cmd := &cobra.Command{
		Use:   "go-dqlite-demo",
		Short: "Demo application using Dqlite",

		RunE: func(cmd *cobra.Command, args []string) error {
			dir := filepath.Join(dir, db)
			if err := os.MkdirAll(dir, 0755); err != nil {
				return fmt.Errorf("can't create %s: %v", dir, err)
			}

			// Set own address and specify all existing nodes in the cluster.
			options := []app.Option{app.WithAddress(db), app.WithCluster(*join)}

			// Set up Dqlite application
			app, err := app.New(dir, options...)
			if err != nil {
				return err
			}
			log.Println("App created")

			// Create a database 'my-database' or just open it if
			// it already exists.
			db, err := app.Open(context.Background(), "my-database")
			if err != nil {
				log.Fatal(err)
			}
			log.Println("Database created")

			// wait until we received a termination signal
			ch := make(chan os.Signal, 32)
			signal.Notify(ch, unix.SIGPWR)
			signal.Notify(ch, unix.SIGINT)
			signal.Notify(ch, unix.SIGQUIT)
			signal.Notify(ch, unix.SIGTERM)

			<-ch

			db.Close()
			// Transfer all responsibilities (leader/voting rights) to other node
			app.Handover(context.Background())
			app.Close()

			return nil
		},
	}

	flags := cmd.Flags()
	flags.StringVarP(&db, "db", "d", "", "address used for internal database replication")
	join = flags.StringSliceP("join", "j", nil, "database addresses of existing nodes")
	flags.StringVarP(&dir, "dir", "D", "/tmp/dqlite-demo", "data directory")
	cmd.MarkFlagRequired("db")

	if err := cmd.Execute(); err != nil {
		os.Exit(1)
	}
}
```

Let's test this code by starting multiple nodes:

```bash
# Start the first node (as a background process)
go run go-dqlite-demo.go --db 127.0.01:9001 &

# Start the second node and automatically join it 
# with the first node to a form a cluster.
go run go-dqlite-demo.go --db 127.0.01:9002 --join 127.0.0.1:9001 &
```

That's it, you have successfully created your first Dqlite cluster!
Let's check out how we can interact with it in the next section.

### Exposing a simple HTTP interface 

As a final step in this tutorial, we want to provide a simple way to interact with the database. For that, a simple HTTP endpoint is exposed that accepts `GET` and `PUT` requests to read and write values from/to the database.

First, define the SQL command templates:
```Go
const (
	schema = "CREATE TABLE IF NOT EXISTS model (key TEXT, value TEXT, UNIQUE(key))"
	query  = "SELECT value FROM model WHERE key = ?"
	update = "INSERT OR REPLACE INTO model(key, value) VALUES(?, ?)"
)
```
Those templates will serve as the frame for the SQL commands that are executed against the database.

Extend the existing Cobra command with the following HTTP handler:

```Go
// Create a database 'my-database' or just open it if
// it already exists.
db, err := app.Open(context.Background(), "my-database")
if err != nil {
    log.Fatal(err)
}

+ // Create the database schema if it doesn't exist yet.
+ if _, err := db.Exec(schema); err != nil {
+    return err
+ }
+
+ // HTTP endpoint to provide basic Read/write operations according to
+ // a fixed SQL schema.
+ http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
+     key := strings.TrimLeft(r.URL.Path, "/")
+     result := ""
+     switch r.Method {
+     case "GET":
+         row := db.QueryRow(query, key)
+         if err := row.Scan(&result); err != nil {
+             result = fmt.Sprintf("Error: %s", err.Error())
+         }
+         break
+     case "PUT":
+         result = "done"
+         value, _ := io.ReadAll(r.Body)
+         if _, err := db.Exec(update, key, string(value[:])); err != nil {
+             result = fmt.Sprintf("Error: %s", err.Error())
+         }
+     default:
+         result = fmt.Sprintf("Error: unsupported method %q", r.Method)
+ 
+     }
+     fmt.Fprintf(w, "%s\n", result)
+ })
+ 
+ listener, err := net.Listen("tcp", api)
+ if err != nil {
+     return err
+ }
+ 
+ go http.Serve(listener, nil)

// wait until we received a termination signal
ch := make(chan os.Signal, 32)
signal.Notify(ch, unix.SIGPWR)
signal.Notify(ch, unix.SIGINT)
signal.Notify(ch, unix.SIGQUIT)
signal.Notify(ch, unix.SIGTERM)
```

>Read more on [HTTP servers in Go](https://pkg.go.dev/net/http).

Finally, add an `api` flag that defines the address of the HTTP API. The final code looks like this:

```Go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"path/filepath"
	"strings"

	"github.com/canonical/go-dqlite/app"
	"github.com/spf13/cobra"
	"golang.org/x/sys/unix"
)

const (
	schema = "CREATE TABLE IF NOT EXISTS model (key TEXT, value TEXT, UNIQUE(key))"
	query  = "SELECT value FROM model WHERE key = ?"
	update = "INSERT OR REPLACE INTO model(key, value) VALUES(?, ?)"
)

func main() {
	var db string
	var api string
	var join *[]string
	var dir string

	cmd := &cobra.Command{
		Use:   "go-dqlite-demo",
		Short: "Demo application using Dqlite",

		RunE: func(cmd *cobra.Command, args []string) error {
			dir := filepath.Join(dir, db)
			if err := os.MkdirAll(dir, 0755); err != nil {
				return fmt.Errorf("can't create %s: %v", dir, err)
			}

			// Set own address and specify all existing nodes in the cluster.
			options := []app.Option{app.WithAddress(db), app.WithCluster(*join)}

			// Set up Dqlite application
			app, err := app.New(dir, options...)
			if err != nil {
				return err
			}
			log.Println("App created")

			// Create a database 'my-database' or just open it if
			// it already exists.
			db, err := app.Open(context.Background(), "my-database")
			if err != nil {
				log.Fatal(err)
			}

			// Create the database schema if it doesn't exist yet.
			if _, err := db.Exec(schema); err != nil {
				return err
			}

			// HTTP endpoint to provide basic Read/write operations according to
			// a fixed SQL schema.
			http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
				key := strings.TrimLeft(r.URL.Path, "/")
				result := ""
				switch r.Method {
				case "GET":
					row := db.QueryRow(query, key)
					if err := row.Scan(&result); err != nil {
						result = fmt.Sprintf("Error: %s", err.Error())
					}
					break
				case "PUT":
					result = "done"
					value, _ := io.ReadAll(r.Body)
					if _, err := db.Exec(update, key, string(value[:])); err != nil {
						result = fmt.Sprintf("Error: %s", err.Error())
					}
				default:
					result = fmt.Sprintf("Error: unsupported method %q", r.Method)

				}
				fmt.Fprintf(w, "%s\n", result)
			})

			listener, err := net.Listen("tcp", api)
			if err != nil {
				return err
			}

			go http.Serve(listener, nil)

			// wait until we received a termination signal
			ch := make(chan os.Signal, 32)
			signal.Notify(ch, unix.SIGPWR)
			signal.Notify(ch, unix.SIGINT)
			signal.Notify(ch, unix.SIGQUIT)
			signal.Notify(ch, unix.SIGTERM)

			<-ch

			db.Close()
			// Transfer all responsibilities (leader/voting rights) to other node
			app.Handover(context.Background())
			app.Close()

			return nil
		},
	}

	flags := cmd.Flags()
	flags.StringVarP(&api, "api", "a", "", "address used to expose the demo API")
	flags.StringVarP(&db, "db", "d", "", "address used for internal database replication")
	join = flags.StringSliceP("join", "j", nil, "database addresses of existing nodes")
	flags.StringVarP(&dir, "dir", "D", "/tmp/dqlite-demo", "data directory")
	cmd.MarkFlagRequired("db")

	if err := cmd.Execute(); err != nil {
		os.Exit(1)
	}
}
```

Let's spin up three instances of the application to verify that everything works as expected:

```bash
go run go-dqlite-demo.go --api 127.0.0.1:8001 --db 127.0.0.1:9001 &
go run go-dqlite-demo.go --api 127.0.0.1:8002 --db 127.0.0.1:9002 --join 127.0.0.1:9001 &
go run go-dqlite-demo.go --api 127.0.0.1:8003 --db 127.0.0.1:9003 --join 127.0.0.1:9001 &
```

You should now be able to set values:

```bash
ubuntu@dqlite-tutorial:~$ curl -X PUT -d my-value http://127.0.0.1:8001/my-key
done
```

And to retrieve them:
```bash
ubuntu@dqlite-tutorial:~$ curl http://127.0.0.1:8001/my-key
my-value
```

The first node is currently the leader. Let's kill it to check if fail-over to other nodes works as expected:

```bash
ubuntu@dqlite-tutorial:~$ kill -TERM %1; curl http://127.0.0.1:8002/my-key
my-value
```

As you can see, the leader node was killed but we can still retrieve the value from the other API servers.

That's all for this tutorial. Check out the Explanation and Reference sections to find out more.