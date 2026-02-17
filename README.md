# LIME Protocol

**LIME** (*Lightweight Messaging Protocol*) is a JSON-based protocol for messaging, inspired by XMPP. It aims to be simple but extensible, with little verbosity yet providing good readability.

🌐 **Website:** [limeprotocol.org](https://limeprotocol.org/)

---

## Table of Contents

- [Introduction](#introduction)
- [Concepts](#concepts)
- [Node Identity](#node-identity)
- [Envelope Types](#envelope-types)
  - [Message](#message)
  - [Notification](#notification)
  - [Command](#command)
  - [Session](#session)
- [Content Types](#content-types)
- [Resources](#resources)
- [Libraries](#libraries)
- [About](#about)
- [License](#license)

---

## Introduction

The LIME Protocol defines a simple, JSON-based messaging standard as an alternative to XMPP. While XMPP is powerful, its complexity can be excessive for many use cases. LIME covers only the basic messaging structure and uses MIME content-type declarations to remain extensible, without attempting to support the full breadth of scenarios that XMPP handles.

## Concepts

The basic idea of the protocol is the exchange of **JSON documents** (called **envelopes**) between **nodes** in a network.

Nodes communicate over **persistent connections** (TCP or WebSockets) or HTTP. The protocol is **asynchronous**, where there is no guarantee of delivery order except during session negotiation, where only session envelopes are allowed.

### Node Identity

Every connected node has an identity in the `name@domain` format ([RFC 2822, section 3.4.1](http://tools.ietf.org/html/rfc2822#section-3.4.1)). A node can additionally specify an **instance** identifier (e.g. a device name), resulting in a full address of `name@domain/instance` (similar to XMPP's Jabber ID).

- The **domain** must be a valid DNS-resolvable domain.
- The **name** is a valid user account identifier in that domain.
- The **instance** differentiates multiple active connections for the same identity (e.g. `john@example.com/cellphone`, `john@example.com/pc`).

### Client / Server Roles

During a connection, the **server** is the node that received the connection and the **client** is the node that initiated it. The server controls session state management; otherwise, both share the same functionality.

---

## Envelope Types

All JSON documents exchanged must comply with [RFC 4627](http://tools.ietf.org/html/rfc4627) and use [UTF-8](http://tools.ietf.org/html/rfc3629) encoding.

Every envelope may contain these **common properties**:

| Property     | Description |
|-------------|-------------|
| **from**     | Sender node identifier (`name@domain/instance`). If absent, the envelope was originated by the remote party. |
| **to**       | Destination node identifier. If absent, the envelope is addressed to the receiver itself. |
| **pp**       | *Per procurationem* — a delegate node sending on behalf of another identity. |
| **id**       | Identifier of the envelope (any caller-relevant value). |
| **metadata** | Generic key/value information for transport context (should not change server behavior). |

### Message

Provides the transport of content between nodes.

**Required properties:** `type`, `content`

| Property    | Description |
|------------|-------------|
| **type**    | [MIME](http://tools.ietf.org/html/rfc2045) declaration of the content type (e.g. `text/plain`, `application/json`). |
| **content** | The content representation. JSON data is unescaped; binary data uses Base64 encoding. |

**Example** — plain text message:
```json
{
    "from": "skyler@breakingbad.com/bedroom",
    "to": "ww@breakingbad.com",
    "type": "text/plain",
    "content": "Walter, are you in danger?"
}
```

### Notification

Transports information about events related to a previously sent message. Can originate from the server or the destination node.

**Required properties:** `id`, `event`

| Property  | Description |
|----------|-------------|
| **event** | The event name: `accepted`, `dispatched`, `received`, `consumed`, or `failed`. |
| **reason**| Present when event is `failed`. Contains a `code` (integer) and optional `description` (string). |

**Event lifecycle:**
1. **accepted** — Message received and accepted by the server.
2. **dispatched** — Message dispatched to the destination by the server.
3. **received** — Destination has received the message.
4. **consumed** — Destination has read/processed the message.
5. **failed** — A problem occurred during processing.

**Example** — failure notification:
```json
{
    "id": "9d0c4fea-75c7-432a-a164-c1a219bc17a8",
    "to": "skyler@breakingbad.com/bedroom",
    "event": "failed",
    "reason": {
        "code": 42,
        "description": "The message destination was not found"
    }
}
```

### Command

Allows the manipulation of node resources (e.g. server session parameters, network node information).

**Required properties:** `id`, `method`

| Property     | Description |
|-------------|-------------|
| **method**   | `get`, `set`, `merge`, `delete`, `subscribe`, `unsubscribe`, or `observe`. |
| **uri**      | Resource URI ([RFC 3986](http://www.ietf.org/rfc/rfc3986.txt)). Uses the `lime://` scheme (e.g. `/presence`, `/contacts`). |
| **type**     | MIME type of the resource. |
| **resource** | JSON representation of the resource. |
| **status**   | Response status: `success` or `failure`. |
| **reason**   | Failure details with `code` and `description`. |

**Example** — setting presence:
```json
{
    "id": "9cbe5fe1-b6b2-4afe-ab12-0675aa139f36",
    "from": "jesse@breakingbad.com/home",
    "method": "set",
    "uri": "/presence",
    "type": "application/vnd.lime.presence+json",
    "resource": {
        "status": "available",
        "message": "Yo!"
    }
}
```

### Session

Configures and establishes the communication channel between nodes. Handles authentication, encryption, and compression negotiation.

**Required properties:** `state`

**Session states:**

| State              | Description |
|-------------------|-------------|
| **new**            | Client sends to start a session. |
| **negotiating**    | Server and client negotiate encryption and compression options. |
| **authenticating** | Server provides authentication schemes; client authenticates. |
| **established**    | Session is active — messages and commands can be exchanged. |
| **finishing**      | Client requests to end the session. |
| **finished**       | Server confirms the session has ended. |
| **failed**         | Session closed due to an error (includes `reason`). |

**Negotiation options:**
- **Encryption:** `none`, `tls`
- **Compression:** `none`, `gzip`
- **Authentication schemes:** `guest`, `plain`, `transport` (TLS client certificate)

---

## Content Types

The protocol suggests common message content types (not part of the core specification) for interoperability:

| Content Type | MIME Type | Description |
|-------------|-----------|-------------|
| **Text** | `text/plain` | Plain text messages. |
| **Chat State** | `application/vnd.lime.chatstate+json` | Conversation events: `starting`, `composing`, `paused`, `deleting`, `gone`. |
| **Collection** | `application/vnd.lime.collection+json` | A collection of documents with `itemType`, `total`, and `items`. |
| **Select** | `application/vnd.lime.select+json` | A list of text options for selection, with optional `scope` (`transient`, `persistent`, `immediate`). |
| **Document Select** | `application/vnd.lime.document-select+json` | Like Select but with generic documents in header and options. |
| **Location** | `application/vnd.lime.location+json` | Geographic coordinates (`latitude`, `longitude`, optional `altitude`, `course`, `speed`, `accuracy`). |
| **Media Link** | `application/vnd.lime.media-link+json` | External link to media content (`uri`, `type`, `size`, `title`, `text`, preview info). |
| **Web Link** | `application/vnd.lime.web-link+json` | External link to a web page (`uri`, `title`, `text`, `target`). |

---

## Resources

Every connected node has resources that can be queried or changed by other nodes through commands. Common resource types include:

| Resource | MIME Type | URI | Description |
|----------|-----------|-----|-------------|
| **Account** | `application/vnd.lime.account+json` | `/account` | User account info (name, email, phone, photo, password, etc.). |
| **Capability** | `application/vnd.lime.capability+json` | `/capability` | Messaging capabilities of a node (supported content types and resource types). |
| **Receipt** | `application/vnd.lime.receipt+json` | `/receipt` | Event notification preferences for the client. |
| **Presence** | `application/vnd.lime.presence+json` | `/presence` | Availability status (`unavailable`, `available`, `busy`, `away`) and routing rules. |
| **Contact** | `application/vnd.lime.contact+json` | `/contacts` | Contact/roster management (add, get, delete contacts). |
| **Group** | `application/vnd.lime.group+json` | `/groups` | Group management (`temporary`, `private`, `public` groups). |
| **Member** | `application/vnd.lime.groupmember+json` | `/groups/{id}/members` | Group member management with roles (`listener`, `member`, `moderator`, `owner`). |
| **Delegation** | `application/vnd.lime.delegation+json` | `/delegations` | Send envelopes on behalf of another identity. |
| **Ping** | `application/vnd.lime.ping+json` | `/ping` | Network connectivity test. |
| **Quota** | `application/vnd.lime.quota+json` | `/quota` | Session envelope quota (throughput, max envelope size). |

---

## Libraries

Available protocol implementations:

| Language | Description | Link |
|----------|-------------|------|
| C# | Reference implementation | [GitHub](https://github.com/takenet/lime-csharp) |
| Go | Full client/server support | [GitHub](https://github.com/takenet/lime-go) |
| Javascript | Typescript client stack | [GitHub](https://github.com/takenet/lime-js) |
| Java | Client stack (not actively maintained) | [GitHub](https://github.com/takenet/lime-java) |
| Python | Client implementation | [GitHub](https://github.com/takenet/lime-python) |

---

## About

This specification is maintained by [Blip](https://www.blip.ai/) developers.

- 💬 Join the [discussion group](https://groups.google.com/forum/?hl=pt-BR#!forum/limeprotocol)
- 🛠️ Contribute on [GitHub](https://github.com/takenet/lime/)

---

## License

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a>

**Lime Protocol Specification** by [Take](https://take.net) is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).
