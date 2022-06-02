# Introduce DTM for Multi-Tenant Multi-Database Scene

This article shares a new way to solve [issue #10036](https://github.com/abpframework/abp/issues/10036) using the DTM's 2-phase messages pattern.

The module to be used today is EasyAbp's [Abp.EventBus.Boxes.Dtm](https://github.com/EasyAbp/Abp.EventBus.Boxes.Dtm).

## Introduction of the DTM Event Boxes Module

This implementation uses DTM's [2-phase messages](https://en.dtm.pub/practice/msg.html) to support ABP event boxes in the [multi-tenant & multi-database scene](https://github.com/abpframework/abp/issues/10036).

You should see the [DTM docs](https://en.dtm.pub/guide/start.html), which helps to understand this module.

## Differences From the ABP's Default Event Boxes

|                                                     | DTM 2-phase Message Boxes |                 ABP 5.0+ Default Boxes                 |
| :-------------------------------------------------: | :-----------------------: | :----------------------------------------------------: |
|                     Speediness                      |    :heavy_check_mark:     |                          :x:                           |
|                 Less data transfer                  |            :x:            |                   :heavy_check_mark:                   |
|   Be guaranteed to publish<br>(transactional UOW)   |    :heavy_check_mark:     |                   :heavy_check_mark:                   |
| Be guaranteed to publish<br>(non-transactional UOW) |            :x:            | :heavy_check_mark:<br>(consumers idempotency required) |
|          No consumers idempotency required          |    :heavy_check_mark:     |                   :heavy_check_mark:                   |
|            Multi-tenant-database support            |    :heavy_check_mark:     |                          :x:                           |
|        No additional external infrastructure        |            :x:            |                   :heavy_check_mark:                   |
|                 Dashboard and Alarm                 |    :heavy_check_mark:     |                          :x:                           |

## How Does the DTM Outbox Work?

You are publishing events using the ABP event outbox:
```csharp
await _distributedEventBus.PublishAsync(eto1, useOutbox: true);
await _distributedEventBus.PublishAsync(eto2, useOutbox: true);  // The useOutbox is true by default.
```
The DTM outbox collects them temporarily. Let's see what it will do when you complete the current unit of work:
```CSharp
// Code snippet for UnitOfWork.cs
protected override async Task CommitTransactionsAsync()
{
    // Step 1: inserting a record to the DTM barrier table within the current DB transaction,
    //         and then it sends a "prepare" request to the DTM server.
    await DtmMessageManager.InsertBarriersAndPrepareAsync(EventBag);

    // Step 2: committing the current DB transaction.
    await base.CommitTransactionsAsync();

    // Step 3: sending a "submit" request to the DTM server.
    OnCompleted(async () => await DtmMessageManager.SubmitAsync(EventBag));
}
```
Now, the DTM server has received a "submit" request. It invokes the app's `PublishEvents` service with the events' data, and the latter will publish the events to the MQ provider immediately.

[![](https://mermaid.ink/img/pako:eNqFk89uwjAMxl_Fypm-QA5IaOzAAW2IcevFbVyIlqRd_rAhxLsvbVooUG09NfHPX7449pmVtSDGmaOvQKakpcS9RZ0biJ-XXhEsP9bwFnxR_6RdDL42QRdk03rnyGbz-aJpOLyo2hGgAelcoBSPgRheoscCHUXmgGZPiQDn0d9z19M4LISAVYvtGhExEYX7jW2bloQE0JGMd0nkln535spEkx6wixdorRzc3yfExZbskSzvAo2lBi3dyBTMHnyOUDGh2lXmmXmqS6219OAPBN6icVh6WZter6eya6ET7EJZEokHyZG1ae7PS7tQxJT_7ryCb6kUVNJId-hMJ7_P7zCuQNKesh2ptpF4el8ou0aasN276TUX3ZmQwXso1GBk3A-pIusNnyBAak1Cxk5Sp0SvN1e3Az5tdVyz3oOoDU3YHJNrtJ-ALmk6VwXFZkyT1ShFnMFzm56zaFBTznj8FVRhUD5nublENHRj8Cqkry3jFSpHM9aO4_ZkSsa9DTRA_Rz31OUXaXxJzw)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNqFk89uwjAMxl_Fypm-QA5IaOzAAW2IcevFbVyIlqRd_rAhxLsvbVooUG09NfHPX7449pmVtSDGmaOvQKakpcS9RZ0biJ-XXhEsP9bwFnxR_6RdDL42QRdk03rnyGbz-aJpOLyo2hGgAelcoBSPgRheoscCHUXmgGZPiQDn0d9z19M4LISAVYvtGhExEYX7jW2bloQE0JGMd0nkln535spEkx6wixdorRzc3yfExZbskSzvAo2lBi3dyBTMHnyOUDGh2lXmmXmqS6219OAPBN6icVh6WZter6eya6ET7EJZEokHyZG1ae7PS7tQxJT_7ryCb6kUVNJId-hMJ7_P7zCuQNKesh2ptpF4el8ou0aasN276TUX3ZmQwXso1GBk3A-pIusNnyBAak1Cxk5Sp0SvN1e3Az5tdVyz3oOoDU3YHJNrtJ-ALmk6VwXFZkyT1ShFnMFzm56zaFBTznj8FVRhUD5nublENHRj8Cqkry3jFSpHM9aO4_ZkSsa9DTRA_Rz31OUXaXxJzw)

<details>
<summary>See the more detailed sequence diagram</summary>

[![](https://mermaid.ink/img/pako:eNqtVU1v2zAM_SuETy2Q5FwYQ4ps6TYDC7YiLXrJhbaYRKgsefpoFhT976O_YidxgK5YTpH5SD4-ktJrlBlBURw5-h1IZzSXuLGYrzTwz0uvCOYPC_gZfGr-wBietuhBrsFv2fAZvEXtMPPSaMhMnkvvpd6AdOCU2d3WYTB4o0Oekq3Pj47seDqdFUUMX5RxBKjZxQWq7Wxg8xw9puiIMVvUG6oR4Dz6Y9yBXgwzISApYY-FYJjgwM2HZelWBxJAL6S9q4N07kc5E80kPWBlT9Fa2bI_duDDkuwL2bgyFJYKtNQha-P4hGcPKgaiVsqcY850qQSvWtHrw1nuY5ZPyC5rY7k0ZbhVXuY0mUwGvCoW32k_AqYAexNgS5ZuIYEdak5rQA6I9Cm106ud9NuKl8Oc4FsyB3TVeRX1VYrAlmPnPDhuyHUzLcr3eJSjtEbnW_WHdDj0qqFQldeF6Bxbl3FbXePpQpYRiVbnNseJcpfBZ3EfemtxxeQLuLlu9gO-zpIfd_NSJsHjzCqKUCiZ8XjCRooRWKMUz2iK2XMvw6V2JiwdTeChXMucOOvpONQCSg5ZpmRhnJdKcQtMRs7xsk5ggfa5bFCZuUtLiheTqxnqwD8VfEHd_9rBD4l6uce1YB-Rv6Rw2oGhiTlbtFmNHcOvkCrp6vXpX1Sdbov7eAAFMs9JSC5Z7TuPxf1BqtblfdPe8BFG0zt0aEeoiu3cOqhmiDRniUZRTjZHKfiheS0Nq4h553wBxPxX0BqD8qtopd8YGqqr-05Ib2wUr5HHcBSVT8hyr7Mo9jZQC2oeqwb19he08zVf)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNqtVU1v2zAM_SuETy2Q5FwYQ4ps6TYDC7YiLXrJhbaYRKgsefpoFhT976O_YidxgK5YTpH5SD4-ktJrlBlBURw5-h1IZzSXuLGYrzTwz0uvCOYPC_gZfGr-wBietuhBrsFv2fAZvEXtMPPSaMhMnkvvpd6AdOCU2d3WYTB4o0Oekq3Pj47seDqdFUUMX5RxBKjZxQWq7Wxg8xw9puiIMVvUG6oR4Dz6Y9yBXgwzISApYY-FYJjgwM2HZelWBxJAL6S9q4N07kc5E80kPWBlT9Fa2bI_duDDkuwL2bgyFJYKtNQha-P4hGcPKgaiVsqcY850qQSvWtHrw1nuY5ZPyC5rY7k0ZbhVXuY0mUwGvCoW32k_AqYAexNgS5ZuIYEdak5rQA6I9Cm106ud9NuKl8Oc4FsyB3TVeRX1VYrAlmPnPDhuyHUzLcr3eJSjtEbnW_WHdDj0qqFQldeF6Bxbl3FbXePpQpYRiVbnNseJcpfBZ3EfemtxxeQLuLlu9gO-zpIfd_NSJsHjzCqKUCiZ8XjCRooRWKMUz2iK2XMvw6V2JiwdTeChXMucOOvpONQCSg5ZpmRhnJdKcQtMRs7xsk5ggfa5bFCZuUtLiheTqxnqwD8VfEHd_9rBD4l6uce1YB-Rv6Rw2oGhiTlbtFmNHcOvkCrp6vXpX1Sdbov7eAAFMs9JSC5Z7TuPxf1BqtblfdPe8BFG0zt0aEeoiu3cOqhmiDRniUZRTjZHKfiheS0Nq4h553wBxPxX0BqD8qtopd8YGqqr-05Ib2wUr5HHcBSVT8hyr7Mo9jZQC2oeqwb19he08zVf)

[![](https://mermaid.ink/img/pako:eNp1VMtu2zAQ_JUFTy1g6wOEwoFbp62ABjk4QS66rMW1TUQkVXKV1Ajy711Sit_RSdTODmdnSL2pxmtSpYr0tyfX0MLgJqCtHcjDhluCxcMd3Pe88v9gCk9bZDBr4K0UvgMHdBEbNt5B4601zKRhjaYlfTOQYM_e9XZFYVg_RgrT2WzedSX8aH0kQAcmxp6GuhSkvEDGFUYSzBbdhgYEREY-xe3FlTDXGqoEe-w0Jh3oxg_L1DYQaaAXchwHkkP7yZ6VE5EMmOsrDMF8qD9tkMWSwguFMhe6QB0GOiCH4vRM5xFUX2HNzlxiLnzJducgjlIY-UbUdG_0AP45r_7cLiYQfCsByWTN84FcsCmbEu47CpgjHYK8mOd08icU5rUPYlfr3UZOjaWiKK50ZSm_aTcBGQt2voctBbqBCl7RySgezBXjv63C7Mur4W2eNaIl-FUtAGNe1-rYeQUhHeTIECXkr9ed26c7bpDFH4R-YuHYFfumIdLHqZz5cR34qX2VSKUCHtLFsiRBnkcqR38MIjkhUiObtpWRfUMxGrcp4A7DczIk5ZpCVRNlKVg0Wq72WxJQK2G1YlApr5rW2Ldcq9q9C7TP1-VWG_ZBlWtsI01UurbLnWtUyaGnD9D4exhR7_8BefBo_w)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNp1VMtu2zAQ_JUFTy1g6wOEwoFbp62ABjk4QS66rMW1TUQkVXKV1Ajy711Sit_RSdTODmdnSL2pxmtSpYr0tyfX0MLgJqCtHcjDhluCxcMd3Pe88v9gCk9bZDBr4K0UvgMHdBEbNt5B4601zKRhjaYlfTOQYM_e9XZFYVg_RgrT2WzedSX8aH0kQAcmxp6GuhSkvEDGFUYSzBbdhgYEREY-xe3FlTDXGqoEe-w0Jh3oxg_L1DYQaaAXchwHkkP7yZ6VE5EMmOsrDMF8qD9tkMWSwguFMhe6QB0GOiCH4vRM5xFUX2HNzlxiLnzJducgjlIY-UbUdG_0AP45r_7cLiYQfCsByWTN84FcsCmbEu47CpgjHYK8mOd08icU5rUPYlfr3UZOjaWiKK50ZSm_aTcBGQt2voctBbqBCl7RySgezBXjv63C7Mur4W2eNaIl-FUtAGNe1-rYeQUhHeTIECXkr9ed26c7bpDFH4R-YuHYFfumIdLHqZz5cR34qX2VSKUCHtLFsiRBnkcqR38MIjkhUiObtpWRfUMxGrcp4A7DczIk5ZpCVRNlKVg0Wq72WxJQK2G1YlApr5rW2Ldcq9q9C7TP1-VWG_ZBlWtsI01UurbLnWtUyaGnD9D4exhR7_8BefBo_w)
   
</details>

> If you are still confused about how it is guaranteed to publish, see DTM's [2-phase messages doc](https://en.dtm.pub/practice/msg.html) for more information.

## How Does the DTM Inbox Work?

Unlike ABP's default implementation, the DTM inbox gets an event from MQ and handles it at once. After the handlers finish their work, the inbox inserts a barrier within the current DB transaction. Finally, it commits the transaction and returns ACK to MQ.

All the incoming events have a unique MessageId. Events with the same MessageId only are handled once since we cannot insert a barrier with a duplicate gid (MessageId).

[![](https://mermaid.ink/img/pako:eNp9UstuwjAQ_JWVz_ADUUtFAakI5YDaYy6beAmW4jW115QK8e91HgWh0vhke2dmZ0d7VpXTpDIV6DMSV7Q0WHu0BUM6YqQhWH7ksObSnfpPjOI42pJ8_86309nsislgSY05kgdkoCOxPJV-9mVkDwiRTeoCOYWANa11L3DltjooWGKgDFYnEyRxSvTeJLlOojYanm_8l0FgIE3vjbwOTHYCOxf5Qbv54XAzLHvqHfe4VLsz9IasUxpXVBpQQ8AjQbVHrin8NdPJL_pqB9U36Xuvg3iIVUWkaTSZNQfybTRteSye0XQe2HrcbuGsNdLNLR45YCXG8bh2T_l3moTOtxnMFxs1UZa8RaPTDp5bXKFSJ0uFytJV0w5jI4Uq-JKg8aBRaKWNOK-yHTaBJqrdx_dvrlQmPtIvaNjjAXX5AUn_9To)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNp9UstuwjAQ_JWVz_ADUUtFAakI5YDaYy6beAmW4jW115QK8e91HgWh0vhke2dmZ0d7VpXTpDIV6DMSV7Q0WHu0BUM6YqQhWH7ksObSnfpPjOI42pJ8_86309nsislgSY05kgdkoCOxPJV-9mVkDwiRTeoCOYWANa11L3DltjooWGKgDFYnEyRxSvTeJLlOojYanm_8l0FgIE3vjbwOTHYCOxf5Qbv54XAzLHvqHfe4VLsz9IasUxpXVBpQQ8AjQbVHrin8NdPJL_pqB9U36Xuvg3iIVUWkaTSZNQfybTRteSye0XQe2HrcbuGsNdLNLR45YCXG8bh2T_l3moTOtxnMFxs1UZa8RaPTDp5bXKFSJ0uFytJV0w5jI4Uq-JKg8aBRaKWNOK-yHTaBJqrdx_dvrlQmPtIvaNjjAXX5AUn_9To)


<details>
<summary>See the more detailed sequence diagram</summary>

[![](https://mermaid.ink/img/pako:eNqNlMFO6zAQRX_F8hp-IOIVQcsTFSoSgmU2E3uaWjjjYo9LEeLfceKQtFBaskriM3du5tp5l8pplIUM-BKRFM4M1B6akkS62LBFMXtaiDlVbptfQmRHsanQ5-fFw_lkMjCFmKE1G_QCSOAGiS8qP3k1vBIgIpnURSwwBKhxrrPAUNvqAEMFAQtxszWBU00F3psk10nURot_Y_1l78hyxsNAfwMz1zXrG5zvm77u65Yu0i49WEv44qEQV9O7vIpf2AlBcrwruvutV-v1OC1eYR5X5tLa3jRugXSKYqDSdLUIsEGhVkA1hp9mOvlpXu1QPUrve-3FQ1QKUePRWOYU0Le5tMvHshmjmWFQ3lSGasG45ZNZ_Adjs6SOa2sUMLbih1LZdeadtcmReu6mxB4ogGLj6GTDp5HtVFB3Oke2wf2f98GBAA7bn7qmMXzY-2_aueTX3MY9K89kg74Bo9NRf2-5UqZODZaySLcalxAtl7Kkj4TGtU4zv9GGnZfFEmzAM9ke-8c3UrJgH_EL6n8XPfXxCdjtY4c)](https://mermaid-js.github.io/mermaid-live-editor/edit#pako:eNqNlMFO6zAQRX_F8hp-IOIVQcsTFSoSgmU2E3uaWjjjYo9LEeLfceKQtFBaskriM3du5tp5l8pplIUM-BKRFM4M1B6akkS62LBFMXtaiDlVbptfQmRHsanQ5-fFw_lkMjCFmKE1G_QCSOAGiS8qP3k1vBIgIpnURSwwBKhxrrPAUNvqAEMFAQtxszWBU00F3psk10nURot_Y_1l78hyxsNAfwMz1zXrG5zvm77u65Yu0i49WEv44qEQV9O7vIpf2AlBcrwruvutV-v1OC1eYR5X5tLa3jRugXSKYqDSdLUIsEGhVkA1hp9mOvlpXu1QPUrve-3FQ1QKUePRWOYU0Le5tMvHshmjmWFQ3lSGasG45ZNZ_Adjs6SOa2sUMLbih1LZdeadtcmReu6mxB4ogGLj6GTDp5HtVFB3Oke2wf2f98GBAA7bn7qmMXzY-2_aueTX3MY9K89kg74Bo9NRf2-5UqZODZaySLcalxAtl7Kkj4TGtU4zv9GGnZfFEmzAM9ke-8c3UrJgH_EL6n8XPfXxCdjtY4c)

</details>

> As you may have noticed, the inbox has nothing to do with the DTM Server.🤭

## Installation and Usage

Please see https://github.com/EasyAbp/Abp.EventBus.Boxes.Dtm/tree/main#installation.

## Postscript

This DTM event-boxes implementation is not a perfect solution. The main cost is that you need to care about the availability of the DTM Server. But if you run into the multi-tenant multi-database scene, it should be the best choice for now.
