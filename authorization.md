# SMART Backend Services: Authorization Guide

  
## Profile Audience and Scope

This profile is intended to be used by developers of backend services (clients) that
autonomously (or semi-autonomously) need to access resources from FHIR servers 
that have pre-authorized defined scopes of access.  Specifically, this profile 
describes the registration-time metadata required for a client to be pre-authorized,
and the runtime process by which the client acquires an 
access token that can be used to retrieve FHIR resources.  This 
specification is not restricted to use for retrieving bulk data; it may be used 
to connect to any FHIR API endpoint, including both synchronous and asynchronous 
access.

#### **Use this profile** when the following conditions apply:

* The target FHIR server can register the client and pre-authorize access to a 
defined set of FHIR resources.
* The client may run autonomously, or with user interaction that does not
include access authorization.
* The client is able to protect a private key.
* No compelling need exists for a user to authorize the access at runtime.

### Examples

* An analytics platform or data warehouse that periodically performs a bulk data
import from an electronic health record system for analysis of
a population of patients.

* A lab monitoring service that determines which patients are currently
admitted to the hospital, reviews incoming laboratory results, and generates
clinical alerts when specific trigger conditions are met.  Note that in this 
example, the monitoring service may be a backend client to multiple servers.

* A data integration service that periodically queries the EHR for newly
registered patients and synchronizes these with an external database

* A utilization tracking system that queries an EHR every minute for
bed and room usage and displays statistics on a wall monitor.

## Underlying Standards

* [HL7 FHIR RESTful API](http://www.hl7.org/fhir/http.html)
* [RFC5246, The Transport Layer Security Protocol, V1.2](https://tools.ietf.org/html/rfc5246)
* [RFC6749, The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749)
* [RFC7515, JSON Web Signature](https://tools.ietf.org/html/rfc7515)
* [RFC7517, JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517.txt)
* [RFC7519, JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
* [RFC7521, Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants](https://tools.ietf.org/html/rfc7521)
* [RFC7518, JSON Web Algorithms](https://tools.ietf.org/html/rfc7518)
* [RFC7523, JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants](https://tools.ietf.org/html/rfc7523)
* [RFC7591, OAuth 2.0 Dynamic Client Registration Protocol](https://tools.ietf.org/html/rfc7591)

## Conformance Language
This specification uses the conformance verbs SHALL, SHOULD, and MAY as defined 
in [RFC2119](https://www.ietf.org/rfc/rfc2119.txt). Unlike RFC 2119, however, 
this specification allows that different applications may not be able to 
interoperate because of how they use optional features. In particular:

1.  SHALL: an absolute requirement for all implementations
2.  SHALL NOT: an absolute prohibition against inclusion for all implementations
3.  SHOULD/SHOULD NOT: A best practice or recommendation to be considered by 
implementers within the context of their particular implementation; there may 
be valid reasons to ignore an item, but the full implications must be understood 
and carefully weighed before choosing a different course
4.  MAY: This is truly optional language for an implementation; can be included or omitted as the implementer decides with no implications

## Registering a SMART Backend Service (communicating public keys)

Before a SMART client can run against a FHIR server, the client SHALL
obtain a client-to-client digital certificate and SHALL
register with that FHIR server's authorization service.  SMART does not specify a
standards-based registration process, but we encourage FHIR service implementers to
consider using the [OAuth 2.0 Dynamic Client Registration
Protocol](https://tools.ietf.org/html/draft-ietf-oauth-dyn-reg).

No matter how a client registers with a FHIR authorization service, the 
client SHALL register its **client_id** and the **public key** the
client will use to authenticate itself to the SMART FHIR authorization server.  The public key SHALL
be conveyed to the FHIR authorization server in a JSON Web Key (JWK) structure presented within
a JWK Set, as defined in 
[JSON Web Key Set (JWKS)](https://tools.ietf.org/html/rfc7517).  The client SHALL 
protect the associated private key from unauthorized disclosure
and corruption. 

For consistency in implementation, the client's JWK SHALL be shared with the
FHIR server using one of the following techniques:

  1. URL to JWK Set (strongly preferred). This URL communicates the TLS-protected 
  endpoint where the client's public JWK Set can be found. When provided, this URL 
  SHALL match the `jku` header parameter in the client's Authorization JWT. Advantages 
  of this approach are that
  it allows a client to rotate its own keys by updating the hosted content at the 
  JWK Set URL, assures that the public key used by the FHIR server is current, and avoids the 
  need for the FHIR server to maintain and protect the JWK Set.
  
  2. JWK Set directly (strongly discouraged). If a client cannot host the JWK 
  Set at a TLS-protected URL, it MAY supply the JWK Set directly to the FHIR server at 
  registration time.  In this case, the FHIR server SHALL protect the JWK Set from corruption,
  and SHOULD remind the client to send an update whenever the key set changes.  Conveying
  the JWK Set directly carries the limitation that it does not enable the client to 
  rotate its keys in-band.  Incuding both the current and successor keys within the JWK Set
  helps counter this limitation.  However, this approach places increased responsibility 
  on the FHIR server for protecting the integrity of the key(s) over time, and denies the FHIR server the 
  opportunity to validate the currency and integrity of the key at the time it is used.  
  
The client SHALL be capable of generating a JSON Web Signature in accordance
with [RFC7515](https://tools.ietf.org/html/rfc7515), using JSON Web Algorithm (JWA) 
header parameter values `RS384` and `ES384` as defined in [RFC7518](https://tools.ietf.org/html/rfc7518).  
The FHIR authorization server SHALL be capable of validating such signatures. Over time, 
we expect recommended algorithms to evolve, so while this specification recommends algorithms 
for interoperability, it does not mandate the use of any specific algorithm.

No matter how a JWK Set is communicated to the FHIR server, each JWK SHALL represent an 
asymmetric key by including `kty` and `kid` properties, with content conveyed using 
"bare key" properties (i.e., direct base64 encoding of key material as integer values). 
This means that:

* For RSA public keys, each JWK SHALL include `n` and `e` values (modulus and exponent) 
* For ECDSA public keys, each JWK SHALL include `crv`, `x`, and `y` values (curve, 
x-coordinate, and y-coordinate, for EC keys) 

Upon registration, the client SHALL be assigned a `client_id`, which  the client SHALL use when
requesting an access token.

## Obtaining an Access Token

By the time a client has been registered with the FHIR server, the key
elements of organizational trust will have been established. That is, the 
client will be considered "pre-authorized" to access FHIR resources. 
Then, at runtime, the client will need to obtain an access token in 
order to retrieve FHIR resources as pre-authorized. Such access tokens are 
issued by the FHIR authorization server, in accordance with the [OAuth 2.0
Authorization Framework, RFC6749](https://tools.ietf.org/html/rfc6749).  

Because the authorization scope is limited to protected resources previously 
arranged with the authorization server, the client credentials grant flow,
as defined in [Section 4.4 of RFC6749](https://tools.ietf.org/html/rfc6749#page-40), 
may be used to request authorization.  Use of the client credentials grant type
requires that the client SHALL be a "confidential" client capable of 
protecting its authentication credential.  

This specification describes requirements for requesting an access token
through the use of an OAuth 2.0 client credentials flow, with a [JWT
assertion](https://tools.ietf.org/html/rfc7523) as the 
client's authentication mechanism. The exchange, as depicted below, allows the
client to authenticate itself to the FHIR server and to request a short-lived
access token in a single exchange.

To begin the exchange, the client SHALL use the [Transport Layer Security
(TLS) Protocol Version 1.2 (RFC5246)](https://tools.ietf.org/html/rfc5246) to 
authenticate the identity of the FHIR authorization server and to establish an encrypted, 
integrity-protected link for securing all exchanges between the client 
and the authorization server's token endpoint.  All exchanges described herein between the client
and the FHIR server SHALL be secured using TLS V1.2.

<img class="sequence-diagram-raw"  src="http://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgQmFja2VuZCBTZXJ2aWNlIEF1dGhvcml6YXRpb24KCm5vdGUgb3ZlciBBcHA6ICBDcmVhdGUgYW5kIHNpZ24gYXV0aGVudGljACsFIEpXVCBcbntcbiAgImlzcyI6ICJhcHBfY2xpZW50X2lkIiwAFgVzdWIAAxhleHAiOiAxNDIyNTY4ODYwLCAASAVhdWQiOiAiaHR0cHM6Ly97dG9rZW4gdXJsfQBNBiAianRpIjogInJhbmRvbS1ub24tcmV1c2FibGUtand0LWlkLTEyMyJcbn0gLS0-AIE3BndpdGggYXBwJ3MgcHJpdmF0ZSBrZXkgKFJTMzg0KQCBbBBzY29wZT1zeXN0ZW0vKi5yZWFkJlxuZ3JhbnRfdHlwZT0AgV8HY3JlZGVudGlhbHMmXG4AgXQHYXNzZXJ0aW9uACUGdXJuOmlldGY6cGFyYW1zOm9hdXRoOgCCIQYtACMJLXR5cGU6and0LWJlYXJlcgA8Ez17c2lnbmVkAIJ1FGZyb20gYWJvdmV9CgpBcHAtPkVIUgCDXAUAg2kFZXI6ICBQT1NUIACCQxNcbihTYW1lIFVSTCBhcwCCegYARgYpAIQJDABAEUlzc3VlIG5ldyAAgx0FOgCECAUiYWNjZXNzXwCDMAUiOiAic2VjcmV0LQCDQAUteHl6IixcbiJleHBpcmVzX2luIjogMzAwLFxuLi4uXG59CgCBKA8tPgCFBwVbAFAGAGMGIHJlc3BvbnNlXQ&s=default"/>

#### Protocol details

Before a client can request an access token, it SHALL generate a
one-time-use JSON Web Token (JWT) that will be used to authenticate the client to
the FHIR authorization server. The authentication JWT SHALL include the
following claims, and SHALL be signed with the client's private
key (which SHOULD be an RS384 or EC384 signature). For a practical reference on JWT, as well as debugging
tools and client libraries, see https://jwt.io.

<table class="table">
  <thead>
    <th colspan="3">Authentication JWT Header Values</th>
  </thead>
  <tbody>
    <tr>
      <td><code>alg</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The JWA algorithm (e.g., `RS384`, `EC384`) used for signing the authentication JWT.
      </td>
    </tr>
    <tr>
      <td><code>kid</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The identifier of the key-pair used to sign this JWT. This identifier SHALL
          be unique within the client's JWK Set.</td>
    </tr>
    <tr>
      <td><code>typ</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Fixed value: <code>JWT</code>.</td>
    </tr>
    <tr>
      <td><code>jku</code></td>
      <td><span class="label label-info">optional</span></td>
      <td>The URL to the JWK Set containing the public key(s). When present,
      this should match a value that the client supplied to the FHIR server at
      client registration time.  (When absent, the FHIR server SHOULD fall back on the JWK
      Set URL or the JWK Set supplied at registration time.</td>
    </tr>
  </tbody>
</table>


<table class="table">
  <thead>
    <th colspan="3">Authentication JWT Claims</th>
  </thead>
  <tbody>
    <tr>
      <td><code>iss</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Issuer of the JWT -- the client's <code>client_id</code>, as determined during registration with the FHIR  authorization server
        (note that this is the same as the value for the <code>sub<code> claim)</td>
    </tr>
    <tr>
      <td><code>sub</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The service's <code>client_id</code>, as determined during registration with the FHIR authorization server
      (note that this is the same as the value for the <code>iss<code> claim)</td>
    </tr>
    <tr>
      <td><code>aud</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The FHIR authorization server's "token URL" (the same URL to which this authentication JWT will be posted -- see below)</td>
    </tr>
    <tr>
      <td><code>exp</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Expiration time integer for this authentication JWT, expressed in seconds since the "Epoch" (1970-01-01T00:00:00Z UTC). This time SHALL be no more than five minutes in the future.</td>
    </tr>
    <tr>
      <td><code>jti</code></td>
      <td><span class="label label-success">required</span></td>
      <td>A nonce string value that uniquely identifies this authentication JWT.</td>
    </tr>
  </tbody>
</table>

After generating an authentication JWT, the client requests a new access token
via HTTP `POST` to the FHIR authorization server's token endpoint URL, using
content-type `application/x-www-form-urlencoded` with the following parameters:

<table class="table">
  <thead>
    <th colspan="3">Parameters</th>
  </thead>
  <tbody>
    <tr>
      <td><code>scope</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The scope of access requested. See note about scopes below</td>
    </tr>
    <tr>
      <td><code>grant_type</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Fixed value: <code>client_credentials</code></td>
    </tr>
    <tr>
      <td><code>client_assertion_type</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Fixed value: <code>urn:ietf:params:oauth:client-assertion-type:jwt-bearer</code></td>
    </tr>
    <tr>
      <td><code>client_assertion</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Signed authentication JWT value (see above)</td>
    </tr>
  </tbody>
</table>

## Scopes

As the client authorization addressed by this specification involves no user or launch context, 
the existing SMART on FHIR scopes are not appropriate. Instead, clients SHALL use 
"system" scopes that parallel SMART "user" scopes.  System scopes have the format 
`system/(:resourceType|*).(read|write|*)`-- which conveys
the same access scope as the matching user format `user/(:resourceType|*).(read|write|*)`.
However, system scopes are associated with permissions assigned to an authorized 
software client rather than to a human end-user.

## Authorization Server Obligations 

### Signature Verification

The EHR's authorization server SHALL validate the JWT according to the 
processing requirements defined in [Section 3 of RFC7523](https://tools.ietf.org/html/rfc7523#section-3).

In addition, the authentication server SHALL:
* validate the signature on the JWT
* check that the `jti` value has not been previously encountered for the given `iss` within the maximum allowed authentication JWT lifetime (e.g., 5 minutes). This check prevents replay attacks.
* ensure that the `client_id` provided is known and matches the JWT's `iss` claim

To resolve a key to verify signatures, a server SHALL follow this algorithm:

<ol>
  <li>If the <code>jku</code> header is present, verify that the <code>jku</code> is whitelisted (i.e., that it 
    matches the value supplied at registration time for the specified <code>client_id</code>).
    <ol type="a">
      <li>If the <code>jku</code> header is not whitelisted, the signature verification fails.</li>
      <li>If the <code>jku</code> header is whitelisted, create a set of potential keys by dereferencing the <code>jku</code> URL. Proceed to step 3.</li>
    </ol>
  </li>
  <li> If <code>jku</code> is absent, create a set of potential key sources consisting of: all keys found by dereferencing the registration-time JWK Set URL (if any) + any keys supplied in the registration-time JWK Set (if any). Proceed to step 3.</li>
  <li> Filter the potential keys to retain only those where the <code>kid</code> matches the value supplied in the client's JWK header, and the <code>kty</code> is consistent with the signature algorithm used for the JWT (e.g., <code>RSA</code> for a JWT using an RSA-based signature, or <code>EC</code> for a JWT using an EC-based signature).</li>
  <li> Attempt to verify the JWK using each key in the potential keys list.
    <ol type="a">
      <li> If any attempt succeeds, the signature verification succeeds.</li>
      <li> If all attempts fail, the signature verification fails.</li>
    </ol>
  </li>
</ol>
 
If an error is encountered during the authentication process, the server SHALL 
respond with an `invalid_client` error as defined by 
the [OAuth 2.0 specification](https://tools.ietf.org/html/rfc6749#section-5.2). 

### Issuing Access Tokens

Once the client has been authenticated, the authorization server SHALL
mediate the request to assure that the scope requested is within the scope pre-authorized
to the client.

If the access token request is valid and authorized, the authorization server
SHALL issue an access token in response.  The access token response SHALL be a JSON object with 
the following properties: 

<table class="table">
  <thead>
    <th colspan="3">Access token response: property names</th>
  </thead>
  <tbody>
    <tr>
      <td><code>access_token</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The access token issued by the authorization server.</td>
    </tr>
    <tr>
      <td><code>token_type</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Fixed value: <code>bearer</code>.</td>
    </tr>
    <tr>
      <td><code>expires_in</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The lifetime in seconds of the access token. The recommended value is <code>300</code>, for a five-minute token lifetime.</td>
    </tr>
    <tr>
      <td><code>scope</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Scope of access authorized. Note that this can be different from the scopes requested by the app.</td>
    </tr>
  </tbody>
</table>

To minimize risks associated with token redirection, the scope of each access token SHOULD encompass, and be limited to, the resources requested. Access tokens issued under this profile SHALL be short-lived; the `expires_in` 
value SHOULD NOT exceed `300`, which represents an expiration-time of five minutes. 

The authorization server’s response MUST include the HTTP “Cache-Control” response header field with a value of “no-store,” as well as the “Pragma” response header field with a value of “no-cache.”    

If an error is encountered during the authorization process, the server SHALL
respond with the appropriate error message defined in [Section 5.2 of the OAuth 2.0 specification](https://tools.ietf.org/html/rfc6749#page-45).  The server SHOULD include an 
`error_uri` or `error_description` as defined in OAuth 2.0.  

Rules regarding circumstances under which a client is required to obtain and present an access token along with a request are based on risk-management decisions that each FHIR resource service needs to make, considering the workflows involved, perceived risks, and the organization’s risk-management policies.  Each token issued under this profile MUST be short-lived, with an expiration time of no more than five minutes.  Refresh tokens SHOULD NOT be issued. 

#### Access Token Length
The length of access tokens will change across servers, and each server may change the content and encoding of access tokens over time. Use a variable length data type without a specific maximum size to store access tokens.

This specification makes no specific recommendations about the structure of access tokens, however servers may choose to use JWT as a method to declare and sign access tokens. 

## Worked example

Assume that a "bilirubin result monitoring service" client has registered with
the FHIR authorization server, establishing the following

 * JWT "issuer" URL: `bili_monitor`
 * OAuth2 `client_id`: `bili_monitor`
 * JWK identfier: `kid` value (see [example JWK](https://github.com/smart-on-fhir/fhir-bulk-data-docs/blob/master/sample-jwks/RS384.public.json))

The client protects its private key from unauthorized access, use, and modification.  

At runtime, when the bilirubin monitoring service wants to
start monitoring some bilirubin values, it needs to obtain an OAuth 2.0 access 
token with the scopes `system/*.read` and `system/CommunicationRequest.write`. To accomplish
this (see [example](https://github.com/smart-on-fhir/fhir-bulk-data-docs/blob/master/sample-jwks/authorization-example-jwks-and-signatures.ipynb)), the client must first generate a one-time-use authentication JWT with the following claims:

##### 1. Generate a JWT to use for client authentication:

```
{
  "iss": "bili_monitor",
  "sub": "bili_monitor",
  "aud": "https://authorize.smarthealthit.org/token",
  "exp": 1422568860,
  "jti": "random-non-reusable-jwt-id-123"
}
```


##### 2. Digitally sign the claims, as specified in RFC7515.  

Using the client's RSA private key, with SHA-384 hashing (as specified for 
an `RS384` algorithm (`alg`) parameter value in RFC7518), the signed token 
value is:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzM4NCIsImtpZCI6ImVlZTlmMTdhM2I1OThmZDg2NDE3YTk4MGI1OTFmYmU2In0.eyJpc3MiOiJiaWxpX21vbml0b3IiLCJzdWIiOiJiaWxpX21vbml0b3IiLCJhdWQiOiJodHRwczovL2F1dGhvcml6ZS5zbWFydGhlYWx0aGl0Lm9yZy90b2tlbiIsImV4cCI6MTQyMjU2ODg2MCwianRpIjoicmFuZG9tLW5vbi1yZXVzYWJsZS1qd3QtaWQtMTIzIn0.l2E3-ThahEzJ_gaAK8sosc9uk1uhsISmJfwQOtooEcgUiqkdMFdAUE7sr8uJN0fTmTP9TUxssFEAQnCOF8QjkMXngEruIL190YVlwukGgv1wazsi_ptI9euWAf2AjOXaPFm6t629vzdznzVu08EWglG70l41697AXnFK8GUWSBf_8WHrcmFwLD_EpO_BWMoEIGDOOLGjYzOB_eN6abpUo4GCB9gX2-U8IGXAU8UG-axLb35qY7Mczwq9oxM9Z0_IcC8R8TJJQFQXzazo9YZmqts6qQ4pRlsfKpy9IzyLzyR9KZyKLZalBytwkr2lW7QU3tC-xPrf43jQFVKr07f9dA

```

Note: to inspect this example JWT, you can visit https://jwt.io. Paste the signed
JWT value above into the "Encoded"  field, and paste the [sample public signing key](sample-jwks/RS384.public.json) (starting with the `{"kty": "RSA"` JSON object, and excluding the `{ "keys": [` JWK Set wrapping array) into the "Public Key" box.
The plaintext JWT will be displayed in the "Decoded:Payload"  field, and a "Signature Verified" message will appear.

##### 3. Obtain an access token

The client then calls the SMART authentication server's "token endpoint" using the one-time use
authentication JWT as its authentication mechanism:


**Request**

```
POST /token HTTP/1.1
Host: authorize.smarthealthit.org
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=system%2F*.read%20system%2FCommunicationRequest.write&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&client_assertion=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzM4NCIsImtpZCI6ImVlZTlmMTdhM2I1OThmZDg2NDE3YTk4MGI1OTFmYmU2In0.eyJpc3MiOiJiaWxpX21vbml0b3IiLCJzdWIiOiJiaWxpX21vbml0b3IiLCJhdWQiOiJodHRwczovL2F1dGhvcml6ZS5zbWFydGhlYWx0aGl0Lm9yZy90b2tlbiIsImV4cCI6MTQyMjU2ODg2MCwianRpIjoicmFuZG9tLW5vbi1yZXVzYWJsZS1qd3QtaWQtMTIzIn0.l2E3-ThahEzJ_gaAK8sosc9uk1uhsISmJfwQOtooEcgUiqkdMFdAUE7sr8uJN0fTmTP9TUxssFEAQnCOF8QjkMXngEruIL190YVlwukGgv1wazsi_ptI9euWAf2AjOXaPFm6t629vzdznzVu08EWglG70l41697AXnFK8GUWSBf_8WHrcmFwLD_EpO_BWMoEIGDOOLGjYzOB_eN6abpUo4GCB9gX2-U8IGXAU8UG-axLb35qY7Mczwq9oxM9Z0_IcC8R8TJJQFQXzazo9YZmqts6qQ4pRlsfKpy9IzyLzyR9KZyKLZalBytwkr2lW7QU3tC-xPrf43jQFVKr07f9dA
```


**Response**

The response is a bearer token that will enable the client to retrieve  
resources from, and communicate with, the FHIR resource server for a five-minute time period.

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "m7rt6i7s9nuxkjvi8vsx",
  "token_type": "bearer",
  "expires_in": 300,
  "scope": "system/*.read system/CommunicationRequest.write"
}
```

## Presenting an Access Token to FHIR API

With a valid access token, a client MAY issue a FHIR API call to a FHIR resource server or other appropriate endpoint. The request MUST include an ```Authorization``` header that presents the ```access_token``` as a “Bearer” token:

```
Authorization: Bearer {{access_token}}  
```

[where {{access_token}} is replaced with the actual token value]

**Example Request**

```
GET https://ehr.example.org/metadata
Authorization: Bearer m7rt6i7s9nuxkjvi8vsx
```
The server SHALL validate the access token and SHALL ensure that the token has not expired and that its scope includes the requested resource.  The method the server uses to validate the access token is beyond the scope of this specification but generally involves an interaction or coordination between the resource server and the authorization server.

