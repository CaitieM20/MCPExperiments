# Overview
As the Model Context Protocol evolves we'll need to support multiple response types from Tools and other Capabilities. To ensure consistent experiences between clients and servers, the Response Types and interaction patterns need to be well defined Tool Call.

Response Types Map to the Following Categories
- <b>Answer</b>: This is a basic request/response type and is widely used by the protocol. Should be used when responses are expected to be delivered in a short time frame. 
- <b>Promise</b>: Allows for async or long running operations. The response to this request contains a token that can be used to check status or redeemed later to get the result. This is crucial for operations that may exceed typical HTTP timeout windows.

The Result of a Resposne Type can be one of the Following
- <b> Complete Answer </b>: The result is delivered in full as a structured objet.
- <b> Failure </b>: The operation failed in a way that is unrecoverable or unexpected. This is different than the JSON-RPC errors which indicate a protocol failure. 
- <b> Additional Information</b>: The operation requires additional information from the client to proceed. This is different than a Failure as it is recoverable by providing the requested information. Today MCP supports both Elicitation & Sampling as means of gathering additional information. 
- <b> Streaming </b>: The result can be delivered incrementally and in real time instead of waiting for the full result. This reduces latency and improves interactive user experiences.

# Current Protocol Support
With the 11-25-25 release the MCP Protocol supports the following combinations of Response Types and Results:

## Answer
This is the default resposne type for tool calls and is widely used today. The result is delivered in full as a structured object. The Request/Response structure looks like this:

### Request
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "New York"
    }
}
```

### Response
The Response is a <b>Complete Answer</b> delivered in full.
``` json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Current weather in New York:\nTemperature: 72° F\nConditions: Partly cloudy"
      }
    ],
    "isError": false
  }
}
```
The Response can also be an <b>Failure</b>
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Error: Unable to retrieve weather data for New York."
      }
    ],
    "isError": true
  }
}
```

## Promise
MCP added support for Tasks to address this Response Type. It is an Experimental feature as of 11-25-25. This can be used to implement the Promise Concept.

### Request
The `tools/call` request is modified to include the `task` parameter to indicate that the client would like to receive a Promise result.
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "New York"
    },
    "task": {
      "ttl": 60000
    }
  }
}
```

### Response
The Response contains a token that can be used to check status or redeemed later to get the result.
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
      "status": "working",
      "statusMessage": "The operation is now in progress.",
      "createdAt": "2025-11-25T10:30:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    }
  }
}
```

# Future Work

## Additional Information Needed Result
Additional informaiton can be requested by created in the protocol today by creating an `elicitation/create` or `sampling/createMessage`. 

There are two options for how this could be represented in the Protocol as part of a Tool Call Response. 
1. Reuse existing Elicitation & Sampling messages as the Response
2. Create a new Additional Information Response type that can contain multiple Elicitation & Sampling requests.

### Option 1: Response reuses Existing Messages
One option is to have the Response from a Tool call just be an existing Elicitation or Sampling. Tasks today already does this. The below flow illustrates this message flow

<b> Tool Call Request </b>
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "New York"
    }
  }
}
```
<b> Elicitation Response </b>
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "elicitation/create",
  "params": {
    "mode": "form",
    "message": "Please provide your GitHub username",
    "requestedSchema": {
      "type": "object",
      "properties": {
        "name": {
          "type": "string"
        }
      },
      "required": ["name"]
    }
  }
}
```
<b> Tool Call Request #2 </b>
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "New York"
    }
  },
  "elicitation": 
    {
      "name": "name",
      "value": "octocat"
    }
}
```

The same could be done with Sampling requests and responses as well. This has the benefit of reusing existing message types and patterns, and matches what `Tasks` does However it does not allow for multiple Elicitation & Sampling requests to be returned in a single response which may be needed in some scenarios.

### Option 2: Additional Information Response Type 
The Response for Additional information can contain both Elicitation and Sampling requests. Below is an example of how the response could look for a Tool Call that requires additional information to proceed.
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": 
   {
    "AdditionalInfoRequested":{
        "elicitation": [
            {
                "params": {
                "mode": "form",
                "message": "Please provide your GitHub username",
                "requestedSchema": {
                    "type": "object",
                    "properties": {
                        "name": {
                        "type": "string"
                        }
                    },
                    "required": ["name"]
                    }
                }
            }
        ],
        "sampling": [...]
    },
    "isError": false
  }
}
```
For an <b>Answer</b> Response Type, The client would reinvoke the tool call including all required parameters with `call\tool` including the additional data requested.

### Reply to Additional Information
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "New York"
    }
  },
  "elicitation": [
    {
      "name": "name",
      "value": "octocat"
    }
  ],
  "sampling": [...]
}
```

### Promises/Tasks
Tasks provide a state `input_required` to indicate that additional information is needed to proceed. The client then calls `tasks/result` and an elicitation create message is returned. We can extend this to support all Additional Information Needed responses including Sampling by adopting the same Response structure as above. 

The client can then send the Elicitation & Sampling responses as they are processed, reusing existing message structures. The server associates these messages with the running Task by including the `taskId` and assocaited meta data in the request. 

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "action": "accept", // or "decline" or "cancel"
        "content": {
        "propertyName": "value",
        "anotherProperty": 42
        }
    },
    "_meta": {
      "io.modelcontextprotocol/related-task": {
        "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840"
      }
    }
}
```

## Streaming
Streaming is not yet supported in MCP. Below are some ealry mocks of how a Tool Call with streaming response could look.

### Request
#### Answer 
Following the conventions set by `tasks` we could add `stream` parameter to indicate that the client would like to receive a Streaming result back from an </b>Answer</b> request type.
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "New York"
    },
    "stream": {}
  }
}
```

#### Promise 
Similarly for `Promise` response type we could add `stream` parameter to the call to the `task/result` call
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tasks/result",
  "params": {
    "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840"
  },
  "stream": {}
}
```


