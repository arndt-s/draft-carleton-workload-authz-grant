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
  RFC6749:
  RFC7517:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC8414:
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
3. Per access: the Agent presents a Workload Authorization Grant in an
   ordinary OAuth token request.  The Authorization Server matches it to a
   Workload Identity Provider registration, verifies it under that Workload Identity Provider's
   keys,
   allowing for previously unseen `sub` values,
   and issues an access token under its own policy ({{properties}}).

~~~
      Platform                    Authorization               Resource
      (Workload Identity          Server (AS)                 Server (RS)
       Provider; Agents)
            |                          |                          |
  (1)  [administrator registers the Workload Identity Provider]   |
            |                          |                          |
  (2)  [Platform creates Agent; nothing sent to AS or RS]         |
            |                          |                          |
  (3)       |--- POST /token --------->|                          |
            |    grant_type=jwt-bearer |                          |
            |    assertion=<WAG>       |                          |
            |    resource=<RS>         |                          |
            |<-- access token ---------|                          |
            |--- request + access token ------------------------->|
~~~
{: #fig-overview title="One-time Workload Identity Provider registration, then per-request grants"}

# Agent Identity {#identity-model}

An Agent is identified by its Agent Identifier, carried as the `sub` claim in the assertion.  The Agent Identifier is opaque and immutable; it MUST be unique among all Agent Identifiers issued under the same Workload Identity Provider, MUST NOT be reassigned to a different Agent, and is compared as a case-sensitive string {{RFC7519, Section 2}}. An Authorization Server MUST associate records for an Agent with its Workload Identity Provider and `sub` value, never on its `sub` value alone.


# Workload Authorization Grant

An Agent obtains an access token by presenting a JWT as an authorization grant per {{RFC7523, Section 2.1}}, issued by the Workload Identity Provider as a third party in the sense of {{RFC7521, Section 3}}. The token request carries `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`, the JWT in the `assertion` parameter, and the target resource in the `resource` parameter {{RFC8707}}. The `resource` parameter {{RFC8707}} is RECOMMENDED; an Authorization Server SHOULD restrict the audience of the access token it issues to that resource and MAY refuse a request that lacks it with `invalid_target` ({{RFC8707, Section 2}}). An Agent MAY make the token request without client authentication ({{RFC7523, Section 3.1}}), and this specification attaches no meaning to `client_id`. An Authorization Server MUST NOT require a client registration per Agent.

Assertions SHOULD be short-lived.  The Authorization Server MUST NOT issue refresh tokens for this grant and SHOULD NOT issue access tokens that outlive the assertion by a significant period ({{RFC7521, Section 4.1}}).

## JWT Syntax {#authorization-grant-claims}

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


The assertion is signed under a key configured from the Workload Identity Provider (see {{issuer-keys}}) and MAY carry further claims about the Agent. An Authorization Server that publishes metadata {{RFC8414}} SHOULD list the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type in `grant_types_supported`.

```
{
  "iss": "https://acme.agents.platform.example",
  "sub": "agent/7f3d9as3",
  "aud": "https://as.saas.example",
  "exp": 1785271980,
  "iat": 1785271680,
  "jti": "7d0f5a2b-93c8-4f0e-9c33-1b6a0e6d5f10",
  "scope": "issues:read issues:write"
}
```

# Workload Identity Provider Registration {#provider-registration}

Prior to presenting a WAG to an Authorization Server, an administrator registers the Workload Identity Provider at the Authorization Server. During this registration step, the Authorization Server obtains the Workload Identity Provider's issuer identifier, the issuer's key, and tenant information (see {{tenants}}). The Authorization Server also decides on authorization policy for the Workload Identity Provider including optionally mapping claims provided by the Workload Identity Provider to permissions. The specifics of this registration step are outside the scope of this document. It is not a client registration {{RFC7591}} and yields no client identifier or credential.

## Issuer Keys {#issuer-keys}
As part of a Workload Identity Provider registration, the Authorization Server needs to record an issuer identifier and obtain the public keys associated with that issuer.

A Workload Identity Provider may provide its public key via: a JWK Set {{RFC7517}} entered directly, a JWK Set URL the Authorization Server fetches over HTTPS {{RFC9525}}, or the `jwks_uri` in metadata the issuer publishes under its issuer identifier ({{RFC8414, Section 3}} or {{OIDC-DISCOVERY}}).  An Authorization Server that uses issuer metadata MUST NOT use a document whose `issuer` value is not identical to the registration's issuer identifier ({{RFC8414, Section 3.3}}).  A Workload Identity Provider SHOULD publish its keys at a URL, so that keys can rotate without administrator action.

On each assertion the Authorization Server finds the Workload Identity Provider registration the assertion matches: `iss` equals the registration's issuer identifier by Simple String Comparison ({{RFC7523, Section 3}}) and, where the registration names a claim ({{tenants}}), the assertion carries that claim with the registered value.  An Authorization Server MUST ensure that an assertion can match at most one of its registrations.  The Authorization Server MUST reject an assertion that matches no registration, MUST verify the signature only under a key configured or retrieved for the matched registration's issuer identifier - never under key material or key locations carried in the assertion ({{RFC8725, Section 3.8}} and {{RFC8725, Section 3.10}}) - and MUST interpret `sub` and `jti` only within the scope of the matched registration.

## Permissions {#properties}
During Workload Identity Provider registration, the Authorization Server sets local policy for what permissions to assign an access token given in return for a WAG.  This policy MAY involve consulting claims the Workload Identity Provider asserts about the Agent in the WAG. A claim is an assertion by the Workload Identity Provider, meaningful only within the context of that Workload Identity Provider, and an Authorization Server MUST NOT assume that a similarly named value from another Workload Identity Provider means the same thing.

The specific claims a Workload Identity Provider provides, and what permissions an Authorization Server decides to grant are outside the scope of this document.

## Multi-Tenancy {#tenants}

In many cases, a deployment (Workload Identity Provider or AS/RS) will partition its infrastructure by customer organizations, or tenants.  For the purposes of this document, a Workload Identity Provider and Authorization Server / Resource Server refers to a single partition belonging to a single organization ({{conventions}}). A Workload Identity Provider that knows the organization's identifier at the Authorization Server can carry it in the assertion, as the `aud_tenant` claim of {{IDJAG, Section 3.1}} does; this document does not require it.

Where each Workload Identity Provider has its own issuer identifier, the issuer identifier alone identifies the Workload Identity Provider and nothing further in this section applies.  Where several Workload Identity Providers share one issuer identifier, a claim in the assertion tells them apart.  Existing issuers use different claims for this, so this document does not fix the claim's name: the Workload Identity Provider registration includes the claim and the value it carries for that Workload Identity Provider, and the Authorization Server applies both when matching an assertion ({{issuer-keys}}).  An assertion that lacks the named claim, or carries another value, does not match that registration.

It is RECOMMENDED that deployments use dedicated issuers for partitions. If that's not possible, it is RECOMMENDED to use the `tenant` claim ({{IDJAG, Section 3.1}}) in order to simplify interoperability.

How an Authorization Server determines whether a Workload Identity Provider needs a differentiating claim, and which, is left to be discovered out of band of this specification.


# Error Responses {#errors}

When a token request fails, the Authorization Server SHOULD indicate in `error_description` ({{RFC6749, Section 5.2}}) who must act: an administrator of the Authorization Server, if the Workload Identity Provider is not trusted or the Agent holds no permission for the request; or the Platform, if the assertion is invalid.  An untrusted Workload Identity Provider or an invalid assertion yields `invalid_grant` ({{RFC7523, Section 3.1}}); a missing permission yields `invalid_scope` or `invalid_target` ({{RFC8707}}) where a specific scope or resource is refused, otherwise `invalid_grant`.  When an action can be taken to resolve the issue, the Authorization Server SHOULD include a link in `error_uri`.


# Open Issues {#oi}

* Agent ownership: see issue #13.
* Proof of possession: the grant is a bearer assertion and no client authentication is required; whether to name a hardening (sender-constrained access tokens, authenticating the presenting instance, or the Platform authenticating as a client) and which, if any, to require.
* JWT type: whether to define an explicit `typ` for this grant ({{RFC8725, Section 3.11}}), so that another kind of JWT signed by the same issuer for the same audience cannot be taken for it.
* Replay: whether an Authorization Server is required to reject a `jti` it has already accepted while the assertion is still valid, or whether that stays optional as in {{RFC7523, Section 3}}.

# Security Considerations {#security-considerations}

This revision lists the considerations it is aware of; a fuller treatment will follow.

* Agents are accepted on their first assertion, so the set of acceptable Agents grows at the Platform with no action at the Authorization Server, and each new Agent creates state there; an Authorization Server can cap new Agents per Workload Identity Provider registration.
* The assertion is a bearer credential: a short lifetime, its `aud` and, where the Authorization Server enforces it, single use by `jti` bound what a stolen assertion is worth.
* Keys are held per issuer identifier, so that one issuer's key never verifies another's assertion ({{issuer-keys}}); whoever controls an issuer identifier, or the DNS name under it, controls what every trusting Authorization Server accepts.
* Workload Identity Providers under a shared issuer identifier share its keys, so the claim that tells them apart ({{tenants}}) is only as trustworthy as the party signing for all of them, and a Workload Identity Provider registration for a shared issuer identifier that names no claim trusts every Workload Identity Provider under it.
* Where one Authorization Server serves several organizations, a Workload Identity Provider registration created by the wrong organization routes another organization's Agents to it; who may register a given Workload Identity Provider is out of scope.
* Error responses ({{errors}}) tell any presenter which Workload Identity Providers an Authorization Server trusts, and `error_uri` hands a link to an unauthenticated presenter.
* This document defines no explicit JWT type, so an issuer that signs other kinds of JWT for the same audience risks one being taken for this grant ({{RFC8725, Section 3.11}}).

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The editors thank Pieter Kasselman, Karl McGuinness, Kevin Kelley, Emily Lauber, and Maxwell Gerber for discussions that shaped this document.
