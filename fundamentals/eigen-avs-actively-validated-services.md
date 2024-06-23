---
description: An overview of AVS in Eigen
---

# Eigen AVS - Actively Validated Services

## Understanding AVS in EigenLayer

An Actively Validated Service (AVS) is a system that requires its own unique distributed validation methods for verification. Examples of AVSs include sidechains, data availability layers, virtual machines, keeper networks, oracle networks, bridges, threshold cryptography schemes, and trusted execution environments. These systems rely on off-chain layers for executing specific operations, and the entities responsible for this off-chain work are known as "operators."\
\
Huh!, That was a hard explanation let me simplify it. AVS basically is the logic for your system. You write the logic of how much should be staked amount, which stake token needs to be with the operator, and any other conditions concerning your system!

\
In the EigenLayer ecosystem, each AVS consists of some contracts that manage the service's functionality. These are the set of contracts that you have to interact with to build an AVS.\
\
Key components include:-

* The StrategyManager, where stakers deposit their assets
* The DelegationManager, allowing stakers to choose which operators to delegate to
* The AVSDirectory, listing all registered AVSs
* And the ServiceManager, the entry point for each AVS, which you must implement the interface expected by the EigenLayer protocol.
