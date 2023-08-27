---
title: "A Blueprint to Start With Stripe Subscriptions (using .NET)"
slug: "a-blueprint-to-start-with-stripe-subscriptions"
date: "2020-08-14"
summary: ""
references: 
---

#software-development #dotnet

When you end up on Stripe's landing page their UI leaves an impression. However, this is just the topping on the ice given they have done a tremendous job to offer services with which they facilitate an impressive number of payment scenarios. However impressive this is, it can be also be quite overwhelming for someone implementing a Stripe integration for the first time. This post provides a rough blueprint about how you can implement subscriptions within your application using Stripe, with code examples provided in C#.


![](/uploads/stripe_42cff9acf6.jpg)


## Goals

Within this post I'm going to cover the implementation of a scenario where a subscription needs to be maintained. We'll focus on the complete lifetime of a subscription;

1. Create (or update) a subscription
2. Update payment information
3. Cancel a subscription

Personally I try to keep as much data as possible within Stripe, for the simple reason that it is more difficult to keep all data synchronized, than that it is to retrieve data. As such I wont focus too much on data access, but rather on the logic and code paths required to handle subscriptions themselves.

### Requirements

To make the implementation easier I have defined the following constraints;

1. Payment methods to be supported must be card payments, iDeal (mainly used in the Netherlands), SEPA Direct Debit, and manual payment.
2. Upgrading plans will take effect immediately, prorating charges.
3. Downgrading plans will take effect at the end of the billing period.
4. Users are entitled to a 28 day trial period. The trial period will start as soon as they register themselves with the application.

With these constraints I'm able to balance convenience *(point 2)*, accessibility *(points 1 and 4)* and income protection *(point 3)*.


## Practical implications

Personally I like to do some things different than usual. This is the case with regard to the manual payment option, as well with directly starting the trial period upon registration.

**Immediate trial upon registration**

I cannot really count the number of times I have signed up to an application, only to discover that I had to manually activate the trial of any premium features. The way I think is like "well, I'll keep it for later", and in the end I have never even seen the premium features.

Instead the way I'm doing it is to have users try all premium goodies, and automatically fall back to a free plan if they do not choose to subscribe.

**Manual payments**

When a customer decides they want to use my application, I want all roadblocks to be gone before they eventually change their mind. As such I have implemented the "pay later" option. Hitting that button will immediately activate the subscription, and send an invoice to their mail address.

Even if the user does not pay, the loss is usually negligible for a SaaS compared to the grand scheme of things going on running such business. Conversion to automatic card payments is something to worry about later.

Even if conversion to an automatic payment method is not possible, this still allows people without credit card to become a subscriber.

> Note that the SEPA Direct Debit payment method is unavailable when you're just starting out collecting payments. Manual payment is a reasonable alternative in this situation.


## Implementation overview

To be flexible with regard to possible use-cases, we'll implement methods with which we can achieve one of the following three actions;

- One to change the default payment method
- One to set a subscription
- One to cancel a subscription

Out of these three actions, the one to create a subscription is the most complex one due to various edge cases which need to be considered;

- An account needs to be available with Stripe
- A payment method (if available) needs to be connected with said account
- A proper subscription plan needs to be selected
- If a previous subscription was already available, it would need to be updated
- Local information (if relevant - *it probably is*) about the subscription needs to be updated
- Additional validation for a payment method (such as 3DS2) would need to communicated back to the front end for a good user experience

All combined these requirements lead to a fairly complicated process. Although it would be easier to adhere to a strict flow to subscribe such as collecting payment information, selecting a plan, and then creating the subscription, payment processing is the most essential part of a viable product, and it's the one about which I want to be sure it just works, regardless of how it's executed.

Additionally, because Stripe is so well implemented, I'm trying to offload most decision making about the validity of an action to Stripe. They are perfectly able to tell me whether the thing I'm trying to achieve makes sense.

Although Stripe has a lot of documentation, I was not able to find a clear implementation path for my use-case, so that's what I'll describe here instead;

### Creating or updating a subscription
The goal of this method is to create a new subscription or to update an existing one. Not to cancel an existing subscription because that's what I call a destructive action. I accept 3 parameters for this method:

1. The subject (user) for which I create this subscription
2. The plan to sign the user up to.
3. An tokenized payment method (through Stripe Elements) which can be added to the subscription. This is optional.

An alternative to providing the tokenized payment method on this method would be to require the user to register a payment method for their account first, before setting up the subscription itself. To me this seems counter intuitive. It's not like I walk into a store, pay, and then choose what I would be paying for in the first place.

Within this method I'll do a few checks first;

1. Is the user registered as being a customer with Stripe? If not, create a new customer.
2. Does this customer already have any subscriptions?  
    1. If there is a subscription with the status `incomplete` and the payment intent status of `requires_action`, return the client secret so that the subscription can be completed. Doing so has the benefit that calling this method multiple times does not necessarily result in multiple subscriptions being created.  
    2. If there is a subscription with the state of `trial` and no registered payment method, that's great. This is where I'm converting someone from a trial to a paid plan. 
    3. If there is a subscription with the state of `active` I'm up or downgrading someone. 
3. If there is no subscription yet I do have to create a new one. In my situation this would be the case when a trial has expired, the subscription been cancelled, but the user has been converted later on.


Given I have lifted this logic out of my own codebase, I'll show the individual parts with make up the full method; so you don't have to decipher 150 lines of code;

#### Data model

To clearly understand the code below it's important to know what data is used, and what is returned;

- `PaymentResult`; communicates the result of the subscription mutations
- `PaymentStatus`; pretty much self descriptive?
- `User`; the class I'm using to store user information.

```csharp
public class PaymentResult
{
    /// <summary>
    /// The status of the payment.
    /// </summary>
    public PaymentStatus Status { get; set; }

    /// <summary>
    /// Client secret which should be used in case additional authorization is required on the client side.
    /// </summary>
    public string ClientSecret { get; set; }
}

public enum PaymentStatus
{
    Failed,
    Completed,
    RequiresAuthorization
}

public class User
{
    public Guid Id { get; set; }
    public string? Email { get; set; } = "";

    public string? CustomerId { get; set; } = "";
    public string? SubscriptionId { get; set; }
    public DateTime? SubscriptionStart { get; set; }
    public DateTime? SubscriptionEnd { get; set; }
    public DateTime? TrialPeriodEnd { get; set; }
    public string? SubscriptionStatus { get; set; }
}
```

#### Method signature

The method signature accepts three parameters;

- The current user
- The tokenized payment method
- The plan to sign up to
- Whether to prorate or not

More specifically does it look like this;

```csharp
public PaymentResult SetSubscription(
    ref User user,
    string token, 
    string plan, 
    bool prorate = false)
```


#### Checking user existence

First of all we'd check whether the user exists (with Stripe). It is important to note that we expect the `CustomerId` field to be set when a customer is created with Stripe.

```csharp
if (user == null) throw new ArgumentException(nameof(user));

if (string.IsNullOrWhiteSpace(user?.CustomerId))
{
    var customer = await new CustomerService().CreateAsync(new CustomerCreateOptions
    {
        Email = user.Email,
        Source = token
    });

    user.CustomerId = customer.Id;
}
```


#### Check for incomplete subscriptions

As to prevent one customer from creating multiple subscriptions when their subscription creation fails, we're checking whether there are any incomplete subscriptions which can yet be revived. We're specifically looking for subscriptions which require further payment verifications. If found, the client secret is returned so that the subscription can be completed.

```csharp
var subscriptions = await new SubscriptionService()
    .ListAsync(new SubscriptionListOptions
    {
        Customer = user?.CustomerId,
        Expand = new List<string> { "data.latest_invoice.payment_intent" }
    });

var incompleteSubscription = subscriptions
    .FirstOrDefault(q => q.Status == SubscriptionStatuses.Incomplete);

if (incompleteSubscription != null
    && incompleteSubscription.LatestInvoice.PaymentIntent.Status == "requires_action")
{
    return new PaymentResult
    {
        Status = Data.Enums.PaymentStatus.RequiresAuthorization,
        ClientSecret = incompleteSubscription.LatestInvoice.PaymentIntent.ClientSecret
    };
}
```


#### Creating a subscription

When the previous checks are passed, we get to the point where we can actually create the subscription. We're covering two scenarios in this part;

1. There is already a subscription available which we can change. This means we're dealing with an up or downgrade, or the user is currently in trial mode and wants to convert.
2. Any previous subscription has expired and we'd have to create a new one.

These two cases are dealt with within the if statements.

Additionally, when we're creating a new subscription we default to deferred payments, hopefully to improve conversions.

```csharp
var subscription = subscriptions
    .FirstOrDefault(q =>
        q.Status == SubscriptionStatuses.Trialing
        || q.Status == SubscriptionStatuses.Active);

if (subscription != null)
{
    var items = new List<SubscriptionItemOptions> {
        new SubscriptionItemOptions {
            Id = subscription.Items.Data[0].Id,
            Plan = plan
        }
    };

    SubscriptionUpdateOptions subscriptionUpdateOptions = null;

    if (subscriptions.Any(q => q.Status == SubscriptionStatuses.Trialing))
    {
        // Just convert into a paid subscription, keep the trial as is.
        subscriptionUpdateOptions = new SubscriptionUpdateOptions
        {
            Items = items,
            Expand = new List<string> { "latest_invoice.payment_intent" }
        };
    }
    else if (subscriptions.Any(q => q.Status == SubscriptionStatuses.Active))
    {
        subscriptionUpdateOptions = new SubscriptionUpdateOptions
        {
            Items = items,
            Prorate = prorate,
            Expand = new List<string> { "latest_invoice.payment_intent" }
        };

        if (!prorate)
        {
            subscriptionUpdateOptions.TrialEnd = subscriptions.First().CurrentPeriodEnd;
        }
    }

    subscription = await new SubscriptionService().UpdateAsync(subscriptions.First().Id, subscriptionUpdateOptions);

    user.SubscriptionStatus = subscription.Status;
    user.TrialPeriodEnd = subscription.TrialEnd;
    user.SubscriptionStart = subscription.StartDate;
    user.SubscriptionEnd = subscription.CurrentPeriodEnd;
}
else
{
    var collectionMethod = "send_invoice";

    if (!string.IsNullOrWhiteSpace(token))
    {
        collectionMethod = "charge_automatically";
    }
    else
    {
        var paymentMethod = await new PaymentMethodService()
            .ListAsync(new PaymentMethodListOptions
            {
                Customer = user.CustomerId,
                Type = "card"
            });

        if (paymentMethod.Any()) collectionMethod = "charge_automatically";
    }

    subscription = new SubscriptionService()
        .Create(new SubscriptionCreateOptions
        {
            Customer = user.CustomerId,
            Items = new List<SubscriptionItemOptions> {
                new SubscriptionItemOptions {
                    Plan = plan
                }
            },
            Expand = new List<string> { "latest_invoice.payment_intent" },
            CollectionMethod = collectionMethod,
            DaysUntilDue = (collectionMethod == "send_invoice") ? (long?)14 : null
        });

    user.SubscriptionStart = subscription.StartDate;
    user.SubscriptionEnd = subscription.CurrentPeriodEnd;
    user.TrialPeriodEnd = subscription.TrialEnd;
    user.SubscriptionId = subscription.Id;
    user.SubscriptionStatus = subscription.Status;
}
```


#### Final subscription checks

After we have created or updated an subscription we'll do some final checks to see what the next action should be. Note that the `requires_payment_method` status is unlikely to happen, though it'll probably(?!) trigger when updating a subscription with an expired card.

When the `requires_action` status occurs we're returning the client 

```csharp
if (subscription.Status == SubscriptionStatuses.Incomplete
    && subscription.LatestInvoice.PaymentIntent.Status == "requires_payment_method")
{
    return new PaymentResult
    {
        Status = Data.Enums.PaymentStatus.Failed
    };
}

if (subscription.Status == SubscriptionStatuses.Incomplete
    && subscription.LatestInvoice.PaymentIntent.Status == "requires_action")
{
    return new PaymentResult
    {
        Status = Data.Enums.PaymentStatus.RequiresAuthorization,
        ClientSecret = subscription.LatestInvoice.PaymentIntent.ClientSecret
    };
}

return new PaymentResult
{
    Status = Data.Enums.PaymentStatus.Completed
};
```

### Cancelling a subscription

This is by far the least fun method to implement for obvious reasons, but required if you ever want to have a change to welcome customers once again. The logic to cancel an existing subscription is incredibly simple; again assuming we're storing the subscription ID locally.

```csharp
public User CancelSubscription(ref User user) {
    if (user == null) throw new ArgumentException(nameof(user));

    var service = new SubscriptionService();

    await service.UpdateAsync(user.SubscriptionId, new SubscriptionUpdateOptions
    {
        CancelAtPeriodEnd = true
    });

    return user;
}
```

### Changing payment method

Given credit cards expire every now and then this one is quite necessary to keep money flowing in. Thankfully Stripe will help you a bit to collect new credit card information when one expires, or a payment fails, but it's still nice to have the possibility to change that information from your application.

In this method we do not assume the existence of the user with Stripe. If someone provides us with payment information we store it, regardless of whether we would have to create a new user with Stripe or not. Again, this is not really spectacular.

```csharp
public void ChangePaymentMethod(ref User user) {
    var customerService = new CustomerService();

    if (string.IsNullOrWhiteSpace(user.CustomerId))
    {
        var customer = await customerService.CreateAsync(
            new CustomerCreateOptions
            {
                Email = user.Email,
                Source = token
            });

        user.CustomerId = customer.Id;
    }
    else
    {
        await customerService.UpdateAsync(user.CustomerId, new CustomerUpdateOptions
        {
            Source = token
        });
    }
}
```

Sometimes it's possible to manage multiple payment methods in an app. In the spirit of developing a minimum viable product however I choose to store just a single tokenized payment method, for simplicity.


## Implementing webhooks

Although webhooks are not necessarily required to implement a Stripe integration, it gives me piece of mind to know that all events are sent to me. Whenever I do not catch one via the front channels, I'll catch it via a back-channel.

Webhook support is implemented using ASP.NET Core in a regular controller whose general structure looks like this;

```csharp
[Route("hooks/[controller]")]
public class Stripe : Controller
{
    private readonly IConfiguration _config;

    public Stripe(IConfiguration config)
    {
        _config = config;
    }

    [HttpPost]
    public async Task<ActionResult> Index()
    {
        var json = new StreamReader(HttpContext.Request.Body).ReadToEnd();

        try
        {
            // You can find your endpoint's secret in your webhook settings
            var stripeEvent = EventUtility.ConstructEvent(
                json,
                Request.Headers["Stripe-Signature"],
                _config["config:stripeEndpointSecret"]);

            switch (stripeEvent.Type)
            {
                case Events.ChargeSucceeded:
                    var charge = stripeEvent.Data.Object as Charge;
                    break;
                case Events.CustomerCreated:
                    var customer = stripeEvent.Data.Object as Customer;
                    break;
                case Events.InvoiceCreated:
                case Events.InvoiceFinalized:
                case Events.InvoicePaymentActionRequired:
                case Events.InvoicePaymentFailed:
                case Events.InvoicePaymentSucceeded:
                case Events.InvoiceSent:
                case Events.InvoiceUpcoming:
                    var invoice = stripeEvent.Data.Object as Invoice;
                    break;
                case Events.CustomerSubscriptionCreated:
                case Events.CustomerSubscriptionDeleted:
                case Events.CustomerSubscriptionTrialWillEnd:
                case Events.CustomerSubscriptionUpdated:
                    var subscription = stripeEvent.Data.Object as Subscription;
                    break;
            }

            return new StatusCodeResult(200);
        }
        catch (StripeException)
        {
            return BadRequest();
        }
    }
}
```

Given we do not necessarily need webhooks to keep data up to date you can use those at your leisure. Also, do not forget to register your endpoint with Stripe.

## Security implications

Definitely not least important are the security implications of this approach. The most obvious risk would be to be targeted with so called [card testing](https://stripe.com/docs/card-testing), where stolen credit cards are tested against unprotected endpoints. The logic described in this post on itself is not protected against card testing. You would have to make several adjustments yourself to reduce the risk. As described in the document linked earlier;

- Implement authorization on the endpoints
- Add rate limiting to the endpoints triggering this logic
- Use reCAPTCHA to block automated scripts
- Request 3DS2 verification for new accounts

## What's next?

In this post I have described some structural things when it comes to implementing Stripe subscriptions. This however is not the only way to do it, and the way the code is described in this specific post is scrappy, at best, even though it works.

In the future I hope to make a few improvements to the way I have implemented Stripe myself. One thing I would love to implement are tests which validate the workings of the integration. Thankfully Stripe has released a mock API ([stripe-mock](https://github.com/stripe/stripe-mock)) which can be integrated in a test suite.

Like always there are tradeoffs to be made, and now I got this working I'm happy to move on to something else. I'm pretty sure there are significant improvements to the code and approaches I described in this post. If you happen to know better ways of achieving something, please comment!
