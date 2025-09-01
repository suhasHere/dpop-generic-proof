---
title: "Multi-Protocol  Proof of Possession (DPoP) Framework"
abbrev: "dpop-proof"
category: std
docname: draft-nandakumar-oauth-dpop-proof-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "OAuth Working Group"
keyword:
 - DPoP
 - Proof of Possession
 - Authentication
 - Authorization
venue:
  group: "OAuth Working Group"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "oauth-wg/oauth-dpop-generic"
  latest: "https://oauth-wg.github.io/oauth-dpop-generic/draft-generic-dpop-framework.html"

author:
 -
    fullname: Suhas Nandakumar
    organization: Cisco Systems
    email:  snandaku@cisco.com

normative:
  RFC2119:
  RFC7519:
  RFC7515:
  RFC9449:
  MOQTransport: I-D.draft-ietf-moq-transport-13

informative:
  RFC6749:

--- abstract

This document describes a generic framework for Demonstrating Proof of
Possession (DPoP) that extends beyond HTTP-specific implementations. Building
upon RFC 9449, this framework provides a protocol-agnostic approach for
sender-constraining tokens through cryptographic proof of possession, enabling
secure token binding across various application protocols and contexts.

--- middle

# Introduction

RFC 9449 {{RFC9449}} defines a mechanism for sender-constraining OAuth 2.0
tokens via a proof-of-possession mechanism on the application level. This
document presents a generic framework that abstracts the core concepts of DPoP
into a protocol-agnostic approach, enabling proof-of-possession token binding
for various application contexts while maintaining the security properties
established in RFC 9449.

## Conventions and Terminology

{::boilerplate bcp14-tagged}

The terms "access token", "refresh token", "authorization server", "resource
server", "client", and "public client" are defined by "The OAuth 2.0
Authorization Framework" {{RFC6749}}.

The terms "JSON Web Token (JWT)", "JOSE Header", and "JWS" are defined in
{{RFC7519}} and {{RFC7515}}.

# Framework Overview

The Generic DPoP Framework separates the DPoP mechanism into two distinct and
separable parts:

1. Token Acquisition: How a client obtains a DPoP-bound token
2. Proof of Possession: How a client proves control of the private key
   associated with a DPoP token

The framework specified in this document provides a protocol-agnostic approach
for both parts, enabling secure proof-of-possession across various application
protocols and contexts.

# Concept

The main data structure introduced by this specification is a generic DPoP
proof JWT that can be used across various application protocols and contexts,
as described in detail below. A client uses a generic DPoP proof JWT to prove
the possession of a private key corresponding to a certain public key.

Roughly speaking, a generic DPoP proof is a signature over:

* protocol-specific authorization context data,
* a timestamp,
* a unique identifier,
* an optional server-provided nonce, and
* a hash of the associated access token when an access token is present
  within the request.

~~~
+--------+                                          +---------------+
|        |--(A)-- Token Request ------------------->|               |
| Client |        (Generic DPoP Proof)              | Authorization |
|        |                                          |     Server    |
|        |<-(B)-- DPoP-Bound Access Token ----------|               |
|        |        (token_type=DPoP)                 +---------------+
|        |
|        |
|        |                                          +---------------+
|        |--(C)-- DPoP-Bound Access Token --------->|               |
| Client |        (Generic DPoP Proof)              |   Protocol    |
|        |                                          |     Server    |
|        |<-(D)-- Protected Resource ---------------|               |
|        |                                          +---------------+
+--------+
~~~
{: #fig-basic-dpop-flow title="Basic Generic DPoP Flow"}

The basic steps of a protocol flow with generic DPoP (without the optional
nonce) are shown in {{fig-basic-dpop-flow}}.

A. In the token request, the client sends an authorization grant (e.g., an
authorization code, refresh token, etc.) to the authorization server in order
to obtain an access token (and potentially a refresh token). The client
attaches a generic DPoP proof to the request.

B. The authorization server binds (sender-constrains) the access token to the
public key claimed by the client in the DPoP proof; that is, the access token
cannot be used without proving possession of the respective private key. If a
refresh token is issued to a public client, it is also bound to the public key
of the DPoP proof.

C. To use the access token, the client has to prove possession of the private
key by, again, providing a generic DPoP proof for that request to the protocol
server. The protocol server needs to receive information about the public key
to which the access token is bound. This information may be encoded directly
into the access token (for JWT-structured access tokens) or provided via token
introspection endpoint (not shown). The protocol server verifies that the
public key to which the access token is bound matches the public key of the
DPoP proof. It also verifies that the access token hash in the DPoP proof
matches the access token presented in the request.

D. The protocol server refuses to serve the request if the signature check
fails or if the data in the DPoP proof is wrong, e.g., the authorization
context does not match the expected context for the protocol operation. The
access token itself, of course, must also be valid in all other respects.

The generic DPoP mechanism presented herein is not a client authentication
method. In fact, a primary use case of DPoP is for public clients (e.g.,
single-page applications and applications on a user's device) that do not use
client authentication. Nonetheless, generic DPoP is designed to be compatible
with private_key_jwt and all other client authentication methods.

Generic DPoP does not directly ensure message integrity, but it relies on the
underlying transport security layer (such as TLS) for that purpose. See
{{security-considerations}} for details.

# Generic DPoP Proof Structure

## Authorization Context Object

The core extension introduced by this framework is the Authorization Context
(`actx`) object, which replaces protocol-specific fields in the DPoP proof JWT
payload.

A generic DPoP proof JWT MUST contain an Authorization Context object
(`actx`) that specifies the protocol or context for which the proof is being
generated.

### Authorization Context Object Structure

The Authorization Context (`actx`) object MUST contain:

- type (string): A registered identifier specifying the protocol or context
  type
- Additional fields as defined by the specific context type specification

~~~json

{
  "actx": {
    "type": "protocol-identifier"
  }
}
~~~

## JWT Header Requirements

The JWT header for a generic DPoP proof MUST contain the same elements as
specified in {{RFC9449}}:

- typ: MUST be "dpop-proof+jwt"
- alg: An asymmetric signature algorithm identifier
- jwk: The public key used for verification

## JWT Payload Requirements

The JWT payload for a generic DPoP proof MUST contain:

- jti: A unique identifier for the JWT
- iat: Issued-at time
- actx: Authorization Context object (if not using HTTP binding)

Additional claims MAY be included based on context requirements:

- nonce: Server-provided nonce for replay protection
- ath: Access token hash (when presenting access tokens)

# Generic DPoP Proof JWT Examples

This section provides complete examples of generic DPoP proof JWTs,
illustrating the use of the Authorization Context object in place of
HTTP-specific claims.

## Example: MOQ Context DPoP Proof

The following example shows a complete DPoP proof JWT for a MOQ context:

JWT Header:

~~~json

{
  "typ": "dpop-proof+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "x": "l8tFrhx-34tV3hRICRDY9zCkDlpBhF42UQUfWVAWBFs",
    "y": "9VE4jf_Ok_o64zbTTlcuNJajHmt6v9TDVrU0CdvGRDA",
    "crv": "P-256"
  }
}
~~~

JWT Payload:

~~~json

{
  "jti": "unique-request-id-789",
  "iat": 1705123456,
  "actx": {
    "type": "moq",
    "action": "SUBSCRIBE",
    "tns": "moq://example.com/app/scope/video",
    "tn": "camera1"
  },
  "ath": "fUHyO2r2Z3DZ53EsNrWBb1xWXM4VbCqpW5G-o9GqC7Y"
}
~~~

## Comparing with HTTP DPoP

For comparison, an equivalent HTTP DPoP proof would contain `htm` and `htu`
claims instead of the `actx` object:

~~~json
{
  "jti": "unique-request-id-789",
  "iat": 1705123456,
  "htm": "POST",
  "htu": "https://media.example.com/subscribe",
  "ath": "fUHyO2r2Z3DZ53EsNrWBb1xWXM4VbCqpW5G-o9GqC7Y"
}
~~~

The key difference is that the generic framework uses the `actx` object to
provide protocol-specific authorization context, while HTTP DPoP uses `htm` and
`htu` for HTTP method and URI.

# Protocol Type Registry

This framework establishes a registry for protocol type identifiers used in
the `actx.type` field. Each registered type MUST specify:

1. The additional required and optional fields for the Authorization
   Context
2. The semantic meaning and validation rules for those fields
3. Security considerations specific to the protocol context
4. Examples of usage


## MOQ Context Type {#moq-context}

This section defines the normative authorization context for Media Over QUIC
(MOQ) as specified in this framework.

Type Identifier: "moq"

### Required Fields

The MOQ authorization context MUST contain the following fields:

action:

A string specifying the MOQ operation being authorized.

tns:

Track Namespace as defined for MOQ operations.

### Optional Fields

The MOQ authorization context MAY contain the following fields:

tn:

Track Name for MOQ operations.

parameters:

An object containing additional MOQ-specific parameters relevant to the
operation. The structure and contents of this field are context-dependent.

### Validation Requirements

Servers processing MOQ authorization contexts MUST:

1. Verify that the `action` field contains a recognized MOQ operation
2. Validate that the `tns` field represents a properly formatted track
   namespace URI
3. Ensure the requested action is permitted for the specified track
   namespace
4. If present, ensure the requested action is permitted for the specified
   track name
5. If present, validate any parameters according to usage context

### MOQ Examples

JWT Header:

~~~json
{
  "typ": "dpop-proof+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "x": "l8tFrhx-34tV3hRICRDY9zCkDlpBhF42UQUfWVAWBFs",
    "y": "9VE4jf_Ok_o64zbTTlcuNJajHmt6v9TDVrU0CdvGRDA",
    "crv": "P-256"
  }
}
~~~

JWT Payload:

~~~json
{
  "jti": "unique-request-id",
  "iat": 1705123456,
  "actx": {
    "type": "moq",
    "action": "ANNOUNCE",
    "tns": "moq://example.com/app/scope/audio"
  }
}
~~~


# Relationship to RFC 9449

This framework extends and generalizes the concepts defined in RFC 9449
{{RFC9449}} while maintaining full backward compatibility with HTTP-based DPoP
implementations.

## Extensions to Section 7 (Protected Resource Access)

RFC 9449 Section 7 defines protected resource access specifically for HTTP
contexts. This framework extends those concepts to support non-HTTP protocols
while preserving the core security model.

### Key Differences from RFC 9449 Section 7

1. **Authorization Context vs HTTP Parameters**: Instead of requiring `htm`
(HTTP method) and `htu` (HTTP URI) claims, this framework uses the
Authorization Context (`actx`) object to specify protocol-specific request
context.

2. **Protocol-Agnostic Token Binding**: While RFC 9449 Section 7.1 defines
the DPoP authentication scheme for HTTP Authorization headers, this framework
enables token binding for protocols that may not use HTTP-style headers.

3. **Flexible Proof Validation**: The validation rules in RFC 9449 Section
4.3 are adapted to work with the `actx` object rather than HTTP-specific
claims, allowing servers to validate proofs according to their protocol
requirements.

### Compatibility with RFC 9449

This framework is designed to coexist with RFC 9449 implementations:

- HTTP-based DPoP proofs continue to work unchanged using `htm` and `htu`
  claims
- Generic DPoP proofs use the `actx` object for non-HTTP contexts
- The same key pairs and token binding mechanisms apply to both approaches
- Authorization servers MAY support both HTTP and generic DPoP proof formats
  simultaneously

### Migration Path

Existing RFC 9449 implementations can adopt this framework incrementally:

1. **HTTP-Only Phase**: Continue using standard RFC 9449 DPoP for HTTP
   resources
2. **Hybrid Phase**: Support both HTTP DPoP (with `htm`/`htu`) and generic
   DPoP (with `actx`)
3. **Generic Phase**: Migrate to using Authorization Context objects for all
   protocols, including HTTP

The `typ` header value `dpop-proof+jwt` (instead of `dpop+jwt`) signals
support for the generic framework while maintaining the same cryptographic
properties and security model.

# Security Considerations

This framework inherits all security considerations from {{RFC9449}}.
Additional considerations specific to the generic framework include:

## Context Type Validation

Servers MUST validate that the `actx.type` field corresponds to a registered
and supported context type. Unknown or unsupported context types MUST be
rejected.

## Protocol-Specific Security Requirements

Each registered context type MUST specify its own security requirements and
threat model. The generic framework does not impose additional security
properties beyond those provided by the underlying JWT signature.

## Cross-Context Token Binding

DPoP tokens bound using this framework SHOULD be validated only within their
intended context type. Servers MUST NOT accept a DPoP proof with one context
type as valid for a different context type.

# IANA Considerations

## DPoP Authorization Context Types Registry

This document establishes the "DPoP Authorization Context Types" registry
within the "JSON Web Token (JWT)" registry group.

Registry Name: DPoP Authorization Context Types
Registration Policy: Specification Required
Expert Review Guidelines:

Specifications submitted for registration MUST include:

  1. A complete specification of required and optional fields
  2. Semantic definitions and validation rules for all fields
  3. Security considerations specific to the context type
  4. At least one complete example of usage
  5. Considerations for interoperability with existing DPoP implementations

Reference Format: The reference MUST be to a published RFC or an
Internet-Draft that has been adopted by a working group.

### Registration Template

To register a new DPoP Authorization Context Type, the following template
MUST be completed:

Type Identifier:

: The string identifier used in the `actx.type` field

Description:

: A brief description of the context type and its intended use

Required Fields:

: List of mandatory fields in the authorization context object

Optional Fields:

: List of optional fields in the authorization context object

Validation Rules:

: Specific validation requirements for the context type

Security Considerations:

: Context-specific security requirements and threat model


### Initial Registry Contents

| Type | Description | Required Fields | Optional Fields | Reference |
|------|-------------|----------------|-----------------|-----------|
| moq | Media Over QUIC authorization context | action, tns | tn, parameters |
RFCXXXX |

## JWT Claims Registration

This document registers the following JWT claim in the "JSON Web Token
Claims" registry:

Claim Name: actx

Claim Description: Authorization Context Object

Claim Value Type: Object

Change Controller: IETF

Specification Document: This document, Section 4.1

## Media Types Registration

### application/dpop-proof+jwt

This document registers the "application/dpop-proof+jwt" media type for
generic DPoP proof JWTs in the "Media Types" registry.

Type Name: application

Subtype Name: dpop-proof+jwt

Required Parameters: none

Optional Parameters: none

Encoding Considerations: Binary; base64url encoding of JWT

Security Considerations: See Section 8 of this document

Interoperability Considerations: Multi-Protocol DPoP proofs extend
HTTP-specific DPoP while maintaining backward compatibility

Published Specification: This document
Applications that use this media type: Applications implementing generic DPoP
authorization

Fragment Identifier Considerations: none

Additional Information: none

Person and email address to contact: OAuth Working Group
<oauth@ietf.org>

Intended Usage: COMMON

Restrictions on Usage: none

Author: OAuth Working Group

Change Controller: IETF

--- back

# Acknowledgments

TODO
