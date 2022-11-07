# 重视和解决 ABP 分布式事件乱序问题

ABP Framework 5.0 实现了单体应用场景下，收件箱和发件箱的事件严格顺序性。但在微服务或多数据库场景下，由于网络时延和设施效率的限制，
分布式事件将不再是 Linearizability [[1]](#参考) 的，因此必然会存在物理时间上的收件乱序。

借用 Daniel Wu 的文章《消息可靠性和顺序(中文)》[[2]](#参考) 中的插图为您展示问题：

![image](https://user-images.githubusercontent.com/30018771/194262471-d5c7aa5f-adc6-4593-b0b2-9738ac56edab.png)

如果一个处理中的事件，与任何其他的事件之间有因果关系，则有可能因两者的乱序而产生问题。

本文在这个事实下，讨论我们在订阅方可能遇到的情况和解决方案。

## 假设

1. 我们关注的是一个用户积分服务，它是一些分布式事件的订阅方。
2. `m1` 和 `m2` 是 **先后发生** 的两个事件。
3. `t1` 和 `t2` 分别为订阅方服务收到并处理事件 m1 和 m2 的时间。
4. `t1 < t2` 代表 t1 早于 t2，称为正序；`t1 > t2` 代表 t1 晚于 t2，称为乱序。
5. C 代表订阅方服务的状态(Configuration)。`C0` 为初始状态，`CF` 为预期的最终状态，`CW` 为错误的最终状态。

## 场景

### 场景 1：m1 和 m2 没有因果关系

* 事件 m1：用户 A 创建事件
* 事件 m2：用户 B 创建事件
* Handler 的工作：根据 m1 和 m2，分别在本地创建 LocalUser 实体
* 分析：m1 和 m2 顺序不敏感
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s1-disordered](https://user-images.githubusercontent.com/30018771/194247285-e62dc690-e691-4c38-8616-6e4f12a81975.png)](https://excalidraw.com/#json=94-kB06wg4IZdyFTBNoSe,hyQGl4eKRmgHXAU027iy0w)

无需处理。

### 场景 2：m1 和 m2 有因果关系，但 m2 handler 是幂等的

* 事件 m1：用户 A 创建事件
* 事件 m2：订单 1 支付事件
* Handler 的工作：根据 m1，在本地创建`LocalUser`实体；根据 m2，给`LocalUser.Score`增加积分
* 分析：m1 和 m2 顺序敏感，但业务上拦截了乱序，不产生一致性问题
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s2-disordered](https://user-images.githubusercontent.com/30018771/194253204-da17cf54-8b6a-410e-89f7-ed01d6a7fd0a.png)](https://excalidraw.com/#json=4DC1glLm6_BfbYG8n5Z1i,laJPAVucCQX9cq7f79FoHg)

无需处理。待 m1 被处理后，m2 延迟重试处理，实质上达到正序。

### 场景 3：m1 和 m2 有因果关系，m2 handler 不是幂等的，m1 和 m2 是相同实体产生的事件

* 事件 m1：订单 1 支付事件
* 事件 m2：订单 1 取消事件
* Handler 的工作：根据 m1，给`LocalUser.Score`增加积分；根据 m2，如果订单已支付，给`LocalUser.Score`扣减积分
* 分析： m1 和 m2 顺序敏感，产生一致性问题
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s3-s4-disordered](https://user-images.githubusercontent.com/30018771/194257491-ff439083-5a18-4afa-b815-a2853a4b5e97.png)](https://excalidraw.com/#json=83yIcQyZr9Nn8QCewL9LK,CeEjjo-knZoUuSkYbjG0BA)

积分服务在处理订单事件时，于本地冗余`LocalOrder`实体记录订单信息。

```CSharp
public class LocalOrder : AggregateRoot<Guid>
{
    public DateTime? ScoreGrantedTime { get; set; }
    public bool IsCanceled { get; set; }
}
```

* 如果 m1 先被处理
    1. 处理 m1 时，设置`LocalOrder.ScoreGrantedTime = now`，由于`LocalOrder.ScoreGrantedTime != null`，给用户增加积分。
    2. 处理 m2 时，由于`LocalOrder.ScoreGrantedTime != null`，给用户扣除积分。
* 如果 m2 先被处理
    1. 处理 m2 时，设置`LocalOrder.IsCanceled = true`，由于`LocalOrder.ScoreGrantedTime == null`，不再扣除用户的积分。
    2. 处理 m1 时，由于`LocalOrder.IsCanceled == true`，不再增加用户的积分。

实质上达到正序。

#### 处理后

  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s3-resolved](https://user-images.githubusercontent.com/30018771/194258486-03390b1f-9b9e-4802-8099-db9a66d9c0b1.png)](https://excalidraw.com/#json=sgRxqhVcsJ_NfphSDKD2T,1XH7AKhZmSpTSXgjS1feKg)

### 场景 4：m1 和 m2 有因果关系，m2 handler 不是幂等的，m1 和 m2 是不同实体产生的事件

* 事件 m1：用户 A 变更事件 (变更了可用区 `Region`)
* 事件 m2：订单 1 支付事件
* Handler 的工作：根据 m1，由于`UserEto.Region != LocalUser.Region`，清零`LocalUser.Score`。根据 m2，给`LocalUser.Score`增加积分
* 分析：m1 和 m2 顺序敏感，产生一致性问题
  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序)：

    [![s3-s4-disordered](https://user-images.githubusercontent.com/30018771/194257491-ff439083-5a18-4afa-b815-a2853a4b5e97.png)](https://excalidraw.com/#json=83yIcQyZr9Nn8QCewL9LK,CeEjjo-knZoUuSkYbjG0BA)

我们可以通过这些改动解决问题：
  1. 给`User`实体扩展 int 类型属性`RegionVersion`，默认值为 0，每次 Region 变更时，`RegionVersion`递增 1
  2. 在用户支付时，本地服务调用 Identity 远程服务，将查得的`UserDto.RegionVersion`写入`OrderPaidEto.UserRegionVersion`，与事件 m2 一起发布
  3. 处理 m1 时，应将`UserEto.RegionVersion`同步到`LocalUser.RegionVersion`
  4. 处理 m2 时，若`OrderPaidEto.RegionVersion < LocalUser.RegionVersion`，则抛弃事件，结束处理。这是因为变更可用区会清零积分，旧可用区的积分应被抛弃，而不是加到新可用区的积分中
  5. 处理 m2 时，调用 Identity 远程服务，若查得`UserDto.RegionVersion == LocalUser.RegionVersion`，则给用户增加积分，否则抛出错误等待下次重试。这是为了确保 RegionVersion 的同步（包含清空用户积分）工作已完成

#### 处理后

  * t1 < t2 (正序)：

    [![ordered](https://user-images.githubusercontent.com/30018771/194246857-ec06763c-f2be-4d39-85b2-b5243fb37a65.png)](https://excalidraw.com/#json=EzNloyRKYJa6rfvSNgm2l,HFAPhV9l9kZDT4SGJaZ-zA)

  * t1 > t2 (乱序) 且 RegionVersion 非陈旧：

    [![s4-resolved-1](https://user-images.githubusercontent.com/30018771/194259901-bc57228c-f307-4b7c-9753-56b34b2a5b2b.png)](https://excalidraw.com/#json=y_PkS5DOUfJudbS8jE1h-,VVFmDfNuw4CuyOirCG54FA)

  * t1 > t2 (乱序) 且 RegionVersion 陈旧：

    [![s4-resolved-2](https://user-images.githubusercontent.com/30018771/194261319-1785b143-6d41-4f38-b984-d0c4f6d9708e.png)](https://excalidraw.com/#json=74D7htXoXKvDzMXQgJ6aH,6cL_fOdAfwvyHMVqL-21YQ)

#### 更好的处理方案

试着转换一下思路，如果为每位用户在每个 RegionVersion 单独建立实体记录积分，m1 与 m2 就不再是因果关系，顺序性的要求也就不存在了。

## 方案总结

笔者认为，解决事件乱序问题有以下原则。

1. 尽可能保持 DistributedEventHandler 的业务逻辑简单，以便发现潜在的乱序问题。
2. 如果因果关系来源于实体自身状态，可以通过实体状态检查，实现 handler 的幂等。参考上面场景 3 的做法。
3. 如果因果关系来源于其他实体，可以尝试通过设计解除因果关系。如果无法解除因果关系，则手动实现幂等（这是不推荐的，因为会带来更大的复杂度）。参考上面场景 4 的做法。

## ABP 实体同步器

在 ABP 的 DDD 实践中，不同模块之间会通过实体同步器冗余实体数据。一个典型的案例是 Blogging 模块的 BlogUserSynchronizer [[3]](#参考)。这实际上是前文场景 3 的一种衍生。

我们给实体增加 int 类型的 `EntityVersion` 属性，此属性的值从 0 开始，并在每次更新实体时，自动递增 1。在实体同步器处理 `EntityUpdatedEto<UserEto>` 事件时，若 `UserEto.EntityVersion <= BlogUser.EntityVersion`，则跳过处理。就这样，我们解决了问题。我尝试了在 ABP 框架实现以上能力，见 PR #14197 [[4]](#参考)。

## 结论

即使你的应用当前只是单体，也应关心收件乱序问题，为今后可能到来的架构变化做储备。另外，请放弃实现 Linearizability，因为在微服务或多数据库场景下这是不可能的。

本文提到的几个场景，开发者似乎不难找出一致性问题的隐患。但在实际生产中，业务往往更复杂，事件数量也会更多，我们很难顾及周全。即便我们在开发时把所有可能的因果关系都找了出来，并且处理了它们，将来业务变更时，我们还能确保万无一失吗？答案恐怕是否定的。

分布式一致性问题是没有银弹的，它永远都在那里，开发者能做的是降低复杂度，通过设计解除因果关系，或手动实现幂等。

## 参考

1. Herlihy, Maurice P.; Wing, Jeannette M. (1987). "Axioms for Concurrent Objects". Proceedings of the 14th ACM SIGACT-SIGPLAN Symposium on Principles of Programming Languages, POPL '87. p. 13
2. Daniel Wu. (2021). 《消息可靠性和顺序(中文)》. https://danielw.cn/messaging-reliability-and-order-cn
3. GitHub abpframework/abp repository. BlogUserSynchronizer.cs. https://github.com/abpframework/abp/blob/1275f2207fc39d850d23472294e456c8504f20d2/modules/blogging/src/Volo.Blogging.Domain/Volo/Blogging/Users/BlogUserSynchronizer.cs
4. GitHub abpframework/abp repository. PR #14197. https://github.com/abpframework/abp/pull/14197
