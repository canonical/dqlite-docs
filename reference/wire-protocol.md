## Definitions

The following terms are used in this document:

* *server* : A Dqlite node.
* *client* : Either application code (typically wanting to issue database queries) or a Dqlite node (typically pushing replicational data).
* *connection* : A TCP or Unix socket connection established by the client against a server.
* *word* : A sequence of 8 bytes.
* *protocol version* : A positive number stored in a word using little endian representation.
* *message* : A sequence of bytes sent either by the client to the server or by the server to the client. It consists of a header and a body. The header consists of a single word with the following layout:
  * byte 0 to 3: Size of the message body, expressed in words and stored using little endian representation. For example a value of [2 1 0 0] means that the message body consists of 258 bytes.
  * byte 4: Type code identifying the schema of the message body.
  * byte 5: Message schema version. This version is bumped for a particular message type when the expected format of the body for that message type changes. Unless otherwise noted, the message formats described in this document are for schema version 0.
  * byte 6 to 7: Currently unused.

The message body is a sequence of fields as described by the associated schema. Message types and their schemas are listed below.

## Setup

As soon as a connection is established, the client must send to the server a single word containing the protocol version it wishes to use. The current protocol version is 1 (in little-endian representation).

## Conversation

After the setup, communication between client and server happens by message exchange. Typically the client will send to the server a message containing a request and the server will send to the client a message containing a response. Any request may result in a failure response (type `0`); otherwise, the correspondence between request and response types is as follows:

| Request | Response |
| ---     | ---      |
| `0` (Get current leader) | `1` (Leader information) |
| `1` (Client registration) | `2` (Welcome) |
| `3` (Open a database) | `4` (Database information) |
| `4` (Prepare a statement) | `5` (Prepared statement information) |
| `5` (Execute a prepared statement) | `6` (Statement execution result) |
| `6` (Execute a prepared statement yielding rows) | `7` (Batch of table rows) |
| `7` (Finalise a prepared statement) | `8` (Acknowledgement) |
| `8` (Execute a SQL text) | `6` (Statement execution result) |
| `9` (Execute a SQL text yielding rows) | `7` (Batch of table rows) |
| `10` (Interrupt the execution of a statement yielding rows) | `8` (Acknowledgement) |
| `12` (Add a non-voting node to the cluster) | `8` (Acknowledgement) |
| `13` (Assign a role to a node) | `8` (Acknowledgement) |
| `14` (Remove a node from the cluster) | `8` (Acknowledgement) |
| `15` (Dump the content of a database) | `9` (Database files) |
| `16` (List all nodes of the cluster) | `3` (Cluster information) |
| `17` (Transfer leadership to another node) | `8` (Acknowledgement) |
| `18` (Get metadata associated with this node) | `10` (Node metadata) |
| `19` (Set the weight of this node) | `8` (Acknowledgement) |

## Data types

Each field in a message body has a specific data type, as described in the message schema. Available data types are:

### `uint64`

A single word containing an unsigned integer in little endian representation.

### `int64`

A single word containing a two-complement signed integer in little endian representation.

### `uint32`

Four bytes containing an unsigned integer in little endian representation.

### `text`

A sequence of one or more words containing a UTF-8 encoded zero-terminated string. All bytes past the terminating zero byte are zeroed as well.

### `row-tuple`, `params-tuple`, `params32-tuple`

A tuple represents a sequence of typed values. Dqlite uses three distinct
formats to code tuples, depending on the kind of message. They are described in detail below, and the differences between them are summarised by this table:

| Type             | Length field | Type codes  |
| ---              | ---          | ---         |
| `row-tuple`      | implicit     | 4 bits each |
| `params-tuple`   | 1 byte       | 8 bits each |
| `params32-tuple` | 4 bytes      | 8 bits each |

Every tuple ends with a sequence of *values*, each of which is coded according to the corresponding type code. The correspondence between type codes and values is:

| Type code | Value                                                                                |
| ---       | ---                                                                                  |
| `1`       | Integer value stored using the `int64` encoding                                      |
| `2`       | An IEEE 754 floating point number stored in a single word (little endian)            |
| `3`       | A string value using the `text` encoding                                             |
| `4`       | A binary blob: the first word of the value is the length of the blob (little endian) |
| `5`       | A SQL NULL value encoded as a zeroed word |
| `9`       | A Unix timestamp, encoded as an `int64` count of seconds since the Unix epoch |
| `10`      | An ISO-8601 date value using the `text` encoding                                     |
| `11`      | A Boolean value using `uint64` encoding (0 for false and 1 for true)                 |

The `row-tuple` type represents a single database row in the "Batch of table rows" (response message, and is coded as a sequence of 4-bit type codes (two per byte), followed by zero padding if necessary up to a whole number of words, and then a sequence of values corresponding to the type codes. There is no length field: the client is expected to know the length of the tuple based on the database schema.

The `params-tuple` type represents a sequence of SQL statement parameters in the following request messages, when the message schema version is 0:

- Execute prepared statement (5)
- Execute prepared statement yielding rows (6)
- Execute SQL text (8)
- Execute SQL test yielding rows (9)

It is coded as a single byte indicating the number of values, followed by a
sequence of 8-bit (1-byte) type codes, followed by zero padding if necessary up to a whole number of words (including the "number of values" field), and then a sequence of values corresponding to the type codes. Clients that need to send more than 255 parameters for a statement must use the `params32-tuple` type.

The `params32-tuple` type represents a sequence of SQL statement parameters in the four request messages listed above (5, 6, 8, 9), when the message schema version is set to 1. It is coded in the same way as the `params-tuple` type, except that the initial "number of values" field is a 4-byte little-endian integer instead of a single byte.

### `node-info0`, `node-info`

Information about a node in the cluster. `node-info0` consists of a node ID (in `uint64` encoding) followed by a node address (in `text` encoding). `node-info` consists of the same two fields, followed by a role code (in `uint64` encoding). The role codes are:

| Code | Value   |
| ---  | ---     |
| `0`  | Voter   |
| `1`  | Standby |
| `2`  | Spare   |

### `file`

A single database file. It consists of the file name (in `text` encoding), followed by the file size (in `uint64` encoding) and finally a blob with the file content.

## Client messages

The client can send to the server messages with the following type codes and associated schemas:

### `0` - Get current leader

| Type     | Value        |
| ---      | ---          |
| `uint64` | Unused field |

### `1` - Client registration

| Type     | Value            |
| ---      | ---              |
| `uint64` | ID of the client |

### `3` - Open a database

| Type     | Value                    |
| ---      | ---                      |
| `text`   | The name of the database |
| `uint64` | Currently unused         |
| `text`   | Currently unused         |

### `4` - Prepare a statement

| Type     | Value                          |
| ---      | ---                            |
| `uint64` | ID of the open database to use |
| `text`   | SQL text of the statement      |

### `5` - Execute a prepared statement

The format for message schema version 0 is:

| Type           | Value                                                   |
| ---            | ---                                                     |
| `uint32`       | ID of the open database to use                          |
| `uint32`       | ID of the prepared statement to execute                 |
| `params-tuple` | A tuple of parameters to bind to the prepared statement |

For message schema version 1, `params32-tuple` is used instead of `params-tuple` for the last field.

### `6` - Execute a prepared statement yielding rows

The format for message schema version 0 is:

| Type           | Value                                                   |
| ---            | ---                                                     |
| `uint32`       | ID of the open database to use                          |
| `uint32`       | ID of the prepared statement to execute                 |
| `params-tuple` | A tuple of parameters to bind to the prepared statement |

For message schema version 1, `params32-tuple` is used instead of `params-tuple` for the last field.

### `7` - Finalise a prepared statement

| Type     | Value                                    |
| ---      | ---                                      |
| `uint32` | ID of the open database to use           |
| `uint32` | ID of the prepared statement to finalise |

### `8` - Execute a SQL text

The format for message schema version 0 is:

| Type           | Value                          |
| ---            | ---                            |
| `uint64`       | ID of the open database to use |
| `text`         | SQL text to execute            |
| `params-tuple` | A tuple of parameters to bind  |

For message schema version 1, `params32-tuple` is used instead of `params-tuple` for the last field.

The SQL text may consist of multiple statements, but if it does, parameters must not be provided. The server will not crash or otherwise behave dangerously when processing a request that includes both multi-statement text and parameters, but it may respond with a "Failure" message or with a "Statement execution result" message that is not meaningful.

If the SQL text consists of multiple statements, the fields of the "Statement execution result" response pertain to the *last* statement.

### `9` - Execute a SQL text yielding rows

The format for message schema version 0 is:

| Type           | Value                          |
| ---            | ---                            |
| `uint64`       | ID of the open database to use |
| `text`         | SQL text to execute            |
| `params-tuple` | A tuple of parameters to bind  |

For message schema version 1, `params32-tuple` is used instead of `params-tuple` for the last field.

### `10` - Interrupt the execution of a statement yielding rows

| Type     | Value                                                 |
| ---      | ---                                                   |
| `uint64` | ID of the open database currently executing the query |

### `12` - Add a non-voting node to the cluster

| Type         | Value                             |
| ---          | ---                               |
| `node-info0` | ID and address of the node to add |

### `13` - Assign a role to a node

| Type     | Value                    |
| ---      | ---                      |
| `uint64` | ID of the node to update |
| `uint64` | New role                 |

The "new role" field is interpreted as described for `node-info`.

### `14` - Remove a node from the cluster

| Type     | Value                    |
| ---      | ---                      |
| `uint64` | ID of the node to remove |

### `15` - Dump the content of a database

| Type   | Value                        |
| ---    | ---                          |
| `text` | Name of the database to dump |

### `16` - List all nodes of the cluster

| Type     | Value  |
| ---      | ---    |
| `uint64` | Format |

The format field should always be set to 1.

### `17` - Transfer leadership to another node

|Type|Value|
| --- | --- |
|`uint64`|ID of the new leader|

### `18` - Get metadata associated with this node

| Type     | Value  |
| ---      | ---    |
| `uint64` | Format |

The format field should always be set to 0.

### `19` - Set the weight of this node

| Type     | Value      |
| ---      | ---        |
| `uint64` | New weight |

## Server messages

The server can send to the client messages with the following type codes and associated schemas:

### `0` - Failure response

| Type     | Value                             |
| ---      | ---                               |
| `uint64` | Code identifying the failure type |
| `text`   | Human-readable failure message    |

### `1` - Leader information

| Type         | Value                                |
| ---          | ---                                  |
| `node-info0` | Information about the cluster leader |

### `2` - Welcome

| Type     | Value            |
| ---      | ---              |
| `uint64` | Currently unused |

### `3` - Cluster information

| Type        | Value                          |
| ---         | ---                            |
| `uint64`    | Number of nodes in the cluster |
| `node-info` | First node                     |
| `node-info` | Second node (if any)           |
| ...         |                                |

### `4` - Database information

| Type     | Value       |
| ---      | ---         |
| `uint32` | Database ID |
| `uint32` | Unused      |

### `5` - Prepared statement information

| Type     | Value                |
| ---      | ---                  |
| `uint32` | Database ID          |
| `uint32` | Statement ID         |
| `uint64` | Number of parameters |

### `6` - Statement execution result

| Type     | Value                         |
| ---      | ---                           |
| `uint64` | ID of last row inserted, or 0 |
| `uint64` | Number of rows affected or 0  |

### `7` - Batch of table rows

| Type        | Value                                                 |
| ---         | ---                                                   |
| `uint64`    | Number of columns                                     |
| `text`      | Name of first column                                  |
| `text`      | Name of second column (if any)                        |
| ...         |                                                       |
| `row-tuple` | Column values of the first row in the batch           |
| `row-tuple` | Column values of the second row in the batch (if any) |
| ...         |                                                       |
| `uint64`    | End marker                                            |

The end marker is the value `0xffffffffffffffff` if the statement currently yielding rows has completed and there are no more rows, or otherwise `0xeeeeeeeeeeeeeeee` if there are more rows and another batch will be sent.

### `8` - Acknowledgement

| Type     | Value  |
| ---      | ---    |
| `uint64` | Unused |

### `9` - Database files

| Type     | Value                |
| ---      | ---                  |
| `uint64` | Number of files = 2  |
| `file`   | Main database file   |
| `file`   | Write-ahead log file |

### `10` - Node metadata

| Type     | Value          |
| ---      | ---            |
| `uint64` | Failure domain |
| `uint64` | Weight         |

These properties can be used to inform how leaders manage node roles within the cluster to maximise availability.
