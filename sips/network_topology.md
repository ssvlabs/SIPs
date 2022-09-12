| Author      | Title                   | Category | Status |
|-------------|-------------------------|----------|--------|
| Amir Yahalom | Network Topology | P2P     | draft  |

## Overview

...

## Topologies

### 1. Topic per Committee

There is a pubsub topic for each validator in the network.

### 2. Subnets

There is a pubsub topic for a set of validators in the network. \
The number of topics is fixed (128, 256, etc.), and validators are mapped deterministically to subnets using: `hash(publicKey) % <subnets-count>`

### 3. Dynamic Subnets

There is a pubsub topic for a set of validators in the network,
but the number of topics is changing as more validators join the network.

New validators are assigned with a unique incremental index. \
Topics are created on the fly for X number of sequential IDs (topic 0 for IDs 0-9, topic 1 for IDs 10-19, etc).

### 4. Clusters / Groups

There is a pubsub topic for a set of validators in the network,
but the number of topics is changing as there are more permutations of unique operators assigned to validators.

Topics are created on the fly for each unique (ordered) operator group,
i.e. all validators using operators 1,2,3,4 will be part of the same topic.
