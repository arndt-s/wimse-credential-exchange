---
title: "WIMSE Credential Exchange"
abbrev: "WIMSE Credential Exchange"
category: info

docname: draft-schwenkschuster-wimse-credential-exchange-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: AREA
workgroup: "Workload Identity in Multi System Environments"
keyword:
 - workload
 - identity
 - credential
 - exchange
venue:
  group: "Workload Identity in Multi System Environments"
  type: "Working Group"
  mail: "wimse@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/wimse/"
  github: "arndt-s"
  latest: "https://ietf-wg-wimse.github.io/draft-ietf-wimse-s2s-protocol/draft-ietf-wimse-s2s-protocol.html"

author:
 -  fullname: Arndt Schwenkschuster
    organization: SPIRL
    email: arndts.ietf@gmail.com
    role: editor

normative:
  RFC4210:
  RFC7030:
  RFC7521: Assertion flow
  RFC8693: Token exchange

informative:

--- abstract

WIMSE defines Workload Identity and its representation through credentials. Typically, a credential is provisioned to the workload, allowing it to represent itself. The credential format is usually chosen by the platform. Common formats are JSON Web Tokens or X.509 certificates. However, workloads often encounter situations where a different identity or credential is required.

This document describes various situations where a workload requires another credential. It also outlines different ways this can be acchieved and compares them.

--- middle

# Introduction

Workload Identity credentials come in all shapes and forms. JSON Web Tokens are popular but also X.509 certificates are commonly used. When a workload is provisioned it can be assumed that it gets all of the following

* an identity in the form of an identifier.

* one or multiple credentials that allow the workload to represent itself (as that identity). Multiple credentials are often different types but representing the same identity.

* an indication of trust domain. (TODO not sure)

Identity, credential and trust domain enable the workload to interact within its environment, communicate to sibling workloads (same trust domain), access APIs inside that trust domain or provide an API itself.

## Needs

Why a workload cannot use its existing credential can have lots of reasons and subsequent need. The following list highlights the most common ones and is certainly not complete.

### Change in format

Workloads may require a different format representing the same identity in the same trust domain. Some concrete examples are:

* initial credential was an X.509 certificate but infrastructure requires application-level authentication such as JWT or Workload Identity Tokens as defined in (TODO).
* initial credential was a JWT bound to a key, requiring to be presented along with proof of possession but the peer does not support it and requires a Bearer credential.

Credential format is dificult to define. Some formats are opague to the workload and should remain that way. For instance how an OAuth Bearer token is construct and whether it carries claims or not is not a concern of the workload. That a Bearer token is required, however, is. Hence, for example, a change in format between a Bearer token and an X.509 certificate is certainly a change in format the workload can require. A different encoding of a Bearer token on the other hand is not and this specification is not meant for this.

### Change in scope

A credential in the same format representing the same identity but scoped differently. Examples are:

* A JWT credential audienced to interact with the Workload platform but access to other workloads are required. The workload is in need of JWTs with different, dedicated audiences.
* An X.509 credential is constrained to a certain key usage but the workload requires additional ones. For instance the existing certificate allows for `digitalSignature` but `keyEncipherment` or `dataEncipherment` is required.

Generally, scope should already be present and configured approperately with the workload platform only issuing narrowly scoped credentials to the workload.

In some situation the platform may only support the provisioning of a single credential and not support scoping it. If those cannot be requested by the platform itself an exchange may be necessary.

### Change in identity

A workload may be known under multiple identities. For example:

* A workload identity representing an exact physical instance may be eligable for a workload identity representing a logical unit that consists of many phyiscal instances. Another example is a workload running in a specific region being eligable for a more broader, geo scoped identity.
* A workload that can act on behalf of other workloads. These workloads often are part of infrastructure such as API-Gateways, proxies or service meshes in container environments.

### Change in trust domain

A provisioned workload identity is often part of a trust domain that is coupled to infrastructure or deployment. Workloads often require to intercept with other workloads or access outside resources located in other trust domains or reside in different trust domains. This requires the client workload to retrieve an identity of the other trust domain. Examples here include:

* Federation (a workload identity federates to a identity in a different trust domain). In existing workload identity environment OAuth2 with Token Exchange (TODO) and Assertion framework (TODO) are popular.
* A workload requires a credential of "higher trust" to interact with other workloads. This "higher trust" is facilitated by another trust domain. For instance a workload requiring a WebPKI certificate to offer a service to the world wide web.

### Change in lifetime

Credentials often come in time-restricted manners. Or usage may be restricted based on lifetime. For instance:

* A resource denies the long-lived credential the workload has availabe based on policy of maximum lifetime.
* An initial provisioned credentials has expired and renewal is not supported.
* A credential with shorter lifetime is desired to reduce replay risk.

### Missing provisioning support

A workload platform may not support the provisioning of credentials required by the workload. Technically, any of these would likely fall under the reasons above but it's a very common reason and often falls into multiple categories. As an example:

* Workload platform provisions identity & credential in the form of a simple signed document that carries the attributes attested by the platform but gives not access in any way.

### Combinations

Reasons and needs to exchange credentials are often not binary. A change in trust domain effectively is a change in identity too. A change in format can require a change in trust domain because formats come with different trust structures and security promises. E.g. a trust domain issuing JSON Web Tokens may not be able to issue WebPKI certificates.

## Provisioning, re-previsioning & exchange

Workloads have multiple options to aquire credentials in the way they are required. The following terms divides them into 3 main mechanism:

{:vspace}
Initial provisioning
: Credentials are issued during workload creation, the workload gets "born" with them. These credentials are fixed and pre-defined, often by configuration. The workload cannot influence their shape during runtime. Configuration may be changed to adjust initial provisioning based on the needs above.

On-demand provisioning
: Workloads are able to obtain credentials on-demand. Parameters allow the workload to specify exactly the required format, scope, identity, lifetime and more it requires. No authentication is necessary to request on-demand credentials. Workloads may choose to request additional credentials on-demand based on its needs.

Credential exchange
: Workload use a provisioned credential (on-demand or initial) to authenticate and authorize a request of a different credential. Based on parameters the workload can specify the exact attributes of the credential it requires. This is also on-demand based, however, the significant difference here is that this is an **authenticated** action, compared to on-demand provisioning, which is unauthenticated. Workloads may leverage credential exchange to obtain credentials based on its needs.

Based on the need some mechanisms is more feasible and better suited than others. The following table gives some guidance based on the identified need. The security considerations below also highlight some additional considerations, particularly {{use-on-demand-provisioning}}.

| Need | Preferred mechanism | Other options (in order) |
|-----|------|-----|
| Change in trust domain | Credential exchange | None |
| Change in identity | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange |
| Change in scope | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange |
| Change in format | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange |
| Change in lifetime | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange (only decrease, see {{exchange-to-renew}}) |

## Exchange patterns

### Format-specific exchange

Existing trust & identity framework often consist of a protocol or framework to exchange credentials. Leveraging this makes use of existing adoption and specific guidelines.

The following bullets give an overview of the existing patterns and when to use them based on the needs given above:

* OAuth Token Exchange {{RFC8693}} is:
  * meant for a change in scope.
  * meant for a change in identity.
  * to a certain extend meant for a change in format (limited).
  * NOT meant for a change in trust domain.

* OAuth Assertion Framework {{RFC7521}} is:
  * meant for a change in trust domain. As a result of the change in trust domain, a change in identity, scope & potentially format is unavoidable but not the primary use case.
  * NOT meant for exchanges within a trust domain.

* X.509 Certificate Management Protocol {{RFC4210}}:
  * Is this valid here? If yes, write about it.

* Enrollment over Secure Transport {{RFC7030}}:
  * Profile of {{RFC4210}}?
  * Is this valid here? If yes, write about it.

* SPIFFE federation:
  * Not a credential exchange but rather allowing for identities of different trust domains to trust each other and communicate securely. For SPIFFE the credential formats are JWT and X.509.

### On-behalf-of exchange

Workload environments can be highly dynamic and connected with a high variety of resources protected by different identity frameworks and formats. A format-agnostic, exchange component that exchanges credentials on behalf of the workload may be desired to remain control of credential issuance. For instance to enforce policy, collect audit trails or ease management.

~~~aasvg
+-----------------+ 2)request     +------------------------+  4)request      +---------------------+
|                 |   credential  |                        |    credential   |                     |
|  Workload       +-------------->|  Credential Exchanger  +---------------->|  Credential issuer  |
|                 |               |                        |                 |                     |
+-------^---------+               +----------+-------------+                 +---------------------+
        |                                    |
  1)initial provisioning                     |
        |                              3) validate
+-------+-----------------+                  |
|                         |                  |
|  Workload Platform      |<-----------------+
|                         |
+-------------------------+
~~~

TODO: describe steps

The author believes that a specific protocol that fits all credential formats and trust frameworks is not feasable while remaining the existing security promises of the format or framework. It rather believes that a profile for each scenario is the best way forward and welcomes them to profile this specificiation for their concrete use cases. As a general guidance it is reccommended

* to narrowly scope the scenarios and don't build a one-fits-all exchange for a specific format.
* to decouple authentication and access control from the actual exchange as best as possible. E.g. a credential of one profile should be allowed as a mean of authentication to exchange to a credential of a different profile, regardless if the profiles are aware of each other or not.

The "Credential Exchanger" shown in the figure may be the Workload Platform itself that offers this capability. Potentially also in a "re-provisioning" way without authentication.

# Consideration

TODO: add more.

## Credential exchange cannot increase trust

A credential exchange is an authenticated way to retrieve credential(s). Thus, the issued credential cannot have higher trust than the credential that was used to authenticate the request. This is particularly relevant when a credential is required which format and frameworks is of a higher trust than the one that was used to authenticate the request. This includes exchanging credentials not requiring proof of key possession to credentials carrying it.

Generally, these situations are not reccommended and should be avoided. Workloads SHOULD be provisioned with the credential of the highest trust and only retrieve less-trusted credentials via credential exchange.

Alternatively, the authentication request should be enriched with additional identification that increases the level of authentication. E.g. authentication and additional proof of platform attestation.

## Credential exchange cannot replace on-demand or initial provisioning

Because credential exchange is authenticated it cannot replace provisioning. Without an initial or on-demand requested credential a workload cannot facilitate credential exchange as there's no proof the workload is eligable for the requested credential.

## Initial provisioning comes with over-provisioning risk {#use-on-demand-provisioning}

Provisioning credentials preemptively risks being exposed to overprovisioning credentials that are not required. E.g. with initial provisioning, every workload is provisionied with a default credential, even though some don't require it (for instance because its just serving static content). This increases the risk of those credentials being unnecessarily exposed.

On-demand provisioning on the other hand only issues credential when requested and mitigates this. They are exactly in the scope, format, identity and lifetime that is require. This can significantly decrease the amount of unnecessarily issued & provisioned credentials.

## Expanding credential lifetime {#exchange-to-renew}

A change in lifetime of a credential can be critical if it can be used to effectively keep a credential alive. For instance a issued short-lived bearer credential that can be used to exchange for a new, longer lived credentials. Thus, it is highly recommended to only use on-demand provisioning to re-request a new credential.

Leveraging token exchange to request a shorter-lived credential which lifetime is within the bound of the credential used for authenticating the request remains valid.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Document History
<cref>RFC Editor: please remove before publication.</cref>

## draft-schwenkschuster-wimse-credential-exchange-xx

* Fix typo that wrongly said OAuth2 assertion flow is not meant for inter-trust domain exchanges (meant was "intra").
* Rephrased X509 change of scope example to be more clear.
* Sharpened ways of provisioning, renamed "provisioning" to "initial provisioning" and "re-provisioning" to "on-demand provisioning".

## draft-schwenkschuster-wimse-credential-exchange-00

* Initial individual draft & write up.

# Acknowledgments
{:numbered="false"}

Big shoutout to the WIMSE token exchange design team (Dean Saxe, Yaroslav Rosomakho, Andrii Deinega, Dmitry Izumskiy, Ken McCracken and George Fletcher) that have done amazing groundlaying work in this area.
