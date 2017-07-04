- [Communicating via HTTP](#communicating-via-http)
- [Websockets](#websockets)
  - [Authentication](#authentication)
  - [Requests](#requests)
  - [Event listeners](#event-listeners)
  - [Action hooks](#action-hooks)

## Communicating via HTTP

Using HTTP to communicate with the API is often the easiest solution for performing simple tasks.

### Basic authentication

The application supports [cookie-based basic HTTP authentication](https://en.wikipedia.org/wiki/Basic_access_authentication#Client_side) for API requests.

An example for refreshing the whole share with [curl](https://curl.haxx.se):

`curl -H "Content-Type: application/json" -X POST -u myusername:mypassword http://localhost:5600/api/v1/share/refresh`

Note that a session object is still created internally by the application, so the methods available for [current session](http://docs.airdcpp.apiary.io/#reference/sessions/current-session/get-current-session) can still be used (such as if you need information about the current user permissions).


### Session-based authentication

Creating an unique session should generally be done only if you are going to use HTTP calls in conjuction with WebSockets.

Authentication happens through [Session API's authentication method](http://docs.airdcpp.apiary.io/#
/sessions/authentication/create-session). Once you have sent the user credentials, you will receive the session token in the response message:

```json
    ....
    "auth_token": "25793d1e-6d48-407c-9c27-a9dcbd2e1188"
    ...
```

The token should be sent in the `Authorization` HTTP header when making requests:

```Authorization: 25793d1e-6d48-407c-9c27-a9dcbd2e1188```


## WebSockets

If your use case is simple, such as initiating share refreshes for directories downloaded from external sources, plain HTTP calls will be totally sufficient.

However, since the application is heavily based on bidirectional communication and long running operations (chatting, file transfers, share refreshes...), writing
of certain implementations will become significantly easier when the application is able to push information about new events to API consumers directly.

WebSockets are beneficial especially if at least one of the following conditions is met:

- There is an existing socket connector available for your scripting language
- Minimal latencies are important (such bidirectional chatting)
- You are developing a more complex integration

WebSockets may be used in conjunction with regular HTTP request or the sockets may also be used exclusively for all API communication.

Note that this section is used as a reference for low-level socket communication. Socket connectors, such as airdcpp-apisocket-js, will have their own abstactions for API communication.


### Authentication

You should use the following URL format when establishing WebSocket connections:

Unencrypted: `ws://<address>:<http port>/api/v1/`
Encrypted: `wss://<address>:<https port>/api/v1/`

WebSockets require an [unique session created via the Session API](http://docs.airdcpp.apiary.io/#
/sessions/authentication/create-session) (basic HTTP authentication isn't sufficient).


#### Associating socket with an existing session

If only want to use WebSockets for receiving event messages and continue using regular HTTP calls for requests, you may associate the socket with an existing session instead:

```json
{
    "method": "POST",
    "path": "/sessions/socket",
    "callback_id": 1,
    "data": { 
        "auth_token": "046f477f-3f09-45b6-8582-889d5fee0de7"
    }
}
```

Response: 

```json
{
    "code": 204,
    "callback_id": 1
}
```

#### Authentication through the socket

To create a new session directly via the socket, send the following message:

```json
{
    "method": "POST",
    "path": "/sessions/authorize",
    "callback_id": 1,
    "data": { 
        "username": "myusername",
        "password": "mypassword"
    }
}
```

[Authentication method API reference](http://docs.airdcpp.apiary.io/#reference/sessions/authentication/create-session)


### Requests

You may use WebSockets exclusively for all API communication. The message structure emulates regular HTTP requests:

**Request**

```json
{
    "method": "POST",
    "path": "/hubs/chat_message",
    "callback_id": 1,
    "data": { 
        "text": "Sent through WebSocket",
        "hub_urls": [ "adcs://myhub.com:4252" ]
    }
}
```

`callback_id` is an unique identifier (positive integer) for the request that you can use for identifying the correct response message. 
Responses may not be delivered in the same order as the requests were sent. Requests sent without `callback_id` will be processed but no responses are sent.

**Example response (success)**

```json
{
    "code": 200,
    "callback_id": 1,
    "data": { 
        "sent": 4
    }
}
```

**Example response (error)**

```json
{
    "code": 403,
    "callback_id": 1,
    "error": { 
        "message": "The permission hubs_send is required for using this method"
    }
}
```


### Event listeners

#### Generic listeners

The format for adding generic listeners is

`POST /<section>/listeners/<subscription name>`

Such as:

`POST /system/listeners/away_state`

An example event message for this event would be: 

```json
{
    "event": "away_state",
    "data": { 
        "id": "idle"
    }
}
```

The lis may be removed with

`DELETE /system/listeners/away_state`

Note that all added subscriptions will be reset when the socket gets disconnected.

Due to limitations of the current documentation framework, event messages are described as regular responses in the API reference section.
Demonstrated example responses will contain the `data` field content only instead of showing the full socket messages with event names and possible IDs.

#### Entity-based subscriptions

You may add subscriptions for certain individual entities (such as hubs and filelists):

`POST /hubs/0/listeners/hub_status_message`

In such cases event messages will also contain the entitity identifier for which the event was triggered:

```json
{
    "event": "hub_status_message",
    "id": 0,
    "data": {
        "text": "Connecting to adcs://myhub.com:5231 ...",
        "severity": "info"
    }
}
```

If you want to receive the event across all entities managing the subscriptions individually for each entity can become cumbersome. That's why the same listeners may also always be added without binding them to any specific entities:

`POST /hubs/listeners/hub_status_message`

The resulting event message will be the same as described earlier (entity IDs are still included for identification purposes).


### Action hooks

Action hooks allow scripts to perform validations related to various client actions, such as validate the content finished bundles/files or filter incoming chat messages. All events fired by the hook must either be accepted or rejected by the script before the application processes them futher.

**Action hooks vs event listeners**

Action hooks are meant to be used only if you are going to change the behavior how certain actions/events are being processed by the application (or whether the action/event is being processed at all). If you just want to be notified of a certain event, you should use regular event listeners, as unlike action hooks, the application won't stop and wait for the listener events to be processed by subscribers and are thus faster and more lightweight. 

Furthermore, various action hooks can be used to reject events from being processed further. As the hook processing order is undefined, you have no way of knowing whether the subsequent subscribers are going to prevent the event from actually "going live". For example, if you use the bundle completion hook as a trigger to move downloaded files to a different location on disk, you will miss possible content validations being performed by other subscribers. Alternatively, you could be logging chat messages in an incoming message hook before knowing whether they are going to be ignored. The correct solution in both cases would be to use the respective event listeners instead (“Bundle status changed” listener to filter bundle completion events and incoming message listeners for chat message).
 

#### Adding hooks

`POST /hubs/hooks/hub_incoming_message_hook`

Data (required):

```json
{
    "id": "simple_chat_filter",
    "name": "Chat message filter",
}
```

+ id: Hook subscriber ID, which should be globally unique across all sessions for a single hook so it's better avoid using too generic IDs (subscribers with the same ID can't be added simultaneously from multiple sessions). The ID should contain alphanumeric characters only.
+ name: Display name for the subscriber

It's not possible to add multiple subscribers for a single hook from within the same session.

#### Removing hooks

`DELETE /hubs/hooks/hub_incoming_message_hook`

#### Handling hook events

**Hook event format**

```json
{
    "completion_id": 1,
    "event": "hub_incoming_message_hook",
    "data": "(hook-specific data)"
}
```

*completion_id* should be put in the URL used for accepting/rejecting the event.


**Accepting the event**

`POST /hubs/hooks/hub_incoming_message_hook/1/resolve`


**Rejecting the event**

`POST /hubs/hooks/hub_incoming_message_hook/1/reject`

Data (required):

```json
{
    "reject_id": "filtered",
    "message": "Filter due to message matching the pattern ignoretest"
}
```

Data passed in the reject call is required and will be used for debugging purposes and, depending on the hook, may also be displayed to the user.

+ reject_id: Subscriber-specific reject ID
+ message: Display reason for the rejection
