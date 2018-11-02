# SMART Backend Services: Authorization Guide

  
## Profile audience and scope

This profile is intended to be used by developers of back-end services that
need to access FHIR resources by requesting access tokens from OAuth 2.0
compliant authorization servers. This profile assumes that a backend service
has been authorized up-front, at registration time, and describes the runtime
process by which the service acquires an access token that can be used to
communicate with a FHIR Resoure Server.

#### **Use this profile** when the following conditions apply:

* The service runs automatically, without user interaction
* The service is able to protect a private key

### Examples

* An analytics platform or data warehouse that periodically performs a bulk data
export from an electronic health record system to provide insights into
a population of patients.

* A lab monitoring service that determines which patients are currently
admitted to the hospital, reviews incoming laboratory results, and generates
clinical alerts when specific trigger conditions are met.

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

Before a SMART backend service can run against an EHR, the service must be
registered with that EHR's authorization service.  SMART does not specify a
standards-based registration process, but we encourage EHR implementers to
consider the [OAuth 2.0 Dynamic Client Registration
Protocol](https://tools.ietf.org/html/draft-ietf-oauth-dyn-reg) for an
out-of-the-box solution.

No matter how a backend service registers with an EHR's authorization service, a 
backend service should communicate its **public key** to the SMART EHR using a
[JSON Web Key Set (JWKS)](https://tools.ietf.org/html/rfc7517) by one of the
following techniques:

  1. JWKS URL (preferred). This URL communicates the TLS-protected endpoint where the service's
  public JSON Web Keys can be found. When provided, this URL will match the `jku` header
  parameter in the service's Authorization JWTs. An advantage of this approach is that
  it allows a client to rotate its own keys by updating the hosted content at the JWKS URL.
  2. JWKS directly (allowed, not preferred). If a backend service cannot host a JWKS at a
  TLS-protected URL, it may supply a JWKS directly to the EHR at registration time. A limitation
  of this approach is that it does not enable the client to rotate its keys in-band.
  
It is recommended that EHRs should be capable of validating signatures using `RS384` and `ES384`; and that
backend services be capable of generating signatures using one of these two
algorithms. Over time, we expect recommended algorithms to evolve, so
while this specification recommends algorithms for interoperability, it does
not mandate any algorithm.

No matter how a JWKS is communicated to the EHR, each key in the JWKS *must be* an asymmetric
key that includes `kty` and `kid` properties, and whose content is conveyed using "bare key" properties (i.e., direct base64 encoding of 
key material as integer values). This means that:

* For RSA public keys, each MUST include `n` and `e` values (modulus and exponent) 
* For ECDSA public keys, each MUST include `crv`, `x`, and `y` values (curve, x-coordinate, and y-coordinate, for EC keys) 

Upon registration, the server assigns a `client_id`, which  the client uses when
obtaining an access token.

## Obtaining an access token

By the time a backend service has been registered with the EHR, the key
elements of organizational trust are already established. That is, the app is
considered "pre-authorized" to access clinical data. Then, at runtime, the
backend service must obtain an access token in order to work with clinical
data. Such access tokens can be issued automatically, without need for human
intervention, and they are short-lived, with a *recommended expiration time of
five minutes*.

To obtain an access token, the service uses an OAuth 2.0 client credentials
flow, with a [JWT
assertion](https://tools.ietf.org/html/rfc7523) as its
client authentication mechanism. The exchange, depicted below, allows the
backend service to authenticate to the EHR and request a short-lived
access token:

<img class="sequence-diagram-raw"  src="http://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgQmFja2VuZCBTZXJ2aWNlIEF1dGhvcml6YXRpb24KCm5vdGUgb3ZlciBBcHA6ICBDcmVhdGUgYW5kIHNpZ24gYXV0aGVudGljACsFIEpXVCBcbntcbiAgImlzcyI6ICJhcHBfY2xpZW50X2lkIiwAFgVzdWIAAxhleHAiOiAxNDIyNTY4ODYwLCAASAVhdWQiOiAiaHR0cHM6Ly97dG9rZW4gdXJsfQBNBiAianRpIjogInJhbmRvbS1ub24tcmV1c2FibGUtand0LWlkLTEyMyJcbn0gLS0-AIE3BndpdGggYXBwJ3MgcHJpdmF0ZSBrZXkgKFJTMzg0KQCBbBBzY29wZT1zeXN0ZW0vKi5yZWFkJlxuZ3JhbnRfdHlwZT0AgV8HY3JlZGVudGlhbHMmXG4AgXQHYXNzZXJ0aW9uACUGdXJuOmlldGY6cGFyYW1zOm9hdXRoOgCCIQYtACMJLXR5cGU6and0LWJlYXJlcgA8Ez17c2lnbmVkAIJ1FGZyb20gYWJvdmV9CgpBcHAtPkVIUgCDXAUAg2kFZXI6ICBQT1NUIACCQxNcbihTYW1lIFVSTCBhcwCCegYARgYpAIQJDABAEUlzc3VlIG5ldyAAgx0FOgCECAUiYWNjZXNzXwCDMAUiOiAic2VjcmV0LQCDQAUteHl6IixcbiJleHBpcmVzX2luIjogOTAwLFxuLi4uXG59CgCBKA8tPgCFBwVbAFAGAGMGIHJlc3BvbnNlXQ&s=default"/>

#### Protocol details

Before a backend service can request an access token, it must generate a
one-time-use JSON Web Token (JWT) that will be used to authenticate the service to
the EHR's authorization server. The authentication JWT is constructed with the
following claims, and then signed with the backend service's private RSA key
(RSA SHA-384 signature). For a practical reference on JWT, as well as debugging
tools and client libraries, see https://jwt.io.

<table class="table">
  <thead>
    <th colspan="3">Authentication JWT Header Values</th>
  </thead>
  <tbody>
    <tr>
      <td><code>alg</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The algorithm used for signing the authentication JWT (e.g., `RS384`, `EC384`).
      </td>
    </tr>
    <tr>
      <td><code>kid</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The identifier of the key-pair used to sign this JWT. This identifier MUST
          be unique within the backend services's JWK Set.</td>
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
      this should match a value that the backend service supplied to the EHR at
      client registration time.</td>
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
      <td>The service's <code>client_id</code>, as determined during registration with the EHR's authorization server
        (note that this is the same as the value for the <code>sub<code> claim)</td>
    </tr>
    <tr>
      <td><code>sub</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The service's <code>client_id</code>, as determined during registration with the EHR's authorization server
      (note that this is the same as the value for the <code>iss<code> claim)</td>
    </tr>
    <tr>
      <td><code>aud</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The EHR authorization server's "token URL" (the same URL to which this authentication JWT will be posted -- see below)</td>
    </tr>
    <tr>
      <td><code>exp</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Expiration time integer for this authentication JWT, expressed in seconds since the "Epoch" (1970-01-01T00:00:00Z UTC). This time MUST be no more than five minutes in the future.</td>
    </tr>
    <tr>
      <td><code>jti</code></td>
      <td><span class="label label-success">required</span></td>
      <td>A nonce string value that uniquely identifies this authentication JWT.</td>
    </tr>
  </tbody>
</table>

After generating an authentication JWT, the service requests a new access token
via HTTP `POST` to the EHR authorization server's token endpoint URL, using
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

The access token response is a JSON object, with the following properties:

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

## Server Obligations for Signature Verification

Servers SHALL follow all requirements defined in [Section 3 of RFC7523](https://tools.ietf.org/html/rfc7523#section-3).

In addition, we require that servers SHALL:
* validate the signature on the JWT
* check that the JWT `exp` claim is valid
* check that the JWT `aud` claim matches the server's OAuth token URL (the URL to which the token was `POST`ed)
* check that this is not a `jti` value previously encountered for the given `iss` within the maximum allowed authentication JWT lifetime (5 minutes). This check prevents replay attacks.
* ensure that the `client_id` provided is known and matches the JWT's `iss` claim

To resolve a key to verify signatures, a server follows this algorithm:

<ol>
  <li>If the <code>jku</code> header is present, verify that the <code>jku</code> is whitelisted (i.e., that it 
matches the value supplied at registration time for the specified `client_id`).
    <ol type="a">
      <li>If the <code>jku</code> header is not whitelisted, the signature verification fails.</li>
      <li>If the <code>jku</code> header is whitelisted, create a set of potential keys by dereferencing the <code>jku</code> URL. Proceed to step 3.</li>
    </ol>
  </li>
  <li> If <code>jku</code> is absent, create a set of potential key sources consisting of: all keys found by dereferencing the registration-time JWKS URI (if any) + any keys supplied in the registration-time JWKS (if any). Proceed to step 3.</li>
  <li> Filter the potential keys to retain only those where the <code>kid</code> matches the value supplied in the client's JWK header, and the <code>kty</code> is consistent with the signature algorithm used for the JWT (e.g., <code>RSA</code> for a JWT using an RSA-based signature, or <code>EC</code> for a JWT using an EC-based signature).</li>
  <li> Attempt to verify the JWK using each key in the potential keys list.
    <ol type="a">
      <li> If any attempt succeeds, the signature verification succeeds.</li>
      <li> If all attempts fail, the signature verification fails.</li>
    </ol>
  </li>
</ol>
 

If an error is encountered during the authorization process, servers SHALL respond with errors as defined by the [OAuth 2 specification](https://tools.ietf.org/html/rfc6749#section-5.2). Servers SHOULD also include an error_uri and error_description as defined by OAuth 2.

## Scopes

As there is no user or launch context when performing backed services authorization, 
the existing SMART on FHIR scopes are not appropriate. Instead, applications use 
system scopes, which have the format `system/(:resourceType|*).(read|write|*)`. These have
the same meanings as their matching `user/(:resourceType|*).(read|write|*)` scopes,
but associated with the permissions of the authorized client instead of a human end-user.

## Worked example

Assume that a "bilirubin result monitoring" service has registered with
the EHR's authorization server, establishing the following

 * JWT "issuer" URL: `bili_monitor`
 * OAuth2 `client_id`: `bili_monitor`
 * RSA [public key](sample-jwks/RS384.public.json)

Separately, the service also maintains its RSA [private key](sample-jwks/authorization-example-jwks-and-signatures.ipynb).

To obtain an access token at runtime, the bilirubin monitoring service wants to
start monitoring some bilirubin values. It needs to obtain an OAuth2 token with
the scopes `system/*.read system/CommunicationRequest.write`. To accomplish
this, the service must first generate a one-time-use authentication JWT with the following claims:

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


##### 2. Generate an RSA SHA-384 signed JWT over these claims

Using the service's RSA private key, the signed token value is:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzM4NCIsImtpZCI6ImVlZTlmMTdhM2I1OThmZDg2NDE3YTk4MGI1OTFmYmU2In0.eyJpc3MiOiJiaWxpX21vbml0b3IiLCJzdWIiOiJiaWxpX21vbml0b3IiLCJhdWQiOiJodHRwczovL2F1dGhvcml6ZS5zbWFydGhlYWx0aGl0Lm9yZy90b2tlbiIsImV4cCI6MTQyMjU2ODg2MCwianRpIjoicmFuZG9tLW5vbi1yZXVzYWJsZS1qd3QtaWQtMTIzIn0.l2E3-ThahEzJ_gaAK8sosc9uk1uhsISmJfwQOtooEcgUiqkdMFdAUE7sr8uJN0fTmTP9TUxssFEAQnCOF8QjkMXngEruIL190YVlwukGgv1wazsi_ptI9euWAf2AjOXaPFm6t629vzdznzVu08EWglG70l41697AXnFK8GUWSBf_8WHrcmFwLD_EpO_BWMoEIGDOOLGjYzOB_eN6abpUo4GCB9gX2-U8IGXAU8UG-axLb35qY7Mczwq9oxM9Z0_IcC8R8TJJQFQXzazo9YZmqts6qQ4pRlsfKpy9IzyLzyR9KZyKLZalBytwkr2lW7QU3tC-xPrf43jQFVKr07f9dA
```

(Note: to inspect this example JWT, you can visit https://jwt.io, choose RS384,
paste in the provided RSA keys, and then paste the JWT value into the "encoded"
field.)

##### 3. Obtain an access token

The service then calls the SMART EHR's "token endpoint" using the one-time use
authentication JWT as its client authentication mechanism:


**Request**

```
POST /token HTTP/1.1
Host: authorize.smarthealthit.org
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=system%2F*.read%20system%2FCommunicationRequest.write&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&client_assertion=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzM4NCIsImtpZCI6ImVlZTlmMTdhM2I1OThmZDg2NDE3YTk4MGI1OTFmYmU2In0.eyJpc3MiOiJiaWxpX21vbml0b3IiLCJzdWIiOiJiaWxpX21vbml0b3IiLCJhdWQiOiJodHRwczovL2F1dGhvcml6ZS5zbWFydGhlYWx0aGl0Lm9yZy90b2tlbiIsImV4cCI6MTQyMjU2ODg2MCwianRpIjoicmFuZG9tLW5vbi1yZXVzYWJsZS1qd3QtaWQtMTIzIn0.l2E3-ThahEzJ_gaAK8sosc9uk1uhsISmJfwQOtooEcgUiqkdMFdAUE7sr8uJN0fTmTP9TUxssFEAQnCOF8QjkMXngEruIL190YVlwukGgv1wazsi_ptI9euWAf2AjOXaPFm6t629vzdznzVu08EWglG70l41697AXnFK8GUWSBf_8WHrcmFwLD_EpO_BWMoEIGDOOLGjYzOB_eN6abpUo4GCB9gX2-U8IGXAU8UG-axLb35qY7Mczwq9oxM9Z0_IcC8R8TJJQFQXzazo9YZmqts6qQ4pRlsfKpy9IzyLzyR9KZyKLZalBytwkr2lW7QU3tC-xPrf43jQFVKr07f9dA
```


**Response**

```
{
  "access_token": "m7rt6i7s9nuxkjvi8vsx",
  "token_type": "bearer",
  "expires_in": 300,
  "scope": "system/*.read system/CommunicationRequest.write"
}
```
