---
title: "Global Token Revocation"
category: std

docname: draft-parecki-oauth-global-token-revocation-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - oauth
 - token revocation
 - logout
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "aaronpk/global-token-revocation"
  latest: "https://aaronpk.github.io/global-token-revocation/draft-parecki-oauth-global-token-revocation.html"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
    uri: https://aaronparecki.com

normative:
  RFC6749:
  RFC8414:
  IANA.OAuth.Parameters:
  I-D.ietf-secevent-subject-identifiers:

informative:
  RFC6750:
  RFC7523:
  RFC9068:
  OpenID:
    title: OpenID Connect Core 1.0
    target: https://openid.net/specs/openid-connect-core-1_0.html
    date: November 8, 2014
    author:
      - ins: N. Sakimura
      - ins: J. Bradley
      - ins: M. Jones
      - ins: B. de Medeiros
      - ins: C. Mortimore


--- abstract

TODO Abstract


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This specification uses the terms "Access Token", "Authorization Code",
"Authorization Endpoint", "Authorization Server" (AS), "Client", "Client Authentication",
"Client Identifier", "Client Secret", "End-User", "Grant Type", "Protected Resource",
"Redirection URI", "Refresh Token", "Resource Owner", "Resource Server" (RS)
and "Token Endpoint" defined by {{RFC6749}},
and the terms "OpenID Provider" (OP) and "ID Token" defined by {{OpenID}}.

TODO: Replace RFC6749 references with OAuth 2.1


# Token Revocation

Upon receiving a token revocation request, implementations MUST support the revocation of refresh tokens and active user sessions, and SHOULD support the revocation of access tokens (see {{implementation-notes}}).

A revocation request is a POST request to the Global Token Revocation endpoint. This URL MUST conform to the rules given in {{RFC6749}}, Section 3.1.

The means to obtain the location of the revocation endpoint is out of the scope of this specification.  For example, the authorization server may publish documentation of the location of the endpoint, or may manually register it with tools that will use it.



## Revocation Request

The request is a POST request with an `application/json` body containing a single property `subject`,  the value of which is a Security Event Token Subject Identifier as defined in {{I-D.ietf-secevent-subject-identifiers}}.

In practice, this means the value of `subject` is a JSON object with a property `format`, and at least one additional property depending on the value of `format`.

The request MUST also be authenticated, the particular authentication method and means by which the authentication is established is out of scope of this specification, but may include an OAuth 2.0 Bearer Token ({{RFC6750}}) or a JWT ({{RFC7523}}).

The following example requests all tokens for a user identified by an email address to be revoked:

    POST /global-token-revocation
    Host: example.com
    Content-Type: application/json
    Authorization: Bearer f5641763544a7b24b08e4f74045

    {
      "subject": {
        "format": "email",
        "email": "user@example.com"
      }
    }

To request revocation of all tokens for a user identified by a user ID at the authorization server, use the "opaque subject" identifier:

    POST /global-token-revocation
    Host: example.com
    Content-Type: application/json
    Authorization: Bearer f5641763544a7b24b08e4f74045

    {
      "subject": {
        "format": "opaque",
        "id": "U1234567890"
      }
    }


## Revocation Response

This specification indicates success and error conditions by using HTTP response codes, and does not define the response body format or content.

To indicate that the request was successful and revocation of the requested set of tokens has begun, the server returns an HTTP 204 response.


### Error Response

The following HTTP response codes can be used to indicate various error conditions:

* **400 Bad Request**: The request was malformed, e.g. an unrecognized type of subject identifier.
* **401 Unauthorized**: Authentication provided was invalid.
* **403 Forbidden**: Insufficient authorization, e.g. missing scopes.
* **404 User Not Found**: The user indicated by the subject identifier was not found.
* **422 Unable to Process Request**: Unable to log out the user.



# Implementation Notes {#implementation-notes}

OAuth 2.0 allows deployment flexibility with respect to the style of
   access tokens.  The access tokens may be self-contained (e.g. {{RFC9068}}) so that a
   resource server needs no further interaction with an authorization
   server issuing these tokens to perform an authorization decision of
   the client requesting access to a protected resource.  A system
   design may, however, instead use access tokens that are handles
   referring to authorization data stored at the authorization server.

While these are not the only options, they illustrate the
   implications for revocation.  In the latter case of reference tokens, the authorization
   server is able to revoke an access token by removing it from storage. In the former case, without storing tokens, it may be impossible to revoke tokens without taking additional measures.

For this reason, revocation of access tokens is optional in this specification, since it may post too significant of a burden for implementers. It is not required to revoke access tokens to be able to return a success code to the caller.


# Security Considerations

TODO Security


# IANA Considerations

## OAuth Authorization Server Metadata

IANA has (TBD) registered the following values in the IANA "OAuth Authorization Server Metadata" registry of {{IANA.OAuth.Parameters}} established by {{RFC8414}}.


**Metadata Name**: `global_token_revocation_endpoint`

**Metadata Description**: URL of the authorization server's global token revocation endpoint.

**Change Controller**: IESG

**Specification Document**: Section X of [[ this specification ]]


**Metadata Name**: `global_token_revocation_endpoint_auth_methods_supported`

**Metadata Description**: OPTIONAL. JSON array containing a list of client authentication methods supported by this introspection endpoint.  The valid client authentication method values are those registered in the IANA "OAuth Token Endpoint Authentication Methods" registry {{IANA.OAuth.Parameters}} or those registered in the IANA "OAuth Access Token Types" registry {{IANA.OAuth.Parameters}}.  (These values are and will remain distinct, due to Section 7.2.)  If omitted, the set of supported authentication methods MUST be determined by other means.

**Change Controller**: IESG

**Specification Document**: Section X of [[ this specification ]]



--- back

# Document History

(( To be removed from the final specification ))

-00

* Initial Draft


# Acknowledgments
{:numbered="false"}

The authors would like to thank the following people for their contributions and reviews of this specification: George Fletcher, Karl McGuinness, Mike Jones.


