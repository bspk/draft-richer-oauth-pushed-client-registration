---
title: "OAuth Pushed Client Registration"
category: std

docname: draft-richer-oauth-pushed-client-registration-latest
submissiontype: IETF
consensus: true
v: 3
area: SEC
workgroup: WG Working Group
keyword:
 - oauth
 - client
 - registration
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
  - ins: J. Richer
    name: Justin Richer
    organization: Bespoke Engineering
    email: ietf@justin.richer.org
    uri: https://bspk.io/
  - ins: A. Parecki
    name: Aaron Parecki
    email: aaron@parecki.com
    organization: Okta
    uri: https://aaronparecki.com

normative:
  RFC7591:
  RFC7636:
  RFC8414:
  RFC8705:
  RFC8792:
  RFC9126:
  RFC9449:

informative:


--- abstract

This specification provides a way for an ephemeral or transactional OAuth 2.0 client to signal to the AS that the client does not need or expect to have an identified state outside the existence of any issued access and refresh tokens. The specification also enables a client to push its registration information to the AS during a pushed authorization request.

--- middle

# Introduction

The OAuth protocol family was originally designed around a model of two relatively stable web sites connecting to each other's APIs on behalf of a user. This design, coupled with the need to begin transactions through an in-browser redirect in the authorization code and implicit grant types, leads naturally to an identifier for the client that can be presented in the front channel and referenced in the back channel along with client authentication. The client identifier could uniquely identify the client at the AS and be used to set policy and track its requests over time.

Not all clients are sufficiently stable or monolithic to fit this model well. To compensate, OAuth 2.0 introduced the concept of "public clients", whereby many instances of a client (such as a native application or single-page browser application) would share a single identifier. OAuth Dynamic Client Registration {{RFC7591}} enabled client software that did not get a client identifier at configuration time to obtain an identifier at runtime through a separate request to the AS. This enables dynamic configuration, but comes at a cost of needing to securely manage issuing and tracking these additional identifiers.

Some forms of client software in open ecosystem patterns, such as the Model Context Protocol or many legacy software authentication systems like IMAP, are built around a different model whereby any compliant piece of client software should be able to connect to a compliant server, without a managed preregistration. For these clients, this specification defines a mechanism to indicate to the AS that a client does not need a longstanding identifier but instead desires authorization within a single delegation transaction.

By using OAuth Pushed Authorization Requests {{RFC9126}}, a Pushed Client Registration can enable open world protocols to use the OAuth family of protocols as their security layer.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document contains non-normative examples of partial and complete HTTP messages. Some examples use a single trailing backslash (\) to indicate line wrapping for long values, as per {{RFC8792}}. The \ character and leading spaces on wrapped lines are not part of the value.

# Protocol Flow

Use of this specification consists of the following steps:

- The client makes a pushed authorization request using a defined static client_id value
- The AS returns a request_uri which the AS can associate to the specific request
- The client makes an authorization request using the request_uri
- The AS processes the authorization request based on the parameters pushed during the pushed authorization request
- The AS creates and returns an authorization code to the client
- The client submits the authorization code to the AS, again using the static client_id value

## Pushed Authorization Request and Response

The client MUST start the transaction using a pushed authorization request per {{RFC9126}}. The request MUST include a `client_id` value of `urn:ietf:oauth:parameters:dynamic`. The request MUST include a `state` value and consist of a random number. The request MUST include a `redirect_uri`value. The request MUST include an OAuth PKCE request per {{RFC7636}}.

The request MAY include a `client_registration` field, which consists of a form-encoded serialization of the JSON client metadata document defined in {{RFC7591}}.

~~~
NOTE: '\' line wrapping per RFC 8792

POST /par HTTP/1.1
Host: as.example.com
DPoP: eyj…
DPoP-Key: ejy…

client_id=urn:ietf:oauth:parameters:dynamic\
  &response_type=code\
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb\
  &state=superrandom4789\
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM\
  &code_challenge_method=S256\
  &client_registration=%7B%22client_name%22%3A%22ClaudeDesktop%22%2C%0A\
      %20%22logo_uri%22%3D%22https%3A%2F%2Fanthropic.com%2Flogo.png%22%7D
~~~

Where the value of `client_registration_ is the form-encoded JSON value:

~~~
{
  "client_name":"ClaudeDesktop",
  "logo_uri"="https://anthropic.com/logo.png"
}
~~~

If the client intends to use a sender-contstrained OAuth access token, such as via DPoP {{RFC9449}} or mTLS {{RFC8705}, the client MUST send the bound authentication parameters appropriate to the method to the PAR endpoint, as in the above example.

The AS creates a pending authorization request and associates it with an internal identifier, which it returns as a URN to the client:

~~~
HTTP 200 OK

{
   "request_uri": "urn:example:g012h340-4j138fha",
   "expires_in": 90
 }
~~~

## Authorization Request and Response

The client takes the result of the pushed authorization request and uses it to create an authorization request to the AS. The client MUST NOT include any parameters other than the syntactically required client_id, which is set to the static value in {{client-identifier}}.

~~~
NOTE: '\' line wrapping per RFC 8792

GET /authorize?client_id=urn:ietf:oauth:parameters:dynamic\
  &request_uri=urn%3Aexample%3Ag012h340-4j138fha HTTP/1.1
Host: as.example.com
~~~

The AS parses the value of the `request_uri` parameter and associates the incoming request with the values saved during the call to the PAR endpoint.

When the AS has received appropriate authorization from the RO, the details of which are out of scope of this specification, the AS returns an authorization code to the client, along with its initial state value.

~~~
NOTE: '\' line wrapping per RFC 8792

HTTP 301 Found
Location: https://client.example.org/cb?\
  code=98765cd-oe\
  &state=superrandom4789
~~~

When the client receives this request, the client compares the value of `state` to the stored state value to ensure they are equal to the one the client is expecting.

## Toiken Request and Response

The client takes the authorization code and sends it to the AS in exchange for a token. In this example, the client is requesting a DPoP bound access token.

~~~
NOTE: '\' line wrapping per RFC 8792

POST /token HTTP/1.1
Host: as.example.com
Content-type: application/www-form-urlencoded
DPoP: eyj…
DPoP-Key: ejy…

client_id=urn:ietf:oauth:parameters:dynamic\
  &code=98765cd-oe\
  &grant_type=authorization_code\
  &code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
~~~

The AS processes the authorization code and issues an access token to the client.

~~~
NOTE: '\' line wrapping per RFC 8792

HTTP 200 OK
Content-Type: application/json

{
  "access_token": "654edfgh-jkoiu789.0plkjhg34k",
  "token_type": "DPoP"
}
~~~

The client can then use this token to call the RS.

## Token Expiration and Refresh

The AS MAY issue a refresh token to the client, which can be used to get a new access token with the exact access rights of the original request.

When the access token expires and the client has no valid refresh token, the client MUST create a new pushed authorization request to get a new token.

# Client Identifier {#client-identifier}

The client MUST use a client_id value of `urn:ietf:oauth:parameters:dynamic` in all requests.

When receiving a client_id value of `urn:ietf:oauth:parameters:dynamic`, the AS MUST process it according to this specification. An AS MUST NOT assign the value `urn:ietf:oauth:parameters:dynamic` to any single client registration.

# Discoverable Support

An AS supporting this extension that publishes its metadata using {{RFC8414}} MUST indicate its support by including the key:

"pushed_client_registration_supported":
: A boolean value indicating that this specification is supported and accepted by the AS.

# Security Considerations

Support for this specification does not imply that it is supported for all possible access requests. In fact, an AS would be expected to determine through policy and configuration whether the request being made is suitable for a pushed client.


# IANA Considerations

- Register the "dynamic" value in the IETF OAuth URN namespace
- register the "client_registration" authorization parameter
- extend the parameters table to indicate that a parameter can only be used inside a PAR request?
- register the AS metadata key

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank Darin McAdams, Den Delimarsky, David Soria Perra, Adrian Frei, Pieter Kasselman, Jerome Swannack, Basil Hosmer, and Fei Yuan for their discussion and input to the concepts behind this draft.
