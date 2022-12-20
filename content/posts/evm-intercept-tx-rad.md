---
title: "Intercept pending transactions with Rust"
date: 2022-12-20T00:03:47+02:00
tags:
  - blockchain
  - oracle
  - arbitrage
  - bot
  - rust
  - alchemy
  - frontrunning
  - web3
image: "/images/post_pics/evm-intercept.jpg"
---

Transactions on blockchain are not instant. They are pending until they are confirmed by the network. This is a security feature of the blockchain. This is why it is important to wait for the transaction to be confirmed before sending another transaction. In this article we will see how to intercept pending transactions with rust.

## 1. Transaction lifecycle

An Ethereum transaction lifecycle is as follows:

1. The transaction is created and signed by the sender.
2. The transaction is broadcasted to the network.
3. The transaction is pending until it is confirmed by the network.
4. The transaction is confirmed by the network.

The transaction is confirmed when it is included in a block. On Ethereum proof of stake network, the block is mined by a validator. The validator is a node that is running the Ethereum client and is participating in the consensus. The validator is selected randomly from the network. The validator is selected based on the stake that the validator has in the network.

## 2. The Mempool

The mempool is a pool of pending transactions. The transactions are pending until they are confirmed by the network. On Ethereum we don't have a single universal mempool. Each node has its own mempool. Even different clients use different jargon for the mempool.

- On Geth the mempool is called the transaction pool.
- On Parity the mempool is called the transaction queue.

### 2.1 Why intercept pending transactions?

There are many reasons and one of them is money, some bots intercept pending transactions to make a profit. Front running means that a bot will intercept a pending transaction and will execute another transaction before the original sender. It's possibile to frontrun a transaction by increasing the gas price. The gas price is the amount of money that the sender is willing to pay for the transaction to be confirmed. The higher the gas price, the higher the priority of the transaction. The transaction with the highest gas price is confirmed first.

Let's make an example, you are trading a token on Uniswap and you want to buy 100 tokens. You set the gas price to 10 Gwei and the transaction is pending. A bot sees your transaction in the pool and increases the gas price to 20 Gwei. Your transaction is now pending and the bot's transaction is confirmed before yours. The bot knows that your transaction will be executed and he will sell the tokens to you at a higher price. **This is called frontrunning**.

### 2.2 Do you want a sandwich?

The Ethereum network is a public network and anyone can see the pending transactions. It is possible to intercept pending transactions and make a profit. The bot will increase the gas price of the transaction and will execute the transaction before the original sender. **The sandwich attack is a type of frontrunning attack**, [it's about placing a trade before and after a victim trade, in order to exploit the slippage that has been created](https://github.com/Defi-Cartel/salmonella). The bot will buy the token and sell it to the original sender at a higher price. The bot will make a profit by selling the token at a higher price.

In a future article we will see how to make a sandwich attack, but for now let's see how to intercept and decode pending transactions.

## 3. Intercept pending transactions with rust

In this section we will see how to intercept pending transactions with rust. We will use the web3 library to interact with the Ethereum network. We will use the web3 library to intercept pending transactions and we will use the dotenv library to load the environment variables of alchemy.

### 3.1 Create a new rust project

Create a new rust project with cargo:

```bash
cargo new intercept_tx
```

Add the dependencies to the Cargo.toml file:

```toml
[package]
name = "evm-intercept-tx"
version = "0.1.0"
edition = "2021"

[dependencies]
dotenv = "0.15.0"
hex = "0.4.3"
tokio = "1.21.2"
web3 = "0.18.0"
```

### 3.2 Load the environment variables

Create a .env file and add the alchemy api key:

```bash
ALCHEMY_API_KEY=your_alchemy_api_key
```

Add the dotenv library to the main.rs file:

```rust
use dotenv::dotenv;
use hex;
use web3::futures::TryStreamExt;
use web3::types::TransactionId;

#[tokio::main]
async fn main() -> web3::Result {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
}
```

### 3.3 Connect to the Ethereum network

Add the web3 library to the main.rs file:

```rust
async fn main() -> web3::Result {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::WebSocket::new(&alchemy_api_key).await?);
}
```

Now you should be able to connect to the Ethereum network.

### 3.4 Intercept pending transactions

Here we will use the 'subscribe_new_pending_transactions' method of web3 to intercept pending transactions. The 'subscribe_new_pending_transactions' method returns a stream of pending transactions.

```rust
use dotenv::dotenv;
use hex;
use web3::futures::TryStreamExt;
use web3::types::TransactionId;

#[tokio::main]
async fn main() -> web3::Result {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::WebSocket::new(&alchemy_api_key).await?);

    let mut pending_transactions = web3
        .eth_subscribe()
        .subscribe_new_pending_transactions()
        .await?;

    while let Some(pending_transaction_hash) = pending_transactions.try_next().await? {
        let transaction = web3
            .eth()
            .transaction(TransactionId::from(pending_transaction_hash))
            .await?;
        if let Some(transaction) = transaction {
            println!("Transaction hash: {}", transaction);
        }
    }

    Ok(())
}
```

Now we are logging every pending transaction in the node mempool. The 'subscribe_new_pending_transactions' method returns a stream of pending transactions. The stream is an iterator that returns a pending transaction hash. We can use the 'transaction' method of web3 to get the transaction details. The 'transaction' method returns a transaction object. The transaction object contains the transaction details.

### 3.5 Filter pending transactions

We can filter pending transactions by the destination address. We can use the 'to' field of the transaction object to filter the transactions. The 'to' field contains the destination address of the transaction.

The address 0x31c8eacbffdd875c74b94b077895bd78cf1e64a3 is the RAD token. We will intercept the pending transactions of the RAD token.

```rust
use dotenv::dotenv;
use hex;
use web3::futures::TryStreamExt;
use web3::types::TransactionId;

#[tokio::main]
async fn main() -> web3::Result {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::WebSocket::new(&alchemy_api_key).await?);

    let mut pending_transactions = web3
        .eth_subscribe()
        .subscribe_new_pending_transactions()
        .await?;

    //filter transaction based on address
    let address = "0x31c8eacbffdd875c74b94b077895bd78cf1e64a3";

    while let Some(pending_transaction_hash) = pending_transactions.try_next().await? {
        let transaction = web3
            .eth()
            .transaction(TransactionId::from(pending_transaction_hash))
            .await?;
        if let Some(transaction) = transaction {
            //filter transaction based on address and method hash
            if transaction.to == Some(address.parse().unwrap()) {
                //decode input data bytes to hex
                println!("transaction: {:?}", transaction);
            }
        }
    }

    Ok(())
}
```

You should be able to intercept the pending transactions of the RAD token and see a log like this:

```bash
transaction: Transaction { hash: 0x4d5564bbedd6eb902e91b3c6a1d10a4c4029a036e9c4610fd6375932e2636e95, nonce: 5442921, block_hash: Some(0x15ba42779e7d34714607312e6b4f33f9a926738e527593402ff4ad15c1c7c7c2), block_number: Some(16228505), transaction_index: Some(28), from: Some(0x28c6c06298d514db089934071355e5743bf21d60), to: Some(0x31c8eacbffdd875c74b94b077895bd78cf1e64a3), value: 0, gas_price: Some(16185590269), gas: 207128, input: Bytes([169, 5, 156, 187, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 33, 160, 138, 28, 191, 39, 255, 75, 202, 197, 95, 205, 201, 124, 116, 141, 91, 81, 39, 96, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 136, 108, 144, 48, 150, 116, 138, 0, 0]), v: Some(0), r: Some(55005093721274821805649449657760135373297623073244198928033742057513000728837), s: Some(19584657317945855950856553141997014012848398946606485974435162940290921668925), raw: None, transaction_type: Some(2), access_list: Some([]), max_fee_per_gas: Some(102000000000), max_priority_fee_per_gas: Some(2000000000) }
```

### 3.6 Decode input data

Now we can intercept a pending transaction, but how to decode the input data? The input data is a byte array. We can use the 'ethabi' crate to decode the input data, but in this simple case I will use the 'hex' crate to decode the input data.

```rust
use dotenv::dotenv;
use hex;
use web3::futures::TryStreamExt;
use web3::types::TransactionId;

#[tokio::main]
async fn main() -> web3::Result {
    dotenv().ok();
    let alchemy_api_key = dotenv::var("ALCHEMY_API_KEY").expect("ALCHEMY_API_KEY must be set");
    let web3 = web3::Web3::new(web3::transports::WebSocket::new(&alchemy_api_key).await?);

    let mut pending_transactions = web3
        .eth_subscribe()
        .subscribe_new_pending_transactions()
        .await?;

    //filter transaction based on address
    let address = "0x31c8eacbffdd875c74b94b077895bd78cf1e64a3";

    while let Some(pending_transaction_hash) = pending_transactions.try_next().await? {
        let transaction = web3
            .eth()
            .transaction(TransactionId::from(pending_transaction_hash))
            .await?;
        if let Some(transaction) = transaction {
            //filter transaction based on address and method hash
            if transaction.to == Some(address.parse().unwrap()) {
                //decode input data bytes to hex
                let tx_clone = transaction.clone();

                let input_data = transaction.input.0;
                let input_data_hex = hex::encode(input_data);
                println!("tx input hex: {:?}", input_data_hex);

                //decode using abi

                if input_data_hex.starts_with("a9059cbb") {
                    let raw_amount = input_data_hex[74..].to_string();
                    println!("Raw Amount cutted hex: {:?}", raw_amount);
                    //decode raw amount to u256
                    let raw_amount = hex::decode(raw_amount).unwrap();
                    //convert to u256
                    let raw_amount = web3::types::U256::from_big_endian(&raw_amount);

                    println!("//---------------------------------------//");
                    println!("Raw Amount: {:?}", raw_amount);
                    println!("Transaction: {:?}", tx_clone);
                    println!("//---------------------------------------//");
                    println!("Transfer in pending towards RAD token contract");
                }
            }
        }
    }

    Ok(())
}
```

Let's break it down, 'let input_data = transaction.input.0;' is the input data of the transaction, a byte array. We can convert it to a hex string with 'let input_data_hex = hex::encode(input_data);'.

Now we can filter the input data based on the method hash. In this case we are looking for the 'transfer' method hash 'a9059cbb'. If the input data starts with 'a9059cbb' we can cut the first 74 characters of the input data hex string. The first 74 characters are the method hash and the address of the receiver.

The rest of the input data is the amount. We can convert the rest of the input data to a u256 with 'let raw_amount = web3::types::U256::from_big_endian(&raw_amount);'. Now we have the amount of the transfer in pending. We can also print the transaction to see the other data of the transaction. You should see a log like this:

```bash
tx input hex: "a9059cbb000000000000000000000000b51da94ae51c339cec40d78260199f73cebbeba8000000000000000000000000000000000000000000000015954d6c905a060000"
Raw Amount cutted hex: "0000000000000000000000000000000000000000000015954d6c905a060000"
//---------------------------------------//
Raw Amount: 398140000000000000000
Transaction: Transaction { hash: 0xee7013a832ff032fd43fc296e6ea6ffead097db2122981eb971f7f4fd5c5d0dc, nonce: 4931764, block_hash: Some(0x7fc256c3640e02070d726a8f504948609d5100cbc50910515cffa708a4b01b1c), block_number: Some(16228188), transaction_index: Some(35), from: Some(0xdfd5293d8e347dfe59e90efd55b2956a1343963d), to: Some(0x31c8eacbffdd875c74b94b077895bd78cf1e64a3), value: 0, gas_price: Some(20264315737), gas: 207128, input: Bytes([169, 5, 156, 187, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 181, 29, 169, 74, 229, 28, 51, 156, 236, 64, 215, 130, 96, 25, 159, 115, 206, 187, 235, 168, 0, 0, 0, 0, 0, 0,
0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 21, 149, 77, 108, 144, 90, 6, 0, 0]), v: Some(1), r: Some(94476206653139298663972815699772140255840765385329375194390453406687512516529), s: Some(34590948637004618330444175192055263866731258644152094038657143970283606957875), raw: None, transaction_type: Some(2), access_list: Some([]), max_fee_per_gas: Some(102000000000), max_priority_fee_per_gas: Some(2000000000) }
//---------------------------------------//
Transfer in pending towards RAD token contract
```

You can confront the values here with the values [on the Etherscan transaction page](https://etherscan.io/tx/0xee7013a832ff032fd43fc296e6ea6ffead097db2122981eb971f7f4fd5c5d0dc).

## 5. Conclusion

You can see that we know the gas details and the token amount of the transaction, guess what you can do with that information. You can use it to calculate the gas price and the gas cost of the transaction. You can also use it to calculate the amount of tokens that will be transferred when the transaction is confirmed and front-run the transaction.

Maybe in a future article we will see how to front-run a transaction. If you have any questions or suggestions, please let me know.

## 7. Do you need to develop a MEV bot?

You can contact me [Lorenzo Zaccagnini](https://www.linkedin.com/in/lorenzo-zaccagnini/) or [Elisa Romondia](https://www.linkedin.com/in/elisa-romondia/) on LinkedIn. If you want to support me you can donate eth or matic to 0xbf8d0d4be61De94EFCCEffbe5D414f911F11cBF8
