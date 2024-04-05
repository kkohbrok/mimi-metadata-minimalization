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

The main mechanism to hide the user and client identifiers from providers in the
context of individual groups and use user- and client-level pseudonyms instead.

User and client identifiers are still visible to (other) users and their clients
and are still used for discovery and initial establishment of connections.

# Connections

A connection is a mutually agreed-upon relation betwen two users. After a user
has discovered another user using their identifier, they can send a connection
request to that user.

To hide the initiator's identity from the responding user's provider, users
provide an HPKE key that the initiator fetches upon discovery and under which it
encrypts a connection request under.

Before sending a connection request to a user, the initiator creates a group
which contains all of its clients. Connection requests then contain information
required for the responding user to join that group, as well as two values:

- a connection key required to derive keys to link the owners pseudonyms to
  identifiers
- a connection (bearer) token which authorizes the connected user to fetch the
  owner's KeyPackages.

If the responder decides to accept the request, it joins the group via external
commit and sends the connection key and connection token to the initiator via
that group.

# Encrypted Credentials

Each leaf credential of a client contains a plaintext part and a ciphertext
part.

The plaintext part contains two unique pseudonyms: a user pseudonym (shared by
all other leaves of the user's clients in that group), a client pseudonym and a
signature key.

The ciphertext part contains the client's real credential, as well as a
signature over the plaintext part using the client's real credential.

The key to decrypt the ciphertext part is derived from the connection key using
the leaf's pseudonyms as context.

Users connected with the leaf's owner can derive that key to verify the
signature over the plaintext and distribute the decryption key to the group in
the context of which they want to use the KeyPackage.

# Joining groups

When clients join a group (either because it was added, or because it joined via
external commit), other group members must be able to fully authenticate the
joining client, for which they require the key to decrypt the ciphertext in the
client's credential.

## Add flow

When adding a user(s) to a group via a Welcome message, the adding user must do
two things: On the one hand, it must provide the new user(s) with the key
material required to authenticate existing group members. On the other hand, it
must provide the existing group members with the key material required to
authenticate the new user(s).

For the new user(s), the key material can be transported along with the Welcome,
e.g. via a GroupInfo extension.

For existing users, the key material must be transported in some other way, e.g.
via a custom proposal (encrypted under an exporter).

## Join flow

When a user joins a group via external commit, it needs an existing group member
to provide the key material necessary to authenticate existing group members.

The new group member can either wait for a member to come online and provide the
key material, or the key material can be shared out of band prior to the new
user joining.

The new user can then send the key material necessary to authenticate it to
existing group members (e.g. via a custom proposal encrypted under an exporter).

# KeyPackages

The use of user and client pseudonyms requires some additional care when
generating and uploading KeyPackages.

Leaf credentials contain a user and a client pseudonym, both of which must be
unique across groups. For the client pseudonym to be unique across groups,
clients can generate it randomly for each KeyPackage.

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
