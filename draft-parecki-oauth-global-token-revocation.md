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
  latest: "https://drafts.aaronpk.com/global-token-revocation/draft-parecki-oauth-global-token-revocation.html"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com
    uri: https://aaronparecki.com

normative:
  RFC6749:
  RFC8414:
  RFC9493:
  IANA.oauth-parameters:

informative:
  RFC6750:
  RFC7009:
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

Global Token Revocation enables parties such as a security incident management tool or an external Identity Provider to send a request to an Authorization Server to indicate that it should revoke all of the user's existing tokens and require that the user re-authenticates before issuing new tokens.


--- middle

# Introduction

An OAuth Authorization Server issues tokens in response to a user authorizing a client. A party external to the OAuth Authorization Server may wish to instruct the Authorization Server to revoke all tokens belonging to a particular user, and prevent the server from issuing new tokens until the user re-authenticates.

For example, a security incident management tool may detect anomalous behaviour on a user's account, or if the user logged in through an enterprise Identity Provider, the Identity Provider may want to revoke all of a user's tokens in the event of a security incident or on the employee's termination.

This specification describes an API endpoint on an Authorization Server that can accept requests from external parties to revoke all tokens associated with a given user.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This specification uses the terms "Access Token", "Authorization Code",
"Authorization Endpoint", "Authorization Server" (AS), "Client", "Client Authentication",
"Client Identifier", "Client Secret", "End-User", "Grant Type", "Protected Resource",
"Redirection URI", "Refresh Token", "Resource Owner", "Resource Server" (RS)
and "Token Endpoint" defined by {{RFC6749}},
and the terms "OpenID Provider" (OP) and "ID Token" defined by {{OpenID}}.

This specification uses the term "Identity Provider" (IdP) to refer to
the Authorization Server or OpenID Provider that is used for End-User authentication.


TODO: Replace RFC6749 references with OAuth 2.1


## Roles

In a typical OAuth deployment, the OAuth client obtains tokens from the authorization server when a user logs in and authorizes the client. In many cases, the method by which a user logs in at the authorization server is through an external identity provider.

For example, a mobile chat application is an OAuth Client, and obtains tokens from its backend server which stores the chat messages. The mobile chat backend plays the OAuth roles of "Resource Server" and "Authorization Server".

In some cases, the user will log in to the Authorization Server using an external (e.g. enterprise) Identity Provider. In that case, when a user logs in to the chat application, the backend server may play the role of an OAuth client (or OpenID or SAML relying party) to the Identity Provider in a new authorization or authentication flow.



# Token Revocation

A revocation request is a POST request to the Global Token Revocation endpoint, which starts the process of revoking all tokens for the identified subject.

## Revocation Endpoint

The Global Token Revocation endpoint is a URL at the authorization server which accepts HTTP POST requests with parameters in the HTTP request message body using the `application/json` format. The Global Token Revocation endpoint URL MUST use the `https` scheme.

If the authorization server supports OAuth Server Metadata ({{RFC8414}}), the authorization server SHOULD include the URL of their Global Token Revocation endpoint in their authorization server metadata document using the `global_token_revocation_endpoint` parameter as defined in {{authorization-server-metadata}}.

The authorization server MAY alternatively register the endpoint with tools that will use it.


## Revocation Request

The request is a POST request with an `application/json` body containing a single property `subject`, the value of which is a Security Event Token Subject Identifier as defined in "Subject Identifiers for Security Event Tokens" {{RFC9493}}.

In practice, this means the value of `subject` is a JSON object with a property `format`, and at least one additional property depending on the value of `format`.

The request MUST also be authenticated, the particular authentication method and means by which the authentication is established is out of scope of this specification, but may include OAuth 2.0 Bearer Token {{RFC6750}} or a JWT {{RFC7523}}.

The following example requests that all tokens for a user identified by an email address be revoked:

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

If the user identifier at the authorization server is known by the system making the revocation request, the request can use the "Opaque Identifer" format to provide the user identifier:

    POST /global-token-revocation
    Host: example.com
    Content-Type: application/json
    Authorization: Bearer f5641763544a7b24b08e4f74045

    {
      "subject": {
        "format": "opaque",
        "id": "e193177dfdc52e3dd03f78c"
      }
    }

If it is expected that the authorization server knows about the user identifier at the IdP, the request can use the "Issuer and Subject Identifier" format:

    POST /global-token-revocation
    Host: example.com
    Content-Type: application/json
    Authorization: Bearer f5641763544a7b24b08e4f74045

    {
      "subject": {
        "format": "iss_sub",
        "iss": "https://issuer.example.com/",
        "sub": "af19c476f1dc4470fa3d0d9a25"
      }
    }


## Revocation Expectations {#revocation-expectations}

Upon receiving a revocation request, authorizing the request, and validating the identified user, the Authorization Server:

* MUST revoke all active refresh tokens
* SHOULD invalidate all access tokens, although it is recognized that it might not be technically feasible to invalidate access tokens (see {{access-tokens}} below)
* MUST NOT issue new access tokens or refresh tokens without re-authenticating the user


## Revocation Response

This specification indicates success and error conditions by using HTTP response codes, and does not define the response body format or content.

### Successful Response

To indicate that the request was successful and revocation of the requested set of tokens has begun, the server returns an HTTP 204 response.

### Error Response

The following HTTP response codes can be used to indicate various error conditions:

* **400 Bad Request**: The request was malformed, e.g. an unrecognized or unsupported type of subject identifier.
* **401 Unauthorized**: Authentication provided was invalid.
* **403 Forbidden**: Insufficient authorization, e.g. missing scopes.
* **404 User Not Found**: The user indicated by the subject identifier was not found.
* **422 Unable to Process Request**: Unable to log out the user.



# Revocation of Access Tokens {#access-tokens}

OAuth 2.0 allows deployment flexibility with respect to the style of
access tokens.  The access tokens may be self-contained (e.g. {{RFC9068}}) so that a
resource server needs no further interaction with an authorization
server issuing these tokens to perform an authorization decision of
the client requesting access to a protected resource.  A system
design may, however, instead use access tokens that are handles (also known as "reference tokens")
referring to authorization data stored at the authorization server.

While these are not the only options, they illustrate the
implications for revocation.  In the latter case of reference tokens, the authorization
server is able to revoke an access token by removing it from storage. In the former case, without storing tokens, it may be impossible to revoke tokens without taking additional measures.

For this reason, revocation of access tokens is optional in this specification, since it may pose too significant of a burden for implementers. It is not required to revoke access tokens to be able to return a success code to the caller.


# Authorization Server Metadata

The following authorization server metadata parameters {{RFC8414}} are introduced to signal the server's capability and policy with respect to Global Token Revocation.

"global_token_revocation_endpoint":
: The URL of the authorization server's global token revocation endpoint.

"global_token_revocation_endpoint_auth_methods_supported":
: OPTIONAL. JSON array containing a list of client authentication methods supported by this introspection endpoint.  The valid client authentication method values are those registered in the IANA "OAuth Token Endpoint Authentication Methods" registry {{IANA.oauth-parameters}} or those registered in the IANA "OAuth Access Token Types" registry {{IANA.oauth-parameters}}.  (These values are and will remain distinct, due to Section 7.2.)  If omitted, the set of supported authentication methods MUST be determined by other means.


# Security Considerations

## Enumeration of User Accounts

Typically, an API that accepts a user identifier and returns different statuses depending on whether the user exists would provide an attack vector allowing enumeration of user accounts. This specification does require a "User Not Found" response, so would normally fall under this category. However, requests to the endpoint defined by this specification are required to be authenticated, so this is not considered a public endpoint.

If the tool making the request is compromised, and the attacker can impersonate the requests from this tool (either by coercing the tool to make the request, or by extracting the credentials), then the attacker would be able to enumerate user accounts. However, since the request is not just testing the presence of a user account, but actually revoking the tokens associated with the user if successful, this would likely be easily visible in any audit logs as many users tokens would be revoked in a short period of time.

To mitigate some of the concerns of providing such a powerful API endpoint, the users that a particular client can request revocation for SHOULD be limited, and the authentication of the request SHOULD be used to scope the possible user revocation list to only users authorized to the client.

For example, a multi-tenant identity provider that uses different signing keys for users assciated with different tenants, can also use the same signing keys to authenticate revocation requests, such as creating a JWT to use as client authentication as described in {{RFC7523}}. This enables the authorization server receiving the request to only accept revocation requests for users that are associated with the particular tenant at the identity provider.



# IANA Considerations

## OAuth Authorization Server Metadata

IANA has (TBD) registered the following values in the IANA "OAuth Authorization Server Metadata" registry of {{IANA.oauth-parameters}} established by {{RFC8414}}.


**Metadata Name**: `global_token_revocation_endpoint`

**Metadata Description**: URL of the authorization server's global token revocation endpoint.

**Change Controller**: IESG

**Specification Document**: Section X of [[ this specification ]]


**Metadata Name**: `global_token_revocation_endpoint_auth_methods_supported`

**Metadata Description**: OPTIONAL. Indicates the list of client authentication methods supported by this endpoint.

**Change Controller**: IESG

**Specification Document**: Section X of [[ this specification ]]



--- back

# Relationship to Related Specifications

## RFC7009: Token Revocation

OAuth 2.0 Token Revocation {{RFC7009}} defines an endpoint for authorization servers that an OAuth client can use to notify the authorization server that a previously-obtained access or refresh token is no longer needed.

The request is made by the OAuth client. The input to the Token Revocation request is the token itself, as well as the client's own authentication credentials.

This differs from the Global Token Revocation endpoint which does not take a token as an input, but instead takes a user identifier as input. It is not called by OAuth clients, but is instead called by an external party such as a security monitoring tool or an identity provider that the user used to authenticate at the authorization server.

## OpenID Connect Front-Channel Logout

[OpenID Connect Front-Channel Logout](https://openid.net/specs/openid-connect-frontchannel-1_0.html) provides a mechanism for an OpenID Provider to log users out of Relying Parties by redirecting the user agent.

While the logout request is the same direction as this draft describes, this relies on the redirection of the user agent, so is only applicable when the user is actively interacting with the application in a web browser.

The Global Token Revocation request works regardless of whether the user is actively using the application, and is also applicable to non-web based applications.

## OpenID Connect Back-Channel Logout

[OpenID Connect Back-Channel Logout](https://openid.net/specs/openid-connect-backchannel-1_0.html) provides a mechanism for an OpenID Provider to log users out of a Relying Party by making a back-channel POST request containing the user identifier of the user to log out.

This is the most similar existing logout specification to Global Token Revocation. However, there are still a few key differences that make it insufficient for the use cases enabled by Global Token Revocation.

OpenID Connect Back-Channel Logout requires Relying Parties to clear state of any sessions for the user, but doesn't mention anything about access tokens. It also says that refresh tokens issued with the `offline_access` scope "SHOULD NOT be revoked". This is a concretely different outcome than is described by Global Token Revocation, which requires the revocation of all refresh tokens for the user regardless of whether the refresh token was issued with the `offline_access` scope.

Additionally, OpenID Connect Back-Channel Logout assumes that the Relying Party implements OpenID Connect, which creates implementation challenges to use it when the Relying Party actually integrates with the identity provider using other specifications such as SAML.

Global Token Revocation works regardless of the protocol that the user uses to authenticate, so works equally well with OpenID Connect and SAML integrations.

## Shared Signals Framework

The Shared Signals Framework at the OpenID Foundation provides two specifications that have functionality related to session and token revocation.

[Continuous Access Evaluation Profile (CAEP)](https://openid.net/specs/openid-caep-specification-1_0.html) defines several event types that can be sent between cooperating parties. In particular, the "Session Revoked" event can be sent from an identity provider to an authorization server when the user's session at the identity provider was revoked. The main difference between this and the Global Token Revocation request is that teh CAEP event is a signal that may or may not be acted upon by the receiver, whereas the Global Token Revocation request is a command that has a defined list of expected outcomes.

[Risk Incident Sharing and Coordination (RISC)](https://openid.net/specs/openid-risc-profile-specification-1_0.html) defines events that have somewhat stronger defined meanings compared to CAEP. In particular, the "Account Disabled" event has clear meaning and strongly implies that a receiver should also disable the specified account. However, RISC also has a mechanism for a user to opt out of sending events for their account, so it does not provide the same level of assurance as a Global Token Revocation request.

Lastly, it is more complex to set up a receiver for CAEP and RISC events compared to a receiver for the Global Token Revocation request, so if the receiver is only interested in supporting the revocation use cases, it is much simpler to support the single POST request described in this draft.


# Document History

(( To be removed from the final specification ))

-02

* Added security consideration around enumeration of user accounts
* Added an appendix describing the differences between this and related logout specifications

-01

* Clarified revocation expectations
* Better definition of endpoint
* Added section defining endpoint in Authorization Server Metadata

-00

* Initial Draft


# Acknowledgments
{:numbered="false"}

The authors would like to thank the following people for their contributions and reviews of this specification: George Fletcher, Karl McGuinness, Mike Jones.


