<div align="center">

  <h1><code>near-sdk</code></h1>

  <p>
    <strong>Rust library for writing NEAR smart contracts.</strong>
  </p>
  <p>
    Previously known as <code>near-bindgen</code>.
  </p>


  <p>
    <a href="https://crates.io/crates/near-sdk"><img src="https://img.shields.io/crates/v/near-sdk.svg?style=flat-square" alt="Crates.io version" /></a>
    <a href="https://crates.io/crates/near-sdk"><img src="https://img.shields.io/crates/d/near-sdk.svg?style=flat-square" alt="Download" /></a>
    <a href="https://docs.rs/near-sdk"><img src="https://docs.rs/near-sdk/badge.svg" alt="Reference Documentation" /></a>
    <a href="https://discord.gg/gBtUFKR"><img src="https://img.shields.io/discord/490367152054992913.svg" alt="Join the community on Discord" /></a>
    <a href="https://github.com/near/near-sdk-rs/actions"><img src="https://github.com/near/near-sdk-rs/actions/workflows/test.yml/badge.svg" alt="GitHub Actions Build" /></a>
  </p>

   <h3>
      <a href="https://github.com/near/near-sdk-rs#features">Features</a>
      <span> | </span>
      <a href="https://github.com/near/near-sdk-rs#pre-requisites">Pre-requisites</a>
      <span> | </span>
      <a href="https://github.com/near/near-sdk-rs#writing-rust-contract">Writing Rust Contract</a>
      <span> | </span>
      <a href="https://github.com/near/near-sdk-rs#building-rust-contract">Building Rust Contract</a>
      <span> | </span>
      <a href="https://docs.rs/near-sdk">Reference Documentation</a>
      <span> | </span>
      <a href="https://github.com/near/near-sdk-rs#contributing">Contributing</a>
    </h3>
</div>

## Release notes

**Release notes and unreleased changes can be found in the [CHANGELOG](CHANGELOG.md)**

## Example

Wrap a struct in `#[near_bindgen]` and it generates a smart contract compatible with the NEAR blockchain:
```rust
use near_sdk::{near_bindgen, env};

#[near_bindgen]
#[derive(Default, BorshDeserialize, BorshSerialize)]
pub struct StatusMessage {
    records: HashMap<AccountId, String>,
}

#[near_bindgen]
impl StatusMessage {
    pub fn set_status(&mut self, message: String) {
        let account_id = env::signer_account_id();
        self.records.insert(account_id, message);
    }

    pub fn get_status(&self, account_id: AccountId) -> Option<String> {
        self.records.get(&account_id).cloned()
    }
}
```

## Features

### Unit-testable
Writing unit tests is easy with `near-sdk`:

```rust
#[test]
fn set_get_message() {
    let mut contract = StatusMessage::default();
    contract.set_status("hello".to_string());
    assert_eq!("hello".to_string(), contract.get_status("bob_near".to_string()).unwrap());
}
```

Run unit test the usual way:
```bash
cargo test --package status-message
```

### Asynchronous cross-contract calls
Asynchronous cross-contract calls allow parallel execution of multiple contracts in parallel with subsequent aggregation on another contract. `env` exposes the following methods:
* `promise_create` -- schedules an execution of a function on some contract;
* `promise_then` -- attaches the callback back to the current contract once the function is executed;
* `promise_and` -- combinator, allows waiting on several promises simultaneously, before executing the callback;
* `promise_return` -- treats the result of execution of the promise as the result of the current function.

Follow [examples/cross-contract-high-level](examples/cross-contract-calls/high-level)
to see various usages of cross contract calls, including **system-level actions** done from inside the contract like balance transfer (examples of other system-level actions are: account creation, access key creation/deletion, contract deployment, etc).

### Initialization methods
We can define an initialization method that can be used to initialize the state of the contract. `#[init]` verifies that the contract has not been initialized yet (the contract state doesn't exist) and will panic otherwise.

```rust
#[near_bindgen]
impl StatusMessage {
    #[init]
    pub fn new(user: String, status: String) -> Self {
        let mut res = Self::default();
        res.records.insert(user, status);
        res
    }
}
```

Even if you have initialization method your smart contract is still expected to derive `Default` trait. If you don't
want to disable default initialization, then you can prohibit it like this:
```rust
impl Default for StatusMessage {
    fn default() -> Self {
        near_sdk::env::panic_str("Contract should be initialized before the usage.")
    }
}
```
You can also prohibit `Default` trait initialization by using `near_sdk::PanicOnDefault` helper macro. E.g.:
```rust
#[near_bindgen]
#[derive(BorshDeserialize, BorshSerialize, PanicOnDefault)]
pub struct StatusMessage {
    records: HashMap<String, String>,
}
```

### Payable methods
We can allow methods to accept token transfer together with the function call. This is done so that contracts can define a fee in tokens that needs to be payed when they are used. By the default the methods are not payable and they will panic if someone will attempt to transfer tokens to them during the invocation. This is done for safety reason, in case someone accidentally transfers tokens during the function call.

To declare a payable method simply use `#[payable]` decorator:
```rust

#[payable]
pub fn my_method(&mut self) {
...
}
```

### Private methods
Usually, when a contract has to have a callback for a remote cross-contract call, this callback method should
only be called by the contract itself. It's to avoid someone else calling it and messing the state. Pretty common pattern
is to have an assert that validates that the direct caller (predecessor account ID) matches to the contract's account (current account ID).
Macro `#[private]` simplifies it, by making it a single line macro instead and improves readability.

To declare a private method use `#[private]` decorator:
```rust

#[private]
pub fn my_method(&mut self) {
...
}
/// Which is equivalent to

pub fn my_method(&mut self ) {
    if near_sdk::env::current_account_id() != near_sdk::env::predecessor_account_id() {
        near_sdk::env::panic_str("Method my_method is private");
    }
...
}
```

Now, only the account of the contract itself can call this method, either directly or through a promise.

## Pre-requisites
To develop Rust contracts you would need to:
* Install [Rustup](https://rustup.rs/):
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
* Add wasm target to your toolchain:
```bash
rustup target add wasm32-unknown-unknown
```

## Writing Rust Contract
You can follow the [examples/status-message](examples/status-message) crate that shows a simple Rust contract.

The general workflow is the following:
1. Create a crate and configure the `Cargo.toml` similarly to how it is configured in [examples/status-message/Cargo.toml](examples/status-message/Cargo.toml);
2. Crate needs to have one `pub` struct that will represent the smart contract itself:
    * The struct needs to implement `Default` trait which
    NEAR will use to create the initial state of the contract upon its first usage;
    * The struct also needs to implement `BorshSerialize` and `BorshDeserialize` traits which NEAR will use to save/load contract's internal state;

   Here is an example of a smart contract struct:
   ```rust
   use near_sdk::{near_bindgen, env};

   #[near_bindgen]
   #[derive(Default, BorshSerialize, BorshDeserialize)]
   pub struct MyContract {
       data: HashMap<u64, u64>
   }
   ```

3. Define methods that NEAR will expose as smart contract methods:
    * You are free to define any methods for the struct but only public methods will be exposed as smart contract methods;
    * Methods need to use either `&self`, `&mut self`, or `self`;
    * Decorate the `impl` section with `#[near_bindgen]` macro. That is where all the M.A.G.I.C. (Macros-Auto-Generated Injected Code) happens;
    * If you need to use blockchain interface, e.g. to get the current account id then you can access it with `env::*`;

    Here is an example of smart contract methods:
    ```rust
    #[near_bindgen]
    impl MyContract {
        pub fn insert_data(&mut self, key: u64, value: u64) -> Option<u64> {
            self.data.insert(key)
        }
        pub fn get_data(&self, key: u64) -> Option<u64> {
            self.data.get(&key).cloned()
        }
    }
    ```

## Building Rust Contract

### [cargo-near](https://github.com/near/cargo-near)

This can be used as an alternative, to allow building while also generating an [abi](https://github.com/near/abi)

```bash
# Install the near extension if you haven't already
cargo install cargo-near

# Builds the wasm contract and ABI into `target/near`
cargo near build --release
```

### Using cargo build

```bash
RUSTFLAGS='-C link-arg=-s' cargo build --target wasm32-unknown-unknown --release
```

## Building with reproducible builds

Since WebAssembly compiler includes a bunch of debug information into the binary, the resulting binary might be
different on different machines. To be able to compile the binary in a reproducible way, we added a Dockerfile
that allows to compile the binary.

**Use [contract-builder](https://github.com/near/near-sdk-rs/tree/master/contract-builder)**

## Versioning

### Semantic Versioning

This crate follows [Cargo's semver guidelines](https://doc.rust-lang.org/cargo/reference/semver.html). 

State breaking changes (low-level serialization format of any data type) will be avoided at all costs. If a change like this were to happen, it would come with a major version and come with a compiler error. If you encounter one that does not, [open an issue](https://github.com/near/near-sdk-rs/issues/new)!

### MSRV

The minimum supported Rust version is currently `1.56`. There are no guarantees that this will be upheld if a security patch release needs to come in that requires a Rust toolchain increase.

## Contributing

If you are interested in contributing, please look at the [contributing guidelines](CONTRIBUTING.md).
