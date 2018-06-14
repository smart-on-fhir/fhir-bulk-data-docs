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

* A clinician producivity tool runs through a roster of clinicians
and obtains access tokens to act as each one in turn, for the purposes
of perfoming anlaytics queries. (Note that this use case requires
a backend service to act "on behalf of" specific users, to ensure
that the analytics queries are properly scoped.)

## Registering a SMART Backend Service

Before a SMART backend service can run against an EHR, the service must be
registered with that EHR's authorization service.  SMART does not specify a
standards-based registration process, but we encourage EHR implementers to
consider the [OAuth 2.0 Dynamic Client Registration
Protocol](https://tools.ietf.org/html/draft-ietf-oauth-dyn-reg) for an
out-of-the-box solution.

No matter how a backend service registers with an EHR's authorization service,
at registration time **every SMART backend service must**:

* Register a fixed "issuer URL" with the EHR
* Register a public RSA key with the EHR (for RSA SHA-256 signatures)


## Obtaining an access token

By the time a backend service has been registered with the EHR, the key
elements of organizational trust are already established. That is, the app is
considered "pre-authorized" to access clinical data. Then, at runtime, the
backend service must obtain an access token in order to work with clinical
data. Such access tokens can be issued automatically, without need for human
intervention, and they are short-lived, with a *recommended expiration time of
five minutes*.

To obtain an access token, the service uses an OAuth 2.0 client credentials
flow, creating its own authorization grant as a JWT in accordance with a [JWT
assertion](https://tools.ietf.org/html/rfc7523#section-2.1). No explicit
client authentication mechanism is required, because the authorization JWT
can be authenticated directly. The exchange, depicted below, allows the
backend service to request a short-lived access token:

<img class="sequence-diagram-raw"  src="http://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgQmFja2VuZCBTZXJ2aWNlIEF1dGhvcml6YXRpb24KCm5vdGUgb3ZlciBBcHA6ICBDcmVhdGUgYW5kIHNpZ24gYQAjDCBKV1QgXG57XG4gICJpc3MiOiAiaHR0cHM6Ly97YXBwIHVybH0iLAAaBXN1YgADHGV4cCI6IDE0MjI1Njg4NjAsIABQBWF1ZABIDXRva2VuAEwLICJqdGkiOiAicmFuZG9tLW5vbi1yZXVzYWJsZS1qd3QtaWQtMTIzIlxufSAtLT4AgT4Gd2l0aCBhcHAncyBwcml2YXRlIGtleSAoUlNBIFNIQS0yNTYpAIF5EHNjb3BlPXN5c3RlbS8qLnJlYWQmXG5ncmFudF90eXBlPXVybjppZXRmOnBhcmFtczpvYXV0aDoAHAUtIHR5cGU6and0LWJlYXJlciZcbmFzc2VydGlvbj17c2lnbmVkAIJHE2Zyb20gYWJvdmV9CgpBcHAtPkVIUgCDLAUAgzkFZXI6ICBQT1NUIACCDBNcbihTYW1lIFVSTCBhcwCCQwYARgYpAINZDABAEUlzc3VlIG5ldyAAgmYFOgCDWQUiYWNjZXNzXwCCeQUiOiAic2VjcmV0LQCDCQUteHl6IixcbiJleHBpcmVzX2luIjogOTAwLFxuLi4uXG59CgCBKA8tPgCEVwVbAFAGAGMGIHJlc3BvbnNlXQoKCgo&s=default"/>

#### Protocol details

Before a Backend Service can request an access token, it must generate a
one-time-use JSON Web Token that will be used as an authorization grant. The
JWT is authenticated (i.e., signed by the Service) and the EHR's authorization server
(after applying any policies of its own) takes the JWT as authorization to 
issue an access token. The authorization JWT is constructed with the
following claims, and then signed with the backend service's private RSA key
(RSA SHA-256 signature). For a practical reference on JWT, as well as debugging
tools and client libraries, see http://jwt.io.

<table class="table">
  <thead>
    <th colspan="3">Authentication JWT Claims</th>
  </thead>
  <tbody>
    <tr>
      <td><code>iss</code></td>
      <td><span class="label label-success">required</span></td>
      <td>The service's issuer URI, as registered with the EHR's authorization server</td>
    </tr>
    <tr>
      <td><code>sub</code></td>
      <td><span class="label label-success">required</span></td>
      <td><ul>
        <li>If the Backend Service is issuing this request on its own behalf, then <code>sub</code> should be the same as <code>iss</code>.</li>
        <li>If the Backend Service is issuing this request on behalf of a user, then <code>sub</code> should be a user identifier that the EHR understands. (The recommended approach is a full URI, like <code>https:/ehr.example.org/Practitioner/123</code>.)
        </li>
      </td>
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

After generating an authorization JWT, the service requests a new access token
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
      <td>Fixed value: <code>urn:ietf:params:oauth:grant- type:jwt-bearer</code></td>
    </tr>
    <tr>
      <td><code>assertion</code></td>
      <td><span class="label label-success">required</span></td>
      <td>Signed authorization JWT value (see above)</td>
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

## Server Obligations ##

<span class="label label-warning">TODO</span>

Servers SHALL
* validate the signature on the authorization JWT
* check that the JWT `exp` is valid
* check that the JWT `aud` matches the server's OAuth token URL (the URL to which the token was `POST`ed)
* check that this is not a `jti` value previously encountered for the given `sub` within the maximum allowed authentication JWT lifetime (5 minutes). This check prevents replay attacks.
* ensure that the supplied `iss` is associated with the signing key used for the JWT

## Scopes

As there is no user or launch context when performing backed services authorization, 
the existing SMART on FHIR scopes are not appropriate. Instead, applications use 
system scopes, which have the format `system/(:resourceType|*).(read|write|*)`. These have
the same meanings as their matching `user/(:resourceType|*).(read|write|*)` scopes,
but associated with the permissions of the authorized client instead of a human end-user.

## Worked example

Assume that a "bilirubin result monitoring" service has registered with
the EHR's authorization server, establishing the following

 * JWT "issuer" URL: `https://bili-monitoring-service.example.com/`
 * OAuth2 `client_id`: `bili_monitor`
 * RSA [public key](example-rsa-key.pub)

Separately, the service also maintains its RSA [private key](example-rsa-key.priv).

To obtain an access token at runtime, the bilirubin monitoring service wants to
start monitoring some bilirubin values. It needs to obtain an OAuth2 token with
the scopes `system/*.read system/CommunicationRequest.write`. To accomplish
this, the service must first generate a one-time-use authentication JWT with the following claims:

##### 1. Generate an authorization JWT:

```
{
  "iss": "https://bili-monitoring-service.example.com",
  "sub": "https://bili-monitoring-service.example.com",
  "aud": "https://authorize.smarthealthit.org/token",
  "exp": 1422568860,
  "jti": "random-non-reusable-jwt-id-123"
}
```


##### 2. Generate an RSA SHA-256 signed JWT over these claims

Using the service's RSA private key, the signed token value is:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwczovL2JpbGktbW9uaXRvcmluZy1zZXJ2aWNlLmNvbS8iLCJzdWIiOiJiaWxpX21vbml0b3IiLCJhdWQiOiJodHRwczovL2F1dGhvcml6ZS5zbWFydHBsYXRmb3Jtcy5vcmcvdG9rZW4iLCJleHAiOjE0MjI1Njg4NjAsImp0aSI6InJhbmRvbS1ub24tcmV1c2FibGUtand0LWlkLTEyMyJ9.Psqfs2IEw_1GcGiSZDdEZquS-iA_gVBpNSedAghL4R9FkJWdvReXvkeBFtgBIa2PjRIQQSLYR7p3XtaH3YERivuxOKCg7OCla8dkLrlaNujhfSdwEdvn-f1GTrytjNTJWEHg0jEDeRoZn7zYy7jFZBYmF0xsRwZe7wisyaCob1w
```

(Note: to inspect this example JWT, you can visit http://jwt.io, choose RS256,
paste in the provided RSA keys, and then paste the JWT value into the "encoded"
field.

TODO: Update `iss` and `sub` in signed token to match the example.)

##### 3. Obtain an access token

The service then calls the SMART EHR's "token endpoint" using the one-time use
authentication JWT as its client authentication mechanism:


**Request**

```
POST /token HTTP/1.1
Host: authorize.smarthealthit.org
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&scope=system%2F*.read%20system%2FCommunicationRequest.write&assertion=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwczovL2JpbGktbW9uaXRvcmluZy1zZXJ2aWNlLmNvbS8iLCJzdWIiOiJiaWxpX21vbml0b3IiLCJhdWQiOiJodHRwczovL2F1dGhvcml6ZS5zbWFydHBsYXRmb3Jtcy5vcmcvdG9rZW4iLCJleHAiOjE0MjI1Njg4NjAsImp0aSI6InJhbmRvbS1ub24tcmV1c2FibGUtand0LWlkLTEyMyJ9.Psqfs2IEw_1GcGiSZDdEZquS-iA_gVBpNSedAghL4R9FkJWdvReXvkeBFtgBIa2PjRIQQSLYR7p3XtaH3YERivuxOKCg7OCla8dkLrlaNujhfSdwEdvn-f1GTrytjNTJWEHg0jEDeRoZn7zYy7jFZBYmF0xsRwZe7wisyaCob1w
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
