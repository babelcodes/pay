# Stripe

- Part of [Pay](../README.md)

Table Of Content
- [Resources](#resources)
- [Dev](#dev)
- [Concepts](#concepts)
  - [(Web) Elements](#web-elements)
  - [Products and prices](#products-and-prices)
  - [Checkout session](#checkout-session)
  - [Accept a payment & PaymentIntent](#accept-a-payment--paymentintent)
  - [Result notification](#result-notification)
- [Customer (optional)](#customer-optional)
- [PaymentLinks (optional)](#payment-links-optional)
- [Webhooks](#webhooks)

## Resources

- [Create subscriptions with Checkout](https://stripe.com/docs/billing/subscriptions/checkout)
- [How subscriptions work](https://stripe.com/docs/billing/subscriptions/overview)
- [Create fixed-price subscriptions](https://stripe.com/docs/billing/subscriptions/fixed-price)

## Dev
```shell
$ stripe login
$ stripe listen --forward-to localhost:3000/webhook
```
- https://stripe.com/docs/cli/listen

## Concepts

### (Web) Elements

> Stripe Elements is a set of prebuilt UI components for building your web checkout flow.
- https://stripe.com/docs/payments/elements

It’s available as a feature of Stripe.js, our foundational JavaScript library
for building payment flows. Stripe.js tokenizes sensitive payment details
within an Element without ever having them touch your server.

Elements features include:
- Automatic input formatting as customers type
- Complete UI translations to match your customer’s preferred language
- Responsive design to fit seamlessly on any screen size
- Custom styling rules so you can match the look and feel of your site
- One-click checkout with Link


### Products and Prices

- [getting-started](https://stripe.com/docs/products-prices/getting-started)
  - [pricing-models#flat-rate](https://stripe.com/docs/products-prices/pricing-models#flat-rate)
  - [pricing-models#multicurrency](https://stripe.com/docs/products-prices/pricing-models#multicurrency)
  - [manage-prices](https://stripe.com/docs/products-prices/manage-prices)
- [Products API](https://stripe.com/docs/api/products)
  - [Prices API](https://stripe.com/docs/api/prices)

As we have a large catalog of (Korali) products and especially dynamically created
by the user / admin, we need to continuously (programmatically) import / synchronise
the (Korali) catalog into Stripe.
- We can create multiple products
- And multiple prices per product (eg monthly and yearly rates)
- Each product should have exactly one active price

Checklist:
- [ ] Create Stripe Product
- [ ] Store the Stripe Product ID into the (Korali) Product
- [ ] Create Stripe Prices
- [ ] Store each Stripe Price ID: it is needed to use the Stripe API to pay
- [ ] Confirm the creation ([api/products/list](https://stripe.com/docs/api/products/list))
- [ ] Synchronized (Korali) products catalog with Stripe (can use webhooks, [api/products/update](https://stripe.com/docs/api/products/update))
  - [ ] Get existing (active) Stripe price
  - [ ] If amount has changed, create new price (active)
  - [ ] Update previous active one as not active


### Checkout session
> A Checkout Session is the programmatic representation of what your customer sees
> when they’re redirected to the payment form.
- [Checkout Session](https://stripe.com/docs/payments/accept-a-payment?platform=web&ui=checkout#redirect-customers)
  - [Checkout Session for Subscriptions](https://stripe.com/docs/billing/subscriptions/build-subscriptions?ui=checkout#create-session)
- [api/checkout/sessions/create](https://stripe.com/docs/api/checkout/sessions/create)

Checkout Sessions expire 24 hours after creation.

You can configure it with options such as:
- [line items](https://stripe.com/docs/api/checkout/sessions/create#create_checkout_session-line_items) to charge
- currencies to use
- A `success_url`, a page on your website to redirect your customer after they complete the payment.
- A `cancel_url`, a page on your website to redirect your customer if they click on your logo in Checkout.

```javascript
// After creating a Checkout Session, 
// redirect your customer to the URL returned in the response.
app.post('/create-checkout-session', async (req, res) => {
  const session = await stripe.checkout.sessions.create({
    line_items: [
      {
        price_data: {
          currency: 'usd',
          product_data: {
            name: 'T-shirt',
          },
          unit_amount: 2000,
        },
        quantity: 1,
      },
    ],
    mode: 'payment',
    success_url: 'https://example.com/success',
    cancel_url: 'https://example.com/cancel',
  });

  res.redirect(303, session.url);
});
```

#### Success page
- https://stripe.com/docs/payments/accept-a-payment?platform=web&ui=checkout#success-page
```html
    <h1>Thanks for your order!</h1>
    <p>
      We appreciate your business!
      If you have any questions, please email
      <a href="mailto:orders@example.com">orders@example.com</a>.
    </p>
```


### Accept a payment & PaymentIntent

- [Accept a payment | Stripe Documentation](https://stripe.com/docs/payments/accept-a-payment?platform=web&ui=elements)
- [The Payment Intents API | Stripe Documentation](https://stripe.com/docs/payments/payment-intents)
- [api/payment_intents](https://stripe.com/docs/api/payment_intents)

> You can build a custom payment flow by embedding the Payment Element, which allows
> you to accept multiple payment methods using a single integration.

- Building an integration with the Payment Intents API involves two actions: creating and confirming a PaymentIntent.
- Each PaymentIntent typically correlates with a single shopping cart or customer session in your application.
- The PaymentIntent encapsulates details about the transaction, such as the supported payment methods,
  the amount to collect, and the desired currency.
- At least one payment method should be enabled
- Stripe uses a PaymentIntent object to represent your intent to collect payment from a customer,
  tracking charge attempts and payment state changes throughout the process.
- The payment methods shown to customers during the checkout process are also included on the PaymentIntent.
- Each payment method must support the currency passed in the PaymentIntent

Inputs:
- amount
- currency
- automatic_payment_methods

Outputs:
- client_secret

Workflow:
1. `{client_secret}` = `stripe.paymentIntents.create`
1. `stripe.elements({client_secret}).create('payment').mount('#myDomELement')`
1. `stripe.confirmPayment(...)`
1. `https://my.site.com/result?payment_intent=&payment_intent_client_secret` => `stripe.retrievePaymentIntent(clientSecret)`
1. `https://my.site.com/webhook`

See the following section about Webhooks and follow those events:
- `payment_intent.succeeded`
- `payment_intent.processing`
- `payment_intent.failed`

### Result notification & Webhooks
> After you integrate Stripe Checkout or create a Stripe Payment Link to take your
> customers to a payment form, you need notification that you can fulfill their
> order after they pay.
- https://stripe.com/docs/payments/checkout/fulfill-orders
- https://stripe.com/docs/webhooks/quickstart

Checklist:
- [ ] Receive an event notification when a customer pays you.
- [ ] Handle the event.
- [ ] Use Stripe CLI to quickly test your new event handler.
- [ ] Optionally, handle additional payment methods.
- [ ] Turn on your event handler in production (register it).
  - https://stripe.com/docs/webhooks/go-live


## Customer (optional)
> This object represents a customer of your business.
- https://stripe.com/docs/api/customers

It lets you create recurring charges and track payments that belong to the same customer.

### Save payment method details
- https://stripe.com/docs/payments/accept-a-payment?platform=web&ui=checkout#save-payment-method-details


## Payment links (optional)
> Accept payments without building a digital storefront.
- https://stripe.com/docs/payments/payment-links

To accept a payment without building additional standalone websites or applications,
use Payment Links and share the link as many times as you want on social media,
in emails, or on your website.



## Webhooks
> A webhook is an endpoint on your server that receives requests from Stripe, 
> notifying you about events that happen on your account such as a customer 
> disputing a charge or a successful recurring payment.
- https://stripe.com/docs/webhooks/quickstart
- Stripe sends the event data in the request body, structured as an [Event object](https://stripe.com/docs/api/events) with a `type`, `id`, and related Stripe resource nested under data
- You need a Endpoint Secret:
   - If you are testing with the CLI, find the secret by running `stripe listen` (see Dev section above)
   - If you are using an endpoint defined with the API or dashboard, look in your [webhook settings](https://dashboard.stripe.com/webhooks)
- You can use one webhook to handle several different event types at once, or set up individual endpoints for specific events.
- Send a successful 200 response to Stripe as quickly as possible because Stripe retries the event if a response isn’t sent within a reasonable time

```shell
$ npm start
$ stripe login
$ stripe listen --forward-to http://localhost:3000/api/v1/webhooks/stripe
$ stripe trigger payment_intent.succeeded
```
