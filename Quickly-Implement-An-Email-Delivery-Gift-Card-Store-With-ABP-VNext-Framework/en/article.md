# Quickly implement an email delivery gift card store with ABP vNext framework

We will spend about 15 minutes to complete the development of a gift card store application with email delivery.

Yes, you read it right! We only need a short 15 minutes, thanks to the excellent modular design of [ABP framework](https://abp.io) and the [EShop](https://easyabp.io/modules/EShop) module group provided by EasyAbp organization.

## Step 1: Create an EasyMall Application

EasyMall is a startup template for the ABP framework to quickly create an application with pre-installed EShop modules.

We can easily use the ABP CLI to create it, just run the following commands:

```
dotnet tool install -g Volo.Abp.Cli
abp new MyStore -ts https://github.com/EasyAbp/EasyMall/releases/download/latest/latest.zip
```

If you want to install EShop to your existing application, please refer to this [document](https://easyabp.io/modules/EShop/#installation).

## Step 2: Define a Product Group

1. Create a new file `GiftCardProductGroup.cs` in the Domain layer with the following code:
    ```csharp
    [ProductGroupName("GiftCard")]
    public class GiftCardProductGroup
    {
    }
    ```

2. Open the `MyStoreDomainModule.cs` file and add the following code in the `ConfigureServices` method:

    ```csharp
    Configure<EShopProductsOptions>(options =>
    {
        options.Groups.Configure<GiftCardProductGroup>(group =>
        {
            group.DisplayName = "GiftCard";
        });
    });
    ```

## Step 3: Implement Automatic Email Delivery

1. Run the following command in the folder of the `MyStore.Domain.csproj` file to install the ABP Emailing module:

    ```
    abp add-package Volo.Abp.Emailing
    ```

2. Configure your email settings, please refer to the ABP official [document](https://docs.abp.io/en/abp/latest/Emailing#email-settings).

3. Create a new file `GiftCardOrderPaidEventHandler.cs` in the Domain layer with the following code:

    ```csharp
    public class GiftCardOrderPaidEventHandler: IDistributedEventHandler<OrderPaidEto>, ITransientDependency
    {
        private readonly IEmailSender _emailSender;
        private readonly IDistributedEventBus _distributedEventBus;
        private readonly IExternalUserLookupServiceProvider _externalUserLookupServiceProvider;

        public GiftCardOrderPaidEventHandler(
            IEmailSender emailSender,
            IDistributedEventBus distributedEventBus,
            IExternalUserLookupServiceProvider externalUserLookupServiceProvider)
        {
            _emailSender = emailSender;
            _distributedEventBus = distributedEventBus;
            _externalUserLookupServiceProvider = externalUserLookupServiceProvider;
        }
        
        public async Task HandleEventAsync(OrderPaidEto eventData)
        {
            var user = await _externalUserLookupServiceProvider.FindByIdAsync(eventData.Order.CustomerUserId);

            await _emailSender.SendAsync(user.Email, "Here is your gift card", "Card number: 123456, password: 123456");

            await _distributedEventBus.PublishAsync(new CompleteOrderEto
            {
                TenantId = eventData.Order.TenantId,
                OrderId = eventData.Order.Id
            });
        }
    }
    ```

## Step 4: Create a Gift Card Product and Place an Order

The EShop provides management pages to manage your products, but it has not yet provided relevant UI for users to place an order, for now, you can implement the UI yourself, read the [document](https://easyabp.io/modules/EShop/#basic-usage) to learn more.

## Postscript

As you can see, using modules to implement features can make your work easier.

This article only provides a simple example to show you how to use and customize the EShop module group, if you want to implement a real gift card store, you can consider using [EasyAbp.GiftCardManagement](https://easyabp.io/modules/GiftCardManagement) module, which can help you manage the gift card business and even be the provider of inventory.