---
title: "Workload Authorization Grant"
abbrev: "WAG"
category: info

docname: draft-carleton-workload-authz-grant-latest
number:
date:
v: 3
area: "sec"
keyword:
 - agent identity
 - workload identity
 - workload authorization grant
 - jwt authorization grant
venue:
  github: pcarleton/draft-carleton-workload-authz-grant

author:
 -
    fullname: Paul Carleton
    organization: Anthropic
    email: paulc@anthropic.com
    role: editor
 -
    fullname: Nick Steele
    organization: OpenAI
    email: steele@openai.com
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
 -
    fullname: Arndt Schwenkschuster
    organization: Defakto Security
    email: arndts.ietf@gmail.com
 -
    fullname: Brian Campbell
    organization: Ping Identity
    email: bcampbell@pingidentity.com

normative:
  RFC2046:
  RFC6749:
  RFC6838:
  RFC7515:
  RFC7517:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC8414:
  RFC8693:
  RFC8707:
  RFC8725:
  RFC9525:
  OIDC-DISCOVERY:
    title: OpenID Connect Discovery 1.0 incorporating errata set 2
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
    date: 2023-12
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: E. Jay

informative:
  RFC7591:
  IANA.media-types:
  IANA.oauth-parameters:
  RFC7523BIS: I-D.ietf-oauth-rfc7523bis
  IDJAG: I-D.ietf-oauth-identity-assertion-authz-grant

--- abstract

This document defines the Workload Authorization Grant (WAG), by which a
workload hosted on a platform -- an AI agent is the motivating case --
obtains access tokens from a third party's OAuth authorization server
without requiring an administrator to perform a per-workload provisioning
step.  Each workload is identified by an opaque identifier that is never
reassigned.  A Workload Identity Provider operated by the platform signs a JWT
authorization grant ({{RFC7523}}) that names one workload, and the workload
presents it at the token endpoint of an authorization server that has been
configured, once, to trust that Workload Identity Provider.  The authorization
server does not reject a workload because it has not seen it before. The
workload's access is determined by the authorization server's own policy,
which may consult claims the Workload Identity Provider asserts about the
workload.  This document covers workloads acting on
their own behalf.  Access on behalf of a user or other principal is out of
scope, though the grant is intended to compose with delegation mechanisms
in which the workload is the actor.

--- to_be_removed_note_Note_to_Readers

This document is an early, exploratory individual draft, published to solicit
discussion of the deployment pattern it describes.  It is not a working group
document, does not describe a shipped or committed design, and does not
represent a position or roadmap of the editors' employers.  Every aspect of it
is subject to change or withdrawal, including whether this mechanism should
be specified in a separate document at all.  Issues and pull requests:
https://github.com/pcarleton/draft-carleton-workload-authz-grant.

--- middle

# Introduction {#introduction}

Agent platforms host many agents per customer, instantiated and torn down as appropriate for the context of the customer's work environment -- one per channel, repository, or pipeline.  Registering each agent individually at the Authorization Servers it uses is impractical and creates a heavy maintenance burden.  In practice every agent of an installation ends up sharing one credential, at the cost of any attribution of an individual agent's actions.

A platform that hosts many workloads -- an agent platform is a motivating case -- needs each workload to obtain an access token at third-party services without requiring an administrator to perform a per-workload provisioning step.

This document defines one grant for that: a JWT authorization grant {{RFC7523}} signed by the platform's Workload Identity Provider and naming one workload, presented at the token endpoint of an authorization server that has been configured, once, to trust that Workload Identity Provider.

It specifies the grant, and that workloads are trusted based on the Workload Identity Provider registration, allowing a previously unseen workload to receive an access token.

How trust in a Workload Identity Provider is established and what a workload may do are left to deployments.

# Conventions and Terminology {#conventions}

{::boilerplate bcp14-tagged}

Platform: the party that instantiates and manages the lifecycle of workloads ("Agents").

Workload Identity Provider: the component of the Platform that issues Workload Authorization Grants.  Where a provider serves several customer organizations under one issuer identifier, each customer's partition must be uniquely identified (see {{tenants}}).

Workload Identity Provider registration: an Authorization Server's record of one Workload Identity Provider it trusts: the Workload Identity Provider's issuer identifier, its keys ({{issuer-keys}}) and, where several Workload Identity Providers share that issuer identifier, the name of a claim and the value the claim carries for this Workload Identity Provider ({{tenants}}).  How a Workload Identity Provider registration comes to exist is out of scope (note: this is not a client registration {{RFC7591}} and yields no client identifier or credential).

Authorization Server, Resource Server: as in {{RFC6749}}. Where an Authorization Server serves several customer organizations under one issuer identifier, each customer's partition is a separate Authorization Server ({{tenants}}).

# Overview {#overview}

1. Once per Workload Identity Provider and Authorization Server: an
   administrator of the Authorization Server creates a Workload Identity Provider
   registration ({{conventions}}), which records the Workload Identity Provider's
   issuer identifier and how to obtain its keys ({{issuer-keys}}).  Nothing about individual Agents is exchanged.
2. Per Agent: the Platform creates an Agent and assigns it an Agent
   Identifier ({{identity-model}}).  Nothing is sent to the Authorization
   Server or the Resource Server.
3. Per access: the Agent obtains a Workload Authorization Grant from the
   Workload Identity Provider and presents it in an ordinary OAuth token
   request.  The Authorization Server matches it to a
   Workload Identity Provider registration, verifies it under that Workload Identity Provider's
   keys,
   allowing for previously unseen `sub` values,
   and issues an access token under its own policy ({{properties}}).

~~~
 Workload Identity                         Authorization   Resource
     Provider       Agent                   Server (AS)   Server (RS)
         |            |                          |             |
(1)  [administrator registers the Workload Identity Provider]  |
         |            |                          |             |
(2)  [Platform creates Agent; nothing sent to AS or RS]        |
         |            |                          |             |
(3)      |--- WAG --->|                          |             |
         |            |--- POST /token --------->|             |
         |            |    grant_type=jwt-bearer |             |
         |            |    assertion=<WAG>       |             |
         |            |    resource=<RS>         |             |
         |            |<-- access token ---------|             |
         |            |--- request + access token ------------>|
~~~
{: #fig-overview title="One-time Workload Identity Provider registration, then per-request grants"}

# Agent Identity {#identity-model}

An Agent is identified by its Agent Identifier, carried as the `sub` claim in the assertion.  The Agent Identifier is opaque and immutable; it MUST be unique among all Agent Identifiers issued under the same Workload Identity Provider, MUST NOT be reassigned to a different Agent, and is compared as a case-sensitive string {{RFC7519, Section 2}}. An Authorization Server MUST associate records for an Agent with its Workload Identity Provider and `sub` value, never on its `sub` value alone.


# Workload Authorization Grant

An Agent obtains an access token by presenting a JWT as an authorization grant per {{RFC7523, Section 2.1}}, issued by the Workload Identity Provider as a third party in the sense of {{RFC7521, Section 3}}. The token request carries `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`, the JWT in the `assertion` parameter, and the target resource in the `resource` parameter {{RFC8707}}. The `resource` parameter {{RFC8707}} is RECOMMENDED; an Authorization Server SHOULD restrict the audience of the access token it issues to that resource and MAY refuse a request that lacks it with `invalid_target` ({{RFC8707, Section 2}}). An Agent MAY make the token request without client authentication ({{RFC7523, Section 3.1}}), and this specification attaches no meaning to `client_id`. An Authorization Server MUST NOT require a client registration per Agent.

Assertions SHOULD be short-lived.  The Authorization Server MUST NOT issue refresh tokens for this grant and SHOULD NOT issue access tokens that outlive the assertion by a significant period ({{RFC7521, Section 4.1}}).

Where a token type identifier is needed for a Workload Authorization Grant, the value `urn:ietf:params:oauth:token-type:wag` ({{iana-uri}}) is used; {{exchange}} uses it to obtain a Workload Authorization Grant from one Workload Identity Provider in exchange for one issued by another.  How an Agent obtains a Workload Authorization Grant from its own Platform is out of scope.

## JWT Syntax {#authorization-grant-claims}

The `typ` JOSE Header Parameter ({{RFC7515, Section 4.1.9}}) of the JWT MUST be `oauth-wag+jwt` ({{iana-media-type}}), as recommended by {{RFC8725, Section 3.11}}.  A Workload Identity Provider MUST NOT use this type for any other kind of JWT.

The JWT Claims Set contains:

`iss`
: REQUIRED - The issuer identifier of the Workload Identity Provider ({{issuer-keys}}): a URL using the `https` scheme with no query or fragment component, as for `issuer` in {{RFC8414, Section 2}}.

`sub`
: REQUIRED - The Agent Identifier ({{identity-model}}).

`aud`
: REQUIRED - Identifies the Authorization Server: its issuer identifier {{RFC8414}}, as a single value, as in {{IDJAG, Section 3.1}}.  An Authorization Server MUST accept its issuer identifier as the audience; it MAY also accept its token endpoint URL, which {{RFC7523BIS}} continues to permit for authorization grants.

`exp`, `iat`, `jti`
: REQUIRED - As defined in {{RFC7519}}.

`scope`
: OPTIONAL - A space-separated list of scopes ({{RFC6749, Section 3.3}}) the Workload Identity Provider asserts for this request, as in {{IDJAG, Section 3.1}}.  The Authorization Server decides under its own policy which of them to grant, and MAY grant a subset ({{IDJAG, Section 4.4.1}}).

`resource`
: OPTIONAL - The resource the Agent intends to access, as a single absolute URI as for the `resource` parameter ({{RFC8707, Section 2}}), as in {{IDJAG, Section 3.1}}.  REQUIRED in a Workload Authorization Grant issued by exchange ({{exchange}}).  When the claim is present, the Authorization Server MUST reject with `invalid_target` a token request whose `resource` parameter is absent or not identical to the claim, and MUST restrict the audience of the access token it issues to that resource.

`nickname`
: OPTIONAL - A human-readable nickname for the Agent, for display only and never an input to authorization ({{security-considerations}}).  Issuers SHOULD limit `nickname` to 64 characters; recipients MAY truncate it for display or ignore it.


The assertion is signed under a key configured from the Workload Identity Provider (see {{issuer-keys}}) and MAY carry further claims about the Agent.
See {{as-metadata}} for details on how an Authorization Server advertises support for this grant.

The following is a non-normative example of a JWT header and claims set, as the Agent in {{permissions-example}} would present it to the ticket tracker:

~~~
{
  "typ": "oauth-wag+jwt",
  "alg": "ES256",
  "kid": "key-2026-10"
}
.
{
  "iss": "https://acme.agents.platform.example",
  "sub": "agent/7f3d9as3",
  "nickname": "Database Migration helper",
  "roles": ["databases-team"],
  "aud": "https://as.saas.example",
  "exp": 1785271980,
  "iat": 1785271680,
  "jti": "7d0f5a2b-93c8-4f0e-9c33-1b6a0e6d5f10",
  "scope": "tickets:read comments:write"
}
~~~

# Workload Identity Provider Registration {#provider-registration}

Prior to presenting a WAG to an Authorization Server, an administrator registers the Workload Identity Provider at the Authorization Server. During this registration step, the Authorization Server obtains the Workload Identity Provider's issuer identifier, the issuer's key, and tenant information (see {{tenants}}). The Authorization Server also decides on authorization policy for the Workload Identity Provider including optionally mapping claims provided by the Workload Identity Provider to permissions. The specifics of this registration step are outside the scope of this document. It is not a client registration {{RFC7591}} and yields no client identifier or credential.

## Issuer Keys {#issuer-keys}
As part of a Workload Identity Provider registration, the Authorization Server needs to record an issuer identifier and obtain the public keys associated with that issuer.

A Workload Identity Provider may provide its public key via: a JWK Set {{RFC7517}} entered directly, a JWK Set URL the Authorization Server fetches over HTTPS {{RFC9525}}, or the `jwks_uri` in metadata the issuer publishes under its issuer identifier ({{RFC8414, Section 3}} or {{OIDC-DISCOVERY}}).  An Authorization Server that uses issuer metadata MUST NOT use a document whose `issuer` value is not identical to the registration's issuer identifier ({{RFC8414, Section 3.3}}).  A Workload Identity Provider SHOULD publish its keys at a URL, so that keys can rotate without administrator action.

On each assertion the Authorization Server finds the Workload Identity Provider registration the assertion matches: `iss` equals the registration's issuer identifier by Simple String Comparison ({{RFC7523, Section 3}}) and, where the registration names a claim ({{tenants}}), the assertion carries that claim with the registered value.  An Authorization Server MUST ensure that an assertion can match at most one of its registrations.  The Authorization Server MUST reject an assertion whose `typ` header is not `oauth-wag+jwt` ({{authorization-grant-claims}}), comparing as described in {{RFC7515, Section 4.1.9}}, MUST reject an assertion that matches no registration, MUST verify the signature only under a key configured or retrieved for the matched registration's issuer identifier - never under key material or key locations carried in the assertion ({{RFC8725, Section 3.8}} and {{RFC8725, Section 3.10}}) - and MUST interpret `sub` and `jti` only within the scope of the matched registration.

## Permissions {#properties}
During Workload Identity Provider registration, the Authorization Server sets local policy for what permissions to assign an access token given in return for a WAG.  This policy MAY involve consulting claims, other than `nickname`, that the Workload Identity Provider asserts about the Agent in the WAG. A claim is an assertion by the Workload Identity Provider, meaningful only within the context of that Workload Identity Provider, and an Authorization Server MUST NOT assume that a similarly named value from another Workload Identity Provider means the same thing.

The specific claims a Workload Identity Provider provides, and what permissions an Authorization Server decides to grant are outside the scope of this document.

### Permissions Example {#permissions-example}

This example is non-normative.  It illustrates one shape the permission decision can take.

* The Workload Identity Provider is configured to include a claim such as `"roles": ["databases-team"]` in its assertions.
* An administrator configures the Platform so that members of the databases team can create Agents that have that role.
* Each Authorization Server has its own concept of groups, roles, and permissions, and its administrator configures what access to grant based on a `roles` claim containing `databases-team`.
* At a ticket tracker, the role is mapped to less than the team itself has: reading tickets, and writing comments on them, in projects associated with the team.
* At a log service, the role is mapped to exactly the access the team has.
* A member of the team creates an Agent with a nickname "Database Migration helper".  On its first assertion at either Authorization Server, the Agent receives an access token with the permissions mapped there, without further registration.
* A different member of the team creates an Agent with a nickname "Latency optimizer". On its first assertion it also receives an access token with the mapped permissions without further registration.

The claim names are used for illustrative purposes only. Inclusion of such claims is optional, and Workload Identity Providers may define specific claims depending on their deployment, supported access control models and ecosystem.

## Multi-Tenancy {#tenants}

In many cases, a deployment (Workload Identity Provider or AS/RS) will partition its infrastructure by customer organizations, or tenants.  For the purposes of this document, a Workload Identity Provider and Authorization Server / Resource Server refers to a single partition belonging to a single organization ({{conventions}}). A Workload Identity Provider that knows the organization's identifier at the Authorization Server can carry it in the assertion, as the `aud_tenant` claim of {{IDJAG, Section 3.1}} does; this document does not require it.

Where each Workload Identity Provider has its own issuer identifier, the issuer identifier alone identifies the Workload Identity Provider and nothing further in this section applies.  Where several Workload Identity Providers share one issuer identifier, a claim in the assertion tells them apart.  Existing issuers use different claims for this, so this document does not fix the claim's name: the Workload Identity Provider registration includes the claim and the value it carries for that Workload Identity Provider, and the Authorization Server applies both when matching an assertion ({{issuer-keys}}).  An assertion that lacks the named claim, or carries another value, does not match that registration.

It is RECOMMENDED that deployments use dedicated issuers for partitions. If that's not possible, it is RECOMMENDED to use the `tenant` claim ({{IDJAG, Section 3.1}}) in order to simplify interoperability.

How an Authorization Server determines whether a Workload Identity Provider needs a differentiating claim, and which, is left to be discovered out of band of this specification.

# Workload Authorization Grant Exchange {#exchange}

An Agent can hold a Workload Authorization Grant from one Workload Identity Provider while the Authorization Server for the resource it wants to access trusts another.  In that case the Agent exchanges its Workload Authorization Grant, by token exchange {{RFC8693}}, at the Workload Identity Provider the Authorization Server trusts (the target issuer), for a Workload Authorization Grant issued by the target issuer.

The target issuer acts as an Authorization Server in the sense of this document: it holds a Workload Identity Provider registration ({{provider-registration}}) for the Workload Identity Provider that issued the presented Workload Authorization Grant, and the presented Workload Authorization Grant is addressed to it.

~~~
 Workload                  Target Issuer    Authorization   Resource
 Identity Provider  Agent  (Workload Id. P.)  Server (AS)   Server (RS)
      |              |            |                |             |
      |--- WAG ----->|            |                |             |
      |  aud=target  |            |                |             |
      |   issuer     |-- token -->|                |             |
      |              |  exchange  |                |             |
      |              |  audience=AS                |             |
      |              |  resource=RS                |             |
      |              |<-- WAG ----|                |             |
      |              |  aud=AS, resource=RS        |             |
      |              |--- POST /token ------------>|             |
      |              |    grant_type=jwt-bearer    |             |
      |              |    assertion=<WAG>          |             |
      |              |    resource=<RS>            |             |
      |              |<-- access token ------------|             |
      |              |--- request + access token --------------->|
~~~
{: #fig-exchange title="Exchanging a Workload Authorization Grant at the target issuer"}

## Exchange Request {#exchange-request}

The Agent makes a token exchange request ({{RFC8693, Section 2.1}}) to the target issuer's token endpoint with the following parameters:

`grant_type`
: REQUIRED - `urn:ietf:params:oauth:grant-type:token-exchange`.

`subject_token`
: REQUIRED - The Workload Authorization Grant the Agent holds.

`subject_token_type`
: REQUIRED - `urn:ietf:params:oauth:token-type:wag`.

`requested_token_type`
: REQUIRED - `urn:ietf:params:oauth:token-type:wag`.

`audience`
: REQUIRED - The issuer identifier {{RFC8414}} of the Authorization Server at which the Agent will present the issued Workload Authorization Grant, as a single value.

`resource`
: REQUIRED - The resource the Agent intends to access at that Authorization Server ({{RFC8707}}), as a single value.

`scope`
: OPTIONAL - The scopes the Agent requests at that Authorization Server ({{RFC6749, Section 3.3}}).

The request carries no `actor_token`; access on behalf of another principal is out of scope.  As in the token request of a Workload Authorization Grant, the Agent MAY make the exchange request without client authentication, and the target issuer MUST NOT require a client registration per Agent.

## Exchange Processing {#exchange-processing}

The target issuer validates the `subject_token` as an Authorization Server validates a Workload Authorization Grant ({{issuer-keys}}), with its own issuer identifier as the expected `aud`, and identifies the Agent by the `iss` and `sub` of the `subject_token` ({{identity-model}}).  The target issuer MUST reject a request that lacks `audience` or `resource` with `invalid_request`, and a request whose `audience` or `resource` it will not issue a Workload Authorization Grant for with `invalid_target` ({{RFC8693, Section 2.2.2}}).  Whether to issue a Workload Authorization Grant, and with which claims, is decided by the target issuer's local policy, which MAY consult the claims of the `subject_token` ({{properties}}).

The issued Workload Authorization Grant is a Workload Authorization Grant as defined in {{authorization-grant-claims}}, where:

* `iss` is the target issuer's issuer identifier, `aud` is the value of the `audience` parameter, and `resource` is the value of the `resource` parameter.
* `sub` is an Agent Identifier under the target issuer ({{identity-model}}).  It MAY differ from the `sub` of the `subject_token`; how the target issuer maps one to the other is its local policy.
* Other claims MAY differ from those of the `subject_token`.  They are assertions of the target issuer, interpreted within its Workload Identity Provider registration ({{properties}}).
* `exp` SHOULD NOT be later than the `exp` of the `subject_token`.

The issued Workload Authorization Grant MAY carry further claims, including information about the issuer and subject of the `subject_token`; this document defines none.  A recipient MUST NOT use such a claim to identify the Agent.

## Exchange Response {#exchange-response}

On success, the target issuer responds as in {{RFC8693, Section 2.2.1}}, with the issued Workload Authorization Grant in `access_token`, `issued_token_type` set to `urn:ietf:params:oauth:token-type:wag`, and `token_type` set to `N_A`.  The target issuer MUST NOT issue a refresh token.

The Agent MUST treat the issued Workload Authorization Grant as opaque: it does not inspect it or base decisions on its contents.  The Agent presents it unmodified as the `assertion` in a token request to the Authorization Server identified by `audience`, with the same `resource` ({{authorization-grant-claims}}).  The issued Workload Authorization Grant can itself be the `subject_token` of a further exchange.

How an Agent determines the target issuer for an Authorization Server is out of scope.

## Exchange Example {#exchange-example}

The following non-normative example shows the Agent of {{permissions-example}}, whose Workload Identity Provider is not trusted by the ticket tracker, exchanging its Workload Authorization Grant at the target issuer `https://wip.corp.example`, which the ticket tracker's Authorization Server trusts.  Line breaks are for display only.

~~~
POST /token HTTP/1.1
Host: wip.corp.example
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&subject_token=eyJ0eXAiOiJvYXV0aC13YWcrand0Ii...
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Awag
&requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Awag
&audience=https%3A%2F%2Fas.saas.example
&resource=https%3A%2F%2Ftickets.saas.example
&scope=tickets%3Aread%20comments%3Awrite
~~~

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJ0eXAiOiJvYXV0aC13YWcrand0Ii...",
  "issued_token_type": "urn:ietf:params:oauth:token-type:wag",
  "token_type": "N_A",
  "expires_in": 300
}
~~~

The claims of the issued Workload Authorization Grant, in which the target issuer has assigned its own Agent Identifier and mapped the role to one it asserts:

~~~
{
  "iss": "https://wip.corp.example",
  "sub": "wl-51c0e2",
  "groups": ["eng-databases"],
  "aud": "https://as.saas.example",
  "resource": "https://tickets.saas.example",
  "exp": 1785271980,
  "iat": 1785271690,
  "jti": "0b6f3c1e-4a7d-4b8e-a2f9-6c1d9e3f7a20",
  "scope": "tickets:read comments:write"
}
~~~

## Metadata {#exchange-metadata}

A target issuer that publishes metadata {{RFC8414}} SHOULD include `urn:ietf:params:oauth:grant-type:token-exchange` in `grant_types_supported`.


# Error Responses {#errors}

When a token request fails, the Authorization Server SHOULD indicate in `error_description` ({{RFC6749, Section 5.2}}) who must act: an administrator of the Authorization Server, if the Workload Identity Provider is not trusted or the Agent holds no permission for the request; or the Platform, if the assertion is invalid.  An untrusted Workload Identity Provider or an invalid assertion yields `invalid_grant` ({{RFC7523, Section 3.1}}); a missing permission yields `invalid_scope` or `invalid_target` ({{RFC8707}}) where a specific scope or resource is refused, otherwise `invalid_grant`.  When an action can be taken to resolve the issue, the Authorization Server SHOULD include a link in `error_uri`.

The same applies at a target issuer ({{exchange}}): an untrusted Workload Identity Provider or an invalid `subject_token` yields `invalid_grant`, and an `audience` or `resource` the target issuer will not issue a Workload Authorization Grant for yields `invalid_target`.


# Authorization Server Metadata {#as-metadata}

An Authorization Server that publishes metadata {{RFC8414}} SHOULD include the value `urn:ietf:params:oauth:grant-profile:wag` ({{iana-uri}}) in its `authorization_grant_profiles_supported` metadata parameter ({{IDJAG, Section 7.2}}).  An Authorization Server that includes this value MUST also include `urn:ietf:params:oauth:grant-type:jwt-bearer` in `grant_types_supported`.

The following is a non-normative example of Authorization Server metadata advertising support for this specification:

~~~
{
  "issuer": "https://as.example.com",
  "token_endpoint": "https://as.example.com/token",
  "grant_types_supported": [
    "urn:ietf:params:oauth:grant-type:jwt-bearer"
  ],
  "authorization_grant_profiles_supported": [
    "urn:ietf:params:oauth:grant-profile:wag"
  ]
}
~~~


# Open Issues {#oi}

* Agent ownership: see issue #13.
* Proof of possession: the grant is a bearer assertion and no client authentication is required; whether to name a hardening (sender-constrained access tokens, authenticating the presenting instance, or the Platform authenticating as a client) and which, if any, to require.
* Replay: whether an Authorization Server is required to reject a `jti` it has already accepted while the assertion is still valid, or whether that stays optional as in {{RFC7523, Section 3}}.
* Exchange discovery: how an Agent determines the target issuer for an Authorization Server, and whether a target issuer advertises the `subject_token_type` values it accepts ({{exchange}}).
* Exchange provenance: whether to define a claim recording the issuer and subject of the `subject_token` in a Workload Authorization Grant issued by exchange, such as `act` ({{RFC8693, Section 4.1}}) or a new claim.

# Security Considerations {#security-considerations}

This revision lists the considerations it is aware of; a fuller treatment will follow.

* Agents are accepted on their first assertion, so the set of acceptable Agents grows at the Platform with no action at the Authorization Server, and each new Agent creates state there; an Authorization Server can cap new Agents per Workload Identity Provider registration.
* The assertion is a bearer credential: a short lifetime, its `aud` and, where the Authorization Server enforces it, single use by `jti` bound what a stolen assertion is worth.
* Keys are held per issuer identifier, so that one issuer's key never verifies another's assertion ({{issuer-keys}}); whoever controls an issuer identifier, or the DNS name under it, controls what every trusting Authorization Server accepts.
* Workload Identity Providers under a shared issuer identifier share its keys, so the claim that tells them apart ({{tenants}}) is only as trustworthy as the party signing for all of them, and a Workload Identity Provider registration for a shared issuer identifier that names no claim trusts every Workload Identity Provider under it.
* Where one Authorization Server serves several organizations, a Workload Identity Provider registration created by the wrong organization routes another organization's Agents to it; who may register a given Workload Identity Provider is out of scope.
* The `nickname` claim is free text set at the Platform: it is not unique, can change, and can imitate another Agent or a person, including with visually similar characters.  It MUST NOT be used in an authorization decision or as an identifier, by the Authorization Server or by any party it passes the nickname to; {{identity-model}} says what identifies an Agent.
* A target issuer that registers a Workload Identity Provider ({{exchange}}) lets every Agent of that Workload Identity Provider obtain Workload Authorization Grants under the target issuer's own issuer identifier.  Authorization Servers trusting the target issuer see only the target issuer, so its policy is where that trust is bounded.
* A Workload Authorization Grant issued by exchange is bound to one Authorization Server by `aud` and to one resource by `resource`, which bounds what a stolen one is worth.
* Each exchange can extend the lifetime of access unless the issued `exp` is bounded by that of the `subject_token`; chained exchanges compound this.
* Where the target issuer assigns its own Agent Identifier, downstream Authorization Servers and Resource Servers see only that identifier; correlating it with the Agent's identity at its own Workload Identity Provider relies on the target issuer's records.
* Error responses ({{errors}}) tell any presenter which Workload Identity Providers an Authorization Server trusts, and `error_uri` hands a link to an unauthenticated presenter.

# IANA Considerations

## Media Types {#iana-media-type}

This section registers `oauth-wag+jwt`, a new media type {{RFC2046}} in the "Media Types" registry {{IANA.media-types}} in the manner described in {{RFC6838}}.  It can be used to indicate that the content is a Workload Authorization Grant.

## OAuth URI Registration {#iana-uri}

This section registers `urn:ietf:params:oauth:token-type:wag` in the "OAuth URI" subregistry of the "OAuth Parameters" registry {{IANA.oauth-parameters}}.

* URN: urn:ietf:params:oauth:token-type:wag
* Common Name: Token type URI for a Workload Authorization Grant
* Change Controller: IETF
* Specification Document: This document

This section registers `urn:ietf:params:oauth:grant-profile:wag` in the "OAuth URI" subregistry of the "OAuth Parameters" registry {{IANA.oauth-parameters}}, for use in the `authorization_grant_profiles_supported` metadata parameter ({{as-metadata}}).

* URN: urn:ietf:params:oauth:grant-profile:wag
* Common Name: Authorization grant profile identifier for a Workload Authorization Grant
* Change Controller: IETF
* Specification Document: {{as-metadata}} of this document

--- back

# Acknowledgments
{:numbered="false"}

The editors thank Pieter Kasselman, Karl McGuinness, Kevin Kelley, Emily Lauber, and Maxwell Gerber for discussions that shaped this document.
