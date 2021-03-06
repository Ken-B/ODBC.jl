# ODBC.jl

The `ODBC.jl` package provides high-level julia functionality over the low-level ODBC API middleware. In particular, the package allows making connections with any database that has a valid ODBC driver, sending SQL queries to those databases, and streaming the results into a variety of data sinks.

### `ODBC.listdsns()`

Lists pre-configured DSN datasources available to the user. Note that DSNs are "bit-specific", meaning a 32-bit DSN setup with the 32-bit ODBC system admin console will only be accessible through 32-bit julia.

### `ODBC.listdrivers`

Lists valid ODBC drivers on the system which can be used manually in connection strings in the form of `Driver={ODBC Driver Name};` as a key-value pair. Valid drivers are read from the system ODBC library, which can be seen by calling `ODBC.API.odbc_dm`. This library is "detected" automatically when the ODBC.jl package is loaded, but can also be set by calling `ODBC.API.setODBC("manual_odbc_lib")`.


### `ODBC.DSN`

Constructors:

`ODBC.DSN(dsn, username, password) => ODBC.DSN`
`ODBC.DSN(connection_string) => ODBC.DSN`
`ODBC.disconnect!(dsn::ODBC.DSN)`

The first method attempts to connect to a pre-defined DSN that has been pre-configured through your system's ODBC admin console. Settings such as the ODBC driver, server address, port #, etc. are already configured, so all that is required is the username and password to connect.

The second method takes a full connection string. Connection strings are vendor-specific, but follow the format of `key1=value1;key2=value2...` for various key-value pairs, typically including `Driver=X` and `Server=Y`. For help in figuring out how to build the right connection string for your system, see [connectionstrings.com](https://www.connectionstrings.com/).

`ODBC.disconnect!(dsn)` can also be used to disconnect.

### `ODBC.query`

Methods:

`ODBC.query(dsn::ODBC.DSN, sql::AbstractString, sink=DataFrame, args...; append::Bool=false)`
`ODBC.query{T}(dsn::DSN, sql::AbstractString, sink::T; append::Bool=false)`
`ODBC.query(source::ODBC.Source, sink=DataFrame, args...; append::Bool=false)`
`ODBC.query{T}(source::ODBC.Source, sink::T; append::Bool=false)`

`ODBC.query` is a high-level method for sending an SQL statement to a system and returning the results. As is shown, a valid `dsn::ODBC.DSN` and SQL statement `sql` combo can be sent, as well as an already-constructed `source::ODBC.Source`. By default, the results will be returned in a [`DataFrame`](http://juliastats.github.io/DataFrames.jl/latest/), but a variety of options exist for returning results, including `CSV.Sink`, `SQLite.Sink`, or `Feather.Sink`. `ODBC.query` actually utilizes the `DataStreams.jl` framework, so any valid [`Data.Sink`](http://juliadata.github.io/DataStreams.jl/latest/#datasink-interface) can be used to return results. The `append=false` keyword specifies whether the results should be *added to* any existing data in the `Data.Sink`, or if the resultset should fully replace any existing data.

Examples:

```julia
dsn = ODBC.DSN(valid_dsn)

# return result as a DataFrame
df = ODBC.query(dsn, "select * from cool_table")

# return result as a csv file
using CSV
csv = ODBC.query(dsn, "select * from cool_table", CSV.Sink, "cool_table.csv")

# return the result directly into a local SQLite table
using SQLite
db = SQLite.DB()

sqlite = ODBC.query(dsn, "select * from cool_table", SQLite.Sink, db, "cool_table_in_sqlite")

# return the result as a feather-formatted binary file
using Feather
feather = ODBC.query(dsn, "select * from cool_table", Feather.Sink, "cool_table.feather")

```

### `ODBC.load`

Methods:
`ODBC.load{T}(dsn::DSN, table::AbstractString, ::Type{T}, args...; append::Bool=false)`
`ODBC.load(dsn::DSN, table::AbstractString, source; append::Bool=false)`
`ODBC.load{T}(sink::Sink, ::Type{T}, args...; append::Bool=false)`
`ODBC.load(sink::Sink, source; append::Bool=false)`

`ODBC.load` is a sister method to `ODBC.query`, but instead of providing a robust way of *returning* results, it allows one to *send* data to a DB. 

**Please note this is currently experimental and ODBC driver-dependent; meaning, an ODBC driver must impelement certain low-level API methods to enable this feature. This is not a limitation of ODBC.jl itself, but the ODBC driver provided by the vendor. In the case this method doesn't work for loading data, please see the documentation around prepared statements.**

`ODBC.load` takes a valid DB connection `dsn` and the name of an *existing* table `table` to which to send data. Note that on-the-fly creation of a table is not currently supported. The data to send can be any valid [`Data.Source`](http://juliadata.github.io/DataStreams.jl/latest/#datasource-interface) object, from the `DataStreams.jl` framework, including a `DataFrame`, `CSV.Source`, `SQLite.Source`, `Feather.Source`, etc.

Examples:

```julia
dsn = ODBC.DSN(valid_dsn)

# first create a remote table
ODBC.execute!(dsn, "CREATE TABLE cool_table (col1 INT, col2 FLOAT, col3 VARCHAR)")

# load data from a DataFrame into the table
df = DataFrame(col1=[1,2,3], col2=[4.0, 5.0, 6.0], col3=["hey", "there", "sailor"])

ODBC.load(dsn, "cool_table", df)

# load data from a csv file
using CSV

ODBC.load(dsn, "cool_table", CSV.Source, "cool_table.csv")

# load data from an SQLite table
using SQLite

ODBC.load(dsn, "cool_table", SQLite.Source, "select * from cool_table")

# load data from a feather-formatted binary file
using Feather

ODBC.load(dsn, "cool_table", Feather.Source, "cool_table.feather")

```


### `ODBC.prepare`

Methods:

`ODBC.prepare(dsn::ODBC.DSN, querystring::String) => ODBC.Statement`

Prepare an SQL statement `querystring` against the DB and return it as an `ODBC.Statement`. This `ODBC.Statement` can then be executed once, or repeatedly in a more efficient manner than `ODBC.execute!(dsn, querystring)`. Prepared statements can also support parameter place-holders that can be filled in dynamically before executing; this is a common strategy for bulk-loading data or other statements that need to be bulk-executed with changing simple parameters before each execution. Consult your DB/vendor-specific SQL syntax for the exact specifications for parameters.

Examples:

```julia
# prepare a statement with 3 parameters marked by the '?' character
stmt = ODBC.prepare(dsn, "INSERT INTO cool_table VALUES(?, ?, ?)")

# a DataFrame with data we'd like to insert into a table
df = DataFrame(col1=[1,2,3], col2=[4.0, 5.0, 6.0], col3=["hey", "there", "sailor"])

for row = 1:size(df, 1)
    # each time we execute the `stmt`, we pass another row to be bound to the parameters
    ODBC.execute!(stmt, [df[row, x] for x = 1:size(df, 2)])
end
```


### `ODBC.execute!`

Methods:

`ODBC.execute!(dsn::ODBC.DSN, querystring::String)`
`ODBC.execute!(stmt::ODBC.Statement)`
`ODBC.execute!(stmt::ODBC.Statement, values)`

`ODBC.execute!` provides a method for executing a statement against a DB without returning any results. Certain SQL statements known as "DDL" statements are used to modify objects in a DB and don't have results to return anyway. While `ODBC.query` can still be used for these types of statements, `ODBC.execute!` is much more efficient. This method is also used to execute prepared statements, as noted in the documentation for `ODBC.prepare`.


### `ODBC.Source`

Constructors:

`ODBC.Source(dsn::ODBC.DSN, querystring::String) => ODBC.Source`

`ODBC.Source` is an implementation of a `Data.Source` in the [DataStreams.jl](http://juliadata.github.io/DataStreams.jl/latest/#datasource-interface) framework. It takes a valid DB connection `dsn` and executes a properly formatted SQL query string `querystring` and makes preparations for returning a resultset.
