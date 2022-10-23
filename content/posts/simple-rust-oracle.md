---
title: "Develop an Ethereum oracle with Rust"
date: 2022-10-22T00:03:47+02:00
tags:
  - blockchain
  - oracle
  - rust
  - alchemy
  - infura
  - web3
image: "/images/post_pics/oraclecover.jpg"
---

A blockhain oracle is a service that allows smart contracts to interact with external data sources. In this post, we will develop a simple oracle that will track everytime a [Ethereum Name Service NFT](https://etherscan.io/address/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85) is transferred. We will use Rust and the web3 crate to interact with the Ethereum blockchain.

## 1. Why blockchains can't access external data sources

Blockchains can't access external data sources natively because it is a deterministic system. Each node in the network has a copy of the blockchain, they must all agree on the same state. If a smart contract was able to access external data sources, it would break the deterministic nature of the blockchain. The verification of the state of the blockchain would be impossible, remember same inputs always produce the same outputs.

We can solve this problem by using a blockchain oracle. A blockchain oracle is a service that allows smart contracts to interact with external data sources. The oracle will be responsible for fetching the external data and sending it to the smart contract. The smart contract will then be able to access the data.

Let's make an example to illustrate this. Let's say we have a smart contract that stores the current price of a cryptocurrency. If the price of the cryptocurrency is updated every 10 seconds on an external API, the smart contract will have to be updated every 10 seconds, otherwise the smart contract will be out of sync. It would impossible for others to verify the state of the blockchain.

This is why blockchains need oracles, external data must be fed into the blockchain with a transaction, in this way all nodes in the network will have the same data and the blockchain will remain deterministic.

[Read this fantastic answer on stackoverflow to learn more about that](https://ethereum.stackexchange.com/questions/301/why-cant-contracts-make-api-calls)

In my experience I've developed many blockchain oracles, for our project Devoleum and for many other projects. I've used different langauges but Rust is the one that I love the most. I've developed oracles for various use cases:

- Crosschain bridges
- Connecting IoT to the Ethereum blockchain
- Display data from the blockchain for a frontend that does not have access to the blockchain via a browser wallet like metamask.

## 2. What is an oracle?

Most of the time is a simple service that listens to events on the blockchain and updates the state of the smart contract. It can also be a smart contract that is called by other smart contracts to get external data. In this article we will develop a simple service that will listen to events on the ethereum blockchain, more precisely the transfer event of ENS NFTs.

Different types of oracles exist, some of them are:

- **Inbound oracles**: they listen to events on the blockchain
- **Outbound oracles**: they send transactions to the blockchain
- **Hybrid oracles**: they listen to events on the blockchain and send transactions to the blockchain

## 3. Setup the ethereum oracle project

We will develop a simple oracle that will listen to the transfer event of ENS NFTs. We will use Rust and the web3 crate to interact with the Ethereum blockchain. We will use the Alchemy API to interact with the Ethereum blockchain, otherwise it will be necessary to run a full node.

All the code of this article is available on [Github](https://github.com/LorenzoZaccagnini/simple-rust-ethereum-inbound-oracle).

### 3.1. Install Rust

If you don't have Rust installed on your machine, you can follow the [official installation guide](https://www.rust-lang.org/tools/install).

### 3.2. Create a new project

We will use the cargo command to create a new project. Cargo is the Rust package manager and build system.

```bash
cargo new simple-rust-oracle
```

### 3.3. Add the dependencies

We will add the web3 and other dependencies to our **Cargo.toml** file:

- **Web3** crate is a Rust library for interacting with Ethereum and other blockchain nodes. It provides a full set of features for interacting with the blockchain, including sending transactions, reading data from the blockchain, and listening to events.
- **Tokio** is a runtime for asynchronous Rust applications.
- The **dotenv** crate is used to load environment variables from a .env file, we'll use it to load our Alchemy API key without hardcoding it in our code (and exposing it to the world).
- **Ethnum** is used to handle big unsigned integers.

This how our **Cargo.toml** file should look like:

```toml
[dependencies]
web3 = "0.17.0"
tokio = { version= "1", features = ["full"] }
dotenv = "0.15.0"
ethnum = "1.3.0"
```

### 3.4. Create a .env file

We will create a .env file in the root of our project. We will store our [Alchemy API key](https://www.alchemy.com/) in this file and load it with the dotenv crate.

You can use [Infura](https://infura.io/) instead of Alchemy, just replace the Alchemy API key with your Infura API key.

In both cases you have to signup to get an API key. **I will use the mainnet API key** to listen to the ENS NFTs transfer events. You can use the testnet API key if you want to test the oracle on the testnet.

```bash
touch .env
```

The .env file should look like this:

```bash
ALCHEMY_API_KEY=wss://eth-mainnet.g.alchemy.com/v2/S0meR4nd0mStr1ng
```

## 4. Develop the ethereum oracle

### 4.1. Load the environment variables

We will load the environment variables with the dotenv crate. We will use the **dotenv::dotenv()** function to load the environment variables from the .env file. We will use the **dotenv::var()** function to get the value of a specific environment variable.

```rust

use dotenv::dotenv;

fn main() {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
}
```

### 4.2. Connect to the Ethereum blockchain

We will use the web3 crate and the Alchemy API to connect to the Ethereum blockchain.

```rust
use dotenv::dotenv;
use web3;

fn main() {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::Http::new(&alchemy_api_key).unwrap());
}
```

### 4.3. Filter to the ENS NFTs transfer events

We need to know the ENS smartcontract address to listen to the transfer events. We can find the address of the ENS smartcontract on [Etherscan](https://etherscan.io/address/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85#code).

**WAIT!** We want to listen to a specific event, no to every event of the smart contract so we need to know the event signature. The event signature is the hash of the event name and the event parameters.

Signature or topic0 = 0x + keccak256("Transfer(address,address,uint256)"))

0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef = Transfer(address,address,uint256)

As you can see here on [Etherscan](https://etherscan.io/address/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85#events) the event signature is 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef.

![](/images/post_pics/simple_rust_oracle/eventetherscan.jpg)

Let's code it!

```rust
use dotenv::dotenv;
use web3;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::WebSocket::new(&alchemy_api_key).await?);

    let contract_address = "0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85";
    let event_signature = "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef";

    let filter = web3::types::FilterBuilder::default()
        .address(vec![contract_address.parse().unwrap()])
        .from_block(web3::types::BlockNumber::Latest)
        .topics(
            Some(vec![event_signature.parse().unwrap()]),
            None,
            None,
            None,
        )
        .build();

    Ok(())
}
```

As you can read I set the contract address and the event signature. I also wrote the filter to listen to the latest block. The filter contains the contract address and the event signature. Note that I've used the **topics** function to filter to the specific event and Tokio to run the code asynchronously.

### 4.4. Listen and print the Ethereum ENS transfer events

Now we need to subscribe to the filter and listen to the events. We will use the **web3.eth_subscribe()** function to subscribe to the filter. We will use the **web3::types::Log** struct to decode the event data.

```rust
use dotenv::dotenv;
use web3;
use web3::futures::{future, StreamExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::WebSocket::new(&alchemy_api_key).await?);

    let contract_address = "0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85";
    let event_signature = "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef";

    let filter = web3::types::FilterBuilder::default()
        .address(vec![contract_address.parse().unwrap()])
        .from_block(web3::types::BlockNumber::Latest)
        .topics(
            Some(vec![event_signature.parse().unwrap()]),
            None,
            None,
            None,
        )
        .build();

    let transfer_listen = web3.eth_subscribe().subscribe_logs(filter).await?;

    transfer_listen
        .for_each(|log| {
            println!("log: {:?}", log);
            future::ready(())
        })
        .await;

    Ok(())
}
```

I've used **future::ready** to run the code asynchronously. I've also used the **for_each** function to iterate over the events.

**The result should be this:**

![](/images/post_pics/simple_rust_oracle/transferlog.jpg)

### 4.5. Decode the event data

We need to decode the event data to get the transfer details. First we import ethnum, after we decode the event data in the **transfer_listen** loop. We get the hex string of token id from the fourth topic, after I use **from_str_radix()** from **ethnum** to convert the hex string to a U256.

```rust
use dotenv::dotenv;
use ethnum::U256;
use web3;
use web3::futures::{future, StreamExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::WebSocket::new(&alchemy_api_key).await?);

    let contract_address = "0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85";
    let event_signature = "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef";

    let filter = web3::types::FilterBuilder::default()
        .address(vec![contract_address.parse().unwrap()])
        .from_block(web3::types::BlockNumber::Latest)
        .topics(
            Some(vec![event_signature.parse().unwrap()]),
            None,
            None,
            None,
        )
        .build();

    let transfer_listen = web3.eth_subscribe().subscribe_logs(filter).await?;

    transfer_listen
        .for_each(|log| {
            let id = format!("{:?}", log.unwrap().topics[3]);
            println!("id NOT decoded: {:?}", id);
            let id_decoded = U256::from_str_radix(&id[2..], 16).unwrap();
            println!("id decoded: {:?}", id_decoded);
            println!("----------");
            future::ready(())
        })
        .await;

    Ok(())
}
```

**The result should be this:**

![](/images/post_pics/simple_rust_oracle/transferid.jpg)

## 5. Do you need to develop an oracle or a bridge?

You can contact me [Lorenzo Zaccagnini](https://www.linkedin.com/in/lorenzo-zaccagnini/) or [Elisa Romondia](https://www.linkedin.com/in/elisa-romondia/) on LinkedIn. If you want to support me you can donate eth or matic to 0xbf8d0d4be61De94EFCCEffbe5D414f911F11cBF8
