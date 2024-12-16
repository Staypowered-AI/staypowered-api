# StayPowered AI API
**Version**: 1.01 

## Overview

The StayPowered API allows you to exchange messages with a StayPowered AI Agent via API calls. Your incoming request would be executed by the agent and the response will be delivered to a configured webhook.

## Authenticating

Use your API tenant key in a bearer token when sending requests:

```
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

```
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

Response Format:

Upon success (200 OK):
```
{
  "success": true,
  "result": {
    "message_id": "1234567890"
  }
}
```
Use the returned message Id to identify th eresponse for your request. 

Upon failure (500, 400, 401):
```
{
  "success": false,
  "message": "<REASON FOR FAILURE>"
}

