---
title: "Develop an Ethereum bridge with Rust"
date: 2022-11-14T00:03:47+02:00
tags:
  - blockchain
  - oracle
  - bridge
  - rust
  - alchemy
  - infura
  - web3
image: "/images/post_pics/oraclecover.jpg"
---

A blockchain bridge is a system that allows the transfer of assets between two different blockchains. It is a crucial component of the blockchain ecosystem as it allows the interoperability of different blockchains. In this article, we will develop a bridge between two EVM-compatible blockchains. We will use the Rust programming language.

## 1. Bridges are oracles

Bridges are two-way oracles, which means that they can be used to send data from one blockchain to another. The data can be anything, but in this article, we will focus on sending and burning tokens in two different EVM-compatible blockchains.

More information about oracles can be found in the following article:
[Develop an Ethereum oracle with Rust](https://lorenzozaccagnini.it/posts/simple-rust-oracle/)

## 2. A bridge architecture

The bridge architecture is very simple from a general point of view. It consists of two smart contracts, one on each blockchain. The first contract is called the **bridge contract**. It is deployed on the source blockchain and it is responsible for locking or burning the tokens that need to be transferred. The second contract is called the **destination contract**. It is deployed on the destination blockchain and it is responsible for minting the tokens that need to be transferred. This architecture can work in both directions, but in this article, we will focus on the transfer of tokens from the source blockchain to the destination blockchain.

## 3. The bridge contract

In this case we will develop a "Burn and mint" architecture, so the smart contract on the source blockchain will be responsible for burning the tokens that need to be transferred. The smart contract on the destination blockchain will be responsible for minting the tokens that need to be transferred. **Again this can work in both directions**.

### 3.1 The bridge token

The bridge token will be a simple ERC20 token. ERC20 token is a standard for tokens on the Ethereum blockchain. It is a very simple standard that allows the creation of tokens that can be transferred, received, and burned. I will use openzeppelin contracts to develop the bridge token. Openzeppelin is a very popular library for smart contracts development. It contains a lot of useful contracts that can be used to develop smart contracts.

Burning means that the tokens are destroyed. Burning tokens is a very useful feature for a token. It allows the token to be deflationary. Technically means to send the tokens to the address 0x0000000000000000000000000000000000000000. This address is called the zero address and it is a special address that is used to burn tokens. **No one can access the zero address, so the tokens are destroyed forever.**

Transferring assets between two blockchains without burning or locking will cause a double spending problem. The double spending problem is a problem that occurs when the same asset is spent more than once. In this case, the asset is the token. If the token is not burned or locked, it can be spent on both blockchains. This will cause a double spending problem and makes the token worthless **like your developer skills.**

This will happen with a probability of 101% because bridges are oracles. Oracles are not 101% reliable. They can fail and they often fail. If this bridge fails, the token will be spent on both blockchains and people on twitter will call you a scammer. So if you are a scammer you can skip this step.

### 3.2 Coding the bridge contract

The smart contract code will be the same on both blockchains, but deployed obviously on both blockchains. The code is very simple and it is composed of two functions: `burn` and `mint`. The `burn` function is responsible for burning the tokens that need to be transferred. The `mint` function is responsible for minting the tokens that need to be transferred.

Only the owner of the smart contract can call the `mint` function. The owner of the smart contract is the address that deployed the smart contract. If someone that is not the owner of the smart contract can call the `mint` function, the bridge will be vulnerable to attacks and people on twitter will call you a scammer, again. So please use a multisig wallet to deploy the smart contract, so people on twitter will not call only you a scammer, but also the other people in the multisig wallet. LGTM.

Please don't be the owner of all the multisig wallets, because people on twitter will call you a scammer and a dictator.

The `burn` function is responsible for burning the tokens that need to be transferred and can be called by any token holder that has a balance greater than zero.

```solidity
pragma solidity ^0.8.9;

import "@openzeppelin/contracts@4.8.0/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts@4.8.0/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts@4.8.0/access/Ownable.sol";

contract GBridgeToken is ERC20, ERC20Burnable, Ownable {
    constructor() ERC20("gBridgeToken", "GBT") {
        _mint(msg.sender, 1000 * 10**decimals());
    }

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
}
```

The functions with the underscore are inherited from the openzeppelin contracts. The `mint` function is responsible for minting the tokens. The `burn` function is responsible for burning the tokens. The `onlyOwner` modifier is responsible for checking if the caller of the function is the owner of the smart contract. The `owner` is the address that deployed the smart contract. The `msg.sender` is the address that called the function.

In the constructor I premint 1000 tokens and I assign them to the address that deployed the smart contract. **This is not necessary, but it is a good practice to premint some tokens to the address that deployed the smart contract.** This will allow the owner of the smart contract to test the bridge before deploying it on the mainnet.

## 3.3 Deploy the bridge contracts

The bridge contracts can be deployed on any EVM-compatible blockchain. In this article, I will deploy the bridge contracts on two local ganache blockchains. One on `localhost:8545` and one on `localhost:7545`.

I use remix to connect to the ganache blockchains. Remix is a web IDE for smart contracts development. It is very useful for testing smart contracts. It allows you to connect to different blockchains and to deploy smart contracts. It also allows you to interact with the smart contracts.

Copy the code into remix, compile, select the ganache local chain and deploy the smart contract. If you don't know how to do this you are on the dunning-kruger curve bad side and you should not be developing smart contracts. If you are scammer you can skip this step.

Jokes apart, learn the basics before developing smart contracts and oracles, people can get really angry on twitter.

## 4. Events

Events are a very useful feature of smart contracts. They allow you to log data in the blockchain. The data can be anything, but in this case, we will listen to the transfer event of the bridge token. The transfer event is emitted every time a token is transferred.

If the contracts are deployed correctly you will see transfer events `mint` or `burn` are called. The `burn` event should look like this:

```javascript
"topics": [
    "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef",
    "0x000000000000000000000000399cf2e8d5c14ac04f1599c844a42be4d712b3eb",
    "0x0000000000000000000000000000000000000000000000000000000000000000"
]
```

The event signature for ERC20 transfers, equals sha3("Transfer(address,address,uint256)"), to know more about events and topic check my other article [Develop an Ethereum oracle with Rust](https://lorenzozaccagnini.it/posts/simple-rust-oracle/)

## 5. The bridge rust code

The rust code is in part taken from the article [Develop an Ethereum oracle with Rust](https://lorenzozaccagnini.it/posts/simple-rust-oracle/). The rust code is responsible for listening to the transfer events of the bridge token and for calling the `mint` function of the bridge contract on the destination blockchain when a transfer event to the zero address (a burn) is detected.

### 5.1 The bridge crates

This the Cargo.toml file used:

```toml
[package]
name = "blockchain_oracle"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
web3 = "0.17.0"
tokio = { version= "1", features = ["full"] }
hex = "0.4.3"
ethnum = "1.3.0"
```

- **web3** is the crate used to interact with the blockchain.
- **tokio** is the crate used to run the async code.
- **hex** is the crate used to convert the bytes to hex. ethnum is the crate used to convert the bytes to u256.

### 5.2 Connecting to the blockchain

Let's start by connecting to the blockchain.

```rust
use ethnum::U256;
use web3::contract::{Contract, Options};
use web3::futures::StreamExt;

#[tokio::main]
async fn main() -> web3::contract::Result<()> {
    let web3_source_chain_ws =
        web3::Web3::new(web3::transports::WebSocket::new("ws://localhost:8545").await?);
}
```

The first part imports the crates, the second part is the main function, the third part is the code that connects to the blockchain. The `web3_source_chain_ws` is the web3 instance used to connect to the source blockchain.

### 5.3 Listening to the transfer events

Here I filter and decode the transfer event, dividing the burn and normal transfer events.

```rust
use ethnum::U256;
use web3::contract::{Contract, Options};
use web3::futures::StreamExt;

#[tokio::main]
async fn main() -> web3::contract::Result<()> {
    let web3_source_chain_ws =
        web3::Web3::new(web3::transports::WebSocket::new("ws://localhost:8545").await?);

    let event_signature = "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef";

    let source_sc_address = "0xB9d01d2E0FF04A2Ff2f0720Dd69e73F7671b55CE";


    let filter_source_transfer = web3::types::FilterBuilder::default()
        .address(vec![source_sc_address.parse().unwrap()])
        .from_block(web3::types::BlockNumber::Latest)
        .topics(
            Some(vec![event_signature.parse().unwrap()]),
            None,
            None,
            None,
        )
        .build();

    let sub_ganache = web3_source_chain_ws
        .eth_subscribe()
        .subscribe_logs(filter_source_transfer)
        .await?;

    let sub_ganache_logging = sub_ganache.for_each(|log| async move {
        let address = format!("{:?}", log.clone().unwrap().topics[2]);

        match address.as_str() {
            "0x0000000000000000000000000000000000000000000000000000000000000000" => {
                println!("Burned");
                let amount_decoded =
                    U256::from_str_radix(&hex::encode(log.unwrap().data.0), 16).unwrap();
                println!("Amount burned: {}", amount_decoded);
            }
            _ => {
                println!("Transferred");
            }
        }
    });

    sub_ganache_logging.await;

    Ok(())
}
```

The amount is not indexed so I decode it from the data field. The `address` is the second topic of the event, is where the token are sent, so I check if it is the zero address to know if it is a burn or a normal transfer.

If everything is working correctly you should see the `Burned` and `Amount burned` printed in the console, when you call the `burn` function on the source blockchain.

```
Burned
Amount burned: 666
```

### 5.4 Load the smart contract

Now we can listen, half of the work is done. Now we need to load the smart contract on the destination blockchain. We create a new `function mint_tokens`

```rust
async fn mint_tokens(amount: u64, account_target: &str, smart_contract_address: &str) {
    let web3_destination_chain =
        web3::Web3::new(web3::transports::Http::new("http://localhost:7545").unwrap());

    let web3_destination_chain_contract = Contract::from_json(
        web3_destination_chain.eth(),
        smart_contract_address.parse().unwrap(),
        include_bytes!("GBridgeToken.json"),
    )
    .unwrap();
```

The `web3_destination_chain_contract` is the contract instance on the destination blockchain. The `include_bytes!` macro is used to include the ABI of the smart contract. The ABI is generated by the `solc` compiler. The json file is the ABI that is generated by the `solc` compiler or remix IDE.

### 5.5 Minting the tokens

Now we can mint the tokens on the destination blockchain when we receive a burn event on the source blockchain.

```rust
async fn mint_tokens(amount: u64, account_target: &str, smart_contract_address: &str) {
    let web3_destination_chain =
        web3::Web3::new(web3::transports::Http::new("http://localhost:7545").unwrap());

    let web3_destination_chain_contract = Contract::from_json(
        web3_destination_chain.eth(),
        smart_contract_address.parse().unwrap(),
        include_bytes!("GBridgeToken.json"),
    )
    .unwrap();

    let ganache_accounts = web3_destination_chain.eth().accounts().await.unwrap();
    let account = ganache_accounts[0];

    //convert account_target to address
    let account_target_address =
        web3::types::Address::from_slice(&hex::decode(account_target.replace("0x", "")).unwrap());

    web3_destination_chain_contract
        .call(
            "mint",
            (account_target_address, amount),
            account,
            Options::default(),
        )
        .await
        .unwrap();
}
```

First I get the accounts from the destination blockchain, then I convert the `account_target` to an address. The `account_target` sends token to the same address on the destination blockchain.

The `account` is the account that will call the `mint` function, it should be the owner of the smart contract on the destination blockchain. The `Options::default()` is used to set the gas price and gas limit. The `web3_destination_chain_contract.call` is used to call the `mint` function.

The `amount` is the amount of tokens that will be minted on the destination blockchain, it is same quantity of the burned tokens on the source blockchain.

If you see the token balance of the `account_target` on the destination blockchain, you should see the added the amount of tokens that you burned on the source blockchain.

### 5.6 Putting it all together

Now we can put it all together. We need to listen to the transfer events on the source blockchain and mint the tokens on the destination blockchain.

```rust
use ethnum::U256;
use web3::contract::{Contract, Options};
use web3::futures::StreamExt;

#[tokio::main]
async fn main() -> web3::contract::Result<()> {
    let web3_source_chain_ws =
        web3::Web3::new(web3::transports::WebSocket::new("ws://localhost:8545").await?);

    let event_signature = "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef";

    let source_sc_address = "0xB9d01d2E0FF04A2Ff2f0720Dd69e73F7671b55CE";
    let destionation_sc_address = "0x4641B307794E29062906dc5fEd72152faEBB1C77";

    let filter_source_transfer = web3::types::FilterBuilder::default()
        .address(vec![source_sc_address.parse().unwrap()])
        .from_block(web3::types::BlockNumber::Latest)
        .topics(
            Some(vec![event_signature.parse().unwrap()]),
            None,
            None,
            None,
        )
        .build();

    let sub_ganache = web3_source_chain_ws
        .eth_subscribe()
        .subscribe_logs(filter_source_transfer)
        .await?;

    let sub_ganache_logging = sub_ganache.for_each(|log| async move {
        let address = format!("{:?}", log.clone().unwrap().topics[2]);
        let address_from_raw = format!("{:?}", log.clone().unwrap().topics[1]);
        let address_from_decoded = format!("0x{}", &address_from_raw[26..66]);

        match address.as_str() {
            "0x0000000000000000000000000000000000000000000000000000000000000000" => {
                println!("Burned");
                let amount_decoded =
                    U256::from_str_radix(&hex::encode(log.clone().unwrap().data.0), 16).unwrap();
                println!("Amount burned: {}", amount_decoded);

                //mint tokens on the destination chain
                mint_tokens(
                    amount_decoded.as_u64(),
                    &address_from_decoded,
                    &destionation_sc_address,
                )
                .await;

                println!("Burned from: {}", address_from_decoded);
            }
            _ => {
                println!("Transferred");
            }
        }
    });

    sub_ganache_logging.await;

    Ok(())
}

async fn mint_tokens(amount: u64, account_target: &str, smart_contract_address: &str) {
    let web3_destination_chain =
        web3::Web3::new(web3::transports::Http::new("http://localhost:7545").unwrap());

    let web3_destination_chain_contract = Contract::from_json(
        web3_destination_chain.eth(),
        smart_contract_address.parse().unwrap(),
        include_bytes!("GBridgeToken.json"),
    )
    .unwrap();

    let ganache_accounts = web3_destination_chain.eth().accounts().await.unwrap();
    let account = ganache_accounts[0];

    //convert account_target to address
    let account_target_address =
        web3::types::Address::from_slice(&hex::decode(account_target.replace("0x", "")).unwrap());

    web3_destination_chain_contract
        .call(
            "mint",
            (account_target_address, amount),
            account,
            Options::default(),
        )
        .await
        .unwrap();
}
```

I've added the mint function to the main function when a burn event is detected. The `mint_tokens` function is the same as the one we used in the previous section.

## 6. Aggregate transactions on a bridge oracle

An efficient bridge oracle will aggregate the transactions before sending them to the destination blockchain. It's really important to aggregate the transactions because it will reduce the gas cost and the transaction fees, imagine a bridge oracle use thousand of times every day that will send a transaction for each transfer event, it will be really expensive.

In order to do that is possible to use data structure like a Merkle tree. A Merkle tree is a data structure that allows to aggregate the transactions and verify the aggregated transactions. In the future we will see the use of Verkle trees that are more efficient than Merkle trees. Maybe this topic will be covered in a future article.

## 7. Do you need to develop an oracle or a bridge?

You can contact me [Lorenzo Zaccagnini](https://www.linkedin.com/in/lorenzo-zaccagnini/) or [Elisa Romondia](https://www.linkedin.com/in/elisa-romondia/) on LinkedIn. If you want to support me you can donate eth or matic to 0xbf8d0d4be61De94EFCCEffbe5D414f911F11cBF8
