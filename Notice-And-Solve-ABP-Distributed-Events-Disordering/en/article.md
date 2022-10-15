# Notice and Solve ABP Distributed Events Disordering

ABP Framework 5.0 implemented the strict ordering of outboxes and inboxes for monolithic apps.
However, in a microservice or multi-database scenario,
distributed events will no longer be linearizability [[1]](#references) due to the limitations of network latency and infrastructure efficiency.
So there will inevitably be physical time disordering events.

Borrowing an illustration from Daniel Wu's article "Messaging Reliability and Ordering" [[2]](#references) to show you the problem:

![image](https://user-images.githubusercontent.com/30018771/194262471-d5c7aa5f-adc6-4593-b0b2-9738ac56edab.png)

If the event being handled has a causality with any other event, there may be problems caused by the two being disordering.

This article will discuss the situations and solutions we may encounter on the event subscriber apps based on the above truth.

## Conventions

1. We focus on a user score service, which is a subscriber to some distributed events.
2. `m1` and `m2` are two events that occurred **successively**.
3. `t1` and `t2` are when the service receives and handles the events m1 and m2, respectively.
4. `t1 < t2` means that t1 is earlier than t2, which is called ordered; `t1 > t2` means that t1 is later than t2, which is called disordered.
5. C (configuration) represents the state of the subscriber service. `C0` is the initial state, `CF` is the expected final state, and `CW` is the wrong final state.

## Scenarios

### Scenario 1: m1 and m2 are non-causal

* Event m1: event of user A created
* Event m2: event of user B created
* Handler jobs: Create LocalUser entities locally according to m1 and m2, respectively
* Analysis: m1 and m2 are ordering insensitive
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s1-disordered](https://user-images.githubusercontent.com/30018771/194247285-e62dc690-e691-4c38-8616-6e4f12a81975.png)](https://excalidraw.com/#json=94-kB06wg4IZdyFTBNoSe,hyQGl4eKRmgHXAU027iy0w)

We don't need to intervene in it.

### Scenario 2: m1 and m2 are causal, m2 handler is idempotent

* Event m1: event of user A created
* Event m2: event of order 1 paid
* Handler jobs: According to m1, create a `LocalUser` entity locally; according to m2, increase `LocalUser.Score`
* Analysis: m1 and m2 are ordering insensitive, but the disordering is intercepted by the business code, which avoids the consistency problem
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s2-disordered](https://user-images.githubusercontent.com/30018771/194253204-da17cf54-8b6a-410e-89f7-ed01d6a7fd0a.png)](https://excalidraw.com/#json=4DC1glLm6_BfbYG8n5Z1i,laJPAVucCQX9cq7f79FoHg)

We don't need to intervene in it. After m1 is handled, m2 delays in retrying handling, essentially reaching the ordered.

### Scenario 3: m1 and m2 are causal, m2 handler is not idempotent, m1 and m2 are events from the same entity

* Event m1: event of order 1 paid
* Event m2: event of order 1 canceled
* Handler jobs: According to m1, increase `LocalUser.Score`; according to m2, deduct `LocalUser.Score` if the order has been paid
* Analysis: m1 and m2 are ordering insensitive, resulting in the consistency problem
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s3-s4-disordered](https://user-images.githubusercontent.com/30018771/194257491-ff439083-5a18-4afa-b815-a2853a4b5e97.png)](https://excalidraw.com/#json=83yIcQyZr9Nn8QCewL9LK,CeEjjo-knZoUuSkYbjG0BA)

We can make m1 and m2 carry the entity data, at least include the `PaidTime` and `CancellationTime`, to judge the actual state of the order and make correct handling, which essentially reordered the events.

#### After the Change

  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s3-resolved](https://user-images.githubusercontent.com/30018771/194258486-03390b1f-9b9e-4802-8099-db9a66d9c0b1.png)](https://excalidraw.com/#json=sgRxqhVcsJ_NfphSDKD2T,1XH7AKhZmSpTSXgjS1feKg)

### Scenario 4: m1 and m2 are causal, m2 handler is not idempotent, m1 and m2 are events from different entities

* Event m1: event of user A updated (`Region` changed)
* Event m2: event of order 1 paid
* Handler jobs: According to m1, clear `LocalUser.Score` if `UserEto.Region != LocalUser.Region`. According to m2, increase `LocalUser.Score`
* Analysis: m1 and m2 are ordering insensitive, resulting in the consistency problem
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s3-s4-disordered](https://user-images.githubusercontent.com/30018771/194257491-ff439083-5a18-4afa-b815-a2853a4b5e97.png)](https://excalidraw.com/#json=83yIcQyZr9Nn8QCewL9LK,CeEjjo-knZoUuSkYbjG0BA)

We can solve the problem with these changes:
  1. Add a new `RegionVersion` property in the `User` entity with the default value of 0. It increases by 1 once the user's region is changed.
  2. When the user pays, the local service invokes the Identity remote service to query, write the found `UserDto.RegionVersion` into `OrderPaidEto.UserRegionVersion`, and publishes it together with the event m2.
  3. When handling m1, `UserEto.RegionVersion` should be synced to `LocalUser.RegionVersion`.
  4. When handling m2, discard the event and end handling if `OrderPaidEto.RegionVersion < LocalUser.RegionVersion`. It is because changing the region will clear the scores, and the scores obtained from the old region should be discarded and not added to the scores of the new region.
  5. When handling m2, invokes the Identity remote service to query whether `UserDto.RegionVersion == LocalUser.RegionVersion`. If so, the handler should increase the user's scores. Otherwise, it throws an exception and tries next time. It is to ensure that the synchronization of the RegionVersion (including clearing the user's existing scores) is done.

#### After the Change

  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered) and RegionVersion is not stale:

    [![s4-resolved-1](https://user-images.githubusercontent.com/30018771/194259901-bc57228c-f307-4b7c-9753-56b34b2a5b2b.png)](https://excalidraw.com/#json=y_PkS5DOUfJudbS8jE1h-,VVFmDfNuw4CuyOirCG54FA)

  * t1 > t2 (disordered) and RegionVersion is stale:

    [![s4-resolved-2](https://user-images.githubusercontent.com/30018771/194261319-1785b143-6d41-4f38-b984-d0c4f6d9708e.png)](https://excalidraw.com/#json=74D7htXoXKvDzMXQgJ6aH,6cL_fOdAfwvyHMVqL-21YQ)

#### A Better Way

Think different. What if we create an entity for each RegionVersion of each user? Then the m1 and m2 are no more causal, so the ordering requirement is also gone.

## Solution Summary

We believe that there are the following principles for solving event disordering.

1. Try to keep the business logic of the DistributedEventHandler simple to spot potential event disordering problems.
2. If the causality is from the entity's own state, we can check the entity state to realize the idempotency of the event handler. Refer to Scenario 3 above.
3. If the causality is from other entities, we can try to dissolve the causality by design. If we fail, implement idempotency manually (this is not recommended since it brings complexity). Refer to Scenario 4 above.

## Conclusion

Even if your app is currently monolithic, you should be concerned about the event disordering. That is a preparation for possible architectural changes in the future. Also, please give up implementing linearizability, as it is impossible in microservices or multi-database scenarios.

In several scenarios mentioned in this article, it seems that it is not difficult for developers to find the hidden dangers of consistency problems. However, the business is more complex in actual production, and the diverse events will be hard-considered. Even if we find out all possible causalities and deal with them during development, can we ensure that nothing will go wrong when the business changes? The answer is probably no.

There is no silver bullet to the distributed consistency problem. It's always there. Developers can only reduce complexity, dissolve the causality or manually implement idempotency.

## References

1. Herlihy, Maurice P.; Wing, Jeannette M. (1987). "Axioms for Concurrent Objects". Proceedings of the 14th ACM SIGACT-SIGPLAN Symposium on Principles of Programming Languages, POPL '87. p. 13
2. Daniel Wu. (2021). "Messaging Reliability and Ordering". https://danielw.cn/messaging-reliability-and-order
