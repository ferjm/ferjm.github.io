---
layout: post
title:  "Behind the scenes of the new Web Payments API from Mozilla"
date:   2013-04-09
categories: ["web", "payments", "api", "firefoxos", "telefonica", "mozilla"]
author: "Fernando Jiménez & The Telefonica Digital team"
---


When we started working on [Firefox OS](http://blog.digital.telefonica.com/?press-release=telefonica-outlines-launch-plans-for-firefox-os), we realized that one of the biggest challenges would be enabling web application developers to securely monetise their content, not only for Firefox OS but for the Open Web in general.

We were looking for the same seamless experience that developers find in existing mobile app stores but we wanted to avoid tying them to any store or proprietary solution, while also allowing them to use the same payment mechanisms in desktop and mobile. We also had the challenge of easing the user’s payment process by adding carrier billing capabilities, along with other payment methods like credit cards, to this solution.

The Mozilla Marketplace team already had an experimental feature based on [google.payments.inapp.buy](https://developers.google.com/commerce/wallet/digital/docs/index) to allow developers to add the capability for in-app payments to their apps. However, this solution was tied to the Firefox Marketplace and, as with Google’s solution, it involved the injection of a JS shim in the application code. So even if we liked this approach, we needed to modify it to fulfill our self-imposed requirements.

With [Andreas Gal’s](http://andreasgal.com/) help, we started writing the first draft of [navigator.mozPay()](https://docs.google.com/document/d/1NLKbHVPQXa9uvDBC3cfgOD7sIrtIxi0qDoXMQrxcCsI/edit) with the intention of proposing the first steps for an API that allows Open Web Apps to initiate payment requests from the user for digital goods with multiple payment providers and carrier billing options.

Once we had an agreement from both the Telefónica and Mozilla teams, we started implementing it for Firefox OS with valuable support from [Fabrice Desré](https://github.com/fabricedesre) and [Kumar McMillan](https://github.com/kumar303). Along with this work, the [BlueVia](http://bluevia.com/) and Firefox Marketplace teams also started working on a first implementation of the WebPaymentProvider [spec](https://wiki.mozilla.org/WebAPI/WebPaymentProvider), one after the other, and we started working with [Bango](http://bango.com/) to enable them as a payment partner.

**How it works**

The navigator.mozPay API allows the developer to create payment requests for different payment providers to charge a user for the purchase of a digital good. In order to create each payment request, the developer needs to create a [JSON Web Token (JWT)](http://openid.net/specs/draft-jones-json-web-token-07.html) for each payment provider signed with the Application Secret given by each corresponding provider. This token contains the details of the payment request, including the Application Key, which uniquely identifies the developer and the product being sold.

    {
      "iss": APPLICATION_KEY,
      "aud": "marketplace.firefox.com",
      ...
      "request": {
        "name": "Magical Unicorn",
        "pricePoint": 1,
        "postbackURL": "https://yourapp.com/postback",
        "chargebackURL": "https://yourapp.com/chargeback"
      }
    }

Applications using navigator.mozPay asynchronously receive responses about the completion of payment requests through Javascript callbacks and through POST notifications done by the payment provider to the URLs specified by the developer within the JWT request as postbackURL (for payments) and chargebackURL (for refunds) parameters. The application must only rely on the server side notification to determine the result of a purchase.

     var request = navigator.mozPay([signedJWT1, signedJWTn]);
     request.onsuccess = function() {
       console.log('Payment flow successfully completed' + evt.target.result);
       // The payment buy flow completed without errors.
       // This does NOT mean the payment was successful.
       waitForServerPostback();
     };
     request.onerror = function(evt) {
       console.log('navigator.mozPay() error: ' + evt.target.errorMsg.name);
     };

For more in-depth documentation read the [navigator.mozPay()](https://wiki.mozilla.org/WebAPI/WebPayment) spec and the Firefox Marketplace [guide](https://developer.mozilla.org/en-US/docs/Apps/Publishing/In-app_payments) to in-app payments.

The Firefox Marketplace itself uses navigator.mozPay() to request payments for application purchases and so it is a good proof of concept of the usage of the WebPayments API.

**What next?**

As Mozilla wrote on their [blog](https://hacks.mozilla.org/2013/04/introducing-navigator-mozpay-for-web-payments/) last week, navigator.mozPay() is an experimental API and it is just a first step towards an Open Web Standard for payments. We are already working on some improvements like removing the [server pre-requisite](https://groups.google.com/forum/?fromgroups=#!topic/mozilla.dev.webapps/0vUFHASyWB4) entirely or a better user experience for the [payment flow](https://groups.google.com/forum/?fromgroups=#!topic/mozilla.dev.b2g/4-FVBgM577I) on the payment provider’s side.

The plan is to keep working closely with Mozilla and others through the W3C to make a flexible API for payments part of the Open Web Standards.

After the launch of the first Firefox OS devices, we will be helping Mozilla to add support for navigator.mozPay() to Firefox desktop and Firefox for Android

<small>Originally published at [BlogThinkBig](http://en.blogthinkbig.com/2013/04/09/mozilla-web-payments-api/)</small>
