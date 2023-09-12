# Set up your development environment

The following section walks you through the necessary steps to set up your environment.
It is recommended to spin up a clean Ubuntu VM for these steps as it is always good to perform experiments in an isolated development environment.
Follow the steps in [Set up an Ubuntu VM with Multipass](https://juju.is/docs/sdk/set-up-your-development-environment#heading--set-up-an-ubuntu-vm-with-multipass) to do this.

## Create a project directory

Create a project directory with
```bash
mkdir ~/go-dqlite-demo
cd ~/go-dqlite-demo
```

This creates a folder called `go-dqlite-demo` in your home directory and enters it. All project files will be put in this directory.

## Install the Dqlite tooling

On a Debian-based system, you can get the latest development release from [Dqlite's development PPA](https://launchpad.net/~dqlite/+archive/ubuntu/dev). First add the PPA repository:

```bash
sudo add-apt-repo ppa:dqlite/dev
sudo apt update
```

Then install the library:

```bash
sudo apt install libdqlite-dev
```

[note type="important" status="Info"]
If you are not using a Debian-based system, you must build `libdqlite` from source. See [Build](https://github.com/canonical/dqlite#build) for instructions on how to do this.
[/note]

# Install Go 

Download and install the latest version of Go following the steps outlined in the [Go documentation](https://go.dev/doc/install).
You can verify that everything is installed correctly by typing the following command in your terminal:

```bash
> go version
go version go1.21.1 linux/amd64
```
Check that the output matches the version that you downloaded earlier.

In your project folder, create a mod file with:

```bash
go mod init go-dqlite-demo
```

Install the application bindings for `go-dqilite/app` with:
```bash
go get github.com/canonical/go-dqlite/app
```

[note type="important" status="Info"]
Read more about Go mod files in the [Go documentation](https://go.dev/ref/mod).
[/note]

# Install a C compiler

A C compiler is required for the package to build. Install `gcc` with:
```bash
sudo apt install gcc
```
You should now be all set to start your first Go application. Continue with [Create an example application](/t/create-an-example-application/73).
