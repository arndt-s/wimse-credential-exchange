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

informative:

--- abstract

WIMSE defines Workload Identity and how it is represented in the form of a credential. In most situations a credential is provisioned to the workload it can use to represent itself. The format of the credential is often given by the platform, commonly a JSON Web Token or an X.509 certificate. Workloads may be faced with situations where the credential does not fit and a different credential is required to perform its duties.

This document outlines various situations where a new credential is necessary and the different ways it can be obtained. It also looks at existing mechanisms and compares them to defined situations.

--- middle

# Introduction

Workload Identity credentials come in all shapes and forms. JSON Web Tokens are popular but also X.509 certificates are commonly used. When a workload is provisioned it can be assumed that it gets all of the following

* an identity in the form of an identifier

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
* A X.509 credential allowing the workload to TODO based on X.509 Key Usage extension but the workload requires additonal usages.

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

### Missing provisioning support

A workload platform may not support the provisioning of credentials required by the workload. Technically, any of these would likely fall under the reasons above but it's a very common reason and often falls into multiple categories. As an example:

* Workload platform provisions identity & credential in the form of a simple signed document that carries the attributes attested by the platform but gives not access in any way.

### Combinations

Reasons and needs to exchange credentials are often not binary. A change in trust domain effectively is a change in identity too. A change in format can require a change in trust domain because formats come with different trust structures and security promises. E.g. a trust domain issuing JSON Web Tokens may not be able to issue WebPKI certificates.

## Provisioning, re-previsioning & exchange

Once a workload requires a different credential it has multiple options to retrieve it. Based on the need the list of options shrinks. We differientate between the following 3 main options:

{:vspace}
Provisioning
: The initial credential(s) the workload is provisioned with are modified. This is often a matter of configuration within the platform. For instance a service account token in Kubernetes is projected with a different audience. Or a new credential is provisioned alongside the existing credential.

Re-provisioning
: Workloads are able to request credentials without authentication. This can be in addition to an initial credential or the way the initial credential is delivered in the first place. This approach allows the workload to request credential in the exact format, scope and identity it requires.

Credential exchange
: Workload use a provisioned credential to authenticate a delivery of a different credential. The significant difference here is that this is an authenticated action, compared to the other above, which are unauthenticated.

If a workload is in need of a different format, identity or scope it is always reccomended to use the initial provisioning or re-provisioning instead of a credential exchange. These approaches have generally a higher assurance and are more secure than exchange an already provisioned credential.

The following table gives some guidance based on the need:

| Need | Preferred mechanism | Other options (in order) |
|-----|------|-----|
| Change in trust domain | Credential exchange | None |
| Change in identity | Provisioning | 1) Re-provisioning<br>2) credential exchange |
| Change in scope | Provisioning | 1) Re-provisioning<br>2) credential exchange |
| Change in format | Provisioning | 1) Re-provisioning<br>2) credential exchange |

## Exchange patterns

### Format-specific exchange

Existing trust & identity framework often consist of a protocol or framework to exchange credentials. Leveraging this makes use of existing adoption and specific guidelines.

The following bullets give an overview of the existing patterns and when to use them based on the needs given above:

* OAuth Token Exchange (TODO) is:
  * meant for a change in scope.
  * meant for a change in identity.
  * NOT meant for a change in trust domain.
  * NOT meant for a change in format.

* OAuth Assertion Framework (TODO) is:
  * meant for a change in trust domain. As a result of the change in trust domain, a change in identity, scope & potentially format is unavoidable but not the primary use case.
  * NOT meant for inter-domain exchanges.

* X.509 Certificate Management Protocol {{RFC4210}}:
  * Is this valid here? If yes, write about it.

* Enrollment over Secure Transport {{RFC7030}}:
  * Profile of {{RFC4210}}?
  * Is this valid here? If yes, write about it.

* SPIFFE federation:
  * Not a credential exchange but rather allowing for identities of different trust domains to trust each other and communicate securely.

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

The author believes that a specific protocol that fits all credential formats and trust frameworks is not feasable while remaining the existing security promises of the format or framework. It rather believes that a profile for each scenario is the best way forward and welcomes them to profile this specificiation for their concrete use cases. As a general guidance it is reccommended
* to narrowly scope the scenarios and don't build a one-fits-all exchange for a specific format.
* to decouple authentication and access control from the actual exchange as best as possible. E.g. a credential of one profile should be allowed as a mean of authentication to exchange to a credential of a different profile, regardless if the profiles are aware of each other or not.

The "Credential Exchanger" shown in the figure may be the Workload Platform itself that offers this capability. Potentially also in a "re-provisioning" way without authentication.

# Consideration

## Credential exchange decreases trust

A credential exchange is an authenticated way to retrieve credential(s). Thus, the issued credential cannot have higher trust than the credential that was used to authenticate the request. This is particularly relevant when a credential is required which format and frameworks is of a higher trust than the one that was used to authenticate the request. This includes exchanging credentials not requiring proof of key possession to credentials carrying it.

Generally, these situations are not reccommended and should be avoided. Workloads SHOULD be provisioned with the credential of the highest trust and only retrieve less-trusted credentials via credential exchange.

Alternatively, the authentication request should be enriched with additional identification that increases the level of authentication. E.g. authentication and additional proof of platform attestation.

## Credential exchange does not replace provisioning

Because credential exchange is authenticated it cannot replace provisioning. Without an initial credential a workload cannot facilitate credential exchange as there's no proof the workload is eligable for the requested credential.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
