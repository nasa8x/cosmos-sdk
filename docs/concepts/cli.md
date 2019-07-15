# Command-Line Interface

## Prerequisites

* [Anatomy of an SDK App](./app-anatomy.md)

## Synopsis

This document describes how to create a commmand-line interface for an SDK application. A separate document for creating a compatible interface for a module can be found [here](./modules.md). 

1. [Building a CLI With Cobra](#building-a-cli-with-cobra)
2. [Commands](#commands)
3. [Flags](#flags)
4. [Application CLI](#application-cli) (maybe rename)


## Building a CLI with Cobra

There is no set way to create a CLI, but SDK modules all use the [Cobra Library](https://github.com/spf13/cobra). Building a CLI with Cobra entails defining commands, arguments, and flags. [**Commands**](#commands) represent the action users wish to take, such as `tx` for creating a transaction and `query` for querying the application. Each command can also have nested subcommands, necessary for naming the specific transaction type. Users also supply **Arguments**, such as account numbers to send coins to, and [**Flags**](#flags) to modify various aspects of the commands, such as gas prices or which node to broadcast to. 
    
## Commands

Every application first constructs a root command by creating `rootCmd`, then adds functionality by aggregating subcommands (usually with further nested subcommands) using `AddCommand`. The bulk of an application's capabilities lies in its transaction and query commands, called `TxCmd` and `QueryCmd` respectively. 

### Root Commands

`RootCmd` most typically must include the following commands to support basic functionality in the application.

* **Status** command from the SDK rpc client tools, which prints information about the status of the connected `Node`. 
* **Config** command from the SDK client tools, which allows the user to edit a `config.toml` file that sets values for [flags](#flags) such as `--chain-id` and which `--node` they wish to connect to.
* **Keys** commands from the SDK client tools, which includes a collection of subcommands for using the key functions in the SDK crypto tools, including adding a new key and saving it to disk, listing all public keys stored in the key manager, and deleting a key. 
* [**Transaction**](#transaction-commands) commands.
* [**Query**](#query-commands) commands.

### Transaction Commands 

Application [transactions](#./transactions.md) are objects that trigger state changes. To enable the creation of transactions in the CLI interface, `TxCmd` should add the following commands:

* **Sign** command from the [`auth`](https://github.com/cosmos/cosmos-sdk/tree/67f6b021180c7ef0bcf25b6597a629aca27766b8/docs/spec/auth) module, which signs messages in a transaction. To enable multisig, it should also add the `auth` module MultiSign command. Since every transaction requires some sort of signature in order to be valid, this command is necessary for every application. 
* **Broadcast** command from the SDK client tools, which broadcasts transactions.
* **Send** command from the [`bank`](https://github.com/cosmos/cosmos-sdk/tree/67f6b021180c7ef0bcf25b6597a629aca27766b8/docs/spec/bank) module, which is a transaction that allows accounts to send coins to one another, including gas and fees for transactions. 
* Any application-specific transaction commands defined by the application developer.
* All commands in each module the application is dependent on, retrieved by calling `GetTxCmd()` on all the modules or using the Module Manager's `AddTxCommands()` function. 

### Query Commands

Application queries are objects that allow users to retrieve information about the application's state. In order to enable basic queries, `QueryCmd` should add the following commands:

* **QueryTx** and/or other transaction query commands from the `auth` module which allow the user to search for a transaction by inputting its hash, a list of tags, or a block height. These various queries allow users to see if transactions have been included in a block.
* **Account** command from the `auth` module, which displays the state (e.g. account balance) of an account given an address.
* **Validator** command from the SDK rpc client tools, which displays the validator set of a given height. 
* **Block** command from the SDK rpc client tools, which displays the block data for a given height.
* Any application-specific query commands defined by the application developer.
* All commands in each module the application is dependent on, retrieved by calling `GetQueryCmd()` on all the modules or using the Module Manager's `AddQueryCommands()` function. 

### Query Example

Here is an example of what a user might enter into their command-line to query how many Atoms they have delegated. The application is called `app`: 

```bash
appcli query staking delegations <delegatorAddress>

```
The root command is `appcli`, which indicates which application the user is interacting with. Their `query` command is to be routed to the `staking` module, which includes `delegations` as one of the possible queries. The argument provided is `delegatorAddress`, and no flags were included which means the user's already-configured flags are used in processing this command. 

When the command is executed, the first thing it will do is create a [`CLIContext`](./context.md) using the application's codec. The `delegatorAddress` provided by the user is used to create query parameters, and a route is constructed using the application's provided `queryRoute`. The query parameters and route and used to make an [ABCI call](https://tendermint.com/docs/spec/abci/abci.html#messages) (a Tendermint RPC), `ABCIQueryWithOptions`. The application inherits [ABCI Query](https://tendermint.com/docs/spec/abci/abci.html#query) capabilities from [BaseApp](./baseapp.md); BaseApp handles the query by unmarshaling the request, parsing the query path, and routing it to the appropriate queryable multistore that stores delegations data for this application. 

    
## Flags

Flags are used to modify commands. Users can explicitly include them in commands or pre-configure them by entering a command in the format `appcli config <flag> <value>` into their command line. Commonly pre-configured flags include the `--node` to connect to and `--chain-id` of the blockchain the user wishes to interact with. All flags have default values; some toggle an option off but others are empty values that the user needs to override to create valid commands. Thus, unless the flag listed below is marked as optional, the user must provide a value. 

### Transaction Flags

To **create** a transaction, the user enters a `tx` command and provides several flags.

* `--from` indicates which account the transaction originates from. This account is used to sign the transaction. 
* `--gas` refers to how much gas, which represents computational resources, Tx consumes. Gas is dependent on the transaction and is not precisely calculated until execution, but can be estimated by providing auto as the value for --gas.
* `--gas-adjustment` (optional) can be used to scale gas up in order to avoid underestimating. For example, users can specify their gas adjustment as 1.5 to use 1.5 times the estimated gas.
* `--gas-prices` specifies how much the user is willing pay per unit of gas, which can be one or multiple denominations of tokens. For example, --gas-prices=0.025uatom, 0.025upho means the user is willing to pay 0.025uatom AND 0.025upho per unit of gas.
* `--fees` specifies how much in fees the user is willing to pay in total. Note that the user only needs to provide either `gas-prices` or `fees`, but not both, because they can be derived from each other.
* `--generate-only` (optional) instructs the application to simply generate the unsigned transaction and output or write to a file. Without this flag, the transaction is created, signed, and broadcasted all in one command. 
* `--dry-run` (optional), similar to `--generate-only`, instructs the application to ignore the `--gas` flag and simulate the transaction running without broadcasting.
* `--indent` (optional) adds an indent to the JSON response.
* `--memo` sends a memo along with the transaction.

For example, the following command creates a transaction to send 1000uatom from `sender-address` to `recipient-address`. The user is willing to pay 0.025uatom per unit gas but wants the transaction to be only generated offline (i.e. not broadcasted) and written, in JSON format, to the file `myUnsignedTx.json`. 

```bash
appcli tx send <recipientAddress> 1000uatom --from <senderAddress> --gas auto -gas-prices 0.025uatom --generate-only > myUnsignedTx.json
```

To **sign** a transaction generated offline using the `--generate-only` flag, the user enters a `tx sign` command (by default, the transaction is automatically signed upon creation). There are four values for flags that must be provided if a transaction is expected to be signed:

* `--from` specifies an address; the corresponding private key is used to sign the transaction.
* `--chain-id` specifies the unique identifier of the blockchain the transaction pertains to.
* `--sequence` is the value of a counter measuring how many transactions have been sent from the account. It is used to prevent replay attacks.
* `--account-number` is an identifier for the account.
* `--validate-signatures` (optional) instructs the process to sign the transaction and verify that all signatures have been provided.
* `--ledger` (optional) lets the user perform the action using a Ledger Nano S, which needs to be plugged in and unlocked. 

For example, the following command signs the inputted transaction, `myUnsignedTx.json`, and writes the signed transaction to the file `mySignedTx.json`.

```bash
appcli tx sign myUnsignedTx.json --from <senderName> --chain-id <chainId> --sequence <sequence> --account-number<accountNumber> > mySignedTx.json
```

To **broadcast** a signed transaction generated offline, the user enters a `tx broadcast` command. Only one flag is required here:

* `--node` specifies which node to broadcast to.
* `--trust-node` (optional) indicates whether or not the node and its response proofs can be trusted.
* `--broadcast-mode` (optional) specifies when the process should return. Options include asynchronous (return immediately), synchronous (return after `CheckTx` passes), or block (return after block commit).

For example, the following command broadcasts the signed transaction, `mySignedTx.json` to a particular node.

```bash
appcli tx broadcast mySignedTx.json --node <node>
```
### Query Flags

Queries also have flags. 

* `--node` indicates which full-node to connect to. 
* `--trust-node` (optional) represents whether or not the connected node is trusted. If the node is not trusted, all proofs in the responses are verified.
* `--indent` (optional) adds an indent to the JSON response.
* `--height` (optional) can be provided to query the blockchain at a specific height.
* `--ledger` (optional) lets the user perform the action using a Ledger Nano S. 


## Application CLI (can rename)

TODO: CLI `cmd/appcli/main.go`, LCD `cmd/appd/main.go`
