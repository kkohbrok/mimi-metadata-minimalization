---
title: MIMI Metadata Minimalization (MIMIMI)
abbrev: MIMIMI
docname: draft-kohbrok-mimi-metadata-minimalization-latest
category: info

ipr: trust200902
area: Security
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: K. Kohbrok
    name: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de
 -  ins: R. Robert
    name: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com

--- abstract

This document describes a proposal to run the MIMI protocol in a way that
reduces the ability of the Hub and service providers to associate messaging
activity of clients with their respective identities.

For now, this document only contains a high-level description of the mechanisms
involved.

--- middle

# Introduction

Since the MIMI protocol uses MLS PublicMessages for Proposals and Commits, both
Hub and Provider can observe and track activities of clients both within an
individual group and across groups.

The goal of the scheme described in this document are as follows:

- For a given client, prevent the Hub and all service providers from associating
  the activity of the client with that client's identifier.
- For a given client, prevent the Hub and all service providers except for the
  client's service provider from associating the activity of that client in one
  group from that in any other.

These propoerties are achieved withouth limiting the ability of the Hub to track
a group's group state and provider public group information to newly joining
clients.

However, the scheme comes with a few operative requirements and the following
general limitations:

- There needs to be the notion of a "connection" between two users. Users that
  are connected can link a pseudonym with a user's real identity and can
  (selectively) allow others to do the same.
- Only connected users can add one-another to groups.
- Users that join a group need help of an existing group member in linking the
  pseudonyms visible in the group to real user identities.

This document only contains a high-level description of the individual changes
required to achieve the goals detailed above.

# Pseudonymization

The main mechanism to hide the user and client identifiers from the hub and
providers in the context of individual groups and use user- and client-level
pseudonyms instead.

The main purpose of the pseudonymization scheme described in this document is to
hide metadata from service providers with as little impact as possible on the
features provided by the base MIMI protocol. In particular, this means that by
default, User and client identifiers are still visible to (other) users and
their clients.

However, the pseudonymization scheme can also be used to hide a user's identity
from other users. See Section {{user-to-user-pseudonymity}} for more details.

# Pseudonymity through credential encryption

To hide their identities from providers, clients use PseudonymousCredentials to
represent them in MIMI rooms.

~~~ tls
struct {
  IdentifierUri client_pseudonym;
  IdentifierUri user_pseudonym;
  opaque signature_public_key;
  opaque identity_link_ciphertext<V>;
} PseudonymousCredential
~~~

As the name suggests, PseudonymousCredentials don't contain a client's client
and user ID. Instead, they use two pseudonyms: a user pseudonym (shared by all
other PseudonymousCredentials of the user's clients in a given room) and a
client pseudonym (unique for each PseudonymousCredential). Each pseudonym is an
IdentifierUri made up of a randomly generated UUID and the user's provider's
domain.

~~~ tls
struct {
  IdentifierUri client_pseudonym;
  IdentifierUri user_pseudonym;
  opaque signature_public_key;
} PseudonymousCredentialTBS

struct {
  /* SignWithLabel(., "PseudonymousCredentialTBS", PseudonymousCredentialTBS) */
  opaque pseudonymous_credential_signature<V>;
  Credential client_credential;
} IdentityLinkTBE
~~~

The `identity_link_ciphertext` is created by encrypting the IdentityLinkTBE,
which contains the client's real credential, as well as a signature over the
PseudonymousCredentialTBS using the client credential's signature key.

The identity link key used for encryption is unique per pseudonymous credential. It is derived from the client's connection key.

~~~ tls
identity_link_key = ExpandWithLabel(connection_key, "IdentityLinkKey", PseudonymousCredentialTBS, KDF.Nh)
~~~

See {{connections}} for more details on the connection key.

The fact that the identity link key is unique per pseudonymous credential along
with the fact that pseudonymous credentials are used in only once (i.e. in one
room) allows managing the capability to de-pseudonymize a client on a per-room
level.

# Identity link key management in rooms

Except in the cases detailed in {{user-to-user-pseudonymity}}, all clients in a
room MUST be able to decrypt the identity link ciphertext of all other clients
in the room.

TODO: The following scheme should be replaced by TreeWrap for FS and PCS. It's a
simple placeholder for now.

To achieve this, the hub holds encrypted copies of the identity link keys of all
clients in the room. The identity link keys are encrypted with the room's identity link wrapper key.

The identity link wrapper key is freshly generated by the room's creator and
does not change throughout the lifetime of the group. All clients that join the
room must upon joining the group:

- Receive a copy of that key, allowing them to learn the identity of all other
  participants
- Provide a copy of their own identity link key encrypted under the identity
  link wrapper key to the hub and all other members.

More on group joining in Section {{joining-group}

- Clients joining through the invite flow are sent the key through a GroupContextExtension
- Clients joining through the join flow need to be sent the key out of band

When the room's membership changes, the copies of the identity link keys held by
the hub are updated accordingly.

- When the room is first created, the creator uploads its own encrypted
  identity link key
- When a client is added to the room, the adder includes the encrypted identity
  link key of the added client in the AAD of the add message
- When a client joins a room, it includes its own encrypted identity link key
  in the AAD of the join message (i.e. the external commit)
- When a client is removed from a room, the hub removes the encrypted identity
  link key of the removed client

## Add flow

If a group member adds a new user, the adder MUST include a
IdentityLinkExtension in the Welcome's GroupInfoExtensions.

~~~ tls
struct {
  opaque identity_link_wrapper_key<V>;
} IdentityLinkExtension
~~~

The adder MUST also include the encrypted identity link key of all added users
in the AAD of the commit message. The order of the encrypted identity link keys
in the AAD MUST match the order of the Add proposals in the Commit message.

TODO: Note here that the sender MUST use the SafeAAD as defined in the extension
doc.

~~~ tls
struct {
  opaque encrypted_identity_link_keys<V>;
} AddAAD
~~~

Existing group members can then decrypt the identity link keys to learn the
real identities of the added users.

The hub, receiving the commit message, adds the encrypted identity link keys to
the room's state.

Finally, the added user receiving the welcome can then obtain the encrypted
identity link keys from the hub and decrypt them using the identity link
wrapper key from the extension and use the decrypted identity link keys to
decrypt the identity link ciphertexts of all other clients in the room.

## Join flow

Members joining a room through the join flow need to obtain the identity link
wrapper key out of band. The key can be shared any other group member.

TODO: When defining features such as join/invitation links the identity link
wrapper key should be included in the link.

When joining the group, the joining client MUST include its own encrypted
identity link key in the AAD of the external commit message via the AddAAD
struct.

# Connections

A connection is a mutually agreed-upon relation betwen two users. Users can
establish connections between one-another through the following process.

Let's call the initiator of the connection establishment Alice and the
responding user Bob.

At the time of registration with their local provider, Alice and Bob generate an
HPKE keypair and upload the public key (called the connection establishment public key) to their respective providers. The providers make the keys
available to all other users without requiring authentication.

To hide Alice's identity from Bob's provider throughout the connection
establishment process, Alice obtains Bob's connection establishment public key
from Bob's provider.

Alice now creates a new conversation with her local provider as hub and adds
all of her clients to that conversation. She then compiles the following
information into a connection request:

~~~ tls
struct {
  Credential requester_client_credential;
  opaque connection_room_id<V>;
  opaque connection_room_identity_link_wrapper_key<V>;
  opaque requester_connection_key<V>;
  opaque connection_response_key<V>;
} ConnectionRequest
~~~

The `connection_response_key` is a fresh HPKE keypair that Alice generates when
she creates the connection request. It is later used by Bob to encrypt his own
connection key if Bob chooses to accept the request.

TODO: The request will likely contain more information s.t. an authorization
token that allows Bob to fetch Alice's KeyPackages, etc. If we want to include
more information here, it should also be included in the ConnectionResponse
below.

Alice then encrypts the connection request using Bob's connection establishment
public key and sends it to Bob's provider.

Bob fetches the connection request from his provider and decrypts it using his connection establishment key.

He can inspect Alice's `requester_client_credential` and decide whether to
accept the request.

If Bob accepts, he creates a connection response:

~~~ tls
struct {
  opaque responder_connection_key<V>;
} ConnectionResponse
~~~

Bob then follows the join flow for the room with the `connection_room_id`,
including the ConnectionResponse encrypted under the `connection_response_key`
in the join message's (i.e. the external commit's) AAD. As part of that
process, he obtains the encrypted identity link keys for Alice's clients and
verifies that they belong to the same user as the `requester_client_credential`
in the connection request.

Alice sees that Bob has accepted her connection request by joining the room and
can decrypt Bob's ConnectionResponse using the connection response key she
initially included in the connection request.

Both Alice and Bob now share a room and can derive identity link keys for each
other's clients, which in turn allows them to add each other to rooms.

# KeyPackages

The use of user and client pseudonyms requires some additional care when
generating and uploading KeyPackages.

The user and a client pseudonyms in the PseudonymousCredential must be unique
across groups. For the client pseudonym to be unique across groups, clients can
generate it randomly for each KeyPackage.

However, to allow the hub to associate a set of leaves with a user, the user
pseudonym must be consistent across the leaves in that group. This means that
clients need to coordinate to provide KeyPackage sets, where each KeyPackage in
a set contains a consistent user pseudonym and no two sets contain the same
pseudonym. Each such set can be used in the context of one group.

A procedure for a client to upload KeyPackages could thus look as follows:

1. Check for which user pseudonyms other clients of the same user have uploaded
   KeyPackages.
2. Identify user pseudonyms for which the KeyPackage set is missing a KeyPackage
   of this client.
3. Generate and upload KeyPackages for those user pseudonyms.
4. Upload as many KeyPackages as desired, thus starting new KeyPackage sets for
   other clients to complete.

At least one KeyPackage set of each user must consist of last-resort
KeyPackages. If such a last-resort set is used more than once, the use of the
same user pseudonym allows the hub to learn that one user is active in two
particular groups. The re-use of such a set cannot be avoided entirely, but
users upload more than one set (and re-upload last-resort sets frequently) to
mitigate this at least to a certain degree.

When uploading KeyPackages, user generally upload KeyPackages to their provider
indexed by their connection token. To avoid leaking metadata to the user's own
provider, the upload must not be connected to the user's identity.

Users can then fetch KeyPackages of connected users from their respective
providers using the connected user's connection token.

# Message Fanout

This protocol assumes that clients have one or more queue IDs that their local
provider can use to route incoming messages to a place that the client can
access.

For each member of a given group, the member's queue ID is stored in a leaf node
extension s.t. the hub can forward that information to the client's local
provider upon fanout.

Since in most cases, there will only be a single queue for each client (as
opposed to one queue per group), the queue ID is encrypted using an HPKE key of
the client's provider.

When receiving the encrypted queue ID with the message fanout, the provider can
decrypt the queue ID and route the message to the correct queue.
