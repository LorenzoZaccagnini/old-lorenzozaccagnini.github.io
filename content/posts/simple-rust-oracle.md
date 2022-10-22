---
title: "Develop an Ethereum oracle with Rust"
date: 2022-22-10T00:03:47+02:00
tags:
  - blockchain
  - oracle
  - rust
  - alchemy
  - infura
  - web3
image: "/images/post_pics/soulbound-nft-cover.jpg"
---

A blockhain oracle is a service that allows smart contracts to interact with external data sources. In this post, we will develop a simple oracle that will track everytime a [CryptoKitty NFT](https://etherscan.io/address/0x06012c8cf97bead5deae237070f9587f8e7a266d) is transferred. We will use Rust and the web3 crate to interact with the Ethereum blockchain.

## 1. Why blockchains can't access external data sources

Blockchains can't access external data sources natively because it is a deterministic system. Each node in the network has a copy of the blockchain, they must all agree on the same state. If a smart contract was able to access external data sources, it would break the deterministic nature of the blockchain. The verification of the state of the blockchain would be impossible, remember same inputs always produce the same outputs.

Let's make an example to illustrate this. Let's say we have a smart contract that stores the current price of a cryptocurrency. If the price of the cryptocurrency is updated every 10 seconds on an external API, the smart contract will have to be updated every 10 seconds, otherwise the smart contract will be out of sync. It would impossible for others to verify the state of the blockchain.

This is why blockchains need oracles, external data must be fed into the blockchain with a transaction, in this way all nodes in the network will have the same data and the blockchain will remain deterministic.

[Read this fantastic answer on stackoverflow to learn more about that](https://ethereum.stackexchange.com/questions/301/why-cant-contracts-make-api-calls)

## 2. What is an oracle?

Most of the time is a simple service that listens to events on the blockchain and updates the state of the smart contract. It can also be a smart contract that is called by other smart contracts to get external data. In this case we will develop a simple service that will listen to events on the ethereum blockchain, more precisely the transfer event of CryptoKitties NFTs.

Different types of oracles exist, some of them are:

- Inbound oracles: they listen to events on the blockchain
- Outbound oracles: they send transactions to the blockchain
- Hybrid oracles: they listen to events on the blockchain and send transactions to the blockchain

## 3. How to develop an oracle

We will develop a simple oracle that will listen to the transfer event of CryptoKitties NFTs. We will use Rust and the web3 crate to interact with the Ethereum blockchain. We will use the Alchemy API to interact with the Ethereum blockchain, otherwise you will need to run a full node.

### 3.1. Install Rust

If you don't have Rust installed on your machine, you can follow the [official installation guide](https://www.rust-lang.org/tools/install).

### 3.2. Create a new project

We will use the cargo command to create a new project. Cargo is the Rust package manager and build system.

```bash
cargo new simple-rust-oracle
```

### 3.3. Add the dependencies

We will use the web3 crate to interact with the Ethereum blockchain. We will add the web3 crate to our **Cargo.toml** file. **Tokio** is a runtime for asynchronous Rust applications. The **dotenv** crate is used to load environment variables from a .env file, we'll use it to load our Alchemy API key without hardcoding it in our code (and exposing it to the world).

```toml
[dependencies]
web3 = "0.17.0"
tokio = { version= "1", features = ["full"] }
dotenv = "0.15.0
```

### 3.4. Create a .env file

We will create a .env file in the root of our project. We will store our [Alchemy API key](https://www.alchemy.com/) in this file. We will load this file in our code with the dotenv crate.

```bash
touch .env
```

The .env file should look like this:

```bash
ALCHEMY_API_KEY=wss://eth-mainnet.g.alchemy.com/v2/S0meR4nd0mStr1ng
```

### 3.5. Create a main.rs file

We will create a main.rs file in the src folder. This is the entry point of our application.

```bash
touch src/main.rs
```

### 3.6. Write the code
