# Kafka Roller KRaft Support

This proposal describes the actions that the KafkaRoller should take when operating 
against a Strimzi cluster in KRaft mode.
The proposal omits details about exactly how this should be implemented because 
the logic of the KafkaRoller is currently being rewritten.
This proposal only describes the expected behaviour so that it can be implemented 
in either the current KafkaRoller or a future iteration of the KafkaRoller.

This proposal assumes that liveness/readiness of nodes is as described in [proposal #46](https://github.com/strimzi/proposals/blob/main/046-kraft-liveness-readiness.md) 
and [PR #8892](https://github.com/strimzi/strimzi-kafka-operator/pull/8892).


## Current situation

When operating on a ZooKeeper based cluster the KafkaRoller has the behaviour 
described below.

### Order to roll nodes
Roll in order:
 - Unready brokers
 - Ready brokers
 - Current controller

### Triggers
The following are some of the triggers that roll a broker:
- Pod is annotated for manual rolling update
- Pod is unready because it is in `CrashLoopBackOff`, `ImagePullBackOff`, `ContainerCreating` or `Pending` and `Unschedulable`, 
- Broker's configuration has changed and cannot be updated dynamically

### Rollability checks
 - Attempts to connect an admin client to the broker and if it can't connect rolls the broker
 - Does not force roll a broker performing log recovery
 - Does not roll a broker if doing so would take the in-sync replicas count below `min.insync.replicas`

### Configuration changes
 - Retrieves the current Kafka configurations of the broker via admin client and compares it with the desired configurations specified in the Kafka CR. 
 - Performs dynamic configuration updates if possible, otherwise rolls broker on configuration change

In KRaft mode the KafkaRoller currently skips controller only nodes, but performs the above steps on any combined or broker only nodes.
This is causing a problem in combined mode because if the quorum has not formed due to some of the nodes not being ready 
the KafkaRoller will wait still try to contact the broker via the admin client.
This call fails because the quorum is not formed, so in some cases this results in the cluster being stuck with some nodes 
in a pending state.

## Motivation

When running in the KRaft mode the controller pods need to be rolled if configuration changes occur, also at the moment the 
existing logic is blocking the full implementation of liveness and readiness checks as described in [proposal #46](https://github.com/strimzi/proposals/blob/main/046-kraft-liveness-readiness.md).

## Proposal

The KafkaRoller behaviour should be unchanged when operating against a ZooKeeper based cluster.

The behaviour when operating against a KRaft cluster is described below.

### Order to roll nodes
Roll in order:
- Unready controller/combined nodes
- Ready controller/combined nodes
- Active controller
- Unready broker-only nodes
- Ready broker-only nodes


### Triggers
The following are some of the triggers that would roll a KRaft controller or combined node:
- Pod is annotated for manual rolling update
- Pod is unready because it is in `CrashLoopBackOff`, `ImagePullBackOff`, `ContainerCreating` or `Pending` and `Unschedulable`, 
- Controller's configuration has changed and cannot be updated dynamically.

The triggers for broker remain the same as Zookeeper mode.

**NOTE** There is a related proposal describing how to diff controller configuration currently being discussed: [PR #82](https://github.com/strimzi/proposals/pull/82)

### Rollability checks

The checks made by the KafkaRoller in different modes is described below.
The checks include a new check for controllers to verify that rolling the node does not affect the quorum health.
The proposed check is:
- Fetch `lastCaughtUpTimestamp` for every controller node. `lastCaughtUpTimestamp` is the last millisecond timestamp at which a replica controller was known to be caught up with the quorum leader
- Identify the `lastCaughtUpTimestamp` of the node that is the quorum leader .
- Retrieve the current value of Kafka configuration `controller.quorum.fetch.timeout.ms`
- Mark a node as caught up if `leaderLastCaughtUpTimestamp - replicaLastCaughtUpTimestamp < controllerQuorumFetchTimeoutMs`
- Count each controller node that is caught up, including the leader (`numOfCaughtUpControllers`)
- Can roll if: `numOfCaughtUpControllers > ceil(double) totalNumOfControllers / 2`

In order to perform quorum check for controller, KafkaRoller creates admin client connection to the brokers. This is required until [KIP-919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum) is implemented. If KafkaRoller cannot connect to any of the brokers, reconcilation for the controllers will be marked as failed due to not being able to determine the quorum health. KafkaRoller would then try to reconcile the brokers, which may help restoring admin client connections to them. The controllers quorum check will be retried in the next round of reconcilation.


#### Separate brokers and controllers
For controller-only:
- Does not force roll a controller performing log recovery
- Does not roll a ready controller if doing so would take the number of caught-up voters (inc leader) to less than half of the quorum
- When rolling an unready controller, does not check quorum state before rolling

For broker-only:
- Attempts to connect an admin client to the broker and if it can't connect, force rolls the broker
- Does not force roll a broker performing log recovery
- Does not roll a broker if doing so would take the in-sync replicas count below `min.insync.replicas`
- **NEW** If a broker is not ready and no admin client connection can be made, check the quorum is formed before force rolling the broker

#### Combined mode
- Does not force roll a combined node performing log recovery
- Does not roll a ready combined node if doing so would take the number of caught-up voters (inc leader) to less than half of the quorum
- Does not roll a ready combined node if doing so would take the in-sync replicas count below `min.insync.replicas`
- When rolling an unready combined node, does not check quorum state, or in-sync replicas count, or try to connect an admin client to the broker before rolling

### Configuration changes
For all kinds of nodes:
- Retrieves the current Kafka configurations of the node via Admin client and compares it with the desired configurations specified in the Kafka CR.
- Performs dynamic configuration updates if possible, otherwise rolls node on configuration change

**NOTE:** At the time of writing this proposal, it is not possible to connect to controller nodes via Admin client to retrieve the current configuration or apply dynamic configuration updates. The related proposal in 
[PR #82](https://github.com/strimzi/proposals/pull/82) describes a workaround for retrieving the current configuration of controller nodes so we can detect if there is any configuration change. However dynamic configuration update for them would not possible until [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum) is implemented.

## Affected/not affected projects

The only affected project is the Strimzi cluster operator.

## Compatibility

This proposal does not affect the ZooKeeper broker KafkaRoller behaviour.
This proposal does change the way that KRaft nodes are rolled, however since KRaft mode is not supported for production use 
and the existing logic is incomplete this is acceptable.

## Rejected alternatives

### Node rolling order
We considered rolling all unready nodes, then all ready nodes, regardless of whether they were controllers or brokers.
However, the problem with this approach is that for broker nodes to become ready the controller quorum must be formed.
