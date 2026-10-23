# Decision log

Design decisions for draft-carleton-workload-authz-grant, with the alternatives considered.
Most recent first within each state. Open items are at the bottom.

## Decided

### D15 - WIP, see PR description

### D13 — WAG is a standalone mechanism, not a profile of AIMS (2026-08-31, closes #1)
The draft no longer calls itself a profile of draft-klrc-aiagent-auth
(AIMS). AIMS moves from the normative to the informative references; the
"Relationship to AIMS" section becomes an informative correspondence table
so that a reader who knows AIMS can place WAG in its model (agent acting on
its own behalf, Section 10.4.2), and a new Concepts/Overview section with an
end-to-end figure lets the document stand on its own. Rationale (issue #1
and the 2026-08-26 adoption review): an implementer at a SaaS authorization
server should not have to read a second draft to implement this one, and
"profile" implied a normative dependence the text never actually used --
every rule here is stated in terms of RFC 7523, RFC 8414 metadata and
published JWK Sets. Alternatives: keep the profile framing and inherit AIMS
policy/compliance sections (rejected: inheritance is exactly the dependence
adopters objected to); drop the AIMS reference entirely (rejected: AIMS
remains the shared vocabulary in the WIMSE/OAuth discussion and the
correspondence is genuinely useful to that audience). The table-of-contents
part of #1 was addressed by PR #5 (Arndt Schwenkschuster).

### D14 — Editor group (2026-08-31)
Nick Steele (OpenAI), Aaron Parecki (Okta), Arndt Schwenkschuster (Defakto
Security) and Brian Campbell (Ping Identity) join Paul Carleton (editor) on
the front page from -01, with push access to the repository. Convention
(CONTRIBUTING.md): proposals arrive as issues; editors write normative text;
outside pull requests to normative text are reviewed as input.


### D12 — Mechanism name: Workload Authorization Grant (WAG); docname draft-carleton-workload-authz-grant (2026-08-03)
Supersedes D1 and D2. Pieter Kasselman's critique of JIF: "just-in-time"
misleads (federation is established ahead of time; only per-workload
registration is removed). WAG names the actual artifact (the RFC 7523
authorization grant), rhymes with ID-JAG whose role it parallels for
workloads on their own behalf, avoids the "federation" term, and does not
tie the mechanism to agents (agent platforms are the motivating deployment,
not the scope). Brian Campbell concurred; Aaron Parecki adopted it and
contributed the JWT claims subsection. Alternatives (re-)considered: WIF
(most recognizable, but claims a term with variable wire semantics); A-WIF /
WIFA (agent-tied); JIF (superseded per above).

Docname `draft-carleton-workload-authz-grant`: bare-acronym docnames are
non-idiomatic for individual drafts (a datatracker sample found only DPoP
and RAR; ID-JAG's own slug spells it out as identity-assertion-authz-grant).
No WG segment in the slug and no `venue: group/type/mail` block: the
mechanism straddles OAuth (7523 grant mechanics) and WIMSE (workload
identity, trust domains), so naming a home before list discussion presumes
the answer. May add back after wimse@ and oauth@ feedback. "WAG" stays the
spoken/abbrev name.

### D11 — OBO is out of scope: gesture at actor-claim compatibility, specify nothing (2026-07-30)
The abstract and Scope now state that the profile covers agents acting on
their own behalf only. On-behalf-of flows are not specified here; the text
gestures at compatibility (the agent as actor rather than subject, via RFC
8693 act/actor_token or IDJAG) and leaves the composition to future
documents. This narrows the O4 debate's owed-regardless item about the
IDJAG pointer: the answer in THIS document is scoping, not normative
composition text. Alternatives: specify the IDJAG composition normatively
(rejected: expands the document beyond its exploratory purpose); say
nothing about OBO at all (rejected: reviewers would assume the design
forecloses it).

### D10 — Retirement: state the bound, defer the signal (2026-07-28)
Retirement is cessation of signing; the draft states the residual-access
bound (remaining assertion lifetime + access-token lifetime) normatively and
keeps epoch/event signaling in Open Issues as a gap shared with WIMSE and
SPIFFE (neither has per-credential revocation either). Broader lifecycle
management (e.g. succession planning for resources owned by a retired agent)
is explicitly out of scope.
Alternatives: specify an issuer- or property-scoped epoch mechanism
(premature, no ecosystem precedent); event push a la SSE/CAEP (unstandardized
everywhere).

### D9 — Terminology: standard AS/RS split (2026-07-28)
"Authorization Server" and "Resource Server" are used in their standard
OAuth senses. Alternatives: composite "Resource Server" meaning the combined
AS+resource role a SaaS product presents (rejected: forces readers to track
an RS containing its own AS, and misreads in both OAuth and WIMSE review
communities); a coined composite term like "SaaS Service" (rejected: not a
real word, and combining the roles does nobody any favors).

### D8 — Multi-issuer scoping is normative (2026-07-28)
Keys are resolved and cached per allowlist entry and never merged across
issuers; sub, jti, and property-to-permission mappings are interpreted only
within the presenting iss; policy/audit keys are (iss, sub) or the complete
URI-form identifier. Alternative: leave tuple semantics implicit (rejected:
an AS indexing agents by sub alone lets tenancy B's agent inherit tenancy
A's mapping).

### D7 — [withdrawn in -01, see D15] Agent Identifier: URI form RECOMMENDED, bare string permitted (2026-07-28)
The identifier MAY (and is RECOMMENDED to) be a URI-form workload identifier
with an opaque path; a bare opaque string remains permitted. Role split: the
AS validates the URI authority against the allowlisted issuer's tenancy once
at issuance; RSes exact-match the complete identifier as opaque and never
derive trust from its components. URI form costs no opacity and carries the
tenancy inside sub for relying parties that evaluate only subject/audience.
Alternatives: bare-string only (conflicted with the normative WIMSE-ID
citation, which requires absolute URIs); URI-required (excludes simple
platforms for no gain).

### D6 — [superseded by D15] aud carries both issuer identifier and token endpoint URL (2026-07-28)
The assertion's aud SHOULD list both values; an AS MUST accept either.
rfc7523bis adds the issuer identifier as an audience option for
authorization grants (issuer-only is mandated only for the separate
client-authentication slot); carrying both keeps one assertion valid across
deployed and future processing.
Alternatives: token-endpoint-only (the value the ecosystem is moving away
from); issuer-only (breaks deployed ASes that match on the endpoint);
SHOULD-issuer with endpoint fallback (more prose, same effect, worse
compatibility story than dual-listing).

### D5 — IdP projection may be just-in-time (2026-07-27)
Projection into an Enterprise IdP (e.g. SCIM) must not be required ahead of
time; performing it just in time, including synchronously during first token
issuance, is acceptable. Supersedes the earlier stricter "MUST NOT gate
agent creation or token issuance".

### D4 — Client identity deliberately unspecified (2026-07-27)
The profile attaches no semantics to client_id. Simplest deployment is
unauthenticated; the door stays open for the platform to authenticate as its
own OAuth client (e.g. CIMD + private_key_jwt) with the grant issued by an
Enterprise IdP. Alternatives: public client with client_id = agent
identifier (earlier position, reversed: forecloses the platform-as-client
composition); mandatory client authentication (too heavy a floor).
See notes/oauth-slots.md.

### D3 — RFC 7523 authorization grant, not token exchange or client credentials (2026-07)
The agent presents a platform-signed JWT in the assertion parameter
(grant slot, RFC 7523 s2.1, third-party issuer per RFC 7521 s3).
Alternatives: RFC 8693 token exchange (requires a subject_token_type no
deployed AS ships; more moving parts at the endpoint); client credentials
with a 7523 client assertion (makes the agent the OAuth client, forcing
sub == client_id, which conflates the subject-of-the-grant with the
client role). Chosen for the adoption goal: jwt-bearer is the one grant
type every mainstream AS already implements.

### D2 — [superseded by D12] docname and repo: draft-carleton-jif (2026-07-27)
Short, speakable, matches how people will refer to it. Alternative:
draft-carleton-aims-agent-platform (descriptive but unpronounceable in
conversation).

### D1 — [superseded by D12] Mechanism name: just-in-time identity federation (JIF) (2026-07-27)
Criteria: pronounceable acronym; shape-legible to workload identity
federation without being called WIF; modest namespace claim; names the
mechanism (no registration step) rather than the audience.
Alternatives: WIFA "workload identity federation for agents" (pronounceable
and shape-clear, but contains WIF verbatim and concedes the agents-are-just-
workloads framing); AWIF (worse mouth-feel, buries the legible part); AIF
"agent identity federation" (unpronounceable, overclaims a broad term);
SCUBA "(S...) Credentials Under Backend Authorization" (memorable, zero
shape-legibility; reserved as a candidate name for a future standalone
protocol document).

## Open

### O1 — Proof of possession mechanism
Bearer grant is the adoption floor (with jti replay rejection and short
lifetimes under consideration as mandatory). Candidates for the hardened
mode: attestation-based client authentication (platform as attester,
instance-signed PoP JWT); DPoP (binds the issued access token, but requires
RS-side changes, cutting against the token-endpoint-only adoption goal);
assertion cnf + key-matched DPoP proof at the token endpoint (binds the
assertion itself to an instance key). See notes/oauth-slots.md.

Refinement (2026-07-28): client authentication also closes assertion theft,
IF the profile adds a binding rule -- "assertions from allowlisted issuer I
are accepted only from its bound, authenticated client C" (RFC 7523 does not
link the grant to client authentication by itself). Given that rule, the
choice between CIMD+private_key_jwt and attestation-based client auth is
topology, not capability: pkjwt authenticates the Platform with one client
key (right when the platform core fronts all token requests; platform-wide
blast radius on key theft), attestation pushes a key into each instance
(right when instances call the AS directly; per-instance blast radius, and
the AS can require attestation sub == grant sub). DPoP remains the
orthogonal token-binding layer (protects stolen access tokens, costs
RS-side changes).

Sharper cut (2026-07-28, paulc): against a whole-request thief (inside TLS
termination) every shape degrades to replay-within-freshness-window -- the
pkjwt/PoP artifacts travel next to the assertion, so signing buys nothing
there; short exp + jti replay caching are the real defense and belong in the
floor. Client authentication only matters when the assertion can exist APART
from the caller's key material: issuer != caller (IdP-issued grants),
at-rest leaks (logs/queues), or the AS wanting a caller identity for
quota/kill-switch. pkjwt vs attest is then just which boundary is being
defended: issuer != caller (platform key) vs platform != instance
(per-instance keys). Caveat (paulc): attest's per-instance blast radius
applies only to edge keys -- the attester key is itself a platform-level
durable key (~= the issuer key; possibly the same key), so at the root
pkjwt and attest carry identical concentration risk.

### O2 — Property claim naming and registration
Unregistered generic names (roles, groups) crossing an inter-organizational
boundary will not survive review. Options under consideration:
(a) reuse existing IANA-registered claims (name from OIDC Core; roles,
groups, entitlements from RFC 9068/SCIM semantics) and register only the
genuinely new ones (namespace, ctx);
(b) one registered container claim (e.g. agent_properties) holding the
vocabulary, avoiding all top-level collisions;
(c) collision-resistant (URI-prefixed) interim names;
(d) present the options in the -00 text and let reviewers weigh in.

### O3 — "Relationship to WIMSE and SPIFFE" section text
Outline agreed (slot argument; identifier compatibility; identity-plane
backing changes nothing at the RS boundary). Drafting waits on O1 and O2
since both feed the section's content.
-01 drops the section altogether (see D15); reopen if it returns.

### O4 — Agent as grant-subject vs agent as OAuth client
The expected flashpoint of external review. D3/D4 put the agent in the
grant slot's sub (RFC 7523 s2.1) and deliberately leave the client slot
empty; the strongest challenge is that the agent should BE the OAuth
client, authenticated per RFC 7523 s2.2. Deliberately unresolved in -00;
both cases at full strength below. Side-by-side request shapes for the
three scenarios (self-access, user-granted email, enterprise OBO) are in
notes/client-models.html.

Case for agent==client: the agent is what RFC 6749 calls a client -- it
initiates token requests, holds keys in the hardened mode (O1), and needs
the quotas, kill switches, and revocation this draft cites as motivation.
Deployed AS machinery for managing callers is keyed on client_id (consent
records, RFC 7009 revocation, per-client limits, client_id in RFC
7662/9068); under (iss, sub) every adopting AS rebuilds a parallel copy,
a cost the draft currently understates. The adopted OAuth WG stack points
this way: CIMD (URL client_ids, zero registration writes, s8.9 prefix
allowlists), attestation-based client auth (platform-vouched instance
keys), SPIFFE client auth (workloads as clients). Three internal
inconsistencies hold regardless of outcome: (a) the Scope pointer to
IDJAG does not compose with the D4 unauthenticated floor -- IDJAG s4.4.1
matches the ID-JAG's client_id against an AUTHENTICATED client, and there
is none; (b) "grant MY email to THAT agent" has no enforceable wire form
-- an authorization request names a client_id and the agent is not one;
(c) the unauthenticated floor leaves the AS no caller identity to
throttle or kill. Agent-as-client yields O1's hardened mode by
construction (stolen assertion useless without the attested cnf key),
gives D10 an enforcement surface (platform stops attesting; refresh
chains die), and names individual agents on consent screens with deployed
machinery. It also settles the tension O3 must otherwise explain: the
draft RECOMMENDS Workload Identifiers -- calls agents workloads -- while
declining the workload-equals-client pattern the WG adopted for
workloads.

Case for agent==subject (the current design): no adopted document mints a
client_id per instance. ATTEST-10 s4 REQUIRES attestation sub == the
shared client_id (instances differ only by cnf key); SPIFFE client auth's
wildcard CIMD document (s3.1.2, s5.1) exists precisely so the AS-visible
unit is the family; and both McGuinness individual drafts
(client-instance-assertion-00, ai-agent-instance-00) keep one logical
client_id and surface the instance as sub when self-acting and act.sub in
OBO tokens -- this draft's exact split. Client-per-agent is itself a
novel composition of three pre-RFC drafts, and it regresses the D3
adoption floor: jwt-bearer ships in every mainstream AS; ATTEST,
CIMD-resolving ASes, and CIMD-resolving IdPs ship essentially nowhere.
The delegation grammar is not missing: act/may_act (RFC 8693 s4.1, s4.4)
are IANA-registered claims and actor_token (s2.1) is the registered
second grant-side slot -- delegation has two parties on the grant side by
design. client_id-keyed machinery assumes low-cardinality application
identity; family-grouping by client_id prefix is the
derive-structure-from-identifier move {{identity-model}} forbids for sub,
while (iss, sub) keeps the family/instance relation structural (D7/D8).
Per-agent refresh tokens reintroduce the AIMS Section 7 antipattern
(durable per-agent credentials, instance-key escrow) against the
no-refresh-token design, and CIMD s8.4.1 retirement is a discretionary
MAY on an eventual re-fetch versus D10's normative cessation bound. Root
blast radius is identical either way (O1's sharpened cut: the attester
key is a platform-level durable key). Deployed proof: GitHub Apps -- one
client, millions of installations as grant-side subjects. Every debt the
client case surfaces closes with profile text inside the current
architecture (below).

Crux: per-agent state (grants, jti replay, audit) scales identically
under both designs; the choice is which invention the draft signs up for.
Agent==client inherits deployed client management but must invent
per-instance client identity -- extending the adopted family-granularity
stack to instance cardinality (per-agent client_ids, consent and refresh
custody semantics, IdPs that resolve them): invent new things for client
management. Agent==subject inherits the registered delegation grammar but
must invent the management and binding plane on (iss, sub) -- quotas,
kill switches, token listing, consent binding, act-in-ID-JAG: invent new
things for representing delegations. No third stack exists; the
tiebreakers are the adoption floor and which primary key the ecosystem's
per-agent state accretes under.

Deciding agent==client commits the draft to: reversing D3/D4; normative
dependence on CIMD-02 + ATTEST-10 (+ IDJAG-04 for OBO), all pre-RFC and
moving; URL-form client_ids and new client-auth code paths through every
AS layer, narrowing the token-endpoint-only pitch to nothing; owning the
family/instance data model and family-level consent semantics (today
formalized only in individual -00s); refresh custody for ephemeral
instances; near-term IdP-side registration of the agent family for OBO;
and two RS token shapes (sub = agent for self-access, sub = user +
client_id = agent for OBO). In exchange: per-agent consent, revocation,
quotas, and introspection arrive already keyed on client_id, and O1
largely dissolves by construction.

Deciding agent==subject commits the draft to: one normative sentence
closing the IDJAG loose end (in the Enterprise-IdP composition the
Platform MUST authenticate as an OAuth client -- D4's reserved
composition -- with the agent as actor_token at the exchange and act in
the ID-JAG, plus an interop note, since IDJAG-04 does not yet specify
act); a per-agent consent binding (RFC 9396 authorization_details naming
the agent; grant keyed (user, tenancy client, agent (iss, sub));
redemption and refresh require a matching fresh assertion as actor_token)
that no off-the-shelf AS enforces today; honest costing of the (iss, sub)
management plane plus at least a SHOULD for client authentication above
the floor; a sub disambiguator so RSes do not misread agent tokens as
human ones; and the O3 text arguing subject-granularity instance identity
versus family-granularity client identity. In exchange: the D3 adoption
floor holds, no per-agent durable credentials exist to steal or escrow,
and retirement keeps its normative bound.

Hybrid candidates: tiered profile (base = today's subject-form
jwt-bearer; OBO tier = IDJAG with mandatory platform client auth and
agent-as-actor; hardened tier = O1 row 3); or a per-agent client
"upgrade" reserved for user-granted personal resources (cost: the same
agent appears as client_id in one flow and sub in others). See
notes/oauth-slots.md.
