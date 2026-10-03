# Concepts: OIDC, OP, RP, and Tokens

## What OIDC is

OpenID Connect (OIDC) is an identity layer built on top of OAuth 2.0. OAuth
2.0 alone is an *authorization* protocol — it grants an application limited
access to a resource on a user's behalf, without necessarily proving who
the user is. OIDC adds *authentication*: a standard way for one party to
prove to another "this user is who they say they are," using a signed
token instead of a shared session or a proprietary handoff.

## The two roles

**OP — OpenID Provider** (also called the Identity Provider). The single
source of truth for identity. It authenticates the user directly — password,
MFA, whatever it's configured to require — and is the only party that ever
handles the user's actual credentials.

**RP — Relying Party**. Any application that wants to know who the user is,
without authenticating them itself. It trusts the OP's tokens instead. In
this guide, each RP is a Web Reverse Proxy instance protecting one or more
applications.

This is a **hub-and-spoke** model: one OP, any number of RPs, each with its
own registered trust relationship (a client ID and secret) to the OP —
rather than every pair of applications needing a direct relationship with
each other.

## How a token carries authentication from OP to RP

1. An unauthenticated user hits an RP-protected resource. The RP redirects
   the browser to the OP's authorization endpoint, identifying itself with
   its registered client ID and a redirect URI.
2. The OP authenticates the user.
3. The OP redirects the browser back to the RP's redirect URI with an
   **authorization code** — short-lived, single-use, not the identity
   itself.
4. The RP exchanges that code directly with the OP, server-to-server (not
   through the browser), for:
   - An **ID token** — a signed JWT containing claims about the user
     (subject, issuer, audience, expiry, and configured profile claims).
     Signed by the OP so the RP can verify it hasn't been forged.
   - An **access token** — usable to call APIs (e.g. a userinfo endpoint)
     on the user's behalf.
5. The RP validates the ID token's signature and claims, then establishes
   its own local session. From here, the RP doesn't need to talk to the OP
   again until the session expires.

## How this differs from CDSSO

CDSSO passed an encrypted token directly between peer reverse proxies —
trust was pairwise, and every peer relationship needed its own
configuration. OIDC centralizes trust in the OP: every RP validates tokens
issued by the same OP, using a standard, publicly documented token format
and validation rules, rather than a proprietary peer-to-peer scheme.

| | CDSSO | OIDC |
|---|---|---|
| Topology | Peer-to-peer | Hub-and-spoke |
| Trust model | Each pair of reverse proxies configured directly | Each RP trusts one shared OP |
| Kickoff | `GET /pkmscdsso?<target-url>` | `GET /pkmsoidc?iss=<op-id>&Target=<encoded-target-url>` |

See [02-cdsso-vs-oidc.md](02-cdsso-vs-oidc.md) for the full comparison and
migration considerations.
