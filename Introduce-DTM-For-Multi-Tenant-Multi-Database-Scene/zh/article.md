# 引入 DTM 以支持 ABP 的多租户多数据库场景

这篇文章分享了使用 DTM 二阶段消息模式解决 [issue #10036](https://github.com/abpframework/abp/issues/10036) 的方法。

今天我们要使用 EasyAbp 的 [Abp.EventBus.Boxes.Dtm](https://github.com/EasyAbp/Abp.EventBus.Boxes.Dtm) 模块。

## DTM 事件箱的介绍

这个模块使用了 DTM 的 [二阶段消息](https://dtm.pub/practice/msg.html) 使得 ABP 的事件箱得以支持 [多租户多数据库场景](https://github.com/abpframework/abp/issues/10036)。

你需要先阅读 [DTM 文档](https://en.dtm.pub/guide/start.html)，它将帮助你理解这个模块。

## 与 ABP 默认事件箱的差异

|                                  | DTM 二阶段消息事件箱 |            ABP 5.0+ 默认事件箱             |
| :------------------------------: | :------------------: | :----------------------------------------: |
|             收发速度             |  :heavy_check_mark:  |                    :x:                     |
|          更少的数据传输          |         :x:          |             :heavy_check_mark:             |
|  保证事件发出<br>(事务工作单元)  |  :heavy_check_mark:  |             :heavy_check_mark:             |
| 保证事件发出<br>(非事务工作单元) |         :x:          | :heavy_check_mark:<br>(要求消费端解决幂等) |
|    避免重复处理(仅限纯DB操作)    |  :heavy_check_mark:  |             :heavy_check_mark:             |
|        支持多租户多数据库        |  :heavy_check_mark:  |                    :x:                     |
|         没有增加外部设施         |         :x:          |             :heavy_check_mark:             |
|          管理面板和报警          |  :heavy_check_mark:  |                    :x:                     |

## DTM 发件箱是如何工作的？

假设你正在使用发件箱发布新的事件:
```csharp
await _distributedEventBus.PublishAsync(eto1, useOutbox: true);
await _distributedEventBus.PublishAsync(eto2, useOutbox: true);  // useOutbox 的默认值即 true
```
DTM 发件箱会临时存储这些事件。接下来我们看看，在你完成当前工作单元时它会怎么做：
```CSharp
// UnitOfWork.cs 的代码片段
protected override async Task CommitTransactionsAsync()
{
    // 第 1 步：在事务内插入一条记录到 DTM 屏障表，接着发送一条“prepare”请求到 DTM 服务器
    await DtmMessageManager.InsertBarriersAndPrepareAsync(EventBag);

    // 第 2 步: 提交当前 DB 事务
    await base.CommitTransactionsAsync();

    // 第 3 步: 发送一条"submit"请求到 DTM 服务器
    OnCompleted(async () => await DtmMessageManager.SubmitAsync(EventBag));
}
```
至此，DTM 服务器已经收到了一条“submit”请求。它会调用 app 的 `PublishEvents` 服务并附上所有事件的数据，后者被调用后会立即发布这些事件到 MQ。

[![](https://mermaid.ink/img/pako:eNqNVMtOwkAU_ZXJrOEHujAx4sIFC4LuuhnoKE1owXa6MMYEY1BESRUxPoKvRBOjkUBMBMvvdKbyF06dCqWUhK7mzj33nnNP7nQX5ksKhhI08baF9TxOqWjLQJqsA_4RlRQxSK2nAbXP3eGX1-mJBLJISbe0HDZE7LVeWa2fXFqiToufJUCrn6OrD3dQcQdvQDVNCwugyHMgu-yyRoc6FxJgrW9qXwsU8OpfrLIfAU8p4AX9Ia0_gjW_YKOsIIIVQJun4iJLeLxSQPoWv3WdE14kuk01iSiwm7T6wtWyuyeB69mj23Z8nR-zdoPWn-gNn9QPywYuIwNP8ON8clZ9qACw2hmtP8Tz_Fs5Dx9rpX3mOs98as4uUONkMtQyDIttGTV8Dn4Ba0wrp6lkMWdY7Zx2D2nnlHOw48qoXQEoT9SSbsbRRSwSRHEOcaDYT0lsSLCc0UEmyiaNl__oQdKnpYOD8DqNvUpnJOC9n9DG5ywonQnLFPl4hRHrAmLhRbzGSAV7PPrpdL3hBbtvuwNHsMAE1LChIVXhT3zX7yNDUsAalqHEjwreRFaRyFDW9zjU-ntLq4pKSgaUNlHRxAnoP_Xsjp6HEjEs_A8KfhMBau8Xdo4RAw)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNqNVMtOwkAU_ZXJrOEHujAx4sIFC4LuuhnoKE1owXa6MMYEY1BESRUxPoKvRBOjkUBMBMvvdKbyF06dCqWUhK7mzj33nnNP7nQX5ksKhhI08baF9TxOqWjLQJqsA_4RlRQxSK2nAbXP3eGX1-mJBLJISbe0HDZE7LVeWa2fXFqiToufJUCrn6OrD3dQcQdvQDVNCwugyHMgu-yyRoc6FxJgrW9qXwsU8OpfrLIfAU8p4AX9Ia0_gjW_YKOsIIIVQJun4iJLeLxSQPoWv3WdE14kuk01iSiwm7T6wtWyuyeB69mj23Z8nR-zdoPWn-gNn9QPywYuIwNP8ON8clZ9qACw2hmtP8Tz_Fs5Dx9rpX3mOs98as4uUONkMtQyDIttGTV8Dn4Ba0wrp6lkMWdY7Zx2D2nnlHOw48qoXQEoT9SSbsbRRSwSRHEOcaDYT0lsSLCc0UEmyiaNl__oQdKnpYOD8DqNvUpnJOC9n9DG5ywonQnLFPl4hRHrAmLhRbzGSAV7PPrpdL3hBbtvuwNHsMAE1LChIVXhT3zX7yNDUsAalqHEjwreRFaRyFDW9zjU-ntLq4pKSgaUNlHRxAnoP_Xsjp6HEjEs_A8KfhMBau8Xdo4RAw)

<details>
<summary>查看更详细的时序图</summary>

[![](https://mermaid.ink/img/pako:eNq1VU1PGkEY_iuTPbWJeG5IY9LWHnrwYGxvXFZ2tCSw2GX30BgTqlIRQfzAgnwoNtKQKlSDQeSj_Bc7M7uc-At9l0FcdGk1afc0H88z7_s-z7szi4LbL2HBKQTwBw3LbjzpEecV0eeSEXyqR_ViNPl2CtH4NmlU9fI5cqDJl4jUN2jkiMW3SP1Yb-6y0FfSTNNvy6S1120ecrKoqX5Z881ihc_1RJGFLx0TE7SegLET0VClkyyRWpDUviNPIKBhDuT7AGR7ZyxWpvVdJ2KJKxpPcRTSI1UW_HQHPJQlEC4bNJJHb0zCuwVJVLGE6E6UL8yoMH_1XpTnYRVqARI_beiQOxnEd2ioANmy3BHHncc76aw9z5yzbAw0ovtQqTldUPCCqOBb_GDfcT97CwGx8BaNHNrHuZFyFN5Wyp5r3EG7bO5lr5fWSf0zLbQ6e22WrHaSF-Pj4_bMgbXJYrcZJa280U7RbJFuJaEtWHibrVRG6Ph8Vpl4QlptoJOrDboeuw5mLVVdB3MsUaXhMz29iuY90tN-j3nV4RxopEhaWZa5oO0TDhklA6nVh6nWxKzemt-A57AW2QNxmlV0S8T7Yv6JZBuFG9VthvXTU_QMsVKh21znHtLjc-OiYCpHNw9pJs9SP0xtUGctRo9jYADNHLBG-jbAX4xmqU19vwVe_wouG-19thqn2xU9B_9n2oT0osGuGZCFvxjBEPmZ01erYLDRzhhHUVIrAZPl14zyGcirN3bZQRYG1jSwN4D76th79VgRRij_H71-tOwPagZ-6j_yiosCu6PPu6nshVv1-GW41uFeobUV6204JOfUNNwEJxs0VrEHTk0P6dXDPPiv6CdBy1GgPFyDXqcN2swaDcuSSxbGBB9WfKJHggdu0dxwCep77MMuwQlDCc-Jmld1CS55CaBa75V4LXlUvyI450Ro1DHBfMRmPspuwakqGr4B9R_JPmrpN52bxD0)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNq1VU1PGkEY_iuTPbWJeG5IY9LWHnrwYGxvXFZ2tCSw2GX30BgTqlIRQfzAgnwoNtKQKlSDQeSj_Bc7M7uc-At9l0FcdGk1afc0H88z7_s-z7szi4LbL2HBKQTwBw3LbjzpEecV0eeSEXyqR_ViNPl2CtH4NmlU9fI5cqDJl4jUN2jkiMW3SP1Yb-6y0FfSTNNvy6S1120ecrKoqX5Z881ihc_1RJGFLx0TE7SegLET0VClkyyRWpDUviNPIKBhDuT7AGR7ZyxWpvVdJ2KJKxpPcRTSI1UW_HQHPJQlEC4bNJJHb0zCuwVJVLGE6E6UL8yoMH_1XpTnYRVqARI_beiQOxnEd2ioANmy3BHHncc76aw9z5yzbAw0ovtQqTldUPCCqOBb_GDfcT97CwGx8BaNHNrHuZFyFN5Wyp5r3EG7bO5lr5fWSf0zLbQ6e22WrHaSF-Pj4_bMgbXJYrcZJa280U7RbJFuJaEtWHibrVRG6Ph8Vpl4QlptoJOrDboeuw5mLVVdB3MsUaXhMz29iuY90tN-j3nV4RxopEhaWZa5oO0TDhklA6nVh6nWxKzemt-A57AW2QNxmlV0S8T7Yv6JZBuFG9VthvXTU_QMsVKh21znHtLjc-OiYCpHNw9pJs9SP0xtUGctRo9jYADNHLBG-jbAX4xmqU19vwVe_wouG-19thqn2xU9B_9n2oT0osGuGZCFvxjBEPmZ01erYLDRzhhHUVIrAZPl14zyGcirN3bZQRYG1jSwN4D76th79VgRRij_H71-tOwPagZ-6j_yiosCu6PPu6nshVv1-GW41uFeobUV6204JOfUNNwEJxs0VrEHTk0P6dXDPPiv6CdBy1GgPFyDXqcN2swaDcuSSxbGBB9WfKJHggdu0dxwCep77MMuwQlDCc-Jmld1CS55CaBa75V4LXlUvyI450Ro1DHBfMRmPspuwakqGr4B9R_JPmrpN52bxD0)

[![](https://mermaid.ink/img/pako:eNqFVMtuGjEU_RVrVq0U-ABUsWjpoouu0u7YTBiHIMFAh5lFFUWiKaSEQIEEyiOQgBQqVAkEIiJkgPIvqe0ZVvmF3okTCmFQvfK1z7k-91zb-4IvLGHBJUTxJw3LPuwJiH5FDHllBEMNqEGMPB_eI5rNk_HQ6PaRA3leI6Kf0FSTZXNEv6JXffO6RSZV-vOQTIv3k0tOFjU1LGuhHazw2Ci0WfLG4XZTvQBzF6KJwbzUIaMYGf1CgWhUwxzI9wHIij2W6VL9zIVY4ZZmyxyFjNSQxb48A6-oBMLNmKYa6J1F-BiRRBVLiJ6m-cK2CvGbPVH2wyrUAiSebSXJMwXZU5pogVpWb3JcPzuv1ux5VsxqGfCIVqBSK4woOCIq-B9-se9YV79EQCyZo6lL-3OerNyEt7XyoWu8gxy12HQspVxv8f0kTc8v2Li6khsovLOQ-SxDpjUOtqtzzRejc0z0I9qazoszVhrOS9dOp9Oeubg0pTboINOGOSvTWpvmSnDhWDLPvg42dOjVjuJ-QaYzoJPbE3qcuYvVlvy6i9VZYUiTPaMaR_6A9HKzc2Skr-paPnH5OtgaygGcYtOjdXc2Ef5jKit_NypT8PVP7NCcVVg8S_MDow6vrGpB-GvVjyxbWPKHGUuQ33UjPgQzzdm52UyTUQeYrPHN7PagYmN8xi5qMOG9F7aEEFZCYkCCT2PfEuQV1D0cwl7BBVMJ74paUPUKXvkAoNrDy3srBdSwIrh2xWAUbwnWx7D9WfYJLlXR8BPo8eN5RB38BaxhZok)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNqFVMtuGjEU_RVrVq0U-ABUsWjpoouu0u7YTBiHIMFAh5lFFUWiKaSEQIEEyiOQgBQqVAkEIiJkgPIvqe0ZVvmF3okTCmFQvfK1z7k-91zb-4IvLGHBJUTxJw3LPuwJiH5FDHllBEMNqEGMPB_eI5rNk_HQ6PaRA3leI6Kf0FSTZXNEv6JXffO6RSZV-vOQTIv3k0tOFjU1LGuhHazw2Ci0WfLG4XZTvQBzF6KJwbzUIaMYGf1CgWhUwxzI9wHIij2W6VL9zIVY4ZZmyxyFjNSQxb48A6-oBMLNmKYa6J1F-BiRRBVLiJ6m-cK2CvGbPVH2wyrUAiSebSXJMwXZU5pogVpWb3JcPzuv1ux5VsxqGfCIVqBSK4woOCIq-B9-se9YV79EQCyZo6lL-3OerNyEt7XyoWu8gxy12HQspVxv8f0kTc8v2Li6khsovLOQ-SxDpjUOtqtzzRejc0z0I9qazoszVhrOS9dOp9Oeubg0pTboINOGOSvTWpvmSnDhWDLPvg42dOjVjuJ-QaYzoJPbE3qcuYvVlvy6i9VZYUiTPaMaR_6A9HKzc2Skr-paPnH5OtgaygGcYtOjdXc2Ef5jKit_NypT8PVP7NCcVVg8S_MDow6vrGpB-GvVjyxbWPKHGUuQ33UjPgQzzdm52UyTUQeYrPHN7PagYmN8xi5qMOG9F7aEEFZCYkCCT2PfEuQV1D0cwl7BBVMJ74paUPUKXvkAoNrDy3srBdSwIrh2xWAUbwnWx7D9WfYJLlXR8BPo8eN5RB38BaxhZok)
   
</details>

> 如果你依然对这个模式“如何确保发送”困惑，请查看 DTM 的 [二阶段消息文档](https://en.dtm.pub/practice/msg.html) 以了解更多。

## DTM 收件箱是如何工作的？

与 ABP 的默认实现不同，DTM 收件箱从 MQ 收到一个事件后会立即处理（handle）。在所有的 handler 完成他们的工作后，收件箱会沿用当前事务，向 DTM 屏障表插入一条记录。最后它提交了事务并向 MQ 返回 ACK。

所有入箱的事件都拥有一条唯一的 MessageId。拥有相同 MessageId 的事件只会被处理一次，因为我们不能插入 gid (MessageId) 重复的记录到 DTM 屏障表。

[![](https://mermaid.ink/img/pako:eNqFk8tKw0AUhl9lmLV9gaAFURciXYgus4nNtAaaVHNZSClUULEt0qgRRFJKQSUI9uKitvH2MjkTu_IVnDptMRptVknOf775zz8zBZzOywQL2CC7FtHSZFmRsrqkihpij6mYOYKWN1OIOr3gqRe2urwgWWZes9QtovPv1HoimYzoBDQsnQ1L-0G_FPTvaPWGuuX5LT0JTpv9QiliGFKWrMoovDoI_Cpr4qQIhEHpRYeetMA_F75K0K0Nr9z3pkcv22Dfwv0luN6Im1VktBDFvrc68HLx8dzg5Ckp8dsrx1L3jpZf4bgTbwV8J3S86WDfTPNK1C1cH4T2EVfBoBe81ZlZ6gyg4sX4mbC5AB4fuD6C_217TP9aiR7bUGnMDBHs0x9B1s7g8IbtCa03J0EGfT8uytk5xtr_xw2t2YF_zcaASnM2net4z9_jsr7UuoAWl9bwHFaJrkqKzA54YaQVsblNVCJigb3KJCNZOVPEolZkUmtHlkyyIitmXsdCRsoZZA6PDvrGnpbGgqlbZCIaX5KxqvgJeeqquw)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNqFk8tKw0AUhl9lmLV9gaAFURciXYgus4nNtAaaVHNZSClUULEt0qgRRFJKQSUI9uKitvH2MjkTu_IVnDptMRptVknOf775zz8zBZzOywQL2CC7FtHSZFmRsrqkihpij6mYOYKWN1OIOr3gqRe2urwgWWZes9QtovPv1HoimYzoBDQsnQ1L-0G_FPTvaPWGuuX5LT0JTpv9QiliGFKWrMoovDoI_Cpr4qQIhEHpRYeetMA_F75K0K0Nr9z3pkcv22Dfwv0luN6Im1VktBDFvrc68HLx8dzg5Ckp8dsrx1L3jpZf4bgTbwV8J3S86WDfTPNK1C1cH4T2EVfBoBe81ZlZ6gyg4sX4mbC5AB4fuD6C_217TP9aiR7bUGnMDBHs0x9B1s7g8IbtCa03J0EGfT8uytk5xtr_xw2t2YF_zcaASnM2net4z9_jsr7UuoAWl9bwHFaJrkqKzA54YaQVsblNVCJigb3KJCNZOVPEolZkUmtHlkyyIitmXsdCRsoZZA6PDvrGnpbGgqlbZCIaX5KxqvgJeeqquw)


<details>
<summary>查看更详细的时序图</summary>

[![](https://mermaid.ink/img/pako:eNqVVNFKG0EU_ZVhnvUHljYgtQ-lpCD1cV_W7JguZDd1M_tQREjBFhMrWXUFibE2kEgQGpNS0rjW9mf2TpKn_kJnczfGrWti52lm7rnnnjvnMps0k9cZVWiBbTjMyrBlQ8vamqlaRC5u8Bwjy6tpIrxecN0btLsY0Byetxxzjdl4Tq8splIxnEJGxYNR8X3QLwb9C7HbFLXSkzU7Bd6lvCJpVihoWfZCJ4PqduDvyiRkipFIUnHUEXtt8A-VcQi6lVG1Nqy3xPEluOfw9RhqrZA3a-jkaZx22O7AzdGfn2eR5hwniMcA3obrtsbi_S5E6RfsdLDsNCOuUmalVxSy9OwlIpil42YWMVKK2gVWSO4efG_gtW7f8s47YST-QNDYHrgfEQVXveD3qexXeFdQbiXomXAjAH58Q3yM_r7siH1cSey4UD6b6xu4-_94VzmAD005BuK0PvEu6PtJ7k2twyRodIffm4-zDk6-kJBathb5XvyUzDJL_MlncV2Fm0Mo7cneoVx_XHHEhqXHBHNG59X_zU6iZTOaEBU38Bt35c-XjjkPWzwderpATWabmqHLf2QzxKqUv2EmU6kitzpb15wcV6lqbUmo81bXOHuuGzxvU2VdyxXYAg3_k9fvrAxVuO2wCSj6iyLU1l-RFVkc)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNqVVNFKG0EU_ZVhnvUHljYgtQ-lpCD1cV_W7JguZDd1M_tQREjBFhMrWXUFibE2kEgQGpNS0rjW9mf2TpKn_kJnczfGrWti52lm7rnnnjvnMps0k9cZVWiBbTjMyrBlQ8vamqlaRC5u8Bwjy6tpIrxecN0btLsY0Byetxxzjdl4Tq8splIxnEJGxYNR8X3QLwb9C7HbFLXSkzU7Bd6lvCJpVihoWfZCJ4PqduDvyiRkipFIUnHUEXtt8A-VcQi6lVG1Nqy3xPEluOfw9RhqrZA3a-jkaZx22O7AzdGfn2eR5hwniMcA3obrtsbi_S5E6RfsdLDsNCOuUmalVxSy9OwlIpil42YWMVKK2gVWSO4efG_gtW7f8s47YST-QNDYHrgfEQVXveD3qexXeFdQbiXomXAjAH58Q3yM_r7siH1cSey4UD6b6xu4-_94VzmAD005BuK0PvEu6PtJ7k2twyRodIffm4-zDk6-kJBathb5XvyUzDJL_MlncV2Fm0Mo7cneoVx_XHHEhqXHBHNG59X_zU6iZTOaEBU38Bt35c-XjjkPWzwderpATWabmqHLf2QzxKqUv2EmU6kitzpb15wcV6lqbUmo81bXOHuuGzxvU2VdyxXYAg3_k9fvrAxVuO2wCSj6iyLU1l-RFVkc)

</details>

> 正如你注意到的，DTM 服务器没有参与收件箱的工作。🤭

## 安装和使用

请阅读 https://github.com/EasyAbp/Abp.EventBus.Boxes.Dtm/tree/main#installation.

## 后记

这样的一个 DTM 事件箱实现并不是完美的解决方案。这里最大的成本是，你需要额外关心 DTM 服务器的可用性。但是如果你遇到了多租户多数据库的场景，它就是现在最佳的选择。
