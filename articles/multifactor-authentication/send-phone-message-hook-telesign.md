---
description: Configuring a Custom SMS Provider for MFA using TeleSign
topics:
  - mfa
  - sms
  - custom-sms-provider 
contentType:
  - how-to
useCase:
  - customize-mfa
---
# Configuring a Custom SMS Provider for MFA using TeleSign

TeleSign provides two different APIs for sending SMS:

* The [TeleSign SMS](https://www.telesign.com/products/sms-api), with which you can build and manage SMS communications and security verification processes.
* The [TeleSign SMS Verify](https://www.telesign.com/products/sms-verify), which helps you manage the SMS verification process and is available in the Enterprise plan. 

Either the SMS API or the SMS Verify API may be used alongside Auth0 for MFA.

## Prerequisites

Before you begin this tutorial, please:

* Log in to your TeleSign portal (either the [TeleSign Enterprise Portal](https://teleportal.telesign.com) or the [TeleSign Standard Portal](https://portal.telesign.com/)).
* Capture the Customer ID and API Keys from your TeleSign account.

## 1. Create a Send Phone Message hook 

You will need to create a [Send Phone Message](/hooks/extensibility-points/send-phone-message) hook, which will hold the code and secrets of your custom implementation.

::: note
You can only have **one** Send Phone Message Hook active at a time.
:::

## 2. Configure Hook secrets

Add a [Hook Secret](/hooks/secrets/create) with keys `TELESIGN_CUSTOMER_ID` and `TELESIGN_API_KEY` for the Customer ID and API Keys from your TeleSign account.

## 3. Implement the Hook 

[Edit](/hooks/update) the Send Phone Message hook code to match the relevant example below.

### SMS API example

If you are using the **SMS API** the code should be:

```js
/**
@param {string} recipient - phone number
@param {string} text - message body
@param {object} context - additional authorization context
@param {string} context.factor_type - 'first' or 'second'
@param {string} context.message_type - 'sms' or 'voice'
@param {string} context.action - 'enrollment' or 'authentication'
@param {string} context.language - language used by login flow
@param {string} context.code - one time password
@param {string} context.ip - ip address
@param {string} context.user_agent - user agent making the authentication request
@param {string} context.client_id - to send different messages depending on the client id
@param {string} context.name - to include it in the SMS message
@param {object} context.client_metadata - metadata from client
@param {object} context.user - To customize messages for the user
@param {function} cb - function (error, response)
*/
module.exports = function (recipient, text, context, cb) {

  const axios = require('axios').default;
  const querystring = require('querystring');

  const customerId = context.webtask.secrets.TELESIGN_CUSTOMER_ID;
  const restApiKey = context.webtask.secrets.TELESIGN_REST_API_KEY;

  const instance = axios.create({
    // If you are using the standard TeleSign plan the URL should be https://rest-api.telesign.com/
    baseURL: "https://rest-ww.telesign.com",
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
  });

  instance({
    method: 'post',
    auth: {
      username: customerId,
      password: restApiKey
    },
    url: '/v1/messaging',
    data: querystring.stringify({
      phone_number: recipient,
      message_type: 'ARN',
      message: text
    })
  })
  .then((response) => {
    cb(null, {});

  })
  .catch((error) => {
    cb(error);
  });
}
```

### SMS Verify API example

For the SMS Verify API, the code should be:

```js
module.exports = function(recipient, text, context, cb) {
  
  const axios = require('axios').default;
  const querystring = require('querystring');

  const customerId =  context.webtask.secrets.TELESIGN_CUSTOMER_ID;
  const restApiKey = context.webtask.secrets.TELESIGN_API_KEY;

  const instance = axios.create({
    baseURL: "https://rest-ww.telesign.com",
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
  });

  instance({
    method: 'post',
    auth: {
      username: customerId,
      password: restApiKey
    },
    url: '/v1/verify/sms',
    data: querystring.stringify({
      phone_number: recipient,
      template: text
    })
  })
  .then((response) => {
      cb(null, {});
  })
  .catch((error) => {
      cb(error);
  });
}
```

## 4. Test your hook implementation

Click the **Run** icon on the top right to test the hook. Edit the parameters to specify the phone number to receive the SMS and click the **Run** button.

## 5. Test the MFA flow

Trigger an MFA flow and double check that everything works as intended. If you do not receive the SMS, please take a look at the [Hook Logs](/hooks/view-logs).

## Troubleshooting

If you do not receive the SMS, please look at the logs for clues and make sure that:

- The Hook is active and the SMS configuration is set to use 'Custom'.
- You have configured the Hook Secrets as per Step 2.
- Those secrets are the same ones provided in the TeleSign portal.
- Your phone number is formatted using the [E.164 format](https://en.wikipedia.org/wiki/E.164).
