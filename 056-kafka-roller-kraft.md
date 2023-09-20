# Kafka Roller KRaft Support

This proposal describes the actions that the KafkaRoller should take when operating 
against a Strimzi cluster in KRaft mode.
The proposal describes the checks the KafkaRoller should take, how it should perform 
those checks and in what order but does not discuss exactly how the KafkaRoller works.
This proposal is expected to apply to both the current KafkaRoller and a future iteration of the KafkaRoller.

This proposal assumes that liveness/readiness of nodes is as described in [proposal #46](https://github.com/strimzi/proposals/blob/main/046-kraft-liveness-readiness.md) 
and [PR #8892](https://github.com/strimzi/strimzi-kafka-operator/pull/8892).


## Current situation

When operating on a ZooKeeper based cluster the KafkaRoller has the behaviour 
described below.

### Order to roll nodes
Roll in order:
 1. Unready brokers
 2. Ready brokers
 3. Current controller

### Triggers
The following are some of the triggers that roll a broker:
- Pod or its StrimziPodSet is annotated for manual rolling update
- Pod is unready
- Broker's configuration has changed

### Rollability checks
 1. Does not force roll a broker performing log recovery
 2. Attempts to connect an admin client to a broker and if it can't connect rolls the broker
 3. Force rolls a broker if the pod is in stuck state
 4. Rolls a broker if the configuration has changed and cannot be updated dynamically
 5. Does not roll a broker if doing so would take the in-sync replicas count below `min.insync.replicas`


#### Unready pod
If pod is unready, KafkaRoller awaits its readiness before doing anything. If it hasn't become ready still, it checks whether it is stuck and considers to force roll. A pod is considered stuck if it is in one of following states:
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `ContainerCreating`
- `Pending` and `Unschedulable`

If pod is not stuck but unready, the next rollability checks are performed.


#### Configuration changes
 - Retrieves the current Kafka configurations of the broker via admin client and compares it with the desired configurations specified in the Kafka CR. 
 - Performs dynamic configuration updates if possible

In KRaft mode the KafkaRoller currently skips controller only nodes, but performs the above steps on any combined or broker only nodes.
This is causing a problem in combined mode because if the quorum has not formed due to some of the nodes not being ready 
the KafkaRoller will still try to contact the broker via the admin client.
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
1. Unready controller/combined nodes
2. Ready controller/combined nodes
3. Active controller (applies to both pure controller and combined node)
4. Unready broker-only nodes
5. Ready broker-only nodes


### Triggers
The following are some of the triggers that would roll a KRaft controller or combined node:
- Pod or its StrimziPodSet is annotated for manual rolling update
- Pod is unready 
- Controller's configuration has changed

The triggers for broker remain the same as Zookeeper mode.

### Rollability checks
The checks made by the KafkaRoller in different modes is described below.

#### The new quorum check
The checks include a new check for controllers to verify that rolling the node does not affect the quorum health.
The proposed check is:
- Create admin client connection to the brokers and call `describeMetadataQuorum` API
- If failed to connect to the brokers, return `UnforceableProblem` which will result in delay and retry for the pod until the maximum attempts is reached.
- From the quorum info returned from the admin API, fetch `lastCaughtUpTimestamp` for every controller except the current controller we are considering to roll. `lastCaughtUpTimestamp` is the last millisecond timestamp at which a replica controller was known to be caught up with the quorum leader.
- Check the quorum leader id using the quorum info and identify the `lastCaughtUpTimestamp` of the quorum leader
- Retrieve value of the Kafka property `controller.quorum.fetch.timeout.ms` from the desired configurations specified in the Kafka CR. If this property does not exist in the desired configurations, then use the hard-coded default value for it which is `2000`. The reason for this is explained further in the **NOTE** below.
- Mark a node as caught up if `leaderLastCaughtUpTimestamp - replicaLastCaughtUpTimestamp < controllerQuorumFetchTimeoutMs`.
- Count each controller node that is caught up, including the leader (`numOfCaughtUpControllers`).
- Can roll if: `numOfCaughtUpControllers > ceil((double) totalNumOfControllers / 2)`.

> NOTE: Until [KIP-919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum) is implemented, KafkaRoller cannot create an admin connection to the controller directly to describe its configuration or the quorum state. Therefore KafkaRoller checks the desired configurations to get the value of `controller.quorum.fetch.timeout.ms` and creates admin client connection to the brokers for the quorum check. If KafkaRoller cannot connect to any of the brokers after the maximum attempts to retry, the controllers will be marked as failed to reconcile because of not being able to determine the quorum health. KafkaRoller would then try to reconcile the brokers, which may help restoring admin client connections to them. In this scenario, the reconcilation will be completed with failure and reported to the operator. The controllers quorum check will be retried in the next round of reconcilation.


#### Separate brokers and controllers
For controller-only:
 1. Does not force roll a controller performing log recovery
 2. Force rolls a broker if the pod is in stuck state
 3. Attempts to connect an admin client to any of the brokers, if cannot connect, skip checks in `4` and `5` 
 4. Rolls a controller if the controller configuration has changed
 5. Does not roll a controller if doing so would take the number of caught-up controllers (inc leader) to less than half of the quorum

For broker-only:
 1. Does not force roll a broker performing log recovery
 2. Force rolls a broker if the pod is in stuck state
 3. **NEW** Attempts to connect an admin client to a broker and if it can't connect rolls the broker 
 (the order of the rollability checks is different, we only attempt admin client connection and force roll if the pod is not stuck)
 4. Rolls a broker if the broker configuration has changed and cannot be updated dynamically
 5. Does not roll a broker if doing so would take the in-sync replicas count below `min.insync.replicas`


#### Combined mode
1. Does not force roll a combined node performing log recovery
2. Force rolls a combined node if the pod is in stuck state
3. Attempts to connect an admin client to any of the brokers, if cannot connect, skip checks in `4`, `5` and `6`
4. Rolls a combined node if the controller configuration has changed OR if the broker configuration has changed and cannot be updated dynamically
6. Does not roll a ready combined node if doing so would take the number of caught-up controllers (inc leader) to less than half of the quorum
7. Does not roll a ready combined node if doing so would take the in-sync replicas count below `min.insync.replicas`


#### Unready pod
Remains the same as Zookeeper mode.


#### Configuration changes
As implemented in [PR #9125](https://github.com/strimzi/strimzi-kafka-operator/pull/9125) until [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum+and+add+Controller+Registration) is implemented:
 - For broker only nodes, configuration changes are handled in the same way as brokers in the ZooKeeper case. If controller configurations changed, they will be ignored and brokers will not roll.
 - For controller only nodes, a hash of the controller configs is calculated. A change to this hash causes a restart of the node.
 - For combined nodes, both controller and broker configurations are checked and handled in the same was as brokers in the ZooKeeper case.

Once KIP 919 is implemented, the configurations of controller-only nodes will be diffed to see what values changed.
If the configurations that were updated are dynamic configurations, the KafkaRoller will call the Admin API to dynamically update 
these values. This will be similar to how dynamic configuration updates are handled in ZooKeeper mode.

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
