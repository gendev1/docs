# Tutorial: Developing your first Secret Contract

## Introduction

In this tutorial you will learn the basics of creating a new Secret Contract from scratch. You will learn the basics of handling messages and storing data for your contract on the chain. The contract that you make will allow individual users to store a private reminder on the secret network that can be read back at a later time. Using a Secret Contract for such a task is probably overkill, but it will teach you the basics of Secret Contract construction which provides a foundation for building more complex secret dapps using the same principles.

### Pre-requisites

Make sure you have completed the [Secret Contracts Quickstart Guide](https://docs.scrt.network/dev/quickstart.html) prior to doing this tutorial. You should also have a working knowledge of the Rust programming language. If you have never used Rust before, you can learn more [here](https://doc.rust-lang.org/book/). You can use any IDE that supports Rust or just a text editor. Here are some options:

* [Visual Studio Code Rust extension](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust)
* [IntelliJ Rust plugin](https://www.jetbrains.com/rust/)
* [Sublime Text 3 Rust Enhanced package](https://github.com/rust-lang/rust-enhanced)

### Generating our project

To generate a new project we follow the directions from the [Secret Contracts](https://docs.scrt.network/dev/secret-contracts.html), however we choose a new name, in this case `reminder`.

```bash
cargo generate --git https://github.com/scrtlabs/secret-template --name reminder
```

Go into the project folder, and you should see the following contents: cargo.toml, Developing.md, Importing.md, LICENSE, Makefile, NOTICE, Publishing.md, README.md, examples, rustfmt.toml, schema, src, and tests.     

In addition to everything we need to compile a contract, this template includes sample code for the simple counter contract. We are going to remove that in order to start from scratch. **Go into the `src` directory and empty the contents of the following three files `contract.rs`, `msg.rs`, and `state.rs`.** Do NOT remove or edit `lib.rs`.

You will also need to replace the contents of the `Cargo.toml` file with the contents of the `Cargo.toml` file found [HERE](https://github.com/darwinzer0/secret-contract-tutorials/blob/main/tutorial1/code/Cargo.toml). This will ensure that you are able to successfully compile the project after completing this tutorial.

Note: When building Secret Contracts using the [Secret Contract Template](https://github.com/scrtlabs/secret-template), the secret-toolkit is not found within the [dev-dependencies] of the `Cargo.toml` file in the root directory of your project. When using the Secret Contract Template you will need to manually add it to the projects Cargo.toml file: 

```toml
secret-toolkit = { git = "https://github.com/scrtlabs/secret-toolkit" }
```

## Secret Contract functions

There are three main functions Secret Contracts execute after network deployment:

1. `init` is the constructor for your contract, and it is only executed once. `init` configures the contract based on user-supplied parameters.
2. `handle` takes a handle message as input from a client, executes transactions based on the content of the message, and outputs a response message to the client.
3. `query` takes a query message as input from a client, reads data from storage to answer the query, and outputs a response message to the client.

The key difference between `handle` and `query` is that `handle` can execute transactions that change the state of the storage, whereas `query` is read-only. `handle` transactions therefore require a gas payment from the requester in order to succeed, but a `query` does not.

We define these three functions (and any additional helper functions) in our `src/contract.rs` file as follows:

```rust
use cosmwasm_std::{to_binary, Api, Binary, Env, Extern, HandleResponse, InitResponse, Querier, StdError, StdResult, Storage};
use std::convert::TryFrom;
use crate::msg::{HandleMsg, InitMsg, QueryMsg, HandleAnswer, QueryAnswer};
use crate::state::{load, may_load, save, State, Reminder, CONFIG_KEY};

pub fn init<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    env: Env,
    msg: InitMsg,
) -> StdResult<InitResponse> {
    // add init constructor functionality here
}

pub fn handle<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    env: Env,
    msg: HandleMsg,
) -> StdResult<HandleResponse> {
    match msg {
        // add handle transaction execution code here 
    }
}

pub fn query<S: Storage, A: Api, Q: Querier>(
    deps: &Extern<S, A, Q>,
    msg: QueryMsg,
) -> StdResult<Binary> {
    match msg {
        // add query execution code here
    }
}
```

At the top of this file, we have a number of module imports, some of which will be defined in other files as we go. Then you see the templates for `init`, `handle`, and `query`. The parameters for these three functions are similar. Both `init` and `handle` take three parameters `deps`, `env`, and `msg`. The `query` function only takes `deps` and `msg`.

* `deps` is a struct containing three external dependencies of the contract:
  * `deps.storage` implements `get()`, `set()`, and `remove()` methods to read, write, and delete data from the private storage of the contract;
  * `deps.api` implements two methods that translate back and forth between Secret Network human address Strings (`"secret..."`), and a binary canonical address format;
  * `deps.querier` implements a number of functions to query other contracts.
* `env` is a struct containing information about the external state of the contract:
  * `env.block`, a struct including the current block height, time, and chain-id;
  * `env.message`, a struct with information about the address that executed the contract (`env.message.sender`), plus any funds that might have been sent to the contract at that time;
  * `env.contract`, the address of the contract;
  * `env.contract-key`, the code id used when instantiating the contract;
  * `env.contract_code_hash`, a hex encoded hash of the code id.
* `msg` contains the message sent by the client. We will discuss message types in more detail in the next section.

A keen eye will note that `deps` is defined as `&mut Extern<S, A, Q>` in `init` and `handle`, but in `query` it is not mutable: `&Extern<S, A, Q>`. This is because a `query` is unable to change the `storage`, it is read-only. In addition, `query` does not have access to the external state of the contract, making the address of the query sender not available from within the `query` function. The reason for this is explained in more detail in the [Privacy Model of Secret Networks](https://docs.scrt.network/dev/privacy-model-of-secret-contracts.html#verified-values-during-contract-execution).

## Secret Contract messages

Now we need to structure input messages and output responses for each of our three main contract functions. We will define these message structures in `src/msg.rs`.

### Sketching out our contract's functionality

Before we specify any message structures, let's step back and detail some functionality we want our contract to implement by coming up with some simple user stories:

* We want users to be able to upload a string of text (the reminder) that will be stored for only each user.
* A user should be able to read their stored message, but it should not be accessible to anyone else.
* When a user receives their reminder they should also get information on when it was added.
* When no message is stored, and a user tries to read their reminder, then the user should receive a message that no reminder exists.
* When a user uploads a new reminder it should replace the old one.
* Anyone should be allowed to query for the total number of stored reminders.
* When the contract is first initialized we want the maximum size (in bytes) of a reminder to be set. If a user tries to upload a reminder larger than this size, then they should receive an error message.

Now let's create a skeleton for the message types that will support this functionality in `msg.rs`:

```rust
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct InitMsg {
    // add InitMsg parameters here
}

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleMsg {
    // add HandleMsg types here
}

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum QueryMsg {
    // add QueryMsg types here
}

/// Responses from handle function
#[derive(Serialize, Deserialize, Debug, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleAnswer {
    // add HandleMsg response types here
}

/// Responses from query function
#[derive(Serialize, Deserialize, Debug, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum QueryAnswer {
    // add QueryMsg response types here
}
```

Since there is only one type of `InitMsg` we can define it as a struct. In contrast, there might be multiple kinds of `HandleMsg` and `QueryMsg` types in our contract, so for clarity we will organize those into enums of structs as shown in the following table. This is not a requirement, however, you can simply define your message and response types individually as differently-named structs in your `msg.rs` code. For the response from `init` we will not define a specific type in this contract, but rather use a default response. However, there is nothing to prevent you from creating your own custom responses to `init` if you want.

| Functions | Messages     | Responses                     |
|-----------|--------------|-------------------------------|
| `init`    | `InitMsg`      | `InitResponse::default()` |
| `handle`  | `HandleMsg::{...}`  | `HandleAnswer::{...}`      |
| `query`   | `QueryMsg::{...}`   | `QueryAnswer::{...}`       |

First, we define `InitMsg` to add a field called `max_size`. This will be saved and used to make sure that reminders do not exceed a specified length.

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct InitMsg {
    /// Maximum size of a reminder message in bytes
    pub max_size: i32,
}
```

Next, we want to define two handle messages: `Record` stores a `reminder` string for a user and `Read` requests the current reminder. `Read` does not need any parameters, because the address of the sender will already be available from `env` in our `handle` function.

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleMsg {
    /// Records a new reminder for the sender
    Record {
        reminder: String,
    },
    /// Requests the current reminder for the sender
    Read { }
}
```

In addition to recording and reading our reminders, we also create a query message type `Stats` that requests basic information about the use of the contract.

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum QueryMsg {
    /// Gets basic statistics about the use of the contract
    Stats { }
}
```

Finally, we need to specify the structure of our contract's responses to handle and query messages:

```rust
/// Responses from handle functions
#[derive(Serialize, Deserialize, Debug, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum HandleAnswer {
    /// Return a status message to let the user know if it succeeded or failed
    Record {
        status: String,
    },
    /// Return a status message and the current reminder and its timestamp, if it exists
    Read {
        status: String,
        reminder: Option<String>,
        timestamp: Option<u64>,
    }
}

/// Responses from query functions
#[derive(Serialize, Deserialize, Debug, JsonSchema)]
#[serde(rename_all = "snake_case")]
pub enum QueryAnswer {
    /// Return basic statistics about contract
    Stats {
        reminder_count: u64,
    }
}
```

Sometimes incoming messages, or responses, will have an optional field. Those are defined with Rust `Option` types. The Secret Contract SDK will include those fields as needed in the response to the client automatically. In our `Read` response we define `reminder` and `timestamp` using an `Option` type, because it is possible that there is no reminder for the user.

### A note about data types between the client and contract

Messages are received by contracts as encrypted and then Base64-encoded versions of the JSON stringified version of the original messages (i.e., Javascript object) defined in the client code. This transformation is transparent to you as a Secret Contract developer, but awareness of this process is important because of how it affects data types. Your contract will create a schema document for each message type if you add the `derive(JsonSchema)` macro to your message definitions, but you might still need to do some additional value checking and type casting in your contract code depending on the context.

In addition, number values in Javascript are limited in range. The maximum safe integer value in Javascript falls somewhere between maximum `i32` and `i64` values in Rust. Therefore, 128-bit integers, for example, need to be sent from the client as string values because 128-bit numbers are commonly used in contracts to represent currency values (e.g. $\mu$SCRT), the Cosmos SDK (which Secret Network is built on) has a pre-defined type `Uint128`. 

A message type with a `Uint128` field will expect a string from the incoming JSON, which is validated to be a correct representation of a 128-bit unsigned integer value. In order to use the `Uint128` field value in your contract code, e.g., to put it in contract storage, you will then need to convert it to a Rust `u128` type.

Likewise, a message type with `HumanAddr` or `CanonicalAddr` as field values will also be sent from the client using string values.

## Secret Contract storage

Now we are going to define how we want to model our contract in the contract storage. We will put that code in `src/state.rs`. Once we have completed that we need to make everything work together in our `init`, `handle`, and `query` functions in `src/contract.rs`, then we will have a fully working Secret Contract!

Conceptually, storage for a Secret Contract is quite simple. It is a key-value store on the chain where each unit of data is identified by a unique key and the value of the data is a serialized representation of some data structure in Rust. Storage is encrypted, and only the contract has access to its own storage.

For our contract we need to store two types of information: 
    1) General state information for the contract 
    2) The reminder messages for each user 
    
Add the following code in `src/state.rs`:

```rust
use std::{any::type_name};
use serde::{Deserialize, Serialize};
use cosmwasm_std::{Storage, ReadonlyStorage, StdResult, StdError};
use serde::de::DeserializeOwned;
use secret_toolkit::serialization::{Bincode2, Serde};

pub static CONFIG_KEY: &[u8] = b"config";

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq)]
pub struct State {
    pub max_size: u16,
    pub reminder_count: u64,
}

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq)]
pub struct Reminder {
    pub content: Vec<u8>,
    pub timestamp: u64,
}
```

First, we define a `static` unique key to point to our `State` struct and give it the value `b"config"`. Note, we will also need unique key values for each `Reminder`, but we will wait to create those in our `handle` function using the address of the sender. Next, we define our `State` struct, which keeps track of the `max_size` of the reminder messages along with a running count of the number of users and total reminders recorded. A `Reminder` consists of the reminder `content` (as a vector of bytes), and the timestamp when it was recorded.

You can serialize your data on storage in any way you want. We recommend you use `bincode2` serialization from the [Secret Contract Development Toolkit](https://github.com/scrtlabs/secret-toolkit) if you do not want numbers and `Option` types encoded on the chain at variable lengths. Other types of serialization, such as json encode numbers as strings, so different values can have different byte lengths in storage, which can lead to data leakage if information is discerned due to that difference (see [here](https://github.com/baedrik/SCRT-sealed-bid-auction/blob/master/WALKTHROUGH.md#staters) and [here](https://docs.scrt.network/dev/privacy-model-of-secret-contracts.html#api-calls-2) for more detailed information).

We now define three helper functions in `state.rs` to read and write data to storage using bincode2 <sup id="a2">[2](#f2)</sup>:

* `save` will serialize a struct using `bincode2` and write it to storage using the storage `set()` method.
* `load` will retrieve the data from storage using the `get()` method, deserialize it, and return a `StdResult` with the data. If the key is not found, a "not found" `StdError` is returned. Using the `?` operator in the calling function will cause the error to be sent back up as the response.
* `may_load` is an alternative implementation of `load` returning an `Option` form of the result. If the key is not found, `Ok(None)` is returned. This version is more convenient if you want to customize error reporting when the data is not found.

Add the following code to the end of the state.rs file: 

```rust
pub fn save<T: Serialize, S: Storage>(storage: &mut S, key: &[u8], value: &T) -> StdResult<()> {
    storage.set(key, &Bincode2::serialize(value)?);
    Ok(())
}

pub fn load<T: DeserializeOwned, S: ReadonlyStorage>(storage: &S, key: &[u8]) -> StdResult<T> {
    Bincode2::deserialize(
        &storage
            .get(key)
            .ok_or_else(|| StdError::not_found(type_name::<T>()))?,
    )
}

pub fn may_load<T: DeserializeOwned, S: ReadonlyStorage>(storage: &S, key: &[u8]) -> StdResult<Option<T>> {
    match storage.get(key) {
        Some(value) => Bincode2::deserialize(&value).map(Some),
        None => Ok(None),
    }
}
```

## Wiring it all together in our Secret Contract functions

Now we are ready to fill in our three contract functions in `contract.rs`: `init`, `handle`, and `query`.

### `init` function

In `state.rs` we have said that `max_size` will be stored as a `u16` type, which means that the highest maximum reminder size is 65535 bytes. If the `i32` value for `max_size` sent in `InitMsg` is outside of those bounds we need to throw an error. We can create a helper function to test that by adding the following cose, which will cause our `init` function to return a `StdError` with an informative error message to the client if `max_size` is out of bounds: 

```rust
pub fn init<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    _env: Env,
    msg: InitMsg,
) -> StdResult<InitResponse> {

    let max_size = match valid_max_size(msg.max_size) {
        Some(v) => v,
        None => return Err(StdError::generic_err("Invalid max_size. Must be in the range of 1..65535."))
    };

    ...
```

Note: We changed `env` to `_env` because we will not use it in our `init` function, and the Rust compiler will emit a warning otherwise.

If no error is returned, we create a new instantiation for the `State` struct that initializes `reminder_count` to 0. Then we call the `save` function to send it to storage, and finally return a default `InitResponse` to the client. Add the following to the end of your `init` function:

```rust
    ...
    let config = State {
        max_size,
        reminder_count: 0_u64,
    };

    save(&mut deps.storage, CONFIG_KEY, &config)?;
    Ok(InitResponse::default())
}
```

### `handle` function

Unlike `init` when you create a `handle` function you will likely need to implement logic for multiple types of handle messages. Instead of one large `handle` function we can easily create a helper function for each type of message. We will name them with the form `try_*` where `*` denotes the message type. The parameters of our message depend on how we defined them in `msg.rs`. In our case, we have two handle message types. The first, `HandleMsg::Record`, has one field `reminder` that we pass on to `try_record` along with `deps` and `env`. The second message type, `HandleMsg:Read` takes no parameters so we only pass on `deps` and `env`.

```rust
pub fn handle<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    env: Env,
    msg: HandleMsg,
) -> StdResult<HandleResponse> {
    match msg {
        HandleMsg::Record { reminder } => try_record(deps, env, reminder),
        HandleMsg::Read { } => try_read(deps, env),
    }
}
```

#### `try_record`

The main logic for our record message handling goes in the `try_record` function:

```rust
fn try_record<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    env: Env,
    reminder: String,
) -> StdResult<HandleResponse> {
    let status: String;
    let reminder = reminder.as_bytes();

    // retrieve the config state from storage
    let mut config: State = load(&mut deps.storage, CONFIG_KEY)?;

    if reminder.len() > config.max_size.into() {
        // if reminder content is too long, set status message and do nothing else
        status = String::from("Message is too long. Reminder not recorded.");
    } else {
        // get the canonical address of sender
        let sender_address = deps.api.canonical_address(&env.message.sender)?;

        // create the reminder struct containing content string and timestamp
        let stored_reminder = Reminder {
            content: reminder.to_vec(),
            timestamp: env.block.time
        };

        // save the reminder using a byte vector representation of the sender's address as the key
        save(&mut deps.storage, &sender_address.as_slice().to_vec(), &stored_reminder)?;

        // increment the reminder_count
        config.reminder_count += 1;
        save(&mut deps.storage, CONFIG_KEY, &config)?;

        // set the status message
        status = String::from("Reminder recorded!");
    }

    // return a HandleResponse with the appropriate status message included in the data field
    Ok(HandleResponse {
        messages: vec![],
        log: vec![],
        data: Some(to_binary(&HandleAnswer::Record {
            status,
        })?),
    })
}
```

First thing we do is define a String that will hold our `status` message and convert the incoming `reminder` message into a byte slice. Next, we load the config state from storage and put it in a variable called `config`. We can use `load` and the `?` operator because we know that it was created in `init`. We test the length of our incoming reminder (# of bytes) against the `max_size` field in our config struct. If the message is too long, we indicate it in `status`.

If the incoming reminder is an acceptable length, then we need to store the new reminder and its timestamp using a key derived from the current sender's address. Once that is done we increment the `reminder_count`. To get the address we use `deps.api.canonical_address` method and pass it the current sender from `env`. The result is stored in `sender_address`. We construct a `Reminder` struct and set the `content` to a `vec<u8>` representation of the reminder byte slice and the current block time, also from `env`. Keys in storage are byte sequences, so to use `sender_address` as a key we need to call `.as_slice().to_vec()`. We use `save` to write the new `Reminder` at this key. Finally, we update the config by incrementing `reminder_count`, overwrite it in storage, and set the `status` message.

The return value of our function is a `StdResult<HandleResponse>`. The `msg` field in `HandleResponse` is a vector of `CosmosMsg`s which are actions taken after execution, and `log` is a vector of logging attributes as key-value pairs. For this contract we do not need those, and can simply return empty vectors for those two fields. We want to send our `HandleAnswer::Record` response back to the client in binary-encoded form so we pass it through the `to_binary` Cosmos SDK function.

#### `try_read`

The logic for handling a read function uses many of the same components. One difference from previous code is that we use the `may_load` function to read the `Reminder` from storage. This allows us to handle situations where a sender tries to read a reminder and none exists on storage. We can send a message in `status` without sending an error message.

```rust
fn try_read<S: Storage, A: Api, Q: Querier>(
    deps: &mut Extern<S, A, Q>,
    env: Env,
) -> StdResult<HandleResponse> {
    let status: String;
    let mut reminder: Option<String> = None;
    let mut timestamp: Option<u64> = None;

    let sender_address = deps.api.canonical_address(&env.message.sender)?;

    // read the reminder from storage
    let result: Option<Reminder> = may_load(&mut deps.storage, &sender_address.as_slice().to_vec()).ok().unwrap();
    match result {
        // set all response field values
        Some(stored_reminder) => {
            status = String::from("Reminder found.");
            reminder = String::from_utf8(stored_reminder.content).ok();
            timestamp = Some(stored_reminder.timestamp);
        }
        // unless there's an error
        None => { status = String::from("Reminder not found."); }
    };

    // Return a HandleResponse with status message, reminder, and timestamp included in the data field
    Ok(HandleResponse {
        messages: vec![],
        log: vec![],
        data: Some(to_binary(&HandleAnswer::Read {
            status,
            reminder,
            timestamp,
        })?),
    })
}
```

### `query` function

The implementation of our `Stats` query is quite straightforward compared to handle requests. One difference is that `query` simply returns a `StdResult<Binary>`, because it only needs to return the binary-encoded `Stats` struct containing `reminder_count`. A `query` does not access `env` and does not affect on-chain transactions, so there is no need to wrap it up with `messages` and `log` like in a `HandleResponse`.

```rust
pub fn query<S: Storage, A: Api, Q: Querier>(
    deps: &Extern<S, A, Q>,
    msg: QueryMsg,
) -> StdResult<Binary> {
    match msg {
        QueryMsg::Stats { } => query_stats(deps)
    }
}

fn query_stats<S: Storage, A: Api, Q: Querier>(deps: &Extern<S, A, Q>) -> StdResult<Binary> {
    // retrieve the config state from storage
    let config: State = load(&deps.storage, CONFIG_KEY)?;
    to_binary(&QueryAnswer::Stats{ reminder_count: config.reminder_count })
}
```

If you have completed this tutorial ssuccessfully, you will be able to compile the contract code without any errors using: 

```bash 
cargo build
```

You now have a working Secret Contract! 

If you are experiencing errors when compiling, or want to view the full contract code, see the completed version [HERE](https://github.com/darwinzer0/secret-contract-tutorials/tree/main/tutorial1/code).

## Next steps

Unlike a normal web service, there is no mechanism for a Secret Contract to repeatedly push information through an open socket connection in response to a handle or query message. Instead, if you want to support that behavior, then you must develop a pull mechanism where the client makes repeated executions of the contract. However, as our contract currently implements the functionality of reading a reminder as a handle execution that would very quickly cost the user a lot of `SCRT` due to gas fees! The solution is to create a private viewing key allowing a user to see their own reminder using a query instead of a handle message. In the next tutorial we will show how that can be done in the context of a simple React application.

## Notes

<b id="f1">1</b>: Although queries do not impose a fee, they are metered by gas. This allows a node to reject a long-running query.[↩](#a1)

<b id="f2">2</b>: These functions are based on the [Sealed Bid Auction contract code](https://github.com/baedrik/SCRT-sealed-bid-auction/blob/master/src/state.rs).[↩](#a2)

## About the author

This tutorial was written by Ben Adams, a senior lecturer in computer science and software engineering at the University of Canterbury in New Zealand.

<div class="cc">
<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution 4.0 International License</a>.
</div>

