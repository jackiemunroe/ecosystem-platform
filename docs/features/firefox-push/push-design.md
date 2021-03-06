---
id: push-design
title: Design, API's, Error Codes
sidebar_label: Design, API's, Error Codes
---

From [the WebPush
specification](https://tools.ietf.org/html/draft-ietf-webpush-protocol-01#section-2):

> A general model for push services includes three basic actors: a user agent, a
> push service, and an application (server).

        +-------+           +--------------+       +-------------+
        |  UA   |           | Push Service |       | Application |
        +-------+           +--------------+       +-------------+
            |                      |                      |
            |      Subscribe       |                      |
            |--------------------->|                      |
            |       Monitor        |                      |
            |<====================>|                      |
            |                      |                      |
            |          Distribute Push Resource           |
            |-------------------------------------------->|
            |                      |                      |
            :                      :                      :
            |                      |     Push Message     |
            |    Push Message      |<---------------------|
            |<---------------------|                      |
            |                      |                      |

In WebPush, the Push Message is used to wake UA code.

Currently the Mozilla Push Service supports a proprietary protocol
using a websocket between the UA and the Push Service.

## Bridge API

Push allows for remote devices to perform some functions using an HTTP
interface. This allows remote devices on Android, iOS, and Amazon to receive
WebPush compatible messages using WebPush endpoints.

See [Push Bridge API Documentation][bridge-api].

## WebPush Proprietary Protocol

This documents the current Push Service Protocol used between the UA and
the Push Service.

The protocol uses a websocket that carries JSON messages in a mostly
request/response style format. The only exception is that after the **Hello**
response the Push Server will send any stored notifications as well as new
notifications when they arrive.

All websocket connections are secured with TLS (wss://). The Push Server
maintains a list of all previously seen UAID's and notifications for UA's that
came in while the UA was disconnected. There is no authentication for a given
UAID when reconnecting.

It is **required** that ChannelID's and UAID's be UUID's.

### Messages

All messages are encoded as JSON. All messages MUST have the following fields:

messageType string
: Defines the message type

### Handshake

After the WebSocket is established, the UserAgent begins communication by
sending a hello message. The hello message contains the UAID if the UserAgent
has one, either generated by the UserAgent for the first handshake or returned
by the server from an earlier handshake. The UserAgent also transmits the
channelIDs it knows so the server may synchronize its state and optionally the
Megaphone broadcastIDs it's subscribed to (if any).

The server MAY respect this UAID, but it is at liberty to ask the UserAgent to
change its UAID in the response.

If the UserAgent receives a new UAID, it MUST delete all existing channelIDs and
their associated versions. It MAY then wake up all registered applications
immediately or at a later date by sending them a push-register message.

A repeated hello message on an established channel may result in the server
disconnecting the client for bad behavior.

The handshake is considered complete, once the UserAgent has received a reply.

An UserAgent MUST transmit a hello message only once on its WebSocket. If the
handshake is not completed in the first try, it MUST disconnect the WebSocket
and begin a new connection.

**NOTE**: Applications may request registrations or unregistrations from the
UserAgent, before or when the handshake is in progress. The UserAgent MAY buffer
these or report errors to the application. But it MUST NOT send these requests
to the PushServer until the handshake is completed.

#### UserAgent -> PushServer

<dl>
    <dt>messageType = "hello"</dt>
    <dd></dd>
    <dt>uaid string REQUIRED</dt>
    <dd>If the UserAgent has a previously assigned UAID, it should send it. Otherwise
  send an empty string.</dd>
    <dt>channelIDs list of strings REQUIRED</dt>
    <dd>If the UserAgent has a list of channelIDs it wants to be notified of, it must
  pass these, otherwise an empty list.</dd>
    <dt>use_webpush bool OPTIONAL</dt>
    <dd>If the UserAgent wants the WebPush data support for notifications it
  may include this field with a value of *true*. This field may appear for legacy reasons.</dd>
    <dt>broadcasts object OPTIONAL</dt>
    <dd>If the UserAgent wants to subscribe to specific megaphone broadcasts, it may
  include this field: an object consisting of broadcastIDs mapped to the
  UserAgent's current broadcast versions.</dt>
</dl>

Extra fields: The UserAgent MAY pass any extra JSON data to the PushServer. This
data may include information required to wake up the UserAgent out-of-band. The
PushServer MAY ignore this data.

**Example**

```json
{
  "messageType": "hello",
  "uaid": "fd52438f-1c49-41e0-a2e4-98e49833cc9c",
  "broadcasts": {
    "test/broadcast1": "v3",
    "test/broadcast2": "v0"
  }
}
```

#### PushServer -> UserAgent

<dl>
    <dt>messageType = "hello"</dt>
    <dd></dd>
    <dt>uaid string REQUIRED</dt>
    <dd>The UserAgent's provided UAID OR a new UAID.</dd>
    <dt>status number REQUIRED</dt>
    <dd>Used to indicate success/failure. MUST be one of:

  * 200 - OK. Successful handshake.

  * 503 - Service Unavailable. Database out of space or offline. Disk space
    full or whatever other reason due to which the PushServer could not grant
    this registration. UserAgent SHOULD avoid retrying immediately.</dd>
    <dt>use_webpush bool OPTIONAL</dt>
    <dd>Must match sent value</dd>
    <dt>broadcasts object OPTIONAL</dt>
    <dd>*Only* broadcasts the UserAgent has requested subscriptions to whose version
values differ from the UserAgent's provided values. It may be empty if the
UserAgent is up to date.</dd>
</dl>

**Example**

```json
{
  "messageType": "hello",
  "status": 200,
  "uaid": "fd52438f-1c49-41e0-a2e4-98e49833cc9c",
  "broadcasts": {
    "test/broadcast2": "v1"
  }
}
```


### Register

The Register message is used by the UserAgent to request that the PushServer
notify it when a channel changes. Since channelIDs are associated with only one
UAID, this effectively creates the channel, while unregister destroys the
channel.

The channelID is chosen by the UserAgent because it also acts like a nonce for
the Register message itself. Because of this PushServers MAY respond out of
order to multiple register messages or messages may be lost without compromising
correctness of the protocol.

The request is considered successful only after a response is received with a
status code of 200. On success the UserAgent MUST:

* Update its persistent storage based on the response
* Notify the application of a successful registration.
* On error, the UserAgent MUST notify the application as soon as possible.

**NOTE**: The register call is made by the UserAgent on behalf of an
application. The UserAgent SHOULD have reasonable timeouts in place so that
the application is not kept waiting for too long if the server does not
respond or the UserAgent has to retry the connection.

#### UserAgent -> PushServer

<dl>
    <dt>messageType = "register"</dt>
    <dd></dd>
    <dt>channelID string REQUIRED</dt>
    <dd>A unique identifier generated by the UserAgent, distinct from any existing
  channelIDs it has registered. It is RECOMMENDED that this is a UUIDv4 token.</dd>
</dl>

**Example**

```json
{
    "messageType": "register",
    "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd"
}
```

#### PushServer -> UserAgent

<dl>
    <dt>messageType = "register"</dt>
    <dd></dd>
    <dt>channelID string REQUIRED</dt>
    <dd>This MUST be the same as the channelID sent by the UserAgent in the register
  request that this message is a response to.</dd>
    <dt>status number REQUIRED</dt>
    <dd>Used to indicate success/failure. MUST be one of:

  * 200 - OK. Success. Idempotent: If the PushServer receives a register for the
    same channelID from a UserAgent which already has a registration for the
    channelID, it should still respond with success.

  * 409 - Conflict. The chosen ChannelID is already in use and NOT associated
    with this UserAgent. UserAgent SHOULD retry with a new ChannelID as soon as
    possible.

  * 500 - Internal server error. Database out of space or offline. Disk space
    full or whatever other reason due to which the PushServer could not grant
    this registration. UserAgent SHOULD avoid retrying immediately.</dd>
    <dt>pushEndpoint string REQUIRED</dt>
    <dd>Should be the URL sent to the application by the UserAgent. AppServers will
  contact the PushServer at this URL to update the version of the channel
  identified by channelID.</dd>
</dl>

**Example**

```json
 {
   "messageType": "register",
   "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd",
   "status": 200,
   "pushEndpoint": "http://pushserver.example.org/d9b74644"
 }
```

### Unregister

Unregistration is **required** for the WebPush.

PushServers MUST support it. UserAgents SHOULD support it.

The unregister is required only between the App and the UserAgent, so that the
UserAgent stops notifying the App when the App is no longer interested in a
pushEndpoint.

The unregister is also useful to the AppServer, because it should stop sending
notifications to an endpoint the App is no longer monitoring. Even then, it is
really an optimization so that the AppServer need not have some sort of garbage
collection mechanism to clean up endpoints at intervals of time.

The PushServer MUST implement unregister, but need not rely on it. Well behaved
AppServers will stop notifying it of unregistered endpoints automatically. Well
behaved UserAgents won't notify their apps of unregistered updates either. So
the PushServer can continue to process notifications and pass them on to
UserAgents, when it has not been told about the unregistration.

When an App calls unregister(endpoint) it is RECOMMENDED that the UserAgent
follow these steps:

1. Remove its local registration first, for example from the database.
   This will allow it to immediately start ignoring updates.
2. Notify the App that unregistration succeeded.
3. Fire off an unregister message to the PushServer

#### UserAgent -> PushServer

<dl>
    <dt>messageType = "unregister"</dt>
    <dd></dd>
    <dt>channelID string REQUIRED</dt>
    <dd>ChannelID that should be unregistered.</dd>
</dl>

**Example**
```json
{
    "messageType": "unregister",
    "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd"
}
```

#### PushServer -> UserAgent

<dl>
    <dt>messageType = "unregister"</dt>
    <dd></dd>
    <dt>channelID string REQUIRED</dt>
    <dd>This MUST be the same as the channelID sent by the UserAgent in the
  unregister request that this message is a response to.</dd>
    <dt>status number REQUIRED</dt>
    <dd>Used to indicate success/failure. MUST be one of:

  * 200 - OK. Success. Idempotent: If the PushServer receives a unregister for a
    non-existent channelID it should respond with success. If the channelID is
    associated with a DIFFERENT UAID, it MUST NOT delete the channelID, but
    still MUST respond with success to this UserAgent.

  * 500 - Internal server error. Database offline or whatever other reason due
    to which the PushServer could not grant this unregistration. UserAgent
    SHOULD avoid retrying immediately.</dd>
</dl>

**Example**

```json
{
    "messageType": "unregister",
    "channelID": "d9b74644-4f97-46aa-b8fa-9393985cd6cd",
    "status": 200
}
```

### Ping

The connection to the Push Server may be lost due to network issues. When the
UserAgent detects loss of network, it should reconnect. There are situations in
which the TCP connection dies without either end discovering it immediately. The
UserAgent MAY send a ping approximately every 30 minutes and expect a reply
from the server in a reasonable time (The Mozilla UserAgent uses 10 seconds). If
no data is received, the connection should be presumed closed and a new
connection started.

A UserAgent MUST NOT send a Ping more frequently than once a minute or its
connection MAY be dropped.

The UserAgent should consider normal communications as an indication that the
socket is working properly. It SHOULD send the ping packet only if no activity
has occurred in the past 30 minutes.

**Note**: This section is included for relevant historical purposes as the
current implementation sends WebSocket Ping frames every 5 minutes which is
sufficient to keep the connection alive. As such, client-sent Ping's are no
longer needed.

#### UserAgent -> PushServer

The 2-character string {} is sent. This is a valid JSON object that requires no
alternative processing on the server, while keeping transmission size small.

#### PushServer -> UserAgent

The PushServer may reply with any data. The UserAgent is only concerned about
the state of the connection. The PushServer may deliver pending notifications or
other information. If there is no pending information to be sent, it is
RECOMMENDED that the PushServer also reply with the string {}.

### Notification

#### AppServer -> PushServer

The AppServer MUST make a HTTP PUT request to the Endpoint received from the
App, or a HTTP POST if using the WebPush extension.

If no request body is present, the server MAY presume the version to be the
current server UTC.

If the request body is present, the request MUST contain the string "version=N"
and the Content-Type MUST be application/x-www-form-urlencoded.

#### PushServer -> AppServer

The HTTP response status code indicates if the request was successful.

* 200 - OK. The PushServer will attempt to deliver a notification to the
  associated UserAgent.

* 500 - Something went wrong with the server. Rare, but the AppServer should try
  again.

The HTTP response body SHOULD be empty.

#### PushServer -> UserAgent

Notifications are acknowledged by the UserAgent. PushServers should retry unacknowledged notifications every 60 seconds. If the version of an unacknowledged notification is updated, the PushServer MAY queue up a new notification for this channelID and the new version, and remove the old notification from the pending queue.

<dl>
    <dt>messageType = "notification"</dt>
    <dd></dd>
</dl>

The following attributes are present
in each notification. A
notification message is sent for every stored notification individually, as well
as new notifications.

<dl>
    <dt>channelID string REQUIRED</dt>
    <dd>ChannelID of the notification</dd>
    <dt>version string REQUIRED</dt>
    <dd>Version of the notification sent</dd>
    <dt>headers map OPTIONAL</dt>
    <dd>Encryption headers sent along with the data. Present only if data was sent.</dd>
    <dt>data string OPTIONAL</dt>
    <dd>Data payload, if included. This string is Base64 URL-encoded.</dd>
</dl>

**Example**

```json
{
    "messageType": "notification",
    "channelID": "431b4391-c78f-429a-a134-f890b5adc0bb",
    "version": "a7695fa0-9623-4890-9c08-cce0231e4b36:d9b74644-4f97-46aa-b8fa-9393985cd6cd"
}
```

#### UserAgent -> PushServer

It is RECOMMENDED that the UserAgent try to batch all pending acknowledgements
into fewer messages if bandwidth is a concern. The ack MUST be sent as soon as
the message has been processed, otherwise the
Push Server MAY cease sending notifications to avoid holding excessive client
state.

<dl>
    <dt>messageType = "ack"</dt>
    <dd></dd>
    <dt>updates list</dt>
    <dd>The list contains one or more {"channelID": channelID, "version": N} pairs.</dd>
</dl>

**Example**

```json
{
    "messageType": "ack",
    "updates": [{ "channelID": "431b4391-c78f-429a-a134-f890b5adc0bb", "version": 23 }, { "channelID": "a7695fa0-9623-4890-9c08-cce0231e4b36", "version": 42 } ]
}
```


### Broadcast

When a UserAgent's subscribed broadcasts are updated, PushServers send
broadcast messages with no expected reply.

#### PushServer -> UserAgent

<dl>
    <dt>messageType = "broadcast"</dt>
    <dd></dd>
    <dt>broadcasts object OPTIONAL</dt>
    <dd>*Only* broadcasts the UserAgent has requested subscriptions to whose new
version values differ from the UserAgent's provided values.</dd>
</dl>

**Example**

```json
{
  "messageType": "broadcast",
  "broadcasts": {
    "test/broadcast1": "v4"
  }
}
```


### Broadcast Subscribe

UserAgents subscribe to broadcasts during the hello handshake but may also
subscribe to additional broadcasts afterwards. There is no expected reply to
this message.

#### UserAgent -> PushServer

<dl>
    <dt>messageType = "broadcast_subscribe"</dt>
    <dd></dd>
    <dt>broadcasts object REQUIRED</dt>
    <dd>New broadcasts the UserAgent is requesting subscriptions to and their current
broadcast versions.</dd>
</dl>


**Example**

```json
{
  "messageType": "broadcast_subscribe",
  "broadcasts": {
    "test/new-broadcast22": "v8"
  }
}
```

## Error Codes

The Mozilla Push Service may return a variety of error codes depending on what
went wrong in the request.


Autopush uses error codes based on [HTTP response codes](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html).
An error response will contain a JSON body including an additional error information.

Unless otherwise specified, all calls return one the following error statuses:

-  20x - **Success** - The message was accepted for transmission to the client. Please note that the message may still be rejected by the User Agent if there is an error with the message's encryption.
-  301 - **Moved + `Location:`** if `{client_token}` is invalid (Bridge API Only) - Bridged services (ones that run over third party services like GCM and APNS), may require a new URL be used. Please stop using the old URL immediately and instead use the new URL provided.
-  400 - **Bad Parameters** -- One or more of the parameters specified is invalid. See the following sub-errors indicated by `errno`

   - errno 101 - Missing necessary crypto keys - One or more required crypto key elements are missing from this transaction. Refer to the [appropriate specification](https://datatracker.ietf.org/doc/draft-ietf-httpbis-encryption-encoding/) for the requested content-type.
   - errno 108 - Router type is invalid - The URL contains an invalid router type, which may be from URL corruption or an unsupported bridge. Refer to [bridge API][bridge-api].
   - errno 110 - Invalid crypto keys specified - One or more of the crytpo key elements are invalid. Refer to the [appropriate specification](https://datatracker.ietf.org/doc/draft-ietf-httpbis-encryption-encoding/) for the requested content-type.
   - errno 111 - Missing Required Header - A required crypto element header is missing. Refer to the [appropriate specification](https://datatracker.ietf.org/doc/draft-ietf-httpbis-encryption-encoding/) for the requested content-type.

       - Missing TTL Header - Include the Time To Live header ([IETF WebPush protocol §6.2](https://tools.ietf.org/html/draft-ietf-webpush-protocol#section-6.2))
       - Missing Crypto Headers - Include the appropriate encryption headers ([WebPush Encryption §3.2](https://webpush-wg.github.io/webpush-encryption/#rfc.section.3.2) and [WebPush VAPID §4](https://tools.ietf.org/html/draft-ietf-webpush-vapid-02#section-4))

   - errno 112 - Invalid TTL header value - The Time To Live "TTL" header contains an invalid or unreadable value. Please change to a number of seconds that this message should live, between 0 (message should be dropped immediately if user is unavailable) and 2592000 (hold for delivery within the next approximately 30 days).
   - errno 113 - Invalid Topic header value - The Topic header contains an invalid or unreadable value. Please use only ASCII alphanumeric values [A-Za-z0-9] and a maximum length of 32 bytes..

-  401 - **Bad Authorization** - `Authorization` header is invalid or missing. See the [VAPID specification](https://datatracker.ietf.org/doc/draft-ietf-webpush-vapid/).

   - errno 109 - Invalid authentication

- 404 - **Endpoint Not Found** - The URL specified is invalid and should not be used again.

   - errno 102 - Invalid URL endpoint

-  410 - **Endpoint Not Valid** - The URL specified is no longer valid and should no longer be used. A User has become permanently unavailable at this URL.

   - errno 103 - Expired URL endpoint
   - errno 105 - Endpoint became unavailable during request
   - errno 106 - Invalid subscription

-  413 - **Payload too large** - The body of the message to send is too large. The max data that can be sent is 4028 characters. Please reduce the size of the message.

   - errno 104 - Data payload too large

-  500 - **Unknown server error** - An internal error occurred within the Push Server.

   - errno 999 - Unknown error

-  502 - **Bad Gateway** - The Push Service received an invalid response from an upstream Bridge service.

   - errno 900 - Internal Bridge misconfiguration
   - errno 901 - Invalid authentication
   - errno 902 - An error occurred while establishing a connection
   - errno 903 - The request timed out

-  503 - **Server temporarily unavailable.** - The Push Service is currently unavailable. See the error number "errno" value to see if retries are available.

   -  errno 201 - Use exponential back-off for retries
   -  errno 202 - Immediate retry ok




[pushapi]: https://developer.mozilla.org/en-US/docs/Web/API/Simple_Push_API
[bridge-api]: ../../api/push_bridge.html
