# OLAF/Neighbourhood protocol v0.4
By James, Jack, Tom, Mia, Valen, Isabelle, Katie & Cubie

# WARNING: THIS IS NOT A COMPLETE SPECIFICATION YET! DO NOT IMPLEMENT!

## Definitions
- **User** A user has a key pair. They are uniquely identified by the public key. Each user connects to one server at a time.
- **Server** A server receives messages from clients and relays them towards the destination.
- **Neighbourhood** Servers organise themselves in a meshed network called a neighborhood. Each server in a neighbourhood is aware of and connects to all other servers

## Main design principles
This protocol specification was obtained by taking parts of the original OLAF protocol combined with the neighbourhood protocol. The network structure resembles the original neighbourhood, while the messages and roles of the servers are similar to OLAF.

## Network Topology
Client-to-client messages travel in the following path:
```
Client (Sender)
  |
  |  Message sent directly
  V  
Server (Owner of the sender)
  |
  |  Message routed to the correct server
  V  
Server (Owner of the receiver)
  |
  |  Message flooded to all receiving clients
  V  
Client (Receiver)
```

If a server "owns" a client, that just means that the client is connected to that server. (Since clients only connect to one server at a time)

The transport layer of this protocol uses Websockets (RFC 6455)

## Protocol defined messages
All messages are sent as UTF-8 JSON objects. 

### Sent by client
Messages include a counter and are signed to prevent replay attacks.

All below messages with `data` follow the below structure:
```JSON
{
    type: "data_container",
    data: { ... },
    counter: <64-bit integer counter>,
    signature: "<Base64 signed hash of data concatenated with counter>"
}
```
`counter` is a monotonically increasing counter. All handlers of a message should track the last counter value sent by a client and reject it if the current value is not greater than the last value. This defeats replay attacks.
The hash used for `signature` follows the SHA256 algorithm.

#### Hello
This message is sent when first connecting to a server to establish your public key.
```JSON
data: {
    type: "hello",
    public_key: "<Base64 encoded RSA public key>"
}
```

### Chat
Sent when a user wants to send a chat message to another user[s]. Chat messages are end-to-end encrypted.

```JSON
data: {
    type: "chat",
    destination_server: "<Address of destination server>",
    symm_key: "<Base64 encoded AES symmetric key, encrypted with recipient's public RSA key>",
    chat: "<Base64 encoded AES encrypted segment>"
}

chat: {
    participants: [
        "<Base64 encoded list of public keys of participants, starting with sender>"
    ],
    message: "<Plaintext message>"
}
```

Group chats are defined similar to how group emails work. Simply send a message to all recipients with multiple `participants`. You will need to specifically encrypt the `key` for each recipient though.

### Public chat
Public chats are not encrypted at all and are broadcasted as plaintext.

```JSON
data: {
    type: "public_chat",
    public_key: "<Base64 encoded public key of sender>",
    message: "<Plaintext message>"
}
```

### Client list
To retrieve a list of all currently connected clients on all servers. Your server will send a JSON response. This does not follow the `data` structure.

```JSON
// Client request:
{
    type: "client_list",
}

// Server response:
{
    type: "client_list",
    servers: [
        {
            address: "<Address of server>",
            clients: [
                "<Base64 encoded public key of client>",
                ...
            ]
        },
        ...
    ]
}
```

### Sent by server
#### Client update
A server will know when a client disconnects as the socket connection will drop off. 

When one of the following things happens, a server should send a `client_update` message to all other servers in the neighbourhood so that they can update their internal state.
1. A client sends `hello`
2. A client disconnects

You don't need to send an update for clients who disconnected before sending `hello`.

The `client_update` advertises all currently connected users on a particular server.
```JSON
{
    type: "client_update",
    clients: [
        "<Base64 encoded public key of client>",
        ...
    ]
}
```

#### Client update request
When a server comes online, it will have no initial knowledge of clients connected elsewhere, so it needs to request a `client_update` from all other servers in the neighbourhood.

```JSON
{
    type: "client_update_request"
}

// All other servers respond by sending client_update
```


## File transfers
TBD but will be submitted/downloaded from a server over HTTP.


## Client Responsibilities
When receiving a message from the server, the client first needs to validate the signature against the public key of the sender.

### How to send a message?
There are two things to know about your recipient: their server address and public key. use these to fill out a `"chat"` message and your server will forward it to the correct destination.

### How do you know when you receive a message for you?
When receiving a chat message, you should attempt to decrypt the `symm_key` field, then use that to decrypt the `chat` field. If the result follows the format, then the message is directed to you. You can also check for your public key in the `participants` list.


## Server Responsibilities
A server is primarily a relay for messages. It does only a minimal amount of message parsing and state storage.

It is a server's responsibility to not forward garbage, so it should check all messages to ensure they follow a standard message format as above. This includes incoming (from other servers) and outgoing (from clients) messages.

A server is located by an address which opionally includes a port. The default port is the same as http[s]. 80 for non-TLS and 443 for TLS
- 10.0.0.27:8001
- my.awesomeserver.net
- localhost:666

### Stored state
- Client list. A server should listen to every `client_update` message and use this to keep an internal list of all connected clients in the neighbourhood.
- Files
- List of other servers in the neighbourhood

### Adding a new server to a neighbourhood
The server admins (Whoever is hosting the server) need to agree and each manually add the new server into the stored list.
If not all servers agree on who is in the neighbourhood, the neighbourhood enters an invalid state and it is not guaranteed that all clients will be able to communicate.


## Underlying technologies
The transport layer uses Websockets, meaning the server will need to be HTTP-capable. There are various websocket libraries for the popular programming languages that will handle this.

## Encryption
### Asymmetric Encryption
Asymmetric encryption and decryption is performed with RSA and the OAEP padding scheme.
- Key size/Modulus length (n) = 2048 bits
- Public exponent (e) = 65537
- Padding scheme: OAEP with SHA-256 digest/hash function

Signing and verification uses the same RSA keys as encryption
- Padding scheme: PSS with SHA-256 digest/hash function
- Salt length: 32 bytes

Symmetric encryption is performed with AES in GCM mode.
- Initialisation vector (IV) = 16 bytes (Must be randomly generated)
- Additional/associated data = not used (empty).
- Key length: 32 bytes (128 bits)