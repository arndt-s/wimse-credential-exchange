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
  RFC7521: Assertion flow
  RFC8693: Token exchange
informative:
  Macaroons:
    title: Cookies with Contextual Caveats for Decentralized Authorization in the Cloud
    target: https://theory.stanford.edu/~ataly/Papers/macaroons.pdf
    date: 2014
    author:
      - ins: A. Birgisson
      - ins: J. G. Politz
      - ins: U. Erlingsson
      - ins: M. Vrable
      - ins: M. Lentczner
  Biscuit:
    title: Biscuit, a bearer token with offline attenuation and decentralized verification
    target: https://doc.biscuitsec.org/reference/specifications

--- abstract

WIMSE defines Workload Identity and its representation through credentials. Typically, a credential is provisioned to the workload, allowing it to represent itself. The credential format is usually chosen by the platform. Common formats are JSON Web Tokens or X.509 certificates. However, workloads often encounter situations where a different identity or credential is required.

This document describes various situations where a workload requires another credential. It also outlines different ways this can be acchieved and compares them.

--- middle

# Introduction

Workloads operating across various platforms typically receive identity credentials in platform-specific formats such as JWT tokens, X509 certificates, Kubernetes Service Accounts Tokens, SPIFFE SVIDs, or cloud provider metadata documents. These credentials, including their format, issuer, subject, and other attributes are determined by the platform infrastructure rather than the workload itself.

When accessing external resources or other workloads, workloads must satisfy different authentication requirements specific to each resource they interact with. Resource access might require OAuth 2.0 Access Tokens, SPIFFE credentials, mutual TLS client certificates, or other authentication mechanisms that the workload cannot control or modify.

This credential mismatch creates a fundamental challenge: workloads must bridge the gap between their platform-issued identity and the various authentication requirements of the resources they need to access. Solutions typically involve credential exchange mechanisms or leveraging platform-specific functionality.

This specification:

* Examines the rationale behind credential exchange requirements in {{rationale}}
* Defines abstract mechanisms for credential delivery and exchange in {{mechanisms}}
* Documents concrete implementation patterns based on these mechanisms in {{patterns}}

## Static secrets

Credential exchange is a broad term and can be interpreted in many ways. This document focuses on the exchange of credentials that are issued to workloads on demand, such as JWTs, X.509 certificates, or other workload identity formats. It does not cover the exchange of static secrets, such as API keys or client secrets, which are typically provisioned out-of-band and do not involve a dynamic exchange process. This document rather sees static secrets as a special kind of resources that require strong authentication and authorization to access, but not as a credential exchange.

Credentials returned as by this specification are:

- short-lived
- issued on demand, but potentially cached
- are individual based on the credential used to authenticate

On contrast, static secrets are:

- medium to long-lived
- provisioned out-of-band, often manually
- do not differ based on authentication, different callers get access to the same credential

# Rationale {#rationale}

There are many reasons for a credential exchange. The following list highlights the most common reasons, and is not complete.

## Change in format

Workloads may require a different format representing the same identity in the same trust domain. Some concrete examples are:

* The initial credential was an X.509 certificate but resources require application-level authentication such as JWT or Workload Identity Tokens as defined in (TODO).
* The initial credential was a JWT bound to a key to be presented along with proof of possession, but the resource does not support it and requires a bearer credential.

"Credential format" is difficult to define abstractly. Some formats are opaque to the workload and should remain that way. For instance, how an OAuth 2.0 Bearer token is constructed, and whether it carries claims or not, is not a concern of the workload. That a bearer token is required, however, is known to the workload. So a change in format between a bearer token and an X.509 certificate is certainly a change in format the workload can require. A different encoding of a bearer token, on the other hand, is not and this specification does not address those cases.

## Change in scope

A credential in the same format may represent the same identity, but is scoped differently. Examples are:

* A JWT credential with an `audience` set to interact with the workload platform, but access to other workloads are required. The workload is in need of JWTs with different, dedicated audiences.
* An X.509 credential is constrained to a certain key usage, but the workload requires difference usage bits set. For instance, the existing certificate allows for `digitalSignature` but `keyEncipherment` or `dataEncipherment` is required.

Generally, scope should already be present and configured approperately with the workload platform only issuing narrowly scoped credentials to the workload. However, the platform may only support the provisioning of a single credential and doesn't allow custom scoping.

## Change in identity

A workload may be known under multiple identities. For example:

* A workload identity representing an exact physical instance of the workload may be eligible for a workload identity representing a logical unit that groups many physical instances together. Another example is a workload running in a specific region being eligible for a broader, geographically scoped identity.
* A workload that can act on behalf of other workloads. These workloads often are part of infrastructure such as API gateways, proxies, or service meshes in container environments.

## Change in trust domain

A provisioned workload identity is often part of a trust domain that is coupled to infrastructure or deployment. Workloads often interact with other workloads or access outside resources located in different trust domains. This may require the client workload to retrieve an identity of the other trust domain. Examples here include:

* Federation (a workload identity federates to a identity in a different trust domain). In existing workload identity environment OAuth2 with Token Exchange (TODO) and Assertion framework (TODO) are popular.
* A workload requires a credential of "higher trust" to interact with other workloads. This "higher trust" is facilitated by another trust domain. For instance, a workload may require a WebPKI certificate to offer a service to clients with "default" trust stores.

## Change in lifetime

Credentials often come with time restrictions, or usage may be restricted based on token lifetime. For instance:

* A resource denies the long-lived workload credential based on a maximum lifetime policy.
* An initial provisioned credentials has expired and renewal is unsupported.
* A credential with shorter lifetime would reduce replay risk.

## Missing provisioning support

A workload platform may not support the provisioning of credentials required by the workload. Technically, any of these would likely fall under the reasons above, but it's a very common reason and often falls into multiple categories. As an example:

* Workload platform provisions identity and credential in the form of a simple signed document that carries the attributes attested by the platform, but gives not access in any way.

## Combinations

Reasons for exchange credentials are often not binary. A change in trust domain is effectively a change in identity as well. A change in format can require a change in trust domain, because formats come with different trust structures and security promises. For example, a trust domain issuing JSON Web Tokens may not be able to issue WebPKI certificates.

# Mechanisms {#mechanisms}

Workloads have multiple options to aquire credentials in the way they are required. The following terms divides them into three primary mechanisms:

{:vspace}
Initial provisioning
: Credentials are issued during workload creation. The workload is "born" with them. These credentials are fixed and pre-defined, often by configuration. The workload cannot influence their shape during runtime. Configuration may be changed to adjust initial provisioning.

On-demand provisioning
: Workloads are able to obtain credentials on-demand. Parameters allow the workload to specify exactly the required format, scope, identity, lifetime, and other customization the workload requires. No authentication is necessary to request on-demand credentials. Workloads may choose to request additional on-demand credentials based on its needs. (TODO may emphasize that this is unauthenticated here)

Credential exchange
: Workloads use a provisioned credential (on-demand or initial) to authenticate and authorize a request of a different credential. Based on parameters, the workload can specify the exact attributes of the credential it requires. This is also on-demand, however, the significant difference here is that this is an **authenticated** action, compared to on-demand provisioning, which is **unauthenticated.** Workloads may leverage credential exchange to obtain credentials based on its needs.

Based on the exchange need, some mechanisms are more feasible and better suited than others. The following table gives some guidance based on the identified need. The security considerations below also highlight some additional considerations, particularly {{use-on-demand-provisioning}}.

| Need | Preferred mechanism | Other options (in order) |
|-----|------|-----|
| Change in trust domain | Credential exchange | None |
| Change in identity | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange |
| Change in scope | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange |
| Change in format | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange |
| Change in lifetime | On-demand provisioning | 1) Initial provisioning<br>2) Credential exchange (only decrease, see {{exchange-to-renew}}) |
| Missing platform support | Credential exchange | None |

# Exchange patterns {#patterns}

## Format-specific exchange

The existing trust and identity framework often consist of a protocol or framework to exchange credentials. Leveraging this makes use of existing adoption and specific guidelines.

The following bullets give an overview of the existing patterns and when to use them based on the needs given above:

* OAuth Token Exchange {{RFC8693}} is:
  * meant for a change in scope.
  * meant for a change in identity.
  * to a certain extend meant for a change in format (limited).
  * NOT meant for a change in trust domain.

* OAuth Assertion Framework {{RFC7521}} is:
  * meant for a change in trust domain. As a result of the change in trust domain, a change in identity, scope and, potentially, format is unavoidable but not the primary use case.
  * NOT meant for exchanges within a trust domain.

## On-behalf-of exchange

Workload environments can be highly dynamic and connected with a high variety of resources protected by different identity frameworks and formats. A format-agnostic component that exchanges credentials on behalf of the workload may be desired to remain control of credential issuance. For instance, it might enforce policy, collect audit trails, or aid management.

~~~aasvg
+-----------------+ 2)request     +------------------------+  4)request      +---------------------+
|                 |   credential  |                        |    credential   |                     |
|  Workload       +-------------->|  Credential Exchanger  +---------------->|  Credential issuer  |
|                 |               |                        |                 |                     |
+-------^---------+               +----------+-------------+                 +---------------------+
        |                                    |
  1) Provisioning                            |
        |                              3) validate
+-------v-----------------+                  |
|                         |                  |
|  Workload Platform      |<-----------------+
|                         |
+-------------------------+
~~~

1. The Workload Platform issues credential to the workload. This can be either "initial", during workload startup or "on-demand", once the workload requires it. See {{mechanisms}} for more details.
2. The Workload requests a new credential from the Credential Exchanger by specifying at least the issuer, format, and identity. Potentially, it also specifies lifetime and scope. It authenticates itself with the credential it has received from the Workload Platform.
3. The Credential Exchanger validates the credential it receives. For simplicity, the diagram shows this as a interaction with the Workload Platform, but other means of validations are also possible.
4. The Credential Exchanger requests a credential from the Credential Issuer. Also, for simplicity this step shows the interaction with a third party. However, this may also be the Workload Platform itself. Authentication and other step details depend on the scenario, format, and trust framework.

The author believes that a specific protocol that fits all credential formats and trust frameworks is infeasable while remaining(maintaining?) the existing security promises. He rather believes that a profile for each scenario is the best way forward and welcomes everyone to profile this specificiation for their concrete use cases. As a general guidance it is recommended to:

* narrowly scope the scenarios, instead of building a one-fits-all exchange for a specific format.
* decouple authentication and access control from the actual exchange as best as possible. For example, a credential of one profile should be allowed as a means of authentication to exchange to a credential of a different profile, whether or not the profiles are aware of each other.
* allow the workload to specify at least issuer, identity and format when requesting a credential. Lifetime and scope could be optionally specified, based on the need and support for it.
* keep multi-stepped issuance in mind. Some formats and trust frameworks may require the workload to perform challenges, like responding to a nonce or providing a signature.

The "Credential Exchanger" shown in the figure MAY be the Workload Platform itself that offers this capability. It MAY be offered during a "re-provisioning" without authentication.

# Consideration

## Credential exchange cannot increase trust

A credential exchange is an authenticated method to retrieve credential(s). Thus, the issued credential cannot be given a higher trust level than the credential that was used to authenticate the request. This is particularly relevant when a required credential, due to its format and framework, is of a higher trust than the one that was used to authenticate the request. This includes exchanging credentials without proof of key possession for credentials that do carry proof of possession.

These situations are not recommended. Workloads SHOULD be provisioned with the credential of the highest trust and only retrieve less-trusted credentials via credential exchange.

Alternatively, the authentication request should be enriched with additional identification that increases the level of authentication. For example, along with authentication, the workload would provide additional proof of platform attestation.

## Credential exchange cannot replace on-demand or initial provisioning

Because credential exchange is authenticated it cannot replace provisioning. Without an initial or on-demand requested credential a workload cannot facilitate credential exchange, as there is no proof the workload is eligible for the requested credential.

## Initial provisioning comes with over-provisioning risk {#use-on-demand-provisioning}

Provisioning credentials preemptively risks being exposed to overprovisioning credentials that are not required. For example, with initial provisioning, every workload is provisioned with a default credential, even though some don't require it. This unnecessarily increases the risk of those credentials being exposed.

On-demand provisioning, on the other hand, only issues credential when requested and mitigates this. They are exactly in the scope, format, identity and lifetime that is requires. This can significantly decrease the number of unnecessarily issued and provisioned credentials.

## Expanding credential lifetime {#exchange-to-renew}

A change in lifetime of a credential can be critical if it can be used to effectively keep a credential alive. One example is an issued short-lived bearer credential that can be used to exchange for a new, longer-lived credentials. Thus, it is highly recommended to only use on-demand provisioning to re-request a new credential.

On the other hand, it is valid to leverage token exchange to request a shorter-lived credential whose lifetime is within the bound of the credential used for authenticating the request.

## Involvement of human, transactional or other contextual credentials

Although this document focuses heavily on workload identity, workloads often deal with other credentials carrying caller, transactional, or contextual information. This could include an access token of the caller used to authorize the request. or an OAuth Transaction Token that was part of the request coming from another workload carrying transactional data.

These credentials and their formats, lifetime, scope, etc. are not covered by this document. However, they may be used as parameters or authentication to request additional credentials that combine multiple identities into a single credential.

Some concrete examples are:

* An access token and a workload identity credentials are used to request an OAuth Transaction Token.
* An on-behalf-of scenario where a workload identity is used as actor, and a different, contextual credential unrepresentative of the workload is used as a subject in an OAuth Token Exchange.

On-demand provisioning or credential exchange MAY be used to issue any of those contextual credentials to the workload. Existing contextual credentials MAY be supplied as parameters. Initial provisioning is not suitable with existing contextual credentials as it does not support parameters. In situations where the workload's identity does not play a role and only the contextual credentials are used as authentication, credential exchange is the preferred mechanism.

## Credential formats supporting offline attenuation {#offline-attenuation}

Some credential formats allow the scope of the credential to be reduced offline, without interaction to an issuing party ("offline attenuation"). In these situations no exchange or on-demand provisioning is required and workloads can "act on their own." Examples of these formats are {{Macaroons}} or {{Biscuit}} tokens. The provisioning of a credential that supports offline attenuation is still required in the first place.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Document History
<cref>RFC Editor: please remove before publication.</cref>

## draft-schwenkschuster-wimse-credential-exchange-01

* Fix typo that wrongly said OAuth2 assertion flow is not meant for inter-trust domain exchanges (meant was "intra").
* Rephrased X509 change of scope example to be more clear.
* Sharpened ways of provisioning, renamed "provisioning" to "initial provisioning" and "re-provisioning" to "on-demand provisioning".
* Add "Change in lifetime" need.
* Add considerations for the involvement of contextual, transactional and human credentials
* Add consideration for credential formats supporting offline-attenuation.
* Describe "Credential Exchanger" pattern.
* Clean up for IETF 122.

## draft-schwenkschuster-wimse-credential-exchange-00

* Initial individual draft & write up.

# Acknowledgments
{:numbered="false"}

Big shoutout to the WIMSE token exchange design team (Dean Saxe, Yaroslav Rosomakho, Andrii Deinega, Dmitry Izumskiy, Ken McCracken and George Fletcher) that have done amazing groundlaying work in this area.
