---
description: Usecases of each Eigen Layer Smart Contract
---

# Eigen layer Smart Contracts Overview

EigenLayer is an innovative protocol that allows Ethereum stakers to enhance the security and efficiency of the network by restaking their assets. This system is built on a series of smart contracts, each designed to handle specific aspects of the restaking process. Here’s a closer look at the key contracts within EigenLayer and their roles in this ecosystem.

### EigenPodManager

#### Use Case

The EigenPodManager is essential for managing the EigenPods, which are unique containers holding stakers' funds. It ensures that these funds are securely restaked from the Ethereum beacon chain to other services within the EigenLayer. By deploying and maintaining EigenPods, the EigenPodManager enables users to verify their validators' withdrawal credentials, balances, and exits. This component plays a pivotal role in facilitating native ETH restaking, ensuring that the entire process is seamless and secure.

### StrategyManager

#### Use Case

The StrategyManager handles the diversification of staked assets by managing various staking strategies for liquid staking tokens (LSTs). It allows users to deposit their LSTs into specific strategies tailored to maximize returns while maintaining security. Each strategy is deployed as an instance that manages the tokens and awards shares to users based on their deposits. This contract makes it easier for stakers to participate in multiple strategies, enhancing their potential rewards and contributing to the overall security of the network.

### DelegationManager

#### Use Case

DelegationManager is at the heart of the delegation process within EigenLayer. It enables ETH holders to delegate their staking rights to third-party operators, who then perform staking on their behalf. This trustless delegation mechanism ensures that stakers retain control over their funds while benefiting from the expertise of specialized operators. The DelegationManager also tracks delegations and manages withdrawals, acting as a central hub for all delegation-related activities in the system.

### RewardsCoordinator

#### Use Case

The RewardsCoordinator is responsible for the fair and accurate distribution of rewards generated from staking activities. It collects reward submissions from AVSs and consolidates these into merkle roots posted on-chain. Stakers and operators can then claim their allocated rewards based on these submissions. This contract ensures that everyone involved in the staking process receives their due rewards, maintaining transparency and fairness within the ecosystem.

### AVSDirectory

#### Use Case

The AVSDirectory maintains a registry of actively validated services (AVSs) and their interactions with the EigenLayer core contracts. Once operators register within EigenLayer, they can opt to provide services to AVSs. The AVSDirectory records these registrations, facilitating smooth communication and service provision between operators and AVSs. This component is crucial for integrating new services into the EigenLayer ecosystem, ensuring they are properly validated and managed.

### Slasher

#### Use Case

The Slasher is a critical component for maintaining the security and integrity of the EigenLayer network. It enforces penalties on validators who engage in malicious behavior or fail to adhere to protocol rules. By identifying and slashing the stakes of such validators, the Slasher helps deter misconduct and protect the network. Although still under development, this contract will play a vital role in ensuring that all participants act in the best interest of the network.
