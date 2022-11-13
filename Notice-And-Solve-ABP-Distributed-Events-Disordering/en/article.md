# Notice and Solve ABP Distributed Events Disordering

ABP Framework 5.0 implemented the strict ordering of outboxes and inboxes for monolithic apps.
However, in a microservice or multi-database scenario,
distributed events will no longer be linearizability [[1]](#references) due to the limitations of network latency and infrastructure efficiency.
So there will inevitably be physical time disordering events.

Use an illustration from Daniel Wu's article "Messaging Reliability and Ordering" [[2]](#references) to show you the problem:

![image](https://user-images.githubusercontent.com/30018771/194262471-d5c7aa5f-adc6-4593-b0b2-9738ac56edab.png)

If the event being handled has a causality with any other event, there may be problems caused by the two being disordering.

This article will discuss the situations and solutions we may encounter on the event subscriber apps based on the above truth.

## Cases

We make the following conventions.

1. We focus on a user score service that subscribes to some distributed events.
2. `m1` and `m2` are two events that occurred **successively**.
3. `t1` and `t2` are when the service receives and handles the events m1 and m2, respectively.
4. `t1 < t2` means that t1 is earlier than t2, which is called ordered; `t1 > t2` means that t1 is later than t2, which is called disordered.
5. C (configuration) represents the state of the subscriber service. `C0` is the initial state, `CF` is the expected final state, and `CW` is the wrong final state.

### Case 1: m1 and m2 are non-causal

* Event m1: event of user A created
* Event m2: event of user B created
* Handler jobs: Create LocalUser entities locally according to m1 and m2, respectively
* Analysis: m1 and m2 are ordering insensitive
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s1-disordered](https://user-images.githubusercontent.com/30018771/194247285-e62dc690-e691-4c38-8616-6e4f12a81975.png)](https://excalidraw.com/#json=94-kB06wg4IZdyFTBNoSe,hyQGl4eKRmgHXAU027iy0w)

We don't need to intervene in it.

### Case 2

* Event m1: event of user A created
* Event m2: event of order 1 paid
* Handler jobs: According to m1, create a `LocalUser` entity locally; according to m2, increase `LocalUser.Score`
* Analysis: m1 and m2 are causal; m1 and m2 are ordering sensitive, but the "entity not found exception" intercepts the disordering, so handlers are idempotent, which avoids the consistency problem
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s2-disordered](https://user-images.githubusercontent.com/30018771/201462287-6155f1b9-dd9f-4452-bb3d-921b2e1b876b.png)](https://excalidraw.com/#json=6azro2d7yq3YVGqmmFkeE,vX5ZLgF_as_otPyRgZX0Yg)

We don't need to intervene in it. After m1 is handled, m2 delays retry handling, essentially reaching the order.

### Case 3

* Event m1: event of order 1 paid
* Event m2: event of order 1 canceled
* Handler jobs: According to m1, increase `LocalUser.Score`; according to m2, deduct `LocalUser.Score`; The score is minimum deducted to 0 and will not be a negative number
* Analysis: m1 and m2 are causal; m1 and m2 are ordering sensitive, and handlers are not idempotent, so there will be a consistency problem
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s3-disordered](https://user-images.githubusercontent.com/30018771/201470772-4a01a4fe-f933-4d2c-82cf-e59fd2905bec.png)](https://excalidraw.com/#json=1wnVTL1RZWpvXu3YkHpj8,uFffnJLeWEMTLk3U33Z2ZA)

The user score service creates `LocalOrder` entities to record the order handling states.

```CSharp
public class LocalOrder : AggregateRoot<Guid>
{
    public bool HasPaidEventHandled { get; set; } // set to true after handling m1
}
```

If the m2 handler finds `OrderCanceledEto.OrderPaidTime != null` but `LocalOrder.HasPaidEventHandled == false`, it throws an exception. After m1 is handled, m2 delays retry handling, essentially reaching the order.

We essentially transformed Case 3 into Case 2, thus achieving idempotency.

#### After the Change

  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s3-resolved](https://user-images.githubusercontent.com/30018771/201462287-6155f1b9-dd9f-4452-bb3d-921b2e1b876b.png)](https://excalidraw.com/#json=6azro2d7yq3YVGqmmFkeE,vX5ZLgF_as_otPyRgZX0Yg)

### Case 4

* Event m1: event of user A updated (`Region` changed)
* Event m2: event of order 1 paid
* Handler jobs: According to m1, clear `LocalUser.Score` if `UserEto.Region != LocalUser.Region`. According to m2, increase `LocalUser.Score`
* Analysis: m1 and m2 are causal; m1 and m2 are ordering sensitive, and handlers are not idempotent, so there will be a consistency problem
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s4-disordered](https://user-images.githubusercontent.com/30018771/201470772-4a01a4fe-f933-4d2c-82cf-e59fd2905bec.png)](https://excalidraw.com/#json=1wnVTL1RZWpvXu3YkHpj8,uFffnJLeWEMTLk3U33Z2ZA)

We can solve the problem with these changes:

1. Add a new `RegionVersion` property in the `User` entity with the default value of 0. It increases by 1 once the user's region is changed.
2. The user score service uses `LocalUserRegion.Score` to record user scores instead of `LocalUser.Score`.
    ```CSharp
    public class LocalUserRegion : AggregateRoot<Guid>
    {
        public Guid UserId { get; set; }
        public string Region { get; set; }
        public int RegionVersion { get; set; }
        public int Score { get; set; }
    }
    ```
3. When handling m1, if `UserEto.RegionVersion` is new, create an extra `LocalUserRegion` entity with the initial score of 0, which equals clearing the user's scores once his region is changed.
4. When the user pays, the local service invokes the Identity remote service to query and sets the found `UserDto.RegionVersion` as `OrderPaidEto.UserRegionVersion` in the event m2.
5. When handling m2, according to m1, increase the corresponding `LocalUserRegion.Score`.

We made the handlers idempotent by disentangling the causality of m1 and m2.

#### After the Change

  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s4-resolved](https://user-images.githubusercontent.com/30018771/201462287-6155f1b9-dd9f-4452-bb3d-921b2e1b876b.png)](https://excalidraw.com/#json=6azro2d7yq3YVGqmmFkeE,vX5ZLgF_as_otPyRgZX0Yg)

### Case 5: ABP Entity Synchronizer

In the DDD practice of the ABP framework, modules use entity synchronizers to redundant data of external entities. A typical case is the BlogUserSynchronizer [[3]](#references) of the Blogging module. What's unique about this case is that stale events can be skipped handling if not strictly required.

* Event m1: event of User A updated
* Event m2: event of User A updated
* Handler jobs: According to m1 and m2, update the user information in the `LocalUser` entity.
* Analysis: Once m2 is handled earlier than m1, the stale data will overwrite the latest data, so there will be a consistency problem
  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s5-disordered](https://user-images.githubusercontent.com/30018771/201468277-40c792ce-9a9f-4b29-b46c-c4392b3b79bb.png)](https://excalidraw.com/#json=SwmSL9qcgrFZA5UV8HuPD,_LiJ20bKVHx8D7x5c-KfAw)

Let's add a new integer property named `EntityVersion`. Its default value is 0 and increments by one every time the entity changes. When the entity synchronizer gets an `EntityUpdatedEto<UserEto>` event, skip handling if `UserEto.EntityVersion <= LocalUser.EntityVersion` is satisfied. That's it, and we solved the problem. I tried implementing the above feature in the ABP framework. See PR #14197 [[4]](#references).

#### After the Change

  * t1 < t2 (ordered):

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (disordered):

    [![s5-resolved](https://user-images.githubusercontent.com/30018771/201468757-793bc2bb-5d47-4c7d-bcff-32e705e24d1e.png)](https://excalidraw.com/#json=L0ZI13yl9EYwWtyQC6hwK,CcVzzXgnznQSGA7x8qLGng)

## Solution Summary

We believe that there are the following ideas for solving event disordering.

1. Try to keep the business logic of the DistributedEventHandler simple to spot potential event disordering problems.
2. In some cases, we can make handlers idempotent by recording the entity's states locally, as Case 3 did above.
3. In some cases, we can disentangle causality by improving the business design, as Case 4 did above.
4. Entity synchronizers can be designed with EntityVersion to avoid synchronizing to stale data.

## Conclusion

Even if your app is currently monolithic, you should be concerned about the event disordering. That is a preparation for possible architectural changes in the future. Also, please give up implementing linearizability, as it is impossible in microservices or multi-database scenarios.

In several cases mentioned in this article, it seems that it is not difficult for developers to find the hidden dangers of consistency problems. However, the business is more complex in actual production, and the diverse events will be hard-considered. Even if we find out all possible causalities and deal with them during development, can we ensure that nothing will go wrong when the business changes? The answer is probably no.

There is no silver bullet to the distributed consistency problem. It's always there. Developers can only reduce complexity, dissolve the causality or manually implement idempotency.

## References

1. Herlihy, Maurice P.; Wing, Jeannette M. (1987). "Axioms for Concurrent Objects". Proceedings of the 14th ACM SIGACT-SIGPLAN Symposium on Principles of Programming Languages, POPL '87. p. 13
2. Daniel Wu. (2021). "Messaging Reliability and Ordering". https://danielw.cn/messaging-reliability-and-order
3. GitHub abpframework/abp repository. BlogUserSynchronizer.cs. https://github.com/abpframework/abp/blob/1275f2207fc39d850d23472294e456c8504f20d2/modules/blogging/src/Volo.Blogging.Domain/Volo/Blogging/Users/BlogUserSynchronizer.cs
4. GitHub abpframework/abp repository. PR #14197. https://github.com/abpframework/abp/pull/14197
