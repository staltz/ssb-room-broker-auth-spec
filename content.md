# SSB Room Broker Authentication

**Revision:** DRAFT. DO NOT IMPLEMENT.

**Author:** Andre Medeiros <contact@staltz.com>

**License:** This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).


## The problem, summarized

By default, connections from unknown peers are forbidden by [ssb-conn-firewall](https://github.com/staltz/ssb-conn-firewall/). This is good, for safety, but it makes onboarding more difficult.

When you **invite people** to the network, you want them to be **pre-approved** and open an exception in the firewall, so they can freely connect with you.

When you **install a new SSB app**, you need it to **bootstrap its database** from your existing SSB app, so that the new app has "your content" (or at least social graph) from the beginning.

## The solution, summarized

A remote room client (either an invited friend, or a new SSB app you own) can connect freely to the room server, and upon connection sends some additional data to the room via the muxrpc APIs `brokerAuth.request()` or `brokerAuth.claim()`, which the room will then broker to the alias owner, such that they can then authorize the tunneled connection from remote room client to alias owner.

There are two cases:

1. The remote peer is asking for the alias owner to grant some specific permissions
2. The alias owner has pre-approved some permissions and has given through a 3rd party channel (e.g. via instant messengers or email) a token to the remote peer

In the first case, the muxrpc API used is `brokerAuth.request()`, and in the second case it is `brokerAuth.claim()`. Unlike with tunneled connections, the arguments given to these muxrpc APIs have to be encrypted to the alias owner and signed by the remote peer, to prevent the room from tampering them (which would effectively let the room authenticate whoever they wanted to).

In the case of "claiming" a token, the token is attached to the alias URL in the URI fragment in order to allow only the room's browser-side JavaScript to detect it, not the room server itself. For example, `https://{alias}.{domain}/#{token}`

### Use case: inviting a friend

#### Pre-approve some permissions and create token

```mermaid
sequenceDiagram
  actor A as Alias owner
  actor B as Friend

  A->>A: Create token T with permissions<br/>"connect" and "followback"<br/>and persist it locally
  A->>B: `https://{alias}.{domain}/♯{token}
```

#### Consuming the tokenized alias

```mermaid
sequenceDiagram
  actor A as Alias owner
  participant R as Room
  actor B as Friend

  alt Friend opens `https://{alias}.{domain}/♯{token}` in their browser
    B->>+R: (http) <br/>`https://{alias}.{domain}/♯{token}`
    R->>R: Browser-side JS extracts<br/>`token` from URI fragment
    R-->>-B: SSB URI with multiserverAddress, roomId, userId, token
    B->>B: Opens SSB app
  else Friend inputs `https://{alias}.{domain}/♯{token}` in their SSB app
    B->>B: Extracts `token` from the URL
    B->>+R: (http) <br/>`https://{alias}.{domain}?encoding=json`
    R-->>-B: JSON with multiserverAddress, roomId, userId
  end

  opt Recommended
   B->>B: Follows userId
  end

  B->>B: Encrypt `=broker-auth-token:{token}` to the<br/>public key of the alias owner, thus `boxedToken`
  B->>B: Sign `boxedToken` with my private key,<br/>creating `sig`
  B->>+R: (muxrpc async) `brokerAuth.claim(userId, boxedToken, sig)`

  alt Alias owner is offline
    R-->>B: Respond to brokerAuth.claim with `false`
  else Alias owner is online
    R->>+A: brokerAuth.claim(friendId, boxedToken, sig)
    A->>A: Confirm that `sig` is from `friendId`
    A->>A: Decrypt `boxedToken` to get `token`
    A->>A: Confirms that `token` is<br/>found locally with permissions<br/>"connect" and "followback"
    A->>A: "connect" permission creates<br/>an exception in the firewall
    A->>A: "followback" permission publishes<br/>a follow message for friendId
    A-->>-R: Respond brokerAuth.claim with `true`
    R-->>-B: Respond brokerAuth.claim with `true`

    note over A,B: Friend establishes tunneled connection with Alias owner
  end
```

Note, one variant of this use case is to also include the invited friend into the same room servers that the alias owner is in too, using the `roomMembership` permission.

### Use case: subapp bootstrap

New app wants to be a subfeed of a metafeed belonging to the alias owner, and wants that subfeed to be added as a member in rooms.

Thus the permissions are

- `connect`
- `subfeed` (note, this may require an additional query param to specify the details for the subfeed)
- `roomMembership` (may also require additional query params)

```mermaid
sequenceDiagram
  actor A as Alias owner
  participant R as Room
  actor B as New app

  note over B: Ask user to input alias URL
  B->>B: `{alias}.{domain}`
  B->>+R: (http) <br/>`https://{alias}.{domain}/?encoding=json`
  R-->>-B: JSON with multiserverAddress, roomId, userId
  B->>B: permissions = `connect,subfeed,roomMembership`
  B->>B: Encrypt `=broker-auth-permissions:{permissions}` to the<br/>public key of the alias owner, thus `boxedPermissions`
  B->>B: Sign `boxedPermissions` with my private key,<br/>creating `sig`

  B->>+R: (muxrpc async) `brokerAuth.request(userId, boxedPermissions, sig)`

  loop Until alias owner is online
    R->>R: wait
  end
  R->>+A: brokerAuth.request(newAppId, boxedPermissions, sig)
  A->>A: Confirm that `sig` is from `friendId`
  A->>A: Decrypt `boxedPermissions` to get `permissions`
  note over A: Ask user to confirm whether<br/>`newAppId` can get these permissions

  alt User disallows
    A-->>R: Respond brokerAuth.request with `false`
    R-->>B: Respond brokerAuth.request with `false`
  else User allows
    A->>A: "connect" permission creates<br/>an exception in the firewall
    A->>A: "subfeed" permission creates<br/>a new subfeed under a metafeed
    A->>A: "roomMembership" permission registers<br/>the new app as member in some rooms
    A-->>-R: Respond brokerAuth.request with `true`
    R-->>-B: Respond brokerAuth.request with `true`

    note over A,B: New app initiates tunneled connection with Alias owner

    A->>A: `details` contains data regarding<br/>permissions `subfeed` and<br/>`roomMembership`

    A->>+B: (muxrpc async) `brokerAuth.grant(details)`
    B-->>-A: true
  end
```

### Use case: fusion dance

New app and old app want to link each other as the same "person".

Thus the permissions are:

- `connect`
- `fusion`


```mermaid
sequenceDiagram
  actor A as Alias owner
  participant R as Room
  actor B as New app

  note over B: Ask user to input alias URL
  B->>B: `{alias}.{domain}`
  B->>+R: (http) <br/>`https://{alias}.{domain}/?encoding=json`
  R-->>-B: JSON with multiserverAddress, roomId, userId
  B->>B: permissions = `connect,fusion`
  B->>B: Encrypt `=broker-auth-permissions:{permissions}` to the<br/>public key of the alias owner, thus `boxedPermissions`
  B->>B: Sign `boxedPermissions` with my private key,<br/>creating `sig`

  B->>+R: (muxrpc async) `brokerAuth.request(userId, boxedPermissions, sig)`

  loop Until alias owner is online
    R->>R: wait
  end

  R->>+A: brokerAuth.request(newAppId, boxedPermissions, sig)
  A->>A: Confirm that `sig` is from `friendId`
  A->>A: Decrypt `boxedPermissions` to get `permissions`
  note over A: Ask user to confirm whether<br/>`newAppId` can get these permissions

  alt User disallows
    A-->>R: Respond brokerAuth.request with `false`
    R-->>B: Respond brokerAuth.request with `false`
  else User allows
    A->>A: "connect" permission creates<br/>an exception in the firewall
    A->>A: "fusion" permission publishes<br/>`fusion/init` and `fusion/invite` for newAppId
    A-->>-R: Respond brokerAuth.request with `true`
    R-->>-B: Respond brokerAuth.request with `true`

    note over A,B: New app initiates tunneled connection with Alias owner,<br/>and they replicate with each other the following fusion messages

    B->>B: publish `fusion/consent`
    A->>A: publish `fusion/entrust`
    B->>B: publish `fusion/proof-of-key`
  end
```

## Security considerations

### Spam on `brokerAuth.request()`

A malicious remote peer could spam the alias owner with several `brokerAuth.request()` calls which in turn spam the end-user with manual approval requests. To mitigate that, there are two tactics: (1) the alias owner can block the SSB ID for that spammy remote peer, (2) the room can rate-limit or ban remote peers.

On the second tactic (room server using rate limiting), the remote peer can never know if a `brokerAuth.request()` call returned `false` due to the alias owner being *offline* or due to the alias owner rejecting it, but the room always knows this information. Thus the room could detect that an alias owner rejected `brokerAuth.request()` calls several times for a specific remote peer (identified by its IP address), and thus ban or rate limit that remote peer by IP address or SSB ID, or both.

## What about off-grid use cases?

This proposal heavily relies on room servers over the internet, so there is no solution given for subapp bootstrapping or fusion identity in a local area network, or even on the same device (say, entirely isolated from other devices).

I believe we should make other proposals that are similar to broker auth, but meant primarily for local area network, or primarily for same-device auth.

For local area network authentication, we could replace alias URLs with local-network domains such as `.local` or `.home.arpa`. For instance, for subapp bootstrapping in the same LAN, the old app could have an address such as `$SSBID.local`, the new app can dial that address, and authentication via muxrpc can proceed.

For same-device authentication, either we can use an overkill such as room broker auth (or LAN auth), or we can use a OS-specific solution such as Unix sockets or others.

# Appendix

## List of new muxrpc APIs

- async
  - `brokerAuth.claim(ssbID, boxedToken, sig)`
  - `brokerAuth.request(ssbID, boxedPermissions, sig)`
  - `brokerAuth.grant(details)`