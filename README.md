# neo4j-go-driver

This is the official Neo4j Go Driver.

## Getting the Driver

### Go version prerequisites

The Go driver only works with actively maintained Go versions.

Read [Go official release policy](https://go.dev/doc/devel/release#policy) to learn more.

### Module version

Make sure your application has been set up to use go modules (there should be a go.mod file in your application root).

#### 5.x

Add the driver with:

```shell
go get github.com/neo4j/neo4j-go-driver/v5
```

#### 4.x

Add the driver with:

```shell
go get github.com/neo4j/neo4j-go-driver/v4
```

#### 1.x

For versions 1.x of the driver (notice the absence of `/v4` or `/v5`), run instead the following:

```shell
go get github.com/neo4j/neo4j-go-driver
```

## Documentation

Drivers manual that describes general driver concepts in depth [here](https://neo4j.com/docs/go-manual/current/).
Go package API documentation [here](https://pkg.go.dev/github.com/neo4j/neo4j-go-driver/v5/neo4j).

## Preview Features

The preview feature is a new feature that is a candidate for a future <abbr title="Generally Available">GA</abbr>
status.

It enables users to try the feature out and maintainers to refine and update it.

The preview features are not considered to be experimental, temporary or unstable.

However, they may change more rapidly, without following the usual deprecation cycle.

Most preview features are expected to be granted the GA status unless some unexpected conditions arise.

Due to the increased flexibility of the preview status, user feedback is encouraged so that it can be considered before
the GA status.

Every preview feature gets a
dedicated [GitHub Discussion](https://github.com/neo4j/neo4j-go-driver/discussions/categories/preview-feature-announcement)
where users can share their initial impressions and thoughts.

## Migrating from previous versions

See [migration guide](MIGRATION_GUIDE.md) for information on how to migrate 
from previous versions of the driver to the current one.

## Minimum Viable Snippet

Connect, execute a statement and handle results

Make sure to use the configuration in the code that matches the version of Neo4j server that you run.

```go
package main

import (
    "context"
    "fmt"
    "github.com/neo4j/neo4j-go-driver/v5/neo4j"
)

func main() {
    dbUri := "neo4j://localhost" // scheme://host(:port) (default port is 7687)
    driver, err := neo4j.NewDriverWithContext(dbUri, neo4j.BasicAuth("neo4j", "letmein!", ""))
    if err != nil {
        panic(err)
    }
    // Starting with 5.0, you can control the execution of most driver APIs
    // To keep things simple, we create here a never-cancelling context
    // Read https://pkg.go.dev/context to learn more about contexts
    ctx := context.Background()
    // Handle driver lifetime based on your application lifetime requirements.
    // driver's lifetime is usually bound by the application lifetime, which usually implies one driver instance per
    // application

    defer driver.Close(ctx) // Make sure to handle errors during deferred calls
    item, err := insertItem(ctx, driver)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%v\n", item)
}

func insertItem(ctx context.Context, driver neo4j.DriverWithContext) (*Item, error) {
    result, err := neo4j.ExecuteQuery(ctx, driver,
        "CREATE (n:Item { id: $id, name: $name }) RETURN n",
        map[string]any{
            "id":   1,
            "name": "Item 1",
        }, neo4j.EagerResultTransformer)
    if err != nil {
        return nil, err
    }
    itemNode, _, err := neo4j.GetRecordValue[neo4j.Node](result.Records[0], "n")
    if err != nil {
        return nil, fmt.Errorf("could not find node n")
    }
    id, err := neo4j.GetProperty[int64](itemNode, "id")
    if err != nil {
        return nil, err
    }
    name, err := neo4j.GetProperty[string](itemNode, "name")
    if err != nil {
        return nil, err
    }
    return &Item{Id: id, Name: name}, nil
}

type Item struct {
    Id   int64
    Name string
}

func (i *Item) String() string {
    return fmt.Sprintf("Item (id: %d, name: %q)", i.Id, i.Name)
}

```

## Server/Driver compatibility

Starting with 5.0, the Neo4j Drivers will be moving to a monthly release cadence.
A minor version will be released on the last Friday of each month so as to maintain versioning consistency with the core product (Neo4j DBMS) which has also moved to a monthly cadence.

As a policy, patch versions will not be released except on rare occasions.
Bug fixes and updates will go into the latest minor version and users should upgrade to that.
Driver upgrades within a major version will never contain breaking API changes.

See also: https://neo4j.com/developer/kb/neo4j-supported-versions/


## Connecting to a causal cluster

You just need to use `neo4j` as the URL scheme and set host of the URL to one of your core members of the cluster.

```go
if driver, err = neo4j.NewDriver("neo4j://localhost:7687", neo4j.BasicAuth("username", "password", "")); err != nil {
	return err // handle error
}
```

There are a few points that need to be highlighted:
* Each `Driver` instance maintains a pool of connections inside, as a result, it is recommended to only use **one driver per application**.
* It is considerably cheap to create new sessions and transactions, as sessions and transactions do not create new connections as long as there are free connections available in the connection pool.
* The driver is thread-safe, while the session or the transaction is not thread-safe.

## Parsing Result Values

### Record Stream
A cypher execution result is comprised of a stream of records followed by a result summary.
The records inside the result can be accessed via `Next()`/`Record()` functions defined on `Result`. It is important to check `Err()` after `Next()` returning `false` to find out whether it is the end of the result stream or an error that caused the end of result consumption.

```go
	// Next returns false upon error
	for result.Next(ctx) {
		record := result.Record()
		handleRecord(record)
	}
	// Err returns the error that caused Next to return false
	if err = result.Err(); err != nil {
		handleError(err)
	}
```

### Accessing Values in a Record
Values in a `Record` can be accessed either by index or by alias. The return value is an `any` which means you need
to convert the interface to the expected type

```go
value := record.Values[0]
```

```go
if value, ok := record.Get('field_name'); ok {
	// a value with alias field_name was found
	// process value
}
```

### Value Types
The driver exposes values in the record as an `any` type. 
The underlying types of the returned values depend on the corresponding Cypher types.

The mapping between Cypher types and the types used by this driver (to represent the Cypher type):

|  Cypher Type | Driver Type        |
|-------------:|:-------------------|
|       *null* | nil                |
|         List | []any              |
|          Map | map[string]any     |
|      Boolean | bool               |
|      Integer | int64              |
|        Float | float              |
|       String | string             |
|    ByteArray | []byte             |
|         Node | neo4j.Node         |
| Relationship | neo4j.Relationship |
|         Path | neo4j.Path         |

### Spatial Types - Point

| Cypher Type | Driver Type   |
|------------:|:--------------|
|       Point | neo4j.Point2D |
|       Point | neo4j.Point3D |

The temporal types are introduced in Neo4j 3.4 series.

You can create a 2-dimensional `Point` value using;

```go
point := neo4j.Point2D {X: 1.0, Y: 2.0, SpatialRefId: srId }
```

or a 3-dimensional point value using;

```go
point := neo4j.Point3D {X: 1.0, Y: 2.0, Z: 3.0, SpatialRefId: srId }
```

NOTE:

* For a list of supported `srId` values, please refer to the docs [here](https://neo4j.com/docs/cypher-manual/current/syntax/spatial/#cypher-spatial-crs-geographic).

### Temporal Types - Date and Time

The temporal types are introduced in Neo4j 3.4 series. Given the fact that database supports a range of different temporal types, most of them are backed by custom types defined at the driver level.

The mapping among the Cypher temporal types and actual exposed types are as follows:

|  Cypher Type  |     Driver Type     |
|:-------------:|:-------------------:|
|     Date      |     neo4j.Date      |
|     Time      |  neo4j.OffsetTime   |
|   LocalTime   |   neo4j.LocalTime   |
|   DateTime    |      time.Time      |
| LocalDateTime | neo4j.LocalDateTime |
|   Duration    |   neo4j.Duration    |


Receiving a temporal value as driver type:
```go
dateValue := record.Values[0].(neo4j.Date)
```

All custom temporal types can be constructed from a `time.Time` value using `<Type>Of()` (`DateOf`, `OffsetTimeOf`, ...) functions.

```go
dateValue := DateOf(time.Date(2005, time.December, 16, 0, 0, 0, 0, time.Local)
```

Converting a custom temporal value into `time.Time` (all `neo4j` temporal types expose `Time()` function to gets its corresponding `time.Time` value):
```go
dateValueAsTime := dateValue.Time()
```

Note:
* When `neo4j.OffsetTime` is converted into `time.Time` or constructed through `OffsetTimeOf(time.Time)`, its `Location` is given a fixed name of `Offset` (i.e. assigned `time.FixedZone("Offset", offsetTime.offset)`).
* When `time.Time` values are sent/received through the driver, if its `Zone()` returns a name of `Offset` the value is stored with its offset value and with its zone name otherwise.

## Logging

Logging at the driver level can be configured by setting `Log` field of `neo4j.Config` through configuration functions that can be passed to `neo4j.NewDriver` function.

### Console Logger

For simplicity, we provide a predefined console logger which can be constructed by `neo4j.ConsoleLogger` function. To enable console logger, you need to specify which level you need to enable (`neo4j.ERROR`, `neo4j.WARNING`, `neo4j.INFO` and `neo4j.DEBUG` which are ordered by the level of detail).

A simple code snippet that will enable console logging is as follows;

```go
useConsoleLogger := func(level neo4j.LogLevel) func(config *neo4j.Config) {
	return func(config *neo4j.Config) {
		config.Log = neo4j.ConsoleLogger(level)
	}
}

// Construct a new driver
if driver, err = neo4j.NewDriver(uri, neo4j.BasicAuth(username, password, ""), useConsoleLogger(neo4j.ERROR)); err != nil {
	return err
}
defer driver.Close()
```

### Custom Logger

The `Log` field of the `neo4j.Config` struct is defined to be of interface `neo4j/log.Logger` which has the following definition:

```go
type Logger interface {
	Error(name string, id string, err error)
	Warnf(name string, id string, msg string, args ...any)
	Infof(name string, id string, msg string, args ...any)
	Debugf(name string, id string, msg string, args ...any)
}
```

For a customised logging target, you can implement the above interface and pass an instance of that implementation to the `Log` field.

## Bolt tracing

Logging Bolt messages is done at the session level and is configured independently of the logging configuration described above.
This is **disabled** by default.

### Console Bolt logger

For simplicity, we provide a predefined console logger which can be constructed by `neo4j.ConsoleBoltLogger`.

```go
boltLogger := neo4j.ConsoleBoltLogger()

# for ExecuteQuery
result, err := neo4j.ExecuteQuery(ctx, driver, query, params, transformer, neo4j.ExecuteQueryWithBoltLogger(boltLogger))

# for the regular session APIs (session.Run, session.BeginTransaction, session.ExecuteRead, session.ExecuteWrite)
session := driver.NewSession(neo4j.SessionConfig{BoltLogger: boltLogger})
```

### Custom Bolt Logger

The `BoltLogger` field of the `neo4j.SessionConfig` struct is defined to be of interface `neo4j/log.BoltLogger` which has the following definition:

```go
type BoltLogger interface {
	LogClientMessage(context string, msg string, args ...any)
	LogServerMessage(context string, msg string, args ...any)
}
```

For a customised logging target, you can implement the above interface and pass an instance of that implementation to the `BoltLogger` field.

## Development

This section describes instructions to use during the development phase of the
driver.

### Prerequisites

The Go driver on this branch requires at least Go 1.18 and relies on [Go
modules](https://go.dev/ref/mod) for dependency resolution.

### Unit Testing

You can run unit tests as follows:

```shell
go test -tags=internal_testkit -short ./...
```

### Integration and Benchmark Testing

For these tests, you'll need to start a Neo4j server.
This can be done with [Neo4j Desktop](https://neo4j.com/download/) or with
the [official Docker image](https://hub.docker.com/_/neo4j):

```shell
docker run --rm \
  --env NEO4J_ACCEPT_LICENSE_AGREEMENT=yes \
  --env NEO4J_AUTH=neo4j/password \
  --publish=7687:7687 \
  --health-cmd 'cypher-shell -u neo4j -p password "RETURN 1"' \
  --health-interval 5s \
  --health-timeout 5s \
  --health-retries 5 \
  neo4j:4.4-enterprise
```

Once the server is running, you run integration tests like this:

```shell
TEST_NEO4J_HOST="localhost" TEST_NEO4J_VERSION=4.4 \
  go test ./neo4j/test-integration/...
```

Likewise, benchmark tests are run like this:

```shell
cd benchmark
go run . neo4j://localhost neo4j password
```

### All-In-One / Acceptance Testing

Tests **require** the
latest [Testkit](https://github.com/neo4j-drivers/testkit/), Python3 and Docker.

Testkit needs to be cloned and configured to run against the Go Driver. Use the
following steps to configure it.

1. Clone the Testkit repository

```shell
git clone https://github.com/neo4j-drivers/testkit.git
```

2. Under the Testkit folder, install the requirements.

```shell
pip3 install -r requirements.txt
```

3. Define the following required environment variables

```shell
export TEST_DRIVER_NAME=go
export TEST_DRIVER_REPO=<path for the root folder of driver repository>
```

To run test against some Neo4j version:

```shell
python3 main.py
```

More details about how to use Testkit can be found on [its repository](https://github.com/neo4j-drivers/testkit/)
