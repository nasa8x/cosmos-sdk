# Module Interfaces

## Prerequisites

## Synopsis

## Module Interfaces 

The module developer can specify possible user interactions by defining HTTP Request Handlers or Cobra [commands](https://github.com/spf13/cobra) accessible through command-line, typically found in the module's `/client` folder. 

An application developer utilizes these capabilities when creating an entrypoint to the application such as a [command-line interface](./cli.md).

## CLI

There are many ways to create command-line interfaces, but the SDK modules all use the [Cobra Package](https://github.com/spf13/cobra). To enable command-line interactions, all the module developer needs to do is define commands for each transaction and query. 

### Transaction Commands

Transactions are typically created in applications, but some SDK modules define module-specific transactions such as `tx sign` from the [`auth`](https://github.com/cosmos/cosmos-sdk/tree/67f6b021180c7ef0bcf25b6597a629aca27766b8/docs/spec/auth) module and module transaction subcommands, such as `tx gov vote` and `tx staking delegate` which belong to the [`gov`](https://github.com/cosmos/cosmos-sdk/tree/67f6b021180c7ef0bcf25b6597a629aca27766b8/docs/spec/gov) and [`staking`](https://github.com/cosmos/cosmos-sdk/tree/67f6b021180c7ef0bcf25b6597a629aca27766b8/docs/spec/staking) modules, respectively. 

### Query Commmands



## REST

Module developers create a `rest.go` file to hold HTTP Handlers for both queries and transactions. The file specifies the client interface using a `RegisterRoutes` function that matches specific HTTP Requests (their routes and request methods) with a Handler. 

### Registering Routes

As to be expected, queries are GET requests and transactions are typically POST or PUT requests. 

### Request Definitions

The file also includes structs for each transaction request with what fields should be included, as well as their JSON representations. 

### Request Handlers

Finally, the developer defines a Handler for each type of request; the Handlers directly create messages and return an unsigned transaction. Note that, for security, this interface only creates transactions - they must be signed and broadcasted separately. The application developer calls `RegisterRoutes` in the `main()` function to enable requests with this method.
