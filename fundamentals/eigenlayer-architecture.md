# EigenLayer Architecture

### EigenLayer Architecture: An In-Depth Overview

EigenLayer is a cutting-edge technology that enhances the security and efficiency of blockchain networks by allowing restaking. This means the same staked cryptocurrency can secure multiple networks, creating a shared security model. Let's dive into its architecture and understand how it works.

EigenLayer's protocol architecture consists of four key components: restakers, operators, actively validated services (AVS), and AVS consumers.

* Restakers: individuals or entities who restake their staked ETH or LSTs to extend security to services in the EigenLayer ecosystem, known as Actively Validated Services (AVS).
* Operators: entities that run specialized node software and perform validation tasks for AVSs built on top of EigenLayer in return for pre-defined rewards. Operators register in EigenLayer, allow restakers to delegate to them, and then opt-in to provide validation services for various AVSs. It's important to note that operators are subject to each AVS's slashing conditions.
* Actively Validated Services (AVS): any system that requires unique distributed validation methods for verification. AVSs can take several forms, including data availability layers, shared sequencers, oracle networks, bridges, coprocessors, applied cryptography systems, and more.&#x20;
* AVS Consumers: end-users or applications that utilize the services provided by EigenLayer.

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

## Exploring the Evolution and Architecture of EigenLayer

EigenLayer is a groundbreaking platform designed to simplify decentralized infrastructure development on Ethereum. While many are familiar with the concepts of restaking and EigenLayer, few understand the complexity behind its core contracts. This section delves into the evolution of EigenLayer, tracing its journey from a simple idea to its current sophisticated architecture.

#### The Problem EigenLayer Addresses

Developers building decentralized infrastructures on Ethereum face significant challenges in establishing economic security. Although Ethereum provides security for smart contracts, infrastructures like bridges or sequencers need their own security mechanisms to enable a distributed network of nodes to reach consensus. Traditional consensus mechanisms such as proof of work and proof of authority have limitations, leading to the adoption of proof of stake (PoS) for many infrastructure projects. However, launching a new PoS network is fraught with difficulties:

1. **Finding Stakers:** There is no centralized location to find stakers.
2. **Investment Requirements:** Stakers must invest heavily in volatile network tokens.
3. **Opportunity Cost:** Stakers might miss out on other rewards, such as Ethereum's 5% returns.
4. **Security Model:** The current model is flawed, as the cost to compromise an application is only as high as its weakest dependency.

EigenLayer was created to address these challenges, offering a platform that connects stakers with infrastructure developers. It allows stakers to provide economic security with any token and enables them to restake and earn rewards while securing other infrastructures.

#### Building the Platform: Initial Steps

EigenLayer aims to connect stakers and infrastructure developers by enabling stakers to make credible commitments. In PoS systems, stakers commit to following protocol rules and risk losing their stake if they act maliciously. Developers create the infrastructure logic, while stakers secure it with their stakes.

The initial design involved a simple `TokenPool` contract where stakers could stake tokens. If a staker acted maliciously, their tokens could be slashed (removed). This basic structure allowed stakers to pledge stakes, be slashed if found malicious, and withdraw from the system.

```solidity
TokenPool {
    mapping(address => uint256) public balance;
    function stake(uint256 amount) public;
    function withdraw() public;
    function slash(address staker, ??? proof) public;
}
```

#### Reducing Opportunity Costs for Stakers

EigenLayer addresses the challenge of opportunity cost by allowing stakers to stake any token and earn multiple rewards simultaneously. By using liquid staking tokens (LST), stakers can retain their native ETH rewards while securing other infrastructures, thus reducing the opportunity cost of staking.

#### Consolidating Security Through Pooled Restaking

EigenLayer also aims to improve security by pooling security measures rather than fragmenting them. This approach mitigates the risk of isolated economic "honey pots" attracting attacks, thereby enhancing the overall security of all dependencies.

The design separates the slashing function from the `TokenPool` into a "slasher" contract. This modular approach allows stakers to participate in different Actively Validated Services (AVSs) without needing separate `TokenPool` contracts for each AVS.

```solidity
TokenPool {
    mapping(address => uint256) public balance;
    mapping(address => address[]) public slasher;
    function stake(uint256 amount) public;
    function withdraw() public;
    function enroll(address slasher) onlyOwner;
}

contract Slasher {
    mapping(address => bool) public isSlashed;
    function slash(address staker, ??? proof) public;
}
```

#### Delegating Operations: Stakers and Operators

Not all stakers want to handle the operational aspects of securing AVSs. EigenLayer distinguishes between stakers, who provide economic security, and operators, who run the necessary software. The `TokenPool` contract is adjusted to include a delegation mechanism, allowing stakers to delegate their stakes to operators.

```solidity
TokenPool {
    mapping(address => uint256) public stakerBalance;
    mapping(address => uint256) public operatorBalance;
    mapping(address => address) public delegation;
    mapping(address => address[]) public slasher;
    function stake(uint256 amount) public;
    function withdraw() public;
    function delegateTo(address operator) public;
    function enroll(address slasher) operatorOnly;
    function exit(address slasher) operatorOnly;
}

contract Slasher {
    mapping(address => bool) public isSlashed;
    function slash(address operator, ??? proof) public;
}
```

#### Modularizing the Operator Role

To keep the core contracts streamlined, operator-specific functions are moved to a `DelegationManager` contract. This modular approach ensures that the core contracts remain focused on their primary responsibilities.

```solidity
TokenPool {
    mapping(address => uint256) public stakerBalance;
    function stake(uint256 amount) public;
    function withdraw() public;
}

contract DelegationManager {
    mapping(address => uint256) public operatorBalance;
    mapping(address => address) public delegation;
    mapping(address => address[]) public slasher;
    function delegateTo(address operator) public;
    function enroll(address slasher) operatorOnly;
    function exit(address slasher) operatorOnly;
}
```

#### Supporting Multiple Tokens

EigenLayer extends its design to support multiple tokens by introducing a `TokenManager` and token-specific `TokenPool` contracts. The `TokenManager` acts as a central hub for staking and withdrawing various tokens.

```solidity
 TokenManager {
    mapping(address => address) tokenPoolRegistry;
    mapping(address => mapping(address => uint256)) stakerPoolShares;
    function stakeToPool(address pool, uint256 amount);
    function withdrawFromPool(address pool);
}

contract TokenPool {
    uint256 public totalShares;
    function stake(uint256 amount) TokenManagerOnly;
    function withdraw(uint256 shares) TokenManagerOnly;
}
```

#### Enhancing AVS Security

To prevent malicious operators from withdrawing their stakes before being slashed, EigenLayer introduces an unbonding period. This delay ensures that any slashable behavior can be penalized before the operator withdraws their stakes.

```solidity
 TokenManager {
    mapping(address => address) public tokenPoolRegistry;
    mapping(address => mapping(address => uint256)) public stakerPoolShares;
    mapping(address => uint256) public withdrawalCompleteTime;
    function stakeToPool(address pool, uint256 amount) public;
    function queueWithdrawal(address pool) public;
    function completeWithdrawal(address pool) public;
}
```

#### Making the Slashing Mechanism More Efficient

To reduce gas costs and simplify development, the slashing mechanism is further modularized. The `SlasherManager` maintains operator status, and individual slasher contracts handle the slashing logic.

```solidity
 SlasherManager {
    mapping(address => bool) public isSlashed;
    mapping(address => mapping(address => bool)) public canSlash;
    function enrollAVS(address slasher) operatorOnly;
    function exitAVS(address slasher) operatorOnly;
}

contract Slasher {
    function slash(address operator, ??? proof) public;
}
```

#### Conclusion: Core Components of EigenLayer

EigenLayer aims to revolutionize decentralized infrastructure by connecting stakers and developers, allowing flexible staking options, and pooling security. Through iterative design, it has developed three core components:

1. **TokenManager:** Manages staking and withdrawals.
2. **DelegationManager:** Registers operators and tracks operator shares.
3. **SlasherManager:** Provides AVS developers with an interface for slashing logic.

These components work together to ensure system safety and support various AVS designs, reducing off-chain complexity and gas fees. EigenLayer simplifies and secures infrastructure development on Ethereum. For more information, visit the [EigenLayer public repository](https://github.com/Layr-Labs/eigenlayer-contracts).
