# 重视和解决 ABP 分布式事件乱序问题

ABP Framework 5.0 实现了单体应用场景下，收件箱和发件箱的事件严格顺序性。但在微服务或多数据库场景下，由于网络时延和设施效率的限制，
分布式事件将不再是 Linearizability [[1]](#参考) 的，因此必然会存在物理时间上的收件乱序。

借用 Daniel Wu 的文章《消息可靠性和顺序(中文)》[[2]](#参考) 中的插图为您展示问题：

![image](https://user-images.githubusercontent.com/30018771/194262471-d5c7aa5f-adc6-4593-b0b2-9738ac56edab.png)

如果一个处理中的事件，与任何其他的事件之间有因果关系，则有可能因两者的乱序而产生问题。

本文在这个事实下，讨论我们在订阅方可能遇到的情况和解决方案。

## 案例

我们做以下假设。

1. 我们关注的是一个用户积分服务，它是一些分布式事件的订阅方。
2. `m1` 和 `m2` 是 **先后发生** 的两个事件。
3. `t1` 和 `t2` 分别为订阅方服务收到并处理事件 m1 和 m2 的时间。
4. `t1 < t2` 代表 t1 早于 t2，称为正序；`t1 > t2` 代表 t1 晚于 t2，称为乱序。
5. C 代表订阅方服务的状态(Configuration)。`C0` 为初始状态，`CF` 为预期的最终状态，`CW` 为错误的最终状态。

### 案例 1

* 事件 m1：用户 A 创建事件
* 事件 m2：用户 B 创建事件
* Handler 的工作：根据 m1 和 m2，分别在本地创建 LocalUser 实体
* 分析：m1 和 m2 没有因果关系，顺序不敏感
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s1-disordered](https://user-images.githubusercontent.com/30018771/194247285-e62dc690-e691-4c38-8616-6e4f12a81975.png)](https://excalidraw.com/#json=94-kB06wg4IZdyFTBNoSe,hyQGl4eKRmgHXAU027iy0w)

无需处理。

### 案例 2

* 事件 m1：用户 A 创建事件
* 事件 m2：订单 1 支付事件
* Handler 的工作：根据 m1，在本地创建`LocalUser`实体；根据 m2，给`LocalUser.Score`增加积分
* 分析：m1 和 m2 有因果关系。m1 和 m2 顺序敏感，但“实体不存在”的异常拦截了乱序，handler 是幂等的，不存在一致性问题
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s2-disordered](https://user-images.githubusercontent.com/30018771/201462287-6155f1b9-dd9f-4452-bb3d-921b2e1b876b.png)](https://excalidraw.com/#json=6azro2d7yq3YVGqmmFkeE,vX5ZLgF_as_otPyRgZX0Yg)

无需处理。待 m1 被处理后，m2 延迟重试处理，实质上达到正序。

### 案例 3

* 事件 m1：订单 1 支付事件
* 事件 m2：订单 1 取消事件
* Handler 的工作：根据 m1，给`LocalUser.Score`增加积分；根据 m2，给`LocalUser.Score`扣减积分；积分最低扣到 0，不会为负数
* 分析： m1 和 m2 有因果关系。m1 和 m2 顺序敏感，handler 不是幂等的，存在一致性问题
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s3-disordered](https://user-images.githubusercontent.com/30018771/201470772-4a01a4fe-f933-4d2c-82cf-e59fd2905bec.png)](https://excalidraw.com/#json=1wnVTL1RZWpvXu3YkHpj8,uFffnJLeWEMTLk3U33Z2ZA)

积分服务在本地创建`LocalOrder`实体记录订单处理状态。

```CSharp
public class LocalOrder : AggregateRoot<Guid>
{
    public bool HasPaidEventHandled { get; set; } // set to true after handling m1
}
```

当 m2 handler 发现`OrderCanceledEto.OrderPaidTime != null`而`LocalOrder.HasPaidEventHandled == false`，则抛出错误。待 m1 被处理后，m2 延迟重试处理，实质上达到正序。

我们实质上把本案例 3 转化成了案例 2 的情况，从而实现了幂等。

#### 处理后

  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s3-resolved](https://user-images.githubusercontent.com/30018771/201462287-6155f1b9-dd9f-4452-bb3d-921b2e1b876b.png)](https://excalidraw.com/#json=6azro2d7yq3YVGqmmFkeE,vX5ZLgF_as_otPyRgZX0Yg)

### 案例 4

* 事件 m1：用户 A 变更事件 (变更了可用区 `Region`)
* 事件 m2：订单 1 支付事件
* Handler 的工作：根据 m1，由于`UserEto.Region != LocalUser.Region`，清零`LocalUser.Score`。根据 m2，给`LocalUser.Score`增加积分
* 分析：m1 和 m2 有因果关系。m1 和 m2 顺序敏感，handler 不是幂等的，存在一致性问题
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s4-disordered](https://user-images.githubusercontent.com/30018771/201470772-4a01a4fe-f933-4d2c-82cf-e59fd2905bec.png)](https://excalidraw.com/#json=1wnVTL1RZWpvXu3YkHpj8,uFffnJLeWEMTLk3U33Z2ZA)

我们可以通过这些改动解决问题：

1. 给`User`实体扩展 int 类型属性`RegionVersion`，默认值为 0，每次 Region 变更时，`RegionVersion`递增 1。
2. 积分服务使用`LocalUserRegion.Score`记录用户的积分，而非使用`LocalUser.Score`。
    ```CSharp
    public class LocalUserRegion : AggregateRoot<Guid>
    {
        public Guid UserId { get; set; }
        public string Region { get; set; }
        public int RegionVersion { get; set; }
        public int Score { get; set; }
    }
    ```
3. 处理 m1 时，若`UserEto.RegionVersion`更新，则创建新的`LocalUserRegion`实体，初始的积分为 0，相当于变更 Region 即清零积分。
4. 在用户支付时，本地服务调用 Identity 远程服务，将查得的`UserDto.RegionVersion`写入事件 m2 的`OrderPaidEto.UserRegionVersion`。
5. 处理 m2 时，根据`OrderPaidEto.UserRegionVersion`，增加对应的`LocalUserRegion.Score`。

我们解除了 m1 和 m2 的因果关系，从而实现了幂等。

#### 处理后

  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序) ：

    [![s4-resolved](https://user-images.githubusercontent.com/30018771/201462287-6155f1b9-dd9f-4452-bb3d-921b2e1b876b.png)](https://excalidraw.com/#json=6azro2d7yq3YVGqmmFkeE,vX5ZLgF_as_otPyRgZX0Yg)

### 案例 5：ABP 实体同步器

在 ABP 的 DDD 实践中，不同模块之间会通过实体同步器冗余实体数据。一个典型的案例是 Blogging 模块的 BlogUserSynchronizer [[3]](#参考)。本案例的特别之处在于，如果不是有极严的要求，过期的事件可以被跳过处理。

* 事件 m1：用户 A 变更事件
* 事件 m2：用户 A 变更事件
* Handler 的工作：根据 m1/m2，更新`LocalUser`实体中的用户资料
* 分析：一旦 m2 早于 m1 被处理，则旧资料会覆盖新资料，存在一致性问题
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s5-disordered](https://user-images.githubusercontent.com/30018771/201468277-40c792ce-9a9f-4b29-b46c-c4392b3b79bb.png)](https://excalidraw.com/#json=SwmSL9qcgrFZA5UV8HuPD,_LiJ20bKVHx8D7x5c-KfAw)

我们给实体增加 int 类型的 `EntityVersion` 属性，此属性的值从 0 开始，并在每次更新实体时，自动递增 1。在实体同步器处理 `EntityUpdatedEto<UserEto>` 事件时，若 `UserEto.EntityVersion <= LocalUser.EntityVersion`，则跳过处理。就这样，我们解决了问题。我尝试了在 ABP 框架实现以上能力，见 PR #14197 [[4]](#参考)。

#### 处理后

  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s5-resolved](https://user-images.githubusercontent.com/30018771/201468757-793bc2bb-5d47-4c7d-bcff-32e705e24d1e.png)](https://excalidraw.com/#json=L0ZI13yl9EYwWtyQC6hwK,CcVzzXgnznQSGA7x8qLGng)

## 思路整理

笔者认为，解决事件乱序问题有以下思路。

1. 尽可能保持 DistributedEventHandler 的业务逻辑简单，以便发现潜在的乱序问题。
2. 某些情况下，我们可以通过在本地记录实体的状态，将 handler 转化为幂等，就如上面案例 3 演示的那样。
3. 某些情况下，我们可以通过调整业务设计，解除因果关系，就如上面案例 4 演示的那样。
4. 实体同步器应采用 EntityVersion 的设计，以避免同步到过期的数据。

## 结论

即使你的应用当前只是单体，也应关心收件乱序问题，为今后可能到来的架构变化做储备。另外，请放弃实现 Linearizability，因为在微服务或多数据库场景下这是不可能的。

本文提到的几个案例，开发者似乎不难找出一致性问题的隐患。但在实际生产中，业务往往更复杂，事件数量也会更多，我们很难顾及周全。即便我们在开发时把所有可能的因果关系都找了出来，并且处理了它们，将来业务变更时，我们还能确保万无一失吗？答案恐怕是否定的。

分布式一致性问题是没有银弹的，它永远都在那里，开发者能做的是降低复杂度，通过设计解除因果关系，或手动实现幂等。

## 参考

1. Herlihy, Maurice P.; Wing, Jeannette M. (1987). "Axioms for Concurrent Objects". Proceedings of the 14th ACM SIGACT-SIGPLAN Symposium on Principles of Programming Languages, POPL '87. p. 13
2. Daniel Wu. (2021). 《消息可靠性和顺序(中文)》. https://danielw.cn/messaging-reliability-and-order-cn
3. GitHub abpframework/abp repository. BlogUserSynchronizer.cs. https://github.com/abpframework/abp/blob/1275f2207fc39d850d23472294e456c8504f20d2/modules/blogging/src/Volo.Blogging.Domain/Volo/Blogging/Users/BlogUserSynchronizer.cs
4. GitHub abpframework/abp repository. PR #14197. https://github.com/abpframework/abp/pull/14197
