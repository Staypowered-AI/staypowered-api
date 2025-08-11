# StayPowered AI API
**Version**: 1.01 

## Overview

The StayPowered API allows you to exchange messages with a StayPowered AI Agent via API calls. Your incoming request would be executed by the agent and the response will be delivered to a configured webhook.

## API Host
The StayPowered AI API hostname is ```https://api.staypowered.ai```
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

### Request Body:

```json
{
  "project": "<PROJECT SLUG>",
  "from_id": "<YOUR UNIQUE USER IDENTIFIER>",
  "new_conversation": false,
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
  "new_conversation": false,
  "message": "Hello there!",
  "params": {
    "address": "1500 5th Ave, New York, NY 10010",
    "unit_number": "521"
  }
}
```

### Notes & Limitations:
1. Obtain your ```project``` slug from the StayPowered Console by looking up your project settings
3. The ```from_id``` property should be used to uniquely identify your user (i.e. email address, phone number or any other unique identifier). When used in consecutive requests it will cause the agent to continue the same conversation unless the conversation has expired (typically 4h) in which case the agent will create a new conversation.   
2. Parameters ```params``` are optional and need to be provided as string values. When provided, these parameters values can be injected into the agent instructions using instruction variables so that the AI can refer to them during execution. 
4. Message size should not exceed 10MB
5. Use the ```new_conversation = true``` to force the agent to create a new conversation. If an active (non-expired) conversation does not exist for the provided ```from_id``` then this property will be ignored and a new conversation will be created for the provided ```from_id```.

### Response Format:

Upon success (200 OK):
```json
{
  "success": true,
  "result": {
    "message_id": "1234567890"
  }
}
```
Use the returned message Id to identify the response for your request once your webhook is called with the agent response. 

Upon failure (500, 400, 401):
```json
{
  "success": false,
  "message": "<REASON FOR FAILURE>"
}
```
## Retrieve Conversation History
```
GET https://api.staypowered.ai/api/v1/project/:project/conversations
```
#### Path variables:
  - **project**: Project Slug
#### Query variables (optionsl):
  - **from**: Start date YYYY-MM-DD in UTC (default: start of day, current date - 1 day, UTC)
  - **to**: End date YYYY-MM-DD in UTC (default: end of day, current date, UTC)
  - **from_id**: Show only conversations for a specific from_id
#### Response Format:
An array of all conversation threads sorted by last activity timestamp, ascending
```json
[
  {
      "from_id": "1902335789",
      "thread_id": "kVhjeRPIPFr3Xge5zxx9e3E7",
      "last_activity": "2024-12-09T03:13:22.721Z",
      "expired": true,
      "conversation_summary": null,
      "user_location": null,
      "message_count": 3,
      "duration_sec": 10
  },
  {
      "from_id": "1788995132",
      "thread_id": "iqTma9jpPJxFDqHYBfzE0Mcb",
      "last_activity": "2024-12-08T16:05:46.462Z",
      "expired": true,
      "conversation_summary": {
          "unit": "601",
          "address": "900 Ivanhoe Dr, Northfield, MN 55057, USA",
          "synopsis": "The tenant, who lives in Northfield, MN, reported a problem with a clogged sink. The issue was reported to maintainance and the tenant was notified. Sentiment was positive",
          "sentiment": "Positive"
      },
      "user_location": {
          "lat": 44.4665839,
          "lng": -93.1735142,
          "city": "Northfield",
          "state": "MN",
          "street": "900 Ivanhoe Dr",
          "address": "900 Ivanhoe Dr, Northfield, MN 55057, USA",
          "country": "US",
          "zipcode": "55057"
      },
      "message_count": 6,
      "duration_sec": 106
  }
]
```
**Notes**:
- ```conversation_summary``` is an optional object that contains an AI generated summary of the conversation messages. The structure of this object is arbitrary and is determined by the summary instructions (configurable from the StayPowered Console). Value will be ```null``` if no summary is available.
- ```user_location``` optionally contains the user geolocation information. The user location can be automatically detected using the web client if browser geolocation is enabled and the user provides their consent (configurable from the StayPowered Console). Value will be ```null``` if no location is available.
- If ```expired``` is true, the conversation has expired and no new messages can be added. The expiration duration is at the project level and can be configured from the StayPowered Console, under project settings. 
## Retrieve Conversation Messages
```
GET https://api.staypowered.ai/api/v1/thread/:thread_id/messages
```
#### Path variables:
  - **thread_id**: The conversation thread Id
#### Query variables (optionsl):
  - None
#### Response Format:
An array of all conversation thread messages sorted by last activity timestamp, descending
```json
{
    "success": true,
    "result": [
        {
            "from_id": "5a939a97-3421-4bf4-bab0-b1e4dd3168b7",
            "to_id": "agent",
            "message": "hello",
            "timestamp": "2024-12-28T08:17:38.603Z",
            "message_format": "markdown"
        },
        {
            "from_id": "agent",
            "to_id": "5a939a97-3421-4bf4-bab0-b1e4dd3168b7",
            "message": "Dear unit 521 guest, welcome to our establishment! I am here to assist you with any requests or issues you might be experiencing. You can ask me all sorts of things like ask a billing question, report an issue and request a lease extension. How can I assist you better? 😊",
            "timestamp": "2024-12-28T08:17:45.794Z",
            "message_format": "markdown"
        }
    ]
}
```
**Notes**:
- ```message_format``` can be either ```markdown``` indicating it is formatted using [markdown](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) or ```json```.
- If a JSON object is provided in the response, the ```message``` value will contain a **stringified** JSON object. In order to return JSON, your agent must be configured to return JSON responses

    
## Receiving Agent Responses using Webhooks

To receive agent responses you will need to install a webhook via the StayPowered UI console (see below). 
StayPowered will call your webhook with the following payload:

```json
{
  "timestamp": "<TIMESTAMP IN ISO FORMAT>",
  "message_format": "<MESSAGE FORMAT - 'markdown' or 'json'>",
  "message": "<MESSAGE TEXT - either markdown or stringified JSON>",
  "reply_to_message_id": "<MESSAGE ID THIS REPLY REFERS TO>",
  "project": "<PROJECT SLUG>"
}
```

For Example:
```json
{
  "timestamp": "2024-12-16T07:31:16.718Z",
  "message_format": "markdown",
  "message": "I **LOVE** cupcakes!",
  "reply_to_message_id": "1234567890",
  "project": "test-project"
}
```
### Notes:
1. If ```message_format``` is json then the message will contain a stringified text of the JSON object. In order to return JSON, your agent must be configured to return JSON responses
2. Correlating requests and responses based on the message request result ```message_id``` and the response ```reply_to_message_id``` is the sole responsiblity of the calling application

## Setting up a Webhook

From the StayPowered Console, define a new webhook to receive reply messages. **You must enable and verify the webhook in order to activate it**.
StayPowered will attempt to call your webhook up to 5 tries using the following strategy:
```
(Next Attempt Timestamp) = (First Attempt Timestamp) + (Retry Attempt) * 5000 mSec 
```

### Authenticating Your Webhook

StayPowered implements a security mechanism for webhook authentication by employing a digital signature. This signature is crafted using a secret key provided during the webhook setup alongside the request body. The resulting data is encapsulated in the signature header property. The process to calculate the signature is as follows:

```HMAC-SHA256 = {Base64(HmacSHA256(body_bytes, YOUR_WEBHOOK_SECRET))}```

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
        .createHmac(hmacParts[0], secret)
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

### A Simple CLI Chat Client
The following code sample implements a simple node.js CLI chat client:
```javascript
var express = require('express')
const crypto = require('crypto');
const readline = require('node:readline');
const axios = require('axios');
const { v4: uuidv4 } = require('uuid');

/////////////////////////////
// API 
let endpoint = 'https://api.staypowered.ai/api/v1/message'

let headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer <YOUR TENANT API KEY>'
}

// Set the request body with some sample parameters
// In your StayPowered project you can refer to these parameters in the instructions
// using instruction variables. e.g. $(room_number) = params?.room_number || 'unknown'  
let body = {
    "project": "<YOUR PROJECT SLUG>",
    "message": "",
    "from_id": uuidv4(), // generate random UUID
    "new_conversation": true,
    "params": {
        "room_number": "889",
        "hotel_address": "400 Soldiers Field Rd, Boston, MA 02134"
    }
}

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

// Show animated prompt
let interval;
let i = 0;
let frames = ['-', '\\', '|', '/'];
function animate() {
  process.stdout.write(`\r${frames[i]}`);
  i = (i + 1) % frames.length;
}

////////////////////////////////
// Command line interface
//
const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
});
rl.setPrompt('Me> ');
rl.prompt();
rl.on('line', async function (line) {
    body.message = line;

    // Start new conversation
    if ( line == '#clear') {
        body.new_conversation = true;
        body.from_id = uuidv4();
        rl.prompt();
        return;
    }

    // Exit
    if ( line == '#exit' || line == '#quit') {
        process.exit(0);
    }

    // Send message
    try {
        await axios.post(endpoint, body, {headers: headers});
        interval = setInterval(animate, 80);
    }
    catch (e) {
        console.log(e?.response?.data || e.message);
    }

    // Next message will be part of the same conversation
    body.new_conversation = false;
});

/////////////////////////
// Web application
//
var app = express()

app.use(express.raw({ type: "*/*" }))

// Main webhook entry point
// To test your webhook, use a reverse proxy such as ngrok.io
app.post('/webhook', function (req, res) {

    // Some initial basic verification
    if (!req.headers.signature) {
        console.log('Signature doesn\'t exist');
        return res.sendStatus(500);
    }

    // Verify source from signature
    if (!verifyHmac(req.headers.signature, req.body, secret)) {
        console.log('Signature verification failed');
        return res.sendStatus(500);
    }

    // Extract payload
    let payload = JSON.parse(req.body);

    // print output
    clearInterval(interval);
    process.stdout.write("\r");
    console.log('Agent> ' + payload.message);
    rl.prompt();

    res.send('Ok')
})

app.listen(port);
```
