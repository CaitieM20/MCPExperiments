Below is a mock up of the new Tasks workflow. It simplifies the existing one by combining the various methods into a single unified method `tasks/get`.


TODO & OpenQuestions:
- Do we need a specific error code if the client is polling too frequently? 
- In the Error Case this is very Tool Specific, do we need generic error handling here?
- This will work with request/response, can also work with streaming. How do we make the mode more explicit?


## Tool Call & Task Creation
1. The client calls a Tool. The Server determines if a Task is created for this request. 
``` json
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

2. The server determines if a Task should be created for this request. If so, it creates a Task and returns the Task ID to the client. 
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
      "lastUpdatedAt": "2025-11-25T10:40:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    }
  }
}
```

3. The Client can then poll `tasks/get` to check on the status of the Task and retrieve any intermediary results.

## Task Get Results
A `Task` can be in one of the following states, below defines the response type the server should send for each state to the following client request.

``` json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tasks/get",
  "params": {
    "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840"
  }
}
```

A. If Server Task Status is `working`
``` json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
      "status": "working",
      "statusMessage": "The operation is in progress.",
      "createdAt": "2025-11-25T10:30:00Z",
      "lastUpdatedAt": "2025-11-25T10:40:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    }
  }
}
```

B. If Server Task Status is `completed` the  server MUST return the `Task` and the ToolCallResult
```json
{
"jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
      "status": "completed",
      "statusMessage": "The operation has completed successfully.",
      "createdAt": "2025-11-25T10:30:00Z",
      "lastUpdatedAt": "2025-11-25T10:40:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    },
    "content": [
        {
            "type": "text",
            "text": "Current weather in New York:\nTemperature: 72°F\nConditions: Partly cloudy"
        }
    ],
    "isError": false
  }
}
```

C. If Server Task Status is `failed` the server MUST return the `Task` and error information.
```json
{
"jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
      "status": "failed",
      "statusMessage": "The operation has failed.",
      "createdAt": "2025-11-25T10:30:00Z",
      "lastUpdatedAt": "2025-11-25T10:40:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    },
    "content": [
        {
            "type": "text",
            "text": "An Error occurred processing this request"
        }
    ],
    "isError": true
  }
}
```

D. If Server Task Status is `input_required` this indicates that the `Task` requires additional input from the client before it can proceed. The server MUST return the `Task` and a `IncompleteResult` from the MRTR SEP
```json
{
"jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
      "status": "input_required",
      "statusMessage": "The operation requires additional input.",
      "createdAt": "2025-11-25T10:30:00Z",
      "lastUpdatedAt": "2025-11-25T10:40:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    },
    "inputRequests": {
      "github_login": {
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
    }
  }
}
```

E. If Server Task Status is `cancelled`
```json
{
"jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
      "status": "cancelled",
      "statusMessage": "The operation has been cancelled.",
      "createdAt": "2025-11-25T10:30:00Z",
      "lastUpdatedAt": "2025-11-25T10:40:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    },
  }
}
```

## Task Cancellation Flow
The Client may indicate to the Server that it wishes to cancel a Task by sending a Cancellation request. This is a hint to the server that the client is no longer interested in the result of the Task. Cancellation provides no guarantees around server state or actions being completed/undone. These guarantees are left to the server's implementation.

If the server supports cancellation it will transition the Task to the `cancelled` state and stop further processing. Not every operation can be cancelled and leave the Server in a consistent statue. A sever MAY choose to ignore the cancellation request and proceed processing the `Task` to `completed` or `failed` states based on its internal logic.

<b> Cancellation Supported Flow </b>
1. Client sends cancellation request:
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tasks/cancel",
    "params": {
        "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840"
    }
}
```
2. Server responds with a `Task` object in the `cancelled` state:
```json
{
"jsonrpc": "2.0",
  "id": 1,
  "result": {
    "task": {
      "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
      "status": "cancelled",
      "statusMessage": "The operation has been cancelled.",
      "createdAt": "2025-11-25T10:30:00Z",
      "lastUpdatedAt": "2025-11-25T10:40:00Z",
      "ttl": 60000,
      "pollInterval": 5000
    },
  }
}
```

## Input Required Flow
Below illustrates the workflow for a Task that requires additional input from the client before it can proceed. Once a Task is in the `input_required` state, the client must provide the requested input before the Task can continue processing.

1. The client performs a `tasks/get` request to retrieve the current status of the Task.
2. The server responds with a `Task` object in the `input_required` state, along with the `inputRequests` that specify the additional information needed from the client.
3. The client requests this information from the user or another source as specified in the `inputRequests` object.
4. The client sends the requested input back to the server on the next `tasks/get` request.

``` json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tasks/get",
  "params": {
    "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
    "inputResponses": {
      "echo_input": {
        "action": "accept",
        "content": {
          "not_requested_parameter": "Information the server did not request."
        }
      }
    }
  }
}
```

5. The server processes the provided input and updates the Task status accordingly. In most cases it will transition back to the `working` state, continuing excution. The server MAY also transition the Task to `completed` or `failed` based on the outcome of processing the input.

### Error Cases - Requested Input Not provided.
There are several scenarios where the client may choose not to respond to input including:
1. the user rejects the Elicitation request 
2. the user does not ever provide the requested input so the client never sends the `InputResponses` for the Task.
3. the client encounters an error and crashses.

for #1, user rejects the Elicitation request, the Client SHOULD send the `InputResponses` with an action of `reject` to indicate that the user has rejected the elicitation request. The server can then decide how it wants to proceed with processing the request, it may ask for additional information, proceed without the requested input, or fail the Task.

``` json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tasks/get",
  "params": {
    "taskId": "786512e2-9e0d-44bd-8f29-789f320fe840",
    "inputResponses": {
      "echo_input": {
        "action": "decline"
      }
    }
  }
}
```

for #2, client never sends the `InputResponses`. The Servers SHOULD leverage `Task` ttl to determine when to transition the `Task` into the `failed` state.

for #3, the client MAY durably store in progress `Tasks` and resume polling `tasks/get` after restart. The Server SHOULD just return the current state of the `Task` without any changes until the client provides the requested input or the `Task` ttl expires.

## Task Schema
The `Task` schema defines the Task metadata and remains unchanged.

### Client Requests for `task/get`
```typescript
interface GetTaskRequest extends JSONRPCRequest {
    method "tasks/get";
    params: {
    /**
     * The task identifier to query.
     */
    taskId: string;
    /**
     * Optional field to allow the client to respond to a server's request for more information 
     * when the task is in `input_required` state.
     */
    inputResponses?: InputResponses;
  };
}
```

### Server Response for `task/get`
```typescript
interface GetTaskResult extends Result
{
    /**
     * Required field containing the Task Metadata Object.
     */
    task: Task;
    /**
     * Optional field containing the InputRequests that specify the additional information needed from the client.
     * Should only be sent when the task is in the `input_required` state.
     */
    inputRequests?: InputRequests;
    /**
     * Optional field containing the Result of a Task if its in the `completed` state.
     */
    [key: string]?: unknown;
}
```

### ResultType
We propose the addition of the `task` ResultType to indicate that a Response contains a Task object. For example in the `tools/call` method can return a `ToolCallResult` an `IncompleteRequest` or a `Task` now. 

```typescript
type ResultType = "complete" | "incomplete" | "task"
```

While not strictly necessary for `task/get`, using the `task` ResultType can help standardize the handling of responses that contain Task objects across different methods.

## Headers 
[SEP-2243](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2243) introduces standard headers in the Streamable HTTP Transport to facilitate more efficient routing. Routing on TaskId is also desirable since there is often state associated with a specific Task that needs to be consistently routed to the same server instance.

For Tasks the following Headers MUST be set by the client when making requests over the Streamable HTTP Transport:
- `Mcp-Method`: `tasks/get`, `tasks/cancel`
- `Mcp-Name`: should contain the `taskId` of the Task being requested or cancelled.
