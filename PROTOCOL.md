---
permalink: /
layout: default
---

# Contents
{:.no_toc}

* Contents go here
{:toc}

---

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC2119.

# Chapter 1. Introduction and basic concepts

## Introduction

**This section is non-normative.**

The Punctum protocol is a federated chat protocol with modern features. The main goal of the punctum.im project is to create a free and open chat platform with plenty of quality-of-life improvements.

The main features of punctum include:

* Federation - all servers compatible with the punctum protocol can federate, which means that people from different servers can still talk to each other.
* Conferences - groups of channels; when an user joins a conference, they gain access to the channels in the conference (given they are permitted to do so).
* Flexible permission system - roles and users in a conference can be assigned permissions for various aspects, and conferences and channels can have default permissions.
* Focus on moderation utilities - besides the permission system, there are various additions such as blocking and muting users, banning and kicking users from conferences/private groups and domain blocking for federation.

### Keywords

* User - User of an instance.
* Account - An account on an instance. Only used when mentioning the object type.
* Instance - The instance (running a punctum protocol compatible server).
* Conference - A group that contains channels.
* Channel - A channel of communication. DMs and group chats are also channels.
* Message - A single message, placed in a text channel.
* Invite

Additional note: all JSON fields that use "..." as the name are intended to be comments and shall be ignored.

## Objects and IDs

Most things, such as accounts, conferences, channels or messages, are **objects**.

Every object is assigned an **ID**. This ID is the object's main identicator and is used to quickly locate and access the object with the given ID. IDs are unique; different objects cannot have the same ID. IDs MUST be strings and MUST NOT contain any characters that are not letters, numbers, dashes or underscores. Additionally, the following IDs MUST NOT be used:

* ``by-name``
* ``by-time``

An object has the following required variables:

* ``id`` - this contains the ID of the object and MUST be a string.
* ``type`` - must be set to ``"object"``.
* ``object_type`` - contains the type of the object.

A standard REST API is used to access, modify and create objects. The API returns information in the JSON format. An example object recieved through an API endpoint:

```json
{
    "id": "1",
    "type": "object",
    "object_type": "account",
    "username": "TestAccount",
    "...": "etc."
}
```

Available object types, with explainations:

* ``instance`` - contains information about the instance, such as the server software, protocol version or domain.
* ``account`` - contains information about a user account. Note that this is not the same as a *registered account*, as account objects are only used to represent the user.
* ``conference`` - contains information about a conference, such as the name, channels it contains, its members or roles.
* ``conference_member`` - contains information about a conference member, such as the account ID of the member, their username, roles or permissions.
* ``channel`` - contains information about a channel in a conference or direct message channel/group, depending on the ``channel_type``.
* ``message`` - contains information about a message, such as its content or reactions.
* ``role`` - contains information about a role, such as its name, color or members.
* ``invite`` - contains information about an invite.
* ``report`` - contains information about a report.
* ``emoji`` - contains information about custom emoji.

To get information about an object by ID, the following API endpoint may be used:

```http
GET /api/v1/ids/{object_id}
```

All object types also have specific API endpoints, such as:

```http
GET /api/v1/accounts/{account_id}
GET /api/v1/conferences/{conference_id}
GET /api/v1/messages/{message_id}
(etc.)
```

**Exception:** The object type-specific endpoint for ``conference_member`` objects is ``/api/v1/conferences/{conference_id}/members/{conference_member_id}``.

The main instance object (our instance) MUST have an ID of "0":

```json
{
    "id": "0",
    "type": "object",
    "object_type": "instance",
    "name": "Example Instance",
    "description": "An example instance",
    "...": "etc."
}
```

Note that some objects may take IDs of other objects as values for certain keys; in these cases, the local ID MUST be used.

```json
{
    "id": "a",
    "type": "object",
    "object_type": "role",
    "members": ["account1_id", "account2_id", "account3_id"],
    "...": "etc."
}
```

Objects that have been recieved via federation are given three extra variables: ``remote_domain`` and ``remote_id``, which contain the domain of the instance the object originated on and its ID on the instance specified in the ``remote_domain`` variable, and ``known_ids`` - a list containing the IDs of the object on other known instances, which is added to every time the object needs to be added from said instance. These variables are properly explained in the Federation section.

### Stashes

Stashes are groups of objects returned in one query. They can be used to request larger amounts of IDs without having to make multiple separate requests, and is useful for federating objects with dependencies.

Stashes are **not** objects, and they do not have IDs. They have a ``type`` of ``stash``. For every object in the stash, a variable is added named after the ID of the object which has the object set as its value. There is a variable named ``id_list`` which is a list of IDs of all objects that were sent in the stash.

An example stash:

```json
{
    "type": "stash",
    "id_list": ["id1", "id2"],
    "id1": {
        "id": "id1",
        "type": "object",
        "object-type": "message",
        "...": "etc."
    },
    "id2": {
        "id": "id2",
        "type": "object",
        "object-type": "message",
        "...": "etc."
    }
}
```

## Pings

*Not to be confused with [mentions](#mentions).*

Pings are short requests sent between a client and a server during client-server communication.

Pings are *not** objects, and they do not have IDs. They have a ``type`` of ``ping``. Pings also have a ``ping_type`` variable, which contains the type of the ping.

### Request pings

Request pings are used to send information from a client to a server, and are equivalent to performing an API query. They are used in [client-server communication](#client-server-communication) and [federation](#federating-objects-and-actions-performed-on-objects). They have a ``ping_type`` of ``client_request`` and are sent over the client socket. They take three variables:

* ``request_method`` - the method of the request (``GET``, ``POST``, ``PATCH``, ``DELETE`` + ``JOIN`` for invites only + ``BLOCK`` for accounts only + ``REACT`` for messages only + ``REPORT`` for objects that support [Reporting](#reporting-objects))
* ``request_target_id`` - the ID to make changes to
* ``request_data`` - the data to be sent with the request (required for ``POST`` and ``PATCH`` requests)

In the case of request pings sent to the federationo inbox, ``JOIN``, ``BLOCK`` and ``REACT`` methods also require an additional variable, ``request_source_user``, which contains the remote ID of the account to perform the action on behalf of.

**Example request ping:**

```json
{
    "type": "ping",
    "ping_type": "client_request",
    "request_method": "PATCH",
    "request_target_id": "example",
    "request_data": {
        "username": "Example Account"
    }
}
```

### Error pings

To make diagnosing API errors easier, the API MUST return errors in the form of error pings.

Error pings have a ``ping_type`` of ``error``. They are returned as a result of errors that occured while processing API endpoint queries. They contain information about an error: the error code (``error_code``, stored as a number) and a human-readable error description (``error``).

**Example error ping:**

```json
{
    "type": "ping",
    "ping_type": "error",
    "error_code": 2,
    "error": "Missing data, or Content-Type is not application/json"
}
```

#### List of error codes and corresponding HTTP response status codes

* ``0`` - No error (unused) (``200 OK``)
* ``1`` - Server-side error (``5xx``); clients MUST be able to tell that there's a server-side error from the HTTP response status code alone
* ``2`` - Missing data, or Content-Type is not application/json (``400 BAD REQUEST``)
* ``3`` - Not permitted to access object (``403 FORBIDDEN``)
* ``4`` - Object not found (``404 NOT FOUND``)
* ``5`` - Type of requested object is not supported by endpoint (for example, when a message is requested through the ``/api/v1/accounts`` endpoint) (``400 BAD REQUEST``)
* ``6`` - Attempted to change non-rewritable variable (``400 BAD REQUEST``)
* ``7`` - Missing required variable in provided data (``400 BAD REQUEST``)
* ``8`` - Object does not belong to object (for example, when trying to use ``/api/v1/conferences/x/channels/y`` when the channel with the ID ``y`` belongs to another conference) (``400 BAD REQUEST``)
* ``9`` - Object with the ID provided in a variable that takes an ID does not exist (``404 NOT FOUND``)
* ``10`` - Object with the ID provided in a variable that takes an ID is not of the correct type for this variable (``400 BAD REQUEST``)
* ``11`` - Too many objects provided (``400 BAD REQUEST``)

## Federation

**Federation** is the ability to exchange information with other instances of software using the same protocol.

Every instance has a **public inbox**. This is where other instances can send in objects for the target instance.

```http
POST /api/v1/federation/inbox
```

When the inbox is queried with a GET request, it returns information about our instance (ID "0").

Objects sent to an instance's inbox MUST contain the object's ID from the sender instance in the "id" value.

Objects recieved through federation get three new variables:

* ``remote_domain`` - contains the origin domain of the remote object. This MUST contain the domain for the instance from which the object originated, not the domain which it was first found on.
* ``remote_id`` - contains the ID of the object on the origin instance.
* ``known_ids`` - contains a dict of remote domains and IDs of the object on those instances. This allows for easily getting the origin ID of an object when we recieve it from a different instance than the origin instance (common case scenario for object dependencies). This MUST NOT be federated, as it can be considered a data leak.

For all federation-related queries and APIs, the remote ID MUST be used.

### Federating objects and actions performed on objects

The federation inbox accepts objects and stashes. When an object is sent to the inbox, it is added to the instance. If the sent object already exists on the target instance, the recieved object will be treated as an updated version of the object.

The federation inbox uses [request pings](#request-pings) to represent actions performed on objects. For example, to federate the deletion of an object, the server would send a request ping with a ``request_type`` of ``DELETE``. Request pings are also used to add users to conferences (which is required for a subscription to said conference).

### Subscriptions

Every account, conference and direct message channel can be subscribed to. Once an instance subscribes to the account, conference or direct message channel, any changes to it, its child objects (for example channels and messages) or its dependencies will automatically be sent to the subscribed instance's inbox.

When any account is requested or when a conference or direct message channel is joined, it will automatically be subscribed to.

To unsubscribe from an object, the ``/api/v1/federation/unsubscribe/<id>`` endpoint may be used.

### Stashing

*This section is non-normative.*

*Not to be confused with stashes.*

When a remote instance goes offline, the sender instance may keep a list of IDs to send to the remote instance. Once the remote instance is back online (which is checked by using the ``GET /api/v1/federation/inbox`` endpoint) the IDs are sent in stashes with a small delay to prevent senders from overloading the remote instance.

### Example federation workflow

An user on our instance wants to join a conference created on a remote instance. They recieve an invite, which is then sent to our instance's inbox by the remote instance as they use it. Once the user has used the invite, we have to communicate that to the remote server.

First, we make sure the user's Account object has already been sent to the remote server. We check the ``known_ids`` of the user and see if the remote instance is present. If it isn't, we send the account to their inbox:

```http
POST remote.instance/api/v1/federation/inbox
```

**Sent data:** our user's Account object

```json
{
    "id": "our_user",
    "type": "object",
    "object_type": "account",
    "username": "TestAccount",
    "...": "etc."
}
```

The remote server takes the request and returns the object we just gave them, including the ID they assigned:

```json
{
    "id": "our_users_id_on_the_remote_server",
    "remote_domain": "our.instance",
    "remote_id": "our_user",
    "type": "object",
    "object_type": "account",
    "username": "TestAccount",
    "...": "etc."
}
```

We then add the ID to our user's known_ids:

```json
"known_ids": {
    "remote.domain": "our_users_id_on_the_remote_server"
}
```

Once we have our user's Account's ID, we can request and subscribe to the conference from the remote instance by joining it:

```http
POST remote.instance/api/v1/federation/join-conference-by-invite/invite_id
```

**Sent data:**

```json
{
    "user_id": "our_users_id_on_the_remote_server"
}
```

The remote instance compiles a stash containing the conference object, the roles in the conference, the chanels available by default and the members that can be found in the available channels. It also sends the 50 latest messages from each available channel.

We soon recieve the stash:

```json
{
    "type": "stash",
    "id_list": ["conference", "channel1", "channel2", "message1", "message2", "role1", "account1", "account2"],
    "conference": {
        "id": "conference",
        "type": "object",
        "object_type": "conference",
        "...": "etc."
   },
   "...": "etc."
}
```

We begin taking each object from the stash one-by-one, giving it a remote_domain and remote_id, as well as assigning it our own local ID:

```json
{
    "remote_domain": "remote.instance",
    "remote_id": "conference",
    "id": "local_id_for_the_conference",
    "type": "object",
    "object_type": "conference",
    "...": "etc."
}
```

We iterate over this stash until we stumble upon an object that has a remote_domain and remote_id already:

```json
    "account2": {
        "remote_domain": "far-away.instance",
        "remote_id": "far_account",
        "id": "account2",
        "type": "object",
        "object_type": "conference",
        "...": "etc."
   }
```

We re-request the object from far-away.instance. This is done for two reasons:

* to verify the integrity of the object;
* and to make sure we're not blocked by far-away.instance.

We save the object from far-away.instance and add the object's ID on remote.instance to ``known_ids``:

```json
{
    "remote_domain": "far-away.instance",
    "remote_id": "far_account",
    "known_ids": { "remote.instance": "account2" },
    "id": "local_id_for_the_account",
    "type": "object",
    "object_type": "account",
    "...": "etc."
}
```

Now that we have added all the objects the remote server has sent us, we can now let our user access the conference.

## Permission system

Permissions can be used to allow or deny access to certain features for conference members. They can be assigned to conference members, roles, channels and conferences.

Conferences, channels and roles can have their own default permission sets. Permissions are applied in the following order:

1. Conference
2. Channel
3. Role
4. Conference member

Where each permission set overrides the previous set.

Permissions are stored as a number:

```json
"permissions": 7
```

To calculate permissions, add all the numbers corresponding to the permissions you want to set. Every permission enabled will toggle a 0 or 1 bit in the binary output, for example:

* 1 - ``10000``
* 2 - ``01000``
* 1 + 2 = 3 - ``11000``
* 1 + 4 = 5 - ``10100``

### Permission numbers

* 1 - See channel
* 2 - Read messages in/connect to channel
* 4 - Send messages to/speak in channel
* 8 - Edit and delete own messages
* 16 - Pin own messages, pin or delete other users' messages
* 32 - Create and modify invites
* 64 - Modify channel information (name, description)
* 128 - Change own nickname
* 256 - Change other users' nicknames
* 512 - Kick users
* 1024 - Ban users
* 2048 - Modify and assign roles
* 4096 - Modify conference

The last two permissions MUST be ignored for direct message channels.

## Authentication

Allowing objects to be accessed by anybody is a large security risk. As such, to use most API endpoints, the user MUST be authenticated. Servers MUST provide authentication features.

Servers SHOULD implement a way for users to create and log into user accounts. Each user account is assigned an Account object which is used to represent the user.

### OAuth2 scopes

*This section is non-normative.*

OAuth2 is RECCOMENDED for client authentication. To unify the authentication process across various implementations, it is reccomended to follow this section.

**TODO:** Write this once we're done with OAuth2 authentication in drywall.

## Client-server communication

Clients can communicate with servers in two ways:

* locally - for example, when the client is installed on the same machine as the server.
* remotely - via a WebSocket.

Clients can implement either of those communication methods, or both at once.

Information is streamed between sockets in the form of objects, stashes and pings (explained below).

For local communication, a socket is set up on the machine where the server and client are located. The client recieves data from and sends data to the server through the created socket.

For remote communication, the client sends and recieves requests through a WebSocket located at:

```http
/api/v1/client/socket
```

The endpoint for the WebSocket MUST require authentication.

## Reporting objects

Objects can be reported to the instance admins by using the ``/api/v1/id/<id>/report`` endpoint.

# Chapter 2. Object types

The following chapter contains information about available object types, their descriptions and explanations, as well as required variables.

**Remember!** IDs are always strings.

## Instance

``"object_type": "instance"``

An **instance** object contains information about an instance, such as its domain, name, description and the server software it's running.

Every instance saves information about itself in an object with the ID of "0".

**Required keys:**
{:.required-keys-heading}

| Key             | Value type | Required? | Require authentication?         | Read/write | Federate? | Notes                                                                                 |
|-----------------|------------|-----------|---------------------------------|------------|-----------|---------------------------------------------------------------------------------------|
| address          | string     | yes       | r: no                          | r          | yes       | Contains the domain name for the instance. Required for federation.  MUST NOT CHANGE. |
| server_software  | string     | yes       | r: no                          | r          | yes       | Contains the name and version of the used server software.                            |
| protocol_version | number     | yes       | r: no                          | r          | yes       | Contains the version of the protocol. For the current revision, this MUST be set to ``1``. |
| name             | string     | yes       | r: no; w: yes [instance:modify] | r[w]       | yes       | Contains the name of the server. This can be changed by an user.                      |
| description      | string     | no        | r: no; w: yes [instance:modify] | r[w]       | yes       | Contains the description of the server. This can be changed by an user.     |
{:.required-key-table}

## Account

``"object_type": "account"``

An **account** object contains information about an account, such as the username, status, bio and friend/blocklists.

Accounts can own conferences, direct message channels and messages.

**Required keys:**
{:.required-keys-heading}

| Key             | Value type | Required?    | Require authentication?         | Read/write | Federate? | Notes                                                                       |
|-----------------|------------|--------------|---------------------------------|------------|-----------|-----------------------------------------------------------------------------|
| username        | string     | yes          | r: no; w: yes [account:modify]  | rw         | yes       | Instance-wide username, used for friend requests and identification purposes. There MUST NOT be two users with the same username. This MUST NOT contain non-alphanumeric characters, but can contain dashes and underscores, and MUST NOT be longer than 128 characters. |
| display_name    | string     | yes          | r: no; w: yes [account:modify]  | rw         | yes       | Display name. Can contain non-alphanumeric characters. |
| short_status    | number     | yes          | r: no; w: yes [account:modify]  | rw         | yes       | Short status. 0 - offline, 1 - online, 2 - away, 3 - do not disturb         |
| status          | string     | no           | r: no; w: yes [account:modify]  | rw         | yes       | User status.                                                                |
| bio             | string     | no           | r: no; w: yes [account:modify]  | rw         | yes       | User bio. Hashtags can be taken as profile tags and used in search engines. |
| index_user      | bool       | yes          | r: no; w: yes [account:modify]  | rw         | yes       | Can the user be indexed in search results? MUST be ``false`` by default.    |
| bot             | bool       | no           | r: no; w: yes [account:modify]  | rw         | yes       | Is the user a bot? See the Accounts > Bots section.                         |
| bot_owner       | string     | if bot=true  | r: no; w: yes [account:modify]  | rw         | yes       | The bot's owner.                                                            |
| friends   | list of IDs | no        | r: yes [user needs to be authenticated]         | r          | no        | Contains IDs of the users the account is friends with. This MUST be handled through friend requests, to prevent users from adding people to their friend list without their prior permission. |
| blocklist | list of IDs | no        | r: yes, w: yes [user needs to be authenticated] | rw         | no        | Contains IDs of blocked users. |
| joined_conferences | dict | yes | r: yes, w: yes [user needs to be authenticated] | rw | no | Contains key-value pairs for conferences that the user is a part of, where the key is the conference ID and the value is the conference member ID of this particular user. |
{:.required-key-table}

### Blocking

Users can block other users.

### Permissions

what

## Conference

``"object_type": "conference"``

A conference is a group comprising of any amount of text and voice channels. Users can join a conference, and roles can be given to them. These roles can have certain permissions assigned to them.

| Key           | Value type  | Required? | Require authentication?           | Read/write | Federate? | Notes                                                                                          |
|---------------|-------------|-----------|-----------------------------------|------------|-----------|------------------------------------------------------------------------------------------------|
| name          | string      | yes       | r: no; w: yes [xx3xx permissions] | rw         | yes       | Name of the conference.                                                                        |
| description   | string      | no        | r: no; w: yes [xx3xx permissions] | rw         | yes       | Description of the conference.                                                                 |
| icon          | string      | yes       | r: no; w: yes [xx3xx permissions] | rw         | yes       | URL of the conference's icon. Servers MUST provide a placeholder.                              |
| owner         | ID          | yes       | r: no; w: yes [user needs to be authenticated and be the owner of the conference] | rw | yes | ID of the conference's owner. MUST be an account. Initially assigned at conference creation by the server. |
| index_conference | bool        | yes       | r: no; w: yes [user needs to be owner] | rw    | yes       | Should the conference be indexed in search results? SHOULD default to ``false``.               |
| permissions   | string      | yes       | r: no; w: yes [xx3xx permissions] | rw         | yes       | Conference-wide permission set, stored as a permission map.                                    |
| creation_date | string      | yes       | r: no                             | r          | yes       | Date of the conference's creation. Assigned by the server.                                     |
| channels      | list of IDs | yes       | r: no                             | r          | yes       | List of IDs of channels present in the conference. Assigned by the server at channel creation. |
| members       | list of IDs | yes       | r: yes [user needs to be authenticated and in the conference] | r | yes | List of conference_member objects (people who have joined the conference). Modified by the server when a user joins. |
| roles         | list of IDs | no        | r: yes [xxx1x permissions]        | r          | yes       | List of IDs of roles present in the conference. Modified by the server when a role is added.   |
{:.required-key-table}

## Conference member

``"object_type": "conference_member"``

A conference member object contains information about a member of a conference specific to that conference.

| Key           | Value type  | Required? | Require authentication?                                           | Read/write | Federate? | Notes                                  |
|---------------|-------------|-----------|-------------------------------------------------------------------|------------|-----------|----------------------------------------|
| user_id       | ID          | yes       | r: yes [must be a part of the conference];                        | r          | yes       | The user's ID on the local server (when federating, set this to the ID that the user has on your server). |
| parent_conference | ID          | yes       | r: yes [must be a part of the conference];                        | r          | yes       | ID of the conference that this conference member object applies to. |
| nickname      | string      | no        | r: yes [must be a part of the conference]; w: yes [xxxx2 and up]; | rw         | yes       | The user's nickname on the conference. |
| roles         | list of IDs | no        | r: yes [must be a part of the conference]; w: yes [xxxx2 and up]; | rw         | yes       | Contains the user's roles' ID.         |
| permissions   | string      | yes       | r: yes [must be a part of the conference]; w: yes [xxxx2 and up]; | rw         | yes       | The user's permissions, in a permission map. |
| banned        | bool        | no        | r: yes [must be a part of the conference];                        | r          | yes       | Is the user banned? Modified by the server at ban/unban time. |
{:.required-key-table}

### Banning a conference member

A user can be banned from a conference. This means they cannot join or access the conference.

If a user was banned, their ID can still be queried through the API endpoint, but only contains the following information:

* "banned" - bool, true
* "permissions" with the conference bit set to 0

These are set by the server at ban time.

## Channel

``"object_type": "channel"``

Channels are channels of communication that can be nested within conferences or stay standalone as group DMs. There are three types of channels: text channels, media channels (voice and video chat) and direct message channels.

| Key             | Value type | Required? | Require authentication?             | Read/write | Federate? | Notes                                                                                 |
|-----------------|------------|-----------|-------------------------------------|------------|-----------|---------------------------------------------------------------------------------------|
| channel_type      | string     | yes       | r: yes*; w: yes**                 | r          | yes       | Contains the type of the channel. Can be ``text``, ``media`` or ``direct_message``. |
| parent_conference | ID         | if channel-type is ``text`` or ``media`` | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the ID of the conference the channel is in. |
| name              | string     | yes       | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the name of the channel. If the channel is a direct message channel with one participant, the name is set to the IDs of the users in the conversations, separated by a space.|
| description       | string     | no        | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the name and version of the used server software.                            |
| permissions       | string     | yes       | r: yes*; w: yes [x3xxx permissions] | rw         | yes       | Contains the permissions for the channel. This is a permission map.                   |
{:.required-key-table}

``* Must require prior per-user authentication. Direct message channels require the user to be a part of the direct message. Channels in conferences require the user to join the conference.

** Only writeable when the object is created, not re-writable.``

### Channel types

#### Text channels

Text channels have the ``channel_type`` of ``text``. They are capable of storing messages.

Text channels MUST be attached to a conference. The conference the channel is placed in is stored in the ``parent_conference`` value.

#### Media channels

Media channels have the ``channel_type`` of ``media``. They are used for the transport of voice and video. They MUST be attached to a conference. The conference the channel is placed in is stored in the ``parent_conference`` value.

> TODO: How are audio and video going to be transported? Do some research on this.

#### Direct message channels

Direct message channels can transport both text and media. They have the same API calls as both text and media channels. They have the ``channel_type`` of ``direct_message``.

Direct message channels MUST NOT be attached to a conference. The ``parent_conference`` MUST be ignored when paired with direct message channels.

#### Categories

Categories have the ``channel_type`` of ``category``. They can group text and media channels in a conference.

Categories MUST be attached to a conference. The conference the channel is placed in is stored in the ``parent_conference`` value.

## Message

``"object_type": "message"``

Message objects contain text messages.

| Key             | Value type | Required? | Require authentication?                                                 | Read/write | Federate? | Notes                                                                              |
|-----------------|------------|-----------|-------------------------------------------------------------------------|------------|-----------|------------------------------------------------------------------------------------|
| content         | string     | yes       | r: no; w: yes [must be authenticated as the user who wrote the message] | rw         | yes       | Message content. Any further writes are counted as edits.                          |
| parent_channel  | ID         | yes       | r: no                                                                   | r          | yes       | ID of the channel in which the message has been posted. Assigned by the server at message creation. |
| author          | string     | yes       | r: no                                                                   | r          | yes       | ID of the message author. Assigned by the server at message creation.              |
| post_date       | string     | yes       | r: no                                                                   | r          | yes       | Date of message creation. Assigned by the server at message creation.              |
| edit_date       | string     | no        | r: no                                                                   | r          | yes       | Date of last message edit. Assigned by the server at message edit.                 |
| edited          | bool       | yes       | r: no                                                                   | r          | yes       | Is the message edited? Defaults to ``false``. Set by the server at message edit.   |
| reactions       | list of strings | no   | r: no; w: yes [must be able to read the message]                        | rw         | yes       | List of emoji shortcodes.                                                          |
| attached_files  | list of strings | no   | r: no; w: yes [must be authenticated as the user who wrote the message] | rw         | yes       | List of files that have been attached to the message. Contains URLs to the files. Any further writes are counted as edits. |
| reply_to        | ID          | no | r: no; w: yes [must be authenticated as the user who wrote the message] | rw | yes | ID of the message that this message is a reply to. |
| replies         | list of IDs | no | r: no | r | yes | List of IDs of messages that are replies to this message |
| embeds          | list of dicts | no | r: no; w: yes [must be authenticated as the user who wrote the message] | rw | yes | List containing embed dicts (see [Embeds](#embeds)). |
{:.required-key-table}

### Formatting

Formatting in messages is stored in basic HTML-esque tags:

* ``<b>``, ``</b>`` - bold text (equivalent to markdown's ``**``)
* ``<i>``, ``</i>`` - italic text (equivalent to markdown's ``*``)
* ``<u>``, ``</u>`` - underlined text (equivalent to markdown's ``__``)
* ``<s>``, ``</s>`` - strikethrough text (equivalent to markdown's ``~~``)
* ``<code>``, ``</code>`` - single-line code blocks (equivalent to markdown's ``)
* ``<pre>``, ``</pre>`` - multi-line code blocks (equivalent to markdown's ```)

### Mentions

It's possible to mention an user, role or everyone in a conference, as well as link to a channel using the ``<@ID>`` tag, where ``ID`` is the ID of the user/role/conference/channel that you want to mention/link to.

### Replies

A message can be set as a reply to another message. This will set the ``reply_to`` variable to the replied-to message's ID and add the message's ID to the ``replies`` list in the replied-to message.

### Attaching files

It is possible to attach a file to a message by adding a link to the file in the ``attached_files`` list.

### Embeds

Embeds are small boxes which contain text, alongside a side-color and an image. Two types of embeds exist: the side-image embed and the main-image embed.

```txt
__________________________      _________________
| Title           -------|      | Title         |
| Description     --IMG--|      | Description   |
|_________________-------|      | ------------- |
                                | -----img----- |
         TYPE 1                 | ------------- |
                                |_______________|

                                     TYPE 2
```

| Key               | Value type | Required? | Require authentication?                        | Read/write | Federate? | Notes                           |
|-------------------|------------|-----------|------------------------------------------------|------------|-----------|---------------------------------|
| embed             | bool       | no        | r: no; w: yes [user needs to be authenticated] | rw         | yes       | Whether there should be an embed or not. |
| embed_title       | string     | yes       | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The title.                      |
| embed_description | string     | no        | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The description.                |
| embed_color       | string     | no        | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The color, in RGB ("R, G, B")   |
| embed_image       | string     | no        | r: no; w: yes [user needs to be authenticated] | rw         | yes       | The link to the embedded image  |
| embed_type        | number     | yes       | r: no; w: yes [user needs to be authenticated] | rw         | yes       | 1 or 2 (1 - type 1, 2 - type 2) |
{:.required-key-table}

## Invite

``"object_type": "invite"``

Users can join a conference through an invite.

| Key           | Value type | Required? | Require authentication?           | Read/write | Federate? | Notes                                  |
|---------------|------------|-----------|-----------------------------------|------------|-----------|----------------------------------------|
| name          | string     | yes       | r: no, w: yes [xx3xx permissions] | rw         | yes       | Contains the invite's name.            |
| conference_id | ID         | yes       | r: no                             | r          | yes       | Contains the ID of the conference the invite leads to. MUST NOT be changeable. SHOULD be verified through the ``/api/v1/conference/invites/$INVITEID`` endpoint. |
| creator       | string     | yes       | r: no                             | r          | yes       | Contains the ID of the account that created the invite. Assigned by the server at invite creation. |
{:.required-key-table}

## Role

``"object_type": "role"``

| Key           | Value type | Required? | Require authentication?           | Read/write | Federate? | Notes                                                                                           |
|---------------|------------|-----------|-----------------------------------|------------|-----------|-------------------------------------------------------------------------------------------------|
| name          | string     | yes       | r: no; w: yes [2048 (Modify and assign roles)] | rw         | yes       | Name of the role.                                                                               |
| description   | string     | no        | r: no; w: yes [2048 (Modify and assign roles)] | rw         | yes       | Short description of the role.                                                                  |
| color         | slist of numbers | no        | r: no; w: yes [2048 (Modify and assign roles)] | rw         | yes       | Color of the role, in RGB ([R, G, B]) (does not support alpha)). |
| permissions   | string     | no        | r: no; w: yes [2048 (Modify and assign roles)] | rw         | yes       | Permissions for the role, as a permission map.                                                  |
{:.required-key-table}

## Report

``"object_type": "report"``

Contains information about a report.

| Key           | Value type | Required? | Require authentication?           | Read/write | Federate? | Notes                                   |
|---------------|------------|-----------|-----------------------------------|------------|-----------|-----------------------------------------|
| target        | ID         | yes       | yes (must be an admin or the report's creator) | rw | yes  | Contains the ID of the reported object. |
| note          | string     | no        | yes (must be an admin or the report's creator) | rw | yes  | Contains a note about the report. |
{:.required-key-table}

## Custom emoji

``"object_type": "emoji"``

Contains information about a custom emoji.

| Key           | Value type | Required? | Require authentication?           | Read/write | Federate? | Notes                                   |
|---------------|------------|-----------|-----------------------------------|------------|-----------|-----------------------------------------|
| shortcode     | ID         | yes       | r: no; w: yes [8192 (Modify conference)]| rw | yes  | Contains the ID of the reported object. |
| note          | string     | no        | yes (must be an admin or the report's creator) | rw | yes  | Contains a note about the report. |
{:.required-key-table}

# Chapter 3. API method reference

## Objects, stashes and our instance

### **GET** /api/v1/instance

**Returns** information about the instance.

**Responses:**

* ``200 OK`` - Success

### **POST** /api/v1/id

**Takes** an object and **sends** it to the server. **Returns** the created object.

**Responses:**

* ``201 CREATED`` - Success
* ``400 BAD REQUEST`` - No data provided, or Content-Type is not application/json

### **GET** /api/v1/id/{id}

**Takes** an object ID and **returns** the object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``403 FORBIDDEN`` - Not permitted to access the object (see [Authentication](#authentication) and [Permissions](#permissions))
* ``404 NOT FOUND`` - No object with given ID found

### **PATCH** /api/v1/id/{id}

**Takes** an object ID and **patches** the object with the given ID. Returns the patched object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided, or Content-Type is not application/json
* ``403 FORBIDDEN`` - Not permitted to access or edit the object (see [Authentication](#authentication) and [Permissions](#permissions))
* ``404 NOT FOUND`` - No object with given ID found

### **DELETE** /api/v1/id/{id}

**Takes** an object ID, **deletes** the object with the given ID and **returns** the ID of the deleted object.

**Responses:**

* ``200 OK`` - Success
* ``403 FORBIDDEN`` - Not permitted to access or edit/delete the object (see [Authentication](#authentication) and [Permissions](#permissions))
* ``404 NOT FOUND`` - No object with given ID found

### **POST** /api/v1/stash/request

**Takes** a list of IDs stored in the ``id_list`` variable and **returns** a stash with the requested IDs.

**Example sent data:**

```json
{
    "id_list": [ "1", "2", "3" ]
}
```

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided, or Content-Type is not application/json

## Federation endpoints

### **GET** /api/v1/federation/inbox

**Returns** information about the instance (ID 0). Used by other instances to check if the remote instance is up.

**Responses:**

* ``200 OK`` - Success

### **POST** /api/v1/federation/inbox

**Sends** data to the instance's inbox. Data can be sent in the form of stashes or objects.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json

### **POST** /api/v1/federation/subscribe/{target_id}

**TODO:** Candidate for rename

**Takes** the ID of the account object on the server it is being requested from (``{target_id}``) and subscribes to it. See [Subscriptions](#subscriptions).

### **POST** /api/v1/federation/join-conference-by-invite/{invite_id}

**TODO:** Candidate for rename

**Takes** the invite ID (``{invite_id}``) on the remote instance and, in the sent data, the ID of the user that will join the conference (``user_id``). Once the remote server recieves the request, it MUST subscribe to the account.

**Example sent data:**

```json
{
    "user_id": "our_user_id"
}
```

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided, missing required variable in data, provided invite ID does not belong to an invite or Content-Type is not application/json
* ``404 NOT FOUND`` - Invite with given ID not found

## Authentication and clients

### /auth/sign_up

Sign-up page, provided by the server. See the [Authentication](#authentication) section for implementation details.

### /auth/login

Login page, provided by the server. See the [Authentication](#authentication) section for implementation details.

### /api/v1/client/socket

Client websocket. See the [Client-server communication](#client-server-communication) section for implementation details.

### /client

Should redirect to a web client.

**OAuth2 endpoints:**
{:.separator-heading}

### GET /auth/oauth2/

## Account endpoints

### **GET** /api/v1/accounts

**Returns** the Account object for the currently authenticated account.

**Responses:**

* ``200 OK`` - Success
* ``401 UNAUTHORIZED`` - Not authenticated

### **POST** /api/v1/accounts

**Creates** a new Account object with the provided data. **Returns** the newly created object. This is not equivalent to creating a new account; see the [Authentication](#authentication) section for more information.

**Responses:**

* ``201 CREATED`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not an account (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated; cannot create objects

### **GET** /api/v1/accounts/{account_id}

**Takes** an account ID (``{account_id}``) and **returns** the Account object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not an account (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/accounts/{account_id}

**Takes** an account ID (``{account_id}``) and patch data, **patches** the Account object with the given ID and **returns** the patched Account object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not an account (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/accounts/{account_id}

**Takes** an account ID (``{account_id}``), **deletes** the Account object with the given ID and **returns** the ID of the deleted Account object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an account (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/accounts/by-name/{account_name}

**Takes** the username of a local account, without the leading @ and trailing @domain (``{account_name}``) and **returns** the Account object with the given username.

**Responses:**

* ``200 OK`` - Success
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/accounts/by-name/{account_name}

**Takes** the username of a local account, without the leading @ and trailing @domain (``{account_name}``) and patch data, **patches** the Account object with the given name and **returns** the patched Account object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/accounts/by-name/{account_name}

**Takes** the username of a local account, without the leading @ and trailing @domain (``{account_name}``), **deletes** the Account object with the given name and **returns** the ID of the deleted Account object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an account (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/accounts/{account_id}/block

Blocks the account with the given ID (``{account_id}``) as the currently authenticated user.

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not an account (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/accounts/by-name/{account_name}/block

Blocks the account with the given local account username (``{account_name}``) as the currently authenticated user.

* ``200 OK`` - Success
* ``401 UNAUTHORIZED`` - Not authenticated
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

## Conference endpoints

### **POST** /api/v1/conferences

**Creates** a new Conference object with the provided data. **Returns** the newly created object.

**Responses:**

* ``201 CREATED`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a conference (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated; cannot create objects

### **GET** /api/v1/conferences/{conference_id}

**Takes** a conference ID (``{conference_id}``) and **returns** the Conference object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a conference (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/conferences/{conference_id}

**Takes** a conference ID (``{conference_id}``) and patch data, **patches** the Conference object with the given ID and **returns** the patched Conference object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID/object given in patch data is not a conference (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/conferences/{conference_id}

**Takes** a conference ID (``conference_id``), **deletes** the Conference object with the given ID and **returns** the ID of the deleted Conference object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a conference (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/conferences/{conference_id}/leave

Leaves the conference as the currently authenticated user.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a conference (error code: 5)
* ``400 BAD REQUEST`` - Currently authenticates user not in conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

**Conference members:**
{:.separator-heading}

### **GET** /api/v1/conferences/{conference_id}/members

**Takes** a conference ID and **returns** a stash containing all conference members that the user is authorized to access in the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a ConferenceMember object (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/conferences/{conference_id}/members/{member_id}

**Takes** a conference ID and the ID of a conference member or account ID of the member and **returns** the ConferenceMember object with the given ID if it belongs to the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a ConferenceMember object nor an account (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/conferences/{conference_id}/members/{member_id}

**Takes** a conference ID and the ID of a conference member or account ID of the member, **patches** the ConferenceMember object and **returns** the patched ConferenceMember object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a ConferenceMember object nor an account/object given in patch data is not a ConferenceMember object (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/conferences/{conference_id}/members/{member_id}

**Takes** a conference ID and the ID of a conference member in the conference, **deletes** the ConferenceMember object with the given ID and **returns** the ID of the deleted ConferenceMember object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a conference member (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID is not a ConferenceMember object nor an account/object given in patch data is not a ConferenceMember object (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

**Conference roles:**
{:.separator-heading}

### **GET** /api/v1/conferences/{conference_id}/roles

**Takes** a conference ID and **returns** a stash containing all roles that the user is authorized to access in the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a role (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/conferences/{conference_id}/roles

**Creates** a new role with the ``parent_conference`` set to the provided conference ID. MUST overwrite the ``parent_conference`` variable if a different one is provided. **Returns** the created Role object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a conference/object given in patch data is not a role (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/conferences/{conference_id}/roles/{role_id}

**Takes** a conference ID and the ID of a role in the conference and **returns** the Role object with the given ID if it belongs to the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a role (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/conferences/{conference_id}/roles/{role_id}

**Takes** a conference ID and the ID of a role in the conference, **patches** the role and **returns** the patched Role object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID/object given in patch data is not a role (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/conferences/{conference_id}/roles/{role_id}

**Takes** a conference ID and the ID of a role in the conference, **deletes** the Role object with the given ID and **returns** the ID of the deleted Role object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a role (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID is not a Role object (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

**Conference invites:**
{:.separator-heading}

### **GET** /api/v1/conferences/{conference_id}/invites

**Takes** a conference ID and **returns** a stash containing all invites that the user is authorized to access in the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/conferences/{conference_id}/invites

**Creates** a new invite with the ``conference_id`` set to the provided conference ID. MUST overwrite the ``conference_id`` variable if a different one is provided. **Returns** the created Channel object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a conference/object given in patch data is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/conferences/{conference_id}/invites/{invite_id}

**Takes** a conference ID and the ID of an invite in the conference and **returns** the Invite object with the given ID if it belongs to the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not an invite (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/conferences/{conference_id}/invites/{invite_id}

**Takes** a conference ID and the ID of an invite in the conference, **patches** the invite and **returns** the patched Invite object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID/object given in patch data is not an invite (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/conferences/{conference_id}/invites/{invite_id}

**Takes** a conference ID and the ID of an invite in the conference, **deletes** the Invite object with the given ID and **returns** the ID of the deleted Invite object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a invite (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID is not an Invite object (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

**Conference channels:**
{:.separator-heading}

### **GET** /api/v1/conferences/{conference_id}/channels

**Takes** a conference ID and **returns** a stash containing all channels that the user is authorized to access in the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a channel (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/conferences/{conference_id}/channels

**Creates** a new Channel object with the ``parent_conference`` set to the provided conference ID. MUST overwrite the ``parent_conference`` variable if a different one is provided. **Returns** the created Channel object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a conference/object given in patch data is not a channel (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/conferences/{conference_id}/channels/{channel_id}

**Takes** a conference ID and the ID of a channel in the conference and **returns** the Channel object with the given ID if it belongs to the given conference.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a channel (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/conferences/{conference_id}/channels/{channel_id}

**Takes** a conference ID and the ID of a channel in the conference, **patches** the channel and **returns** the patched Channel object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID/object given in patch data is not a channel (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/conferences/{conference_id}/channels/{channel_id}

**Takes** a conference ID and the ID of a channel in the conference, **deletes** the ObjCapsType object with the given ID and **returns** the ID of the deleted ObjCapsType object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a channel (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID is not a ObjCapsType object (error code: 5)
* ``400 BAD REQUEST`` - Object with provided ID does not belong to the given conference (error code: 7)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

## Channel endpoints

### **POST** /api/v1/channels

**Creates** a new Channel object with the provided data. **Returns** the newly created object.

**Responses:**

* ``201 CREATED`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a channel (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated; cannot create objects

### **GET** /api/v1/channels/{channel_id}

**Takes** a channel ID (``{channel_id}``) and **returns** the Channel object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a channel (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/channels/{channel_id}

**Takes** a channel ID (``{channel_id}``) and patch data, **patches** the Channel object with the given ID and **returns** the patched Channel object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a channel (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/channels/{channel_id}

**Takes** a channel ID (``channel_id``), **deletes** the Channel object with the given ID and **returns** the ID of the deleted Channel object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a channel (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/channels/{channel_id}/messages

Optionally **takes**:

* ``before_message`` - returns X messages before the message with the provided ID
* ``after_message`` - returns X messages after the message with the given ID
* ``message_count`` - amount of messages that will be sent back; this should have a sensible limit (about 100; this SHOULD be the default)

**Returns** the latest X messages in the channel with the provided ID (``{channel_id}``) as a stash.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a channel/object provided in patch data is not a message (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/channels/{channel_id}/messages

**Creates** a new message in the specified channel (``{channel_id}``). If the ``channel_id`` variable in the Message object does not match the provided channel ID (``{channel_id}``}), the server MUST overwrite the channel ID in the object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a channel/object provided in patch data is not a message (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/channels/{channel_id}/messages/by-time/{minutes}

**Returns** all messages in the given channel ``{channel_id}`` from ``{minutes}`` minutes ago as a stash.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a channel/provided minutes value is not a number (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

## Message endpoints

### **POST** /api/v1/messages

**Creates** a new Message object with the provided data. **Returns** the newly created object.

**Responses:**

* ``201 CREATED`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a message (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated; cannot create objects

### **GET** /api/v1/messages/{message_id}

**Takes** a message ID (``{message_id}``) and **returns** the Message object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Object with provided ID is not a message (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/messages/{message_id}

**Takes** a message ID (``{message_id}``) and patch data, **patches** the Message object with the given ID and **returns** the patched Message object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a message (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/messages/{message_id}

**Takes** a message ID (``message_id``), **deletes** the Message object with the given ID and **returns** the ID of the deleted Message object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a message (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/messages/{message_id}/react

**Takes** a message ID (``{message_id}``) and the reaction (``reaction``) and reacts with the given reaction to the message as the currently authenticated user. If the currently authenticated user already used this reaction on this message, the reaction will be cancelled. **Returns** all message reactions.

**Example sent data:**

```json
{
    "reaction": "thumbs_up"
}
```

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Object with provided ID is not a message (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

## Invites

### **POST** /api/v1/invites

**Creates** a new Invite object with the provided data. **Returns** the newly created object.

**Responses:**

* ``201 CREATED`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated; cannot create objects

### **GET** /api/v1/invites/{invite_id}

**Takes** an invite ID (``invite_id``) and **returns** the Invite object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/invites/{invite_id}

**Takes** an invite ID (``invite_id``) and patch data, **patches** the Invite object with the given ID and **returns** the patched Invite object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/invites/{invite_id}

**Takes** an invite ID (``invite_id``), **deletes** the Invite object with the given ID and **returns** the ID of the deleted Invite object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/invites/{invite_id}/join

**Takes** an invite ID (``invite_id``) and joins the conference using the invite as the currently authenticated user. **Returns** the Conference object of the conference that was joined.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated
* ``403 FORBIDDEN`` - Banned from conference
* ``404 NOT FOUND`` - Object with given ID not found

### **GET** /api/v1/invites/by-name/{invite_name}

**Takes** an invite name (``invite_name``) and **returns** the Invite object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/invites/by-name/{invite_name}

**Takes** an invite name (``invite_name``) and patch data, **patches** the Invite object with the given ID and **returns** the patched Invite object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/invites/by-name/{invite_name}

**Takes** an invite name (``invite_name``), **deletes** the Invite object with the given ID and **returns** the ID of the deleted Invite object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found

### **POST** /api/v1/invites/by-name/{invite_name}/join

**Takes** an invite name (``invite_name``) and joins the conference using the invite as the currently authenticated user. **Returns** the Conference object of the conference that was joined.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not an invite (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated
* ``403 FORBIDDEN`` - Banned from conference
* ``404 NOT FOUND`` - Object with given ID not found

## Roles

### **POST** /api/v1/roles

**Creates** a new Role object with the provided data. **Returns** the newly created object.

**Responses:**

* ``201 CREATED`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Provided object is not a role (error code: 5)
* ``401 UNAUTHORIZED`` - Not authenticated; cannot create objects

### **GET** /api/v1/roles/{role_id}

**Takes** a role ID (``role_id``) and **returns** the Role object with the given ID.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a role (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **PATCH** /api/v1/roles/{role_id}

**Takes** a role ID (``role_id``) and patch data, **patches** the Role object with the given ID and **returns** the patched Role object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - No data provided or Content-Type is not application/json
* ``400 BAD REQUEST`` - Provided object is not a role (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access object
* ``404 NOT FOUND`` - Object with given ID not found

### **DELETE** /api/v1/roles/{role_id}

**Takes** a role ID (``role_id``), **deletes** the Role object with the given ID and **returns** the ID of the deleted Role object.

**Responses:**

* ``200 OK`` - Success
* ``400 BAD REQUEST`` - Provided object is not a role (error code: 5)
* ``403 FORBIDDEN`` - Not permitted to access/delete object
* ``404 NOT FOUND`` - Object with given ID not found
