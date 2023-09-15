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

**NOTE** There is a related proposal describing how to diff controller configuration currently being discussed: [PR #82](https://github.com/strimzi/proposals/pull/82)

### Rollability checks
The checks made by the KafkaRoller in different modes is described below.

#### The new quorum check
The checks include a new check for controllers to verify that rolling the node does not affect the quorum health.
The proposed check is:
- Create admin client connection to the brokers and call `describeMetadataQuorum` API
- If failed to connect to the brokers, return `UnforceableProblem` which will result in delay and retry for the pod until the maximum attempts is reached
- From the quorum info returned from the admin API, fetch `lastCaughtUpTimestamp` for every controller node. `lastCaughtUpTimestamp` is the last millisecond timestamp at which a replica controller was known to be caught up with the quorum leader
- Check the quorum leader id using the quorum info and identify the `lastCaughtUpTimestamp` of the quorum leader
- Retrieve the current value of Kafka configuration `controller.quorum.fetch.timeout.ms`

        **NOTE** Once [PR #82](https://github.com/strimzi/proposals/pull/82) is implemented, we would be able to get the current value for this configuration. Until then we would have to use the default value. 
- Mark a node as caught up if `leaderLastCaughtUpTimestamp - replicaLastCaughtUpTimestamp < controllerQuorumFetchTimeoutMs`
- Count each controller node that is caught up, including the leader (`numOfCaughtUpControllers`)
- Can roll if: `numOfCaughtUpControllers > ceil((double) totalNumOfControllers / 2)`

In order to perform quorum check for controller, KafkaRoller creates admin client connection to the brokers. This is required until [KIP-919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum) is implemented. If KafkaRoller cannot connect to any of the brokers after the maximum attempts to retry, the controllers will be marked as failed to reconcile because of not being able to determine the quorum health. KafkaRoller would then try to reconcile the brokers, which may help restoring admin client connections to them. In this scenario, the reconcilation will be completed with failure and reported to the operator. The controllers quorum check will be retried in the next round of reconcilation.


#### Separate brokers and controllers
For controller-only:
 1. Does not force roll a controller performing log recovery
 2. Force rolls a broker if the pod is in stuck state
 3. Attemps to connect an admin client to any of the brokers, if cannot connect, skip checks in `4` and `5` 
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
3. Attemps to connect an admin client to any of the brokers, if cannot connect, skip checks in `4`, `5` and `6`
4. Rolls a combined node if the controller configuration has changed OR if the broker configuration has changed and cannot be updated dynamically
6. Does not roll a ready combined node if doing so would take the number of caught-up controllers (inc leader) to less than half of the quorum
7. Does not roll a ready combined node if doing so would take the in-sync replicas count below `min.insync.replicas`


#### Unready pod
Remains the same as Zookeeper mode.


#### Configuration changes
For all kinds of nodes:
- Retrieves the current Kafka configurations of the node via Admin client and compares it with the desired configurations specified in the Kafka CR.
- Performs dynamic configuration updates if possible, otherwise rolls node on configuration change

**NOTE:** At the time of writing this proposal, it is not possible to connect to controller nodes via admin client to retrieve the current configuration or apply dynamic configuration updates. The related proposal in 
[PR #82](https://github.com/strimzi/proposals/pull/82) describes a workaround for retrieving the current configuration of controller nodes so we can detect if there is any configuration change. However dynamic configuration update for them would not possible until [KIP 919](https://cwiki.apache.org/confluence/display/KAFKA/KIP-919%3A+Allow+AdminClient+to+Talk+Directly+with+the+KRaft+Controller+Quorum) is implemented. Therefore until then, the controllers will be rolled when there is a configuration change.
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
