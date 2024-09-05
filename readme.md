# OLAF/Neighbourhood protocol v0.8
By James, Jack, Tom, Mia, Valen, Isabelle, Katie & Cubie

# WARNING: THIS IS NOT A COMPLETE SPECIFICATION YET! DO NOT IMPLEMENT!

## Definitions
- **User** A user has a key pair. Each user connects to one server at a time.
- **Server** A server receives messages from clients and relays them towards the destination.
- **Neighbourhood** Servers organise themselves in a meshed network called a neighborhood. Each server in a neighbourhood is aware of and connects to all other servers
- **Fingerprint** A fingerprint is the unique identification of a user. It is obtained by taking SHA-256(exported RSA public key)

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

If a server "owns" a client, that just means that the client is connected to that server (Since clients only connect to one server at a time). The transport layer of this protocol uses Websockets (RFC 6455). 

You can call the server you are connected to your homeserver. You can only connect to users connected to your home server or users on servers directly connected your home server. Refer to the additional file network topology examples, to see examples of neighbourhoods and other potential network arrangments. 

## Protocol defined messages
All messages are sent as UTF-8 JSON objects. 

### Sent by client
Messages include a counter and are signed to prevent replay attacks.

All below messages with `data` follow the below structure:
```JSON
{
    "type": "signed_data",
    "data": {  },
    "counter": 12345,
    "signature": "<Base64 signature of data + counter>"
}
```
`counter` is a monotonically increasing integer. All handlers of a message should track the last counter value sent by a client and reject it if the current value is not greater than the last value. This defeats replay attacks.
The hash used for `signature` follows the SHA-256 algorithm. 
base64 encoding follows RFC 4648.


#### Hello
This message is sent when first connecting to a server to establish your public key.
```JSON
{
    "data": {
        "type": "hello",
        "public_key": "<Exported RSA public key>"
    }
}
```

### Chat
Sent when a user wants to send a chat message to another user[s]. Chat messages are end-to-end encrypted.

```JSON
{
    "data": {
        "type": "chat",
        "destination_servers": [
            "<Address of each recipient's destination server>",
        ],
        "iv": "<Base64 encoded AES initialisation vector>",
        "symm_keys": [
            "<Base64 encoded AES key, encrypted with each recipient's public RSA key>",
        ],
        "chat": "<Base64 encoded AES encrypted segment>"
    }
}

{
    "chat": {
        "participants": [
            "<Base64 encoded list of fingerprints of participants, starting with sender>",
        ],
        "message": "<Plaintext message>"
    }
}
```

Group chats are defined similar to how group emails work. Simply send a message to all recipients with multiple `participants`. The `symm_keys` field is an array which lists the AES key for the message encrypted for each recipient using their respective asymmetric key. Each of the `destination_servers`, `symm_keys`, and `participants` are in the same order, except for the sender, which is only included in the `participants` list.

### Public chat
Public chats are not encrypted at all and are broadcasted as plaintext.

```JSON
{
    "data": {
        "type": "public_chat",
        "sender": "<Base64 encoded fingerprint of sender>",
        "message": "<Plaintext message>"
    }
}
```

### Client list
To retrieve a list of all currently connected clients on all servers. Your server will send a JSON response. This does not follow the `data` structure.

```JSON
{
    "type": "client_list_request",
}
```
Server response:
```JSON
{
    "type": "client_list",
    "servers": [
        {
            "address": "<Address of server>",
            "clients": [
                "<Exported RSA public key of client>",
            ]
        },
    ]
}
```
Servers assume that a client/user is online as long as they have an open websocket with the homeServer.

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
    "type": "client_update",
    "clients": [
        "<Exported RSA public key of client>",
    ]
}
```


#### Client update request
When a server comes online, it will have no initial knowledge of clients connected elsewhere, so it needs to request a `client_update` from all other servers in the neighbourhood.

```JSON
{
    "type": "client_update_request"
}
```
All other servers respond by sending `client_update`

#### Server Hello 
When a server added another server's IP to its list of connected servers. It sends this message to that server. To Let them know that this server is connecting to them. 
```JSON
{
"type": "server_hello"
   "data" {
        "type": "server_to_server_message"
        "sender": "<server IP connecting>"
    }
}
```
This lets server know who is connecting to them. Allows the server to add the other server it list of connected servers, allowing it to intiate the connection. 

#### Server Connection Update
When a server connects to a new server, it sends this message to all the other servers it is directly. Letting them know the ip address of the new server they are connecting to.
This helps maintain neighbourhoods, by making it easier for servers have the right direct connections. 

```JSON
{
"type": "server_connections_update"
  "data" {
      "type": "server_to_server_message"
      "sender": "<IP of server that is already connected>"
      "message": "<IP of new server, the server is connecting to>"
    }
}
```


### Defintion Tables of Types and Sections and additonal explanations

#### tables
| Type | Type Meaning |
|:----:|:------------:| 
| signed_data| data that has a signature confirming the sender|
| client_list_request | request sent by client, to get the list of clients online connected to a server |
| client_update | update send by server letting clients know who has disconnected |
| client_list | reply by server to client_List_request which contains list of users online |
| client_update_request | server asking other servers for the client_update |

|different types in the data section | meaning |
| :-----: | :----: |
| chat | message that has chat message data in it |
| hello | message sent when client connects to a server |
| public_chat | message sent to every one connected in the neighbourhood and homeServer not encrypted |

#### Counter
Every message sent by user, tied to their unique key set, has the counter attached to it. The recipient stores the counter value from the latest message sent to them by each user, then when ever a new message received, the counter value stored is compared to the value in the message. If the new value is larger than the old one, the message has not been resent. The starting value of the count will be 0.

The intention behiend the additon of the counter is as a way to defend against replay attacks. A replay attack is a when a copy is taken of a message you receive, and is then resent to you later. For example, Alice sents Bob a message saying "meet me at the park at 2pm" and a malicious attacker takes a copy of that message. A few weeks later the malicious attacker resends Bob the message. Bob goes to the park and finds the malicious attacker there instead of Alice.

## File transfers
File transfers are performed over an HTTP[S] API.

### Upload file
Uplaod a file in the same format as an HTTP form.
```
"<server>/api/upload" {
    METHOD: POST
    body: file
}
```
The server makes no guarantees that it will accept your file or retain it for any given length of time. It can also reject the file based on an arbitrary file size limit. An appropriate `413` error can be returned for this case.

A successful file upload will result in the following response:
```
response {
    body: {
        file_url: "<...>"
    }
}
```
`file_url` is a unique URL that points to the uploaded file which can be retrieved later.

### Retrieve file
```
"<file_url>" {
    METHOD: GET
}
```
The server will respond with the file data. File uploads and downloads are not authenticated and secured only by keeping the unique URL secret.


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
Asymmetric encryption and decryption is performed with RSA.
- Key size/Modulus length (n) = 2048 bits
- Public exponent (e) = 65537
- Padding scheme: OAEP with SHA-256 digest/hash function
- Public keys are exported in PEM encoding with PKCS8 format.

Signing and verification also uses RSA. It shares the same keys as encryption/decryption.
- Padding scheme: PSS with SHA-256 digest/hash function
- Salt length: 32 bytes

Symmetric encryption is performed with AES in GCM mode.
- Initialisation vector (IV) = 16 bytes (Must be randomly generated)
- Additional/associated data = not used (empty).
- Key length: 32 bytes (128 bits)

### Order to apply different layers of encrpytion  
- message is created
- create a signature by applying the signature scheme RSA-PSS 
- encrpyt the message using the symmetric encyption specified above
- encrypt the symmetric key used to encrypt the message with the public asymmetric encrption key of the intended recipient
- format these to be sent as shown in proctocol defined messages

