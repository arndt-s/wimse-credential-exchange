---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
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

WIMSE defines Workload Identity and how it is represented in the form of a credential. In most situations it is provisioned to the workload or the workload is able to request it without further authentication.

Regardless of the delivery, the credential format is often given based on the platform, often in the form of JWT and not a choice of the workload. Many situations however, require the workload to use a credential in a different form, with specific, alternative identifiers or in a different trust domain. Often existing protocols exist on how this can be acchieved. For instance OAuth2 Token Exchange (TODO) for JSON Web Tokens or ACME (TODO) for X.509 certificates.

This document aims to outline the situations where a credential exchange is necessary and what guard rails should exist. To ease integration for workloads requiring a set of different identity represented in a whole set of credentials shapes it proposes a way to retrieve them in a standardized manner.

--- middle

# Introduction

Workload Identity credentials come in all shapes and forms. JSON Web Tokens are popular but also X.509 certificates are commonly used. Generally it can be assumed that the workload receives all of the following

* an identity as a unique identifier within the trust domain

* one or multiple credentials that allow it to represent itself (as that identity). Multiple credentials are often different types but representing the same identity.

* an indication of authority. (TODO not sure)

Identity, credential and authority enable the workload to interact within its environment, communicate to sibling workloads (same authority), access APIs inside that trust domain or provide an API itself.

## Reasons

### Change in format

A different format representing the same identity on the same logical authority. These can be due to many reasons but concrete examples could be

* initial credential was a X.509 certificate but infrastructure requires application-level authentication such as JWT.
* initial credential was a JWT bound to a key, requiring to be presented along with proof of possession but the peer does not support it and requires a Bearer credential.

A change in format is often in control of the workload. Some frameworks and protocols exist where the credential format is not choosen by the workload but the authority. The returning credential is opague to the workload, for instance OAuth Bearer Tokens (TODO).

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

### Change in authority or trust domain

A provisioned workload identity is often part of a trust domain that is coupled to infrastructure or deployment. Workloads often require to intercept with other workloads or access outside resources that are protected by other authorities or reside in different trust domains. This requires the client workload to retrieve an identity of the other trust domain. Examples here include:

* Federation (a workload identity federates to a identity in a different trust domain). In existing workload identity environment OAuth2 with Token Exchange (TODO) and Assertion framework (TODO) are popular.
* A workload requires a credential of "higher trust" to interact with other workloads. This "higher trust" is facilitated by another authority. For instance a workload requiring a WebPKI certificate to offer a service to the world wide web.

### Missing provisioning support

A workload platform may not support the provisioning of credentials required by the workload. Technically, any of these would likely fall under the reasons above but it's a very common reason and often falls into multiple categories. As an example:

* Workload platform provisions identity & credential in the form of a simple signed document that carries the attributes attested by the platform but gives not access in any way.

### Combinations

Reasons to exchange credentials are often not binary. A change in authority effectively is a change in identity and a change in format is also often a change in authority as often formats come with different authority structures and security promises.

## Provisioning, re-previsioning & exchange

Once a workload has identified the need of a different credential it has multiple option to retrieve it. Based on the reason the list of options differs. This speciciation differs between the following 3 main options:

{:vspace}
Provisiong
: The initial credential(s) the workload is provisioned with are modified. This is often a matter of configuration within the platform. For instance a service account token in Kubernetes is projected with a different audience. Or a new credential is provisioned alongside the existing credential.

Re-provisioning
: Workloads are able to request credentials without authentication. This can be in addition to an initial credential or the way the initial credential is delivered in the first place. This approach allows the workload to request credential in the exact format, scope and identity it requires.

Credential exchange
: Workload use a provisioned credential to authenticate a delivery of a different credential. The significant difference here is that this is an authenticated action, compared to the other above, which are unauthenticated.

If a workload is in need of a different format, identity or scope it is always reccomended to use the initial provisioning or re-provisioning instead of a credential exchange. These approaches have generally a higher assurance and are more secure than exchange an already provisioned credential.

The following table gives some guidance based on the reason a different credential is required:

| Reason | Preferred mechanism | Other options (in order) |
|-----|------|-----|
| Change in authority | Credential exchange | None |
| Change in identity | Provisioning | 1) Re-provisioning<br>2) credential exchange |
| Change in scope | Provisioning | 1) Re-provisioning<br>2) credential exchange |
| Change in format | Provisioning | 1) Re-provisioning<br>2) credential exchange |

## Exchange patterns

### Format-specific exchange

Existing trust & identity framework often consist of a protocol or framework to exchange credentials. Leveraging this makes use of existing adoption and specific guidelines.

The following bullets give an overview of the existing patterns and when to use them based on the reasons given above:

* OAuth Token Exchange (TODO):
  * Change in scope.
  * Change in identity.
  * NOT meant for a change in authority.
  * NOT meant for a change in format.

* OAuth Assertion Framework (TODO):
  * Change in authority.
  * As a result of the change in authority, a change in identity, scope & potentially format is unavoidable but not the primary use case.
  * NOT meant for inter-domain exchanges.

* X.509 Certificate Management Protocol {{RFC4210}}:
  * Is this valid here? If yes, write about it.

* Enrollment over Secure Transport {{RFC7030}}:
  * Profile of {{RFC4210}}?
  * Is this valid here? If yes, write about it.

* SPIFFE federation:
  * Not a credential exchange but rather allowing for identities of different authorities to trust each other and communicate securely.

### On-behalf-of exchange

Workload environments can be highly dynamic and connected with a high variety of resources protected by different identity frameworks and formats. A format-agnostic, exchange component that exchanges credentials on behalf of the workload may be desired to remain control of credential issuance. For instance to enforce policy, collect audit trails or ease management.

~~~aasvg
+-----------------+ 2)request     +------------------------+  4)request      +-------------+
|                 |   credential  |                        |    credential   |             |
|  Workload       +-------------->|  Credential Exchanger  +---------------->|  Authority  |
|                 |               |                        |                 |             |
+-------^---------+               +----------+-------------+                 +-------------+
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
