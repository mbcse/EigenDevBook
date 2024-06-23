---
description: Tutorial to  build a simple Hello World AVS using Rust and Solidity
---

# Building an Hello World AVS in Rust

## Introduction&#x20;

Hello, world avs is the first step to understanding how to build an avs of your own. We will deep dive into all individual sections of this repo and understand how the whole AVS works!\


### Setting Up the Development Environment

**To Get Started Clone this repo of hello world AVS starter template**

​[https://github.com/Layr-Labs/hello-world-avs.git](https://github.com/Layr-Labs/hello-world-avs.git)​

**Dependencies**

1. 1.​[npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)​
2. 2.​[Foundry](https://getfoundry.sh/)​
3. 3.​[Docker](https://www.docker.com/get-started/) 3.1 Make sure Docker is running

### There are two main parts:

* Contracts - Your AVS contract, The core logic of it&#x20;
* operator -  A mock operator who will sign a message

## Hello World on Eigen Layer: Understanding the `start_operator.rs` Code

In this article, we'll dissect the `start_operator.rs` code, which is designed to interact with the Ethereum blockchain using the Eigen Layer. This script showcases how to sign and respond to tasks, monitor for new tasks, and register an operator. Let's break down each section to understand its functionality.

Imports and Dependencies

```rust
rustCopy code#![allow(missing_docs)]
use alloy_network::{Ethereum, EthereumSigner};
use alloy_provider::RootProvider;
use alloy_provider::{Provider, ProviderBuilder};
use alloy_rpc_types::{BlockNumberOrTag, Filter};
use alloy_sol_types::{sol, SolEvent};
use alloy_transport_http::Client;
use chrono::Utc;
use dotenv::dotenv;
use once_cell::sync::Lazy;
use rand::RngCore;
use reqwest::Url;
use HelloWorldServiceManager::Task;
use alloy_primitives::{eip191_hash_message, Address, FixedBytes, U256};
use alloy_signer::Signer;
use alloy_signer_wallet::LocalWallet;
use eigen_client_elcontracts::{
    reader::ELChainReader,
    writer::{ELChainWriter, Operator},
};
use eyre::Result;
use alloy_provider::fillers::{
    ChainIdFiller, FillProvider, GasFiller, JoinFill, NonceFiller, SignerFiller,
};
use std::{env, str::FromStr};
use ECDSAStakeRegistry::SignatureWithSaltAndExpiry;

sol!(
    #[allow(missing_docs)]
    #[sol(rpc)]
    HelloWorldServiceManager,
    "json_abi/HelloWorldServiceManager.json"
);
use eigen_utils::binding::ECDSAStakeRegistry;
```

Here, we import various dependencies required for the script. These include libraries for Ethereum interactions (`alloy_network`, `alloy_provider`), signing (`alloy_signer`), JSON-RPC handling (`alloy_rpc_types`), and more. The `sol!` macro is used to generate Rust bindings from the Solidity ABI of the `HelloWorldServiceManager` contract.\
\
Static Variables

```rust
rustCopy codestatic KEY: Lazy<String> =
    Lazy::new(|| env::var("PRIVATE_KEY").expect("failed to retrieve private key"));
pub static RPC_URL: Lazy<String> =
    Lazy::new(|| env::var("RPC_URL").expect("failed to get rpc url from env"));
pub static HELLO_WORLD_CONTRACT_ADDRESS: Lazy<String> = Lazy::new(|| {
    env::var("CONTRACT_ADDRESS").expect("failed to get hello world contract address from env")
});
static DELEGATION_MANAGER_CONTRACT_ADDRESS: Lazy<String> = Lazy::new(|| {
    env::var("DELEGATION_MANAGER_ADDRESS")
        .expect("failed to get delegation manager contract address from env")
});
static STAKE_REGISTRY_CONTRACT_ADDRESS: Lazy<String> = Lazy::new(|| {
    env::var("STAKE_REGISTRY_ADDRESS")
        .expect("failed to get stake registry contract address from env")
});
static AVS_DIRECTORY_CONTRACT_ADDRESS: Lazy<String> = Lazy::new(|| {
    env::var("AVS_DIRECTORY_ADDRESS")
        .expect("failed to get delegation manager contract address from env")
});
```

These static variables lazily load environment variables that store crucial information such as the private key, RPC URL, and various contract addresses. The `Lazy` type ensures these values are only computed once and reused thereafter.

#### Signing and Responding to Tasks

```rust
rustCopy codeasync fn sign_and_response_to_task(
    task_index: u32,
    task_created_block: u32,
    name: String,
) -> Result<()> {
    let provider = get_provider_with_wallet(KEY.clone());
    let message = format!("Hello, {}", name);
    let msg_hash = eip191_hash_message(message);
    let wallet = LocalWallet::from_str(&KEY.clone()).expect("failed to generate wallet ");
    let signature = wallet.sign_hash(&msg_hash).await?;
    println!("Signing and responding to task : {:?}", task_index);
    let hello_world_contract_address = Address::from_str(&HELLO_WORLD_CONTRACT_ADDRESS)
        .expect("wrong hello world contract address");
    let hello_world_contract =
        HelloWorldServiceManager::new(hello_world_contract_address, &provider);
    hello_world_contract
        .respondToTask(
            Task {
                name,
                taskCreatedBlock: task_created_block,
            },
            task_index,
            signature.as_bytes().into(),
        )
        .send()
        .await?
        .get_receipt()
        .await?;
    println!("Responded to task");
    Ok(())
}
```

This function handles signing and responding to a task on the Ethereum blockchain. It:

1. Creates a provider with the wallet.
2. Constructs a message and hashes it.
3. Signs the hash using the wallet.
4. Interacts with the `HelloWorldServiceManager` contract to respond to the task.

#### Monitoring for New Tasks

```rust
rustCopy codeasync fn monitor_new_tasks() -> Result<()> {
    let provider = get_provider_with_wallet(KEY.clone());
    let hello_world_contract_address = Address::from_str(&HELLO_WORLD_CONTRACT_ADDRESS)
        .expect("wrong hello world contract address");
    println!("hello world contrat address:{:?}", hello_world_contract_address);
    let hello_world_contract =
        HelloWorldServiceManager::new(hello_world_contract_address, &provider);
    println!("heelo contract :{:?}", hello_world_contract);
    let word: &str = "EigenWorld";
    let _new_task_tx = hello_world_contract
        .createNewTask(word.to_owned())
        .send()
        .await?
        .get_receipt()
        .await?;
    let mut latest_processed_block = provider.get_block_number().await?;
    loop {
        println!("Monitoring for new tasks...");
        let filter = Filter::new()
            .address(hello_world_contract_address)
            .from_block(BlockNumberOrTag::Number(latest_processed_block));
        let logs = provider.get_logs(&filter).await?;
        for log in logs {
            match log.topic0() {
                Some(&HelloWorldServiceManager::NewTaskCreated::SIGNATURE_HASH) => {
                    let HelloWorldServiceManager::NewTaskCreated { taskIndex, task } = log
                        .log_decode()
                        .expect("Failed to decode log new task created")
                        .inner
                        .data;
                    println!("New task detected :Hello{:?} ", task.name);
                    let _ = sign_and_response_to_task(taskIndex, task.taskCreatedBlock, task.name)
                        .await;
                }
                _ => {}
            }
        }
        tokio::time::sleep(tokio::time::Duration::from_secs(60)).await;
        let current_block = provider.get_block_number().await?;
        latest_processed_block = current_block;
    }
}
```

This function monitors the blockchain for new tasks:

1. Sets up the provider and contract instances.
2. Creates a new task on the contract.
3. Continuously checks for new logs from the contract.
4. When a new task is detected, it calls `sign_and_response_to_task`.

#### Registering an Operator

```rust
rustCopy codeasync fn register_operator() -> Result<()> {
    let wallet = LocalWallet::from_str(&KEY).expect("failed to generate wallet ");
    let provider = get_provider_with_wallet(KEY.clone());
    let hello_world_contract_address = Address::from_str(&HELLO_WORLD_CONTRACT_ADDRESS)
        .expect("wrong hello world contract address");
    let delegation_manager_contract_address =
        Address::from_str(&DELEGATION_MANAGER_CONTRACT_ADDRESS)
            .expect("wrong delegation manager contract address");
    let stake_registry_contract_address = Address::from_str(&STAKE_REGISTRY_CONTRACT_ADDRESS)
        .expect("wrong stake registry contract address");
    let avs_directory_contract_address = Address::from_str(&AVS_DIRECTORY_CONTRACT_ADDRESS)
        .expect("wrong delegation manager contract address");
    let default_slasher = Address::ZERO;
    let default_strategy = Address::ZERO;
    let elcontracts_reader_instance = ELChainReader::new(
        default_slasher,
        delegation_manager_contract_address,
        avs_directory_contract_address,
        RPC_URL.clone(),
    );
    let elcontracts_writer_instance = ELChainWriter::new(
        delegation_manager_contract_address,
        default_strategy,
        elcontracts_reader_instance.clone(),
        RPC_URL.clone(),
        KEY.clone(),
    );
    let operator = Operator::new(
        wallet.address(),
        wallet.address(),
        Address::ZERO,
        0u32,
        None,
    );
    let _tx_hash = elcontracts_writer_instance
        .register_as_operator(operator)
        .await;
    println!("Operator registered on EL successfully");
    let mut salt = [0u8; 32];
    rand::rngs::OsRng.fill_bytes(&mut salt);
    let salt = FixedBytes::from_slice(&salt);
    let now = Utc::now().timestamp();
    let expiry: U256 = U256::from(now + 3600);
    let digest_hash = elcontracts_reader_instance
        .calculate_operator_avs_registration_digest_hash(
            wallet.address(),
            hello_world_contract_address,
            salt,
            expiry,
        )
        .await
        .expect("not able to calculate operator ");
    let signature = wallet.sign_hash(&digest_hash).await?;
    let operator_signature = SignatureWithSaltAndExpiry {
        signature: signature.as_bytes().into(),
        salt,
        expiry: expiry,
    };
    let contract_ecdsa_stake_registry =
        ECDSAStakeRegistry::new(stake_registry_contract_address, provider);
    println!("initialize new ecdsa ");
    let registeroperator_details = contract_ecdsa_stake_registry
        .registerOperatorWithSignature(wallet.clone().address(), operator_signature);
    let _tx = registeroperator_details
        .send()
        .await?
        .get_receipt()
        .await?;
    println!(
        "Operator registered on AVS successfully :{:?}",
        wallet.address()
    );
    Ok(())
}
```

This function registers an operator:

1. Sets up the wallet and provider.
2. Initializes contract addresses.
3. Creates instances of `ELChainReader` and `ELChainWriter`.
4. Registers the operator on the Eigen Layer.
5. Generates a unique salt and calculates a digest hash.
6. Signs the hash and registers the operator with the signature on AVS.

#### Main Function

```rust
rustCopy code#[tokio::main]
pub async fn main() {
    dotenv().ok();
    if let Err(e) = register_operator().await {
        eprintln!("Failed to register operator: {:?}", e);
        return;
    }
    tokio::spawn(async {
        if let Err(e) = monitor_new_tasks().await {
            eprintln!("Failed to monitor new tasks: {:?}", e);
        }
    });
    loop {
        tokio::time::sleep(tokio::time::Duration::from_secs(60)).await;
    }
}
```

The `main` function:

1. Loads environment variables.
2. Registers the operator.
3. Starts monitoring for new tasks in a separate asynchronous task.
4. Keeps the process running indefinitely.

#### Helper Function

```rust
rustCopy codepub fn get_provider_with_wallet(
    key: String,
) -> FillProvider<
    JoinFill<
        JoinFill<
            JoinFill<JoinFill<alloy_provider::Identity, GasFiller>, NonceFiller>,
            ChainIdFiller,
        >,
        SignerFiller<EthereumSigner>,
    >,
    RootProvider<alloy_transport_http::Http<Client>>,
    alloy_transport_http::Http<Client>,
    Ethereum,
> {
    let wallet = LocalWallet::from_str(&key.to_string()).expect("failed to generate wallet ");
    let url = Url::parse(&RPC_URL.clone()).expect("Wrong rpc url");
    let provider = ProviderBuilder::new()
        .with_recommended_fillers()
        .signer(EthereumSigner::from(wallet.clone()))
        .on_http(url);
    return provider;
}
```

This helper function sets up a provider with the necessary configurations, including gas fillers, nonce fillers, and signing capabilities using the provided wallet.\




## Hello World Service Manager Solidity Contract: Detailed Breakdown

The `HelloWorldServiceManager` Solidity contract is designed as a primary entry point for procuring services from HelloWorld. It manages tasks, their creation, and responses from operators. Let's break down the contract step-by-step to understand its functionality.

### The Interface

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

interface IHelloWorldServiceManager {
    // EVENTS
    event NewTaskCreated(uint32 indexed taskIndex, Task task);

    event TaskResponded(uint32 indexed taskIndex, Task task, address operator);

    // STRUCTS
    struct Task {
        string name;
        uint32 taskCreatedBlock;
    }

    // FUNCTIONS
    // NOTE: this function creates new task.
    function createNewTask(
        string memory name
    ) external;

    // NOTE: this function is called by operators to respond to a task.
    function respondToTask(
        Task calldata task,
        uint32 referenceTaskIndex,
        bytes calldata signature
    ) external;
}
```

#### Imports and Dependencies

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

import "@eigenlayer/contracts/libraries/BytesLib.sol";
import "@eigenlayer/contracts/core/DelegationManager.sol";
import "@eigenlayer-middleware/src/unaudited/ECDSAServiceManagerBase.sol";
import "@eigenlayer-middleware/src/unaudited/ECDSAStakeRegistry.sol";
import "@openzeppelin-upgrades/contracts/utils/cryptography/ECDSAUpgradeable.sol";
import "@eigenlayer/contracts/permissions/Pausable.sol";
import {IRegistryCoordinator} from "@eigenlayer-middleware/src/interfaces/IRegistryCoordinator.sol";
import "./IHelloWorldServiceManager.sol";
```

The contract imports various libraries and other contracts necessary for its operations:

* `BytesLib`: Utility functions for bytes manipulation.
* `DelegationManager`: Manages delegation-related functionality.
* `ECDSAServiceManagerBase`: Base contract for ECDSA service management.
* `ECDSAStakeRegistry`: Manages operator registrations and their stakes.
* `ECDSAUpgradeable`: Utility functions for ECDSA signatures.
* `Pausable`: Allows the contract to be paused and unpaused by authorized accounts.
* `IRegistryCoordinator`: Interface for registry coordination.
* `IHelloWorldServiceManager`: Interface for the HelloWorld service manager.

#### Contract Definition and Inheritance

```solidity
/**
 * @title Primary entrypoint for procuring services from HelloWorld.
 * @authos Eigen Labs, Inc.
 */
contract HelloWorldServiceManager is 
    ECDSAServiceManagerBase,
    IHelloWorldServiceManager,
    Pausable
{
    using BytesLib for bytes;
    using ECDSAUpgradeable for bytes32;
```

The contract inherits from `ECDSAServiceManagerBase`, `IHelloWorldServiceManager`, and `Pausable`, combining their functionalities. It also uses `BytesLib` and `ECDSAUpgradeable` libraries.

#### Storage Variables

```solidity
/* STORAGE */
    // The latest task index
    uint32 public latestTaskNum;

    // mapping of task indices to all tasks hashes
    // when a task is created, task hash is stored here,
    // and responses need to pass the actual task,
    // which is hashed onchain and checked against this mapping
    mapping(uint32 => bytes32) public allTaskHashes;

    // mapping of task indices to hash of abi.encode(taskResponse, taskResponseMetadata)
    mapping(address => mapping(uint32 => bytes)) public allTaskResponses;
```

* `latestTaskNum`: Tracks the latest task index.
* `allTaskHashes`: Maps task indices to their hashes. When a task is created, its hash is stored here, and responses are verified against this hash.
* `allTaskResponses`: Maps operators to their task responses. It stores the hash of `abi.encode(taskResponse, taskResponseMetadata)`.

#### Modifiers

```solidity
/* MODIFIERS */
    modifier onlyOperator() {
        require(
            ECDSAStakeRegistry(stakeRegistry).operatorRegistered(msg.sender) == true, 
            "Operator must be the caller"
        );
        _;
    }
```

The `onlyOperator` modifier ensures that the caller is a registered operator in the `ECDSAStakeRegistry`.

#### Constructor

```solidity
constructor(
        address _avsDirectory,
        address _stakeRegistry,
        address _delegationManager
    )
        ECDSAServiceManagerBase(
            _avsDirectory,
            _stakeRegistry,
            address(0), // hello-world doesn't need to deal with payments
            _delegationManager
        )
    {}
```

The constructor initializes the base contract `ECDSAServiceManagerBase` with the provided addresses for the AVS directory, stake registry, and delegation manager. The payment address is set to `address(0)` as HelloWorld doesn't handle payments.

#### Functions

**`createNewTask`**

```solidity
/* FUNCTIONS */
    // NOTE: this function creates new task, assigns it a taskId
    function createNewTask(
        string memory name
    ) external {
        // create a new task struct
        Task memory newTask;
        newTask.name = name;
        newTask.taskCreatedBlock = uint32(block.number);

        // store hash of task onchain, emit event, and increase taskNum
        allTaskHashes[latestTaskNum] = keccak256(abi.encode(newTask));
        emit NewTaskCreated(latestTaskNum, newTask);
        latestTaskNum = latestTaskNum + 1;
    }
```

This function allows anyone to create a new task. It:

1. Creates a new `Task` struct.
2. Stores the hash of the task in `allTaskHashes`.
3. Emits a `NewTaskCreated` event.
4. Increments `latestTaskNum`.

**`respondToTask`**

```solidity
// NOTE: this function responds to existing tasks.
    function respondToTask(
        Task calldata task,
        uint32 referenceTaskIndex,
        bytes calldata signature
    ) external onlyOperator {
        require(
            operatorHasMinimumWeight(msg.sender),
            "Operator does not have match the weight requirements"
        );
        // check that the task is valid, hasn't been responsed yet, and is being responded in time
        require(
            keccak256(abi.encode(task)) ==
                allTaskHashes[referenceTaskIndex],
            "supplied task does not match the one recorded in the contract"
        );
        // some logical checks
        require(
            allTaskResponses[msg.sender][referenceTaskIndex].length == 0,
            "Operator has already responded to the task"
        );

        // The message that was signed
        bytes32 messageHash = keccak256(abi.encodePacked("Hello, ", task.name));
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();

        // Recover the signer address from the signature
        address signer = ethSignedMessageHash.recover(signature);

        require(signer == msg.sender, "Message signer is not operator");

        // updating the storage with task responses
        allTaskResponses[msg.sender][referenceTaskIndex] = signature;

        // emitting event
        emit TaskResponded(referenceTaskIndex, task, msg.sender);
    }
```

This function allows registered operators to respond to tasks. It:

1. Ensures the caller is a registered operator with sufficient weight.
2. Validates the task against the stored hash.
3. Checks if the task hasn't been responded to yet.
4. Verifies the signature.
5. Stores the task response.
6. Emits a `TaskResponded` event.

#### Helper Function

```solidity
// HELPER
    function operatorHasMinimumWeight(address operator) public view returns (bool) {
        return ECDSAStakeRegistry(stakeRegistry).getOperatorWeight(operator) >= ECDSAStakeRegistry(stakeRegistry).minimumWeight();
    }
```

This helper function checks if an operator has the minimum required weight by querying the `ECDSAStakeRegistry`.

The `HelloWorldServiceManager` contract is an entry point for task management within the HelloWorld ecosystem. It enables:

* Creation of new tasks by any user.
* Secure and authenticated responses from registered operators.
* Validation and storage of task responses.



## To run this whole AVS

**Typescript**

1. Run `yarn install`
2. Run `make start-chain-with-contracts-deployed`
   * This will build the contracts, start an Anvil chain, deploy the contracts to it, and leaves the chain running in the current terminal
3. Open new terminal tab and run `make start-operator`
   * This will compile the AVS software and start monitering new tasks
4. Open new terminal tab and run `make spam-tasks` (Optional)
   * This will spam the AVS with random names every 15 seconds

**Rust lang**

**Anvil**

1. Run `make start-chain-with-contracts-deployed`
   * This will build the contracts, start an Anvil chain, deploy the contracts to it, and leaves the chain running in the current terminal
2. Run `make start-rust-operator`
3. Run `make spam-rust-tasks`

Tests are supported in anvil only . Make sure to run the 1st command before running the tests:

```
cargo test --workspace
```

**Holesky Testnet**

| Contract Name               | Holesky Address                                                                                                               |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Hello World Service Manager | [0x3361953F4a9628672dCBcDb29e91735fb1985390](https://holesky.etherscan.io/address/0x3361953F4a9628672dCBcDb29e91735fb1985390) |
| Delegation Manager          | [0xA44151489861Fe9e3055d95adC98FbD462B948e7](https://holesky.etherscan.io/address/0xA44151489861Fe9e3055d95adC98FbD462B948e7) |
| Avs Directory               | [0x055733000064333CaDDbC92763c58BF0192fFeBf](https://holesky.etherscan.io/address/0x055733000064333CaDDbC92763c58BF0192fFeBf) |

You don't need to run any script for holesky testnet.

1. Use the HOLESKY\_ namespace env parameters in the code , instead of normal parameters.
2. Run `make start-rust-operator`
3. Run `make spam-rust-tasks`
