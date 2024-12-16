# StayPowered AI API
**Version**: 1.01 

## Overview

The StayPowered API allows you to exchange messages with a StayPowered AI Agent via API calls. Your incoming request would be executed by the agent and the response will be delivered to a configured webhook.

## Authenticating

Use your API tenant key in a bearer token when sending requests:

```json
{
  "content-type": "application/json",
  "authorization": "Bearer <YOUR TENANT API KEY>"
}
```

## Sending a Message To an Agent
```
POST https://api.staypowered.ai/api/v1/message
```

Request Body:

```json
{
  "project": "<PROJECT SLUG>",
  "from_id": "<YOUR UNIQUE USER IDENTIFIER>",
  "message": "<MESSAGE TEXT>",
  "params": {
    "param1": "value1",
    "param2": "value2"
  }
}
```

For example:
```json
{
  "project": "minty-sweet",
  "from_id": "user@example.com",
  "message": "Hello there!",
  "params": {
    "address": "1500 5th Ave, New York, NY 10010",
    "unit_number": "521"
  }
}
```

**Notes & Limitations**:
1. Obtain your project slug from the StayPowered Console by looking up your project settings
2. Parameters are optional and need to be provided as strings. When provided, these parameters values can be injected into the agent instructions using instruction variables so that the AI can refer to them during runtime. 
3. The "from_id" property should be used to uniquely identify your user (i.e. email address, phone number or any other unique identifier). When used in consecutive requests it will cause the agent to continue the same conversation unless the conversation has expired (typically 4h) in which case the agent will create a new conversation.   
4. Message size should not exceed 10MB

Response Format:

Upon success (200 OK):
```json
{
  "success": true,
  "result": {
    "message_id": "1234567890"
  }
}
```
Use the returned message Id to identify the response for your request. 

Upon failure (500, 400, 401):
```json
{
  "success": false,
  "message": "<REASON FOR FAILURE>"
}
```

## Receiving Agent Responses

To receive respnses you will need to install a webhook via the StayPowered UI console (see below). 
StayPowered will call your webhook with the following payload:

```json
{
  "timstamp": "<TIMESTAMP IN ISO FORMAT>",
  "message_format": "<MESSAGE FORMAT - 'markdown' or 'json'>",
  "message": "<MESSAGE TEXT - either markdown or stringified JSON>",
  "reply_to_message_id": "<MESSAGE ID THIS REPLY REFERS TO>",
  "project": "<PROJECT SLUG>"
}
```

For Example:
```json
{
  "timstamp": "2024-12-16T07:31:16.718Z",
  "message_format": "markdown",
  "message": "I **LOVE** cupcakes!",
  "reply_to_message_id": "1234567890",
  "project": "test-project"
}
```

## Setting up a Webhook

From the StayPowered Console, define a new webhook to receive reply messages. **You must enable and verify the webhook in order to activate it**.
StayPowered will attempt to call your webhook up to 5 tries using the following strategy:
```
Next Attempt Time = First Attempt Timestamp + (Retry Attempt) * 5000 mSec 
```

### Authenticating Your Webhook

StayPowered implements a security mechanism for webhook authentication by employing a digital signature. This signature is crafted using a secret key provided during the webhook setup alongside the request body. The resulting data is encapsulated in the signature header property. The process to calculate the signature is as follows:

```HMAC-SHA256 = {Base64(HmacSHA256(body_bytes, CLIENT_SECRET))}```

To verify the request came from StayPowered, compute an HMAC digest using your secret key and the body and compare it to the signature portion (after the space) contained in the header. If they match, you can be sure the webhook was sent from StayPowered. Otherwise, ensure your code returns an unspecific error immediately without invoking additional logic.

This signature header property contains the SHA algorithm used to generate the signature, a space, and the signature itself. 

```signature: sha256 HMAC-SHA256```

e.g. signature: sha256 37d2725109df92747ffcee59833a1d1262d74b9703fa1234c789407218b4a4ef

**You must verify your webhook from the StayPowered Console so that it will be called.** 

### Samplle StayPowered Webhook Receiver

Use the StayPowered Console to obtain your unique webhook secret. 

### node.js

```javascript
var express = require('express')
const crypto = require('crypto');
var app = express()

// Set your runtime variables
let secret = '<YOUR WEBHOOK SECRET>';
let port = 8080;

// Signature verification
const verifyHmac = (
    receivedHmacHeader,
    receivedBody,
    secret
) => {
    const hmacParts = receivedHmacHeader.split(' ')
    const receivedHmac = hmacParts[1]
    const hash = crypto
        .createHmac('sha256', secret)
        .update(receivedBody)
        .digest('hex')
    return receivedHmac == hash
}

app.use(express.raw({ type: "*/*" }))

// Main webhook entry point
app.post('/webhook', function (req, res) {

    // Some initial basic verification
    if (!req.headers.signature) {
        console.log('Signature doesn\'t exist');
        return res.sendStatus(500);
    }
    else {
        console.log('Signature: ' + req.headers.signature);
    }

    // Verify source from signature
    if (!verifyHmac(req.headers.signature, req.body, secret)) {
        console.log('Signature verification failed');
        return res.sendStatus(500);
    }

    console.log("Signature verified");

    // Extract payload
    let payload = JSON.parse(req.body);
    console.log(JSON.stringify(payload, null, 4));

    // Parse the actual payload here
    // TODO

    res.send('Ok')
})

console.log(`Listening for events on ` + port)
app.listen(port);
```

### Python

```python
from flask import Flask
from flask import request
import hashlib, hmac, json

app = Flask(__name__)

# Set your runtime variables
secret = '<YOUR WEBHOOK SECRET>'
port = 8080

# Signature verification
def verifyHmac(received_hmac_header, received_body, secret):
    hmac_parts = received_hmac_header.split(' ')
    received_hmac = hmac_parts[1]
    hash = hmac.new(bytes(secret, 'UTF-8'), received_body, hashlib.sha256).hexdigest()
    return received_hmac == hash

# Main webhook entry point
@app.route('/webhook', methods=['POST'])
def webhook():
    # Some initial basic verification
    if 'signature' not in request.headers:
        print("Signature doesn't exist")
        return '', 500
    else:
        print('Signature:', request.headers['signature'])

    # Verify source from signature
    if not verifyHmac(request.headers['signature'], request.data, secret):
        print('Signature verification failed')
        return '', 500

    print('Signature verified')

    # Extract payload
    payload = request.get_json()
    print(json.dumps(payload, indent=4))

    # Parse the actual payload here
    # TODO

    return 'Ok'

if __name__ == '__main__':
    print('Listening for events on ' + str(port))
    app.run(port=port)
```


