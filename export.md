# DRAFT FHIR Bulk Data Export Implementation Guide

## Audience and Scope

This implementation guide is intended to be used by developers of backend services (clients) and FHIR Resource Servers (e.g., EHR systems, data warehouses, and other clinical and administrative systems) that aim to interoperate by sharing large FHIR datasets.  The guide defines the application programming interfaces (APIs) through which an authenticated and authorized client may request a bulk-data export from a server, receive status information regarding progress in the generation of the requested files, and receive these files.  It also includes recommendations regarding the FHIR resources that might be exposed throuh the export interface.  

The scope of this document does NOT include:

* A legal framework for sharing data between partners, including Business Associate Agreements, Service Level Agreements, and Data Use Agreements
* Real-time data exchange
* Data transformations that may be required by the client
* Patient matching (although identifiers may be incuded in the exported FHIR resources)
* Management of FHIR groups within the clinical system; the bulk data operation may include a valid group id, but this guide does not specify how FHIR Group resources are created and maintained within a system 

## Referenced Specifications

* Newline-delimited JSON.  http://ndjson.org
* The OAuth 2.0 Authorization Framework, RFC6749, https://tools.ietf.org/html/rfc6749 
* HL7 FHIR Release 3, https://www.hl7.org/fhir/
* The JavaScript Object Notation (JSON) Data Interchange Format, RFC7159.  https://tools.ietf.org/html/rfc7159
* Transport Layer Security (TLS) Protocol Version 1.2.  RFC5246).  https://tools.ietf.org/html/rfc5246 
* The OAuth 2.0 Authorization Framework: Bearer Token Usage, RFC6750.  https://tools.ietf.org/html/rfc6750

## Terminology

This profile inherits terminology from the standards referenced above.
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this specification are to be interpreted as described in RFC2119.


## Bulk Data Export Requests 

### Bulk Data Kick-off Request

This FHIR Operation initiates the asynchronous process of a client's request for the generation of as set of data to which the client is authorized -- whether that be all patients, a subset (defined group) of patients, or all available data contained in a FHIR server.

The FHIR server MUST limit the data returned to only those FHIR resources for which the client is authorized.

#### Endpoint - All Patients

```GET [fhir base]/Patient/$export```

#### Endpoint - Group of Patients

```GET [fhir base]/Group/[id]/$export```

FHIR Operation to obtain data on all patients listed in a single [FHIR Group Resource](https://www.hl7.org/fhir/group.html). 

If a FHIR server supports Group-level data export, it SHOULD support reading and searching for `Group` resource. This enables  clients to discover available groups based on stable characteristics such as `Group.identifier`. 

Note: How these groups are defined is implementation specific for each FHIR system. For example, a payer may send a healthcare institution a roster file that can be imported into their EHR to create or update a FHIR group. Group membership could be based upon explicit attributes of the patient, such as: age, sex or a particular condition such as PTSD or Chronic Opioid use, or on more complex attributes, such as a recent inpatient discharge or membership in the population used to calculate a quality measure. FHIR-based group management is out of scope for the current version of this implementation guide.

#### Endpoint - System Level Export

```GET [fhir base]/$export```

Export data from a FHIR server whether or not it is associated with a patient. This supports use cases like backing up a server or exporting terminology data by restricting the resources returned using the ```_type``` parameter.

#### Headers

- ```Accept``` (required)

  Specifies the format of the optional OperationOutcome response to the kick-off request. Currently, only application/fhir+json is supported.

- ```Prefer``` (required)

  Specifies whether the response is immediate or asynchronous. The header MUST be set to ```respond-async```.

#### Query Parameters

- ```_outputFormat``` (string, optional, defaults to ```application/fhir+ndjson```)

  The format for the requested bulk data files to be generated. Servers MUST support [Newline Delimited JSON](http://ndjson.org), but MAY choose to support additional output formats. Servers MUST accept the full content type of ```application/fhir+ndjson``` as well as the abbreviated representations ```application/ndjson``` and ```ndjson```.

- ```_since``` (FHIR instant type, optional)  

  Resources updated after this period will be included in the response

  Note: This parameter was named ```start``` in an earlier version of this proposal

- ```_type``` (string of comma-delimited FHIR resource types, optional)

  Only resources of the specified resource types(s) SHOULD be included in the response. If this parameter is omitted, the server SHOULD return all supported resources within the scope of the client authorization. For non-system-level requests, the [Patient Compartment](https://www.hl7.org/fhir/compartmentdefinition-patient.html) SHOULD be used as a point of reference for recommended resources to be returned as well as other resources outside of the patient compartment that are helpful in interpreting the patient data such as Organization and Practitioner.
  
  Resource references MAY be relative URIs with the format `<resource type>/<id>`, or absolute URIs with the same structure rooted in the base URI for the server from which the export was performed. References will be resolved looking for a resource with the specified type and id within the file set.

  Note: Implementations MAY limit the resources returned to specific subsets of FHIR, such as those defined in the [Argonaut Implementation Guide](http://www.fhir.org/guides/argonaut/r2/)

##### Experimental Query Parameters

As a community, we've identified use cases for finer-grained, client-specified filtering. For example, some clients may want to retrieve only active prescriptions (rather than historical prescriptions), or only laboratory observations (rather than all observations). We have considered several approaches to finer-grained filtering, including FHIR's `GraphDefinition`, the Clinical Query Language, and FHIR's REST API search parameters. We expect this will be an area of active exploration, so for the time being this document defines an experimental syntax based on search parameters that works side-by-side with the coarse-grained `_type`-based filtering.

To request finer-grained filtering, a client MAY supply a `_typeFilter` parameter alongside the `_type` parameter. The value of the `_typeFilter` parameter is a comma-separated list of FHIR REST API queries that further restrict the results of the query. Understanding `_typeFilter` is OPTIONAL for FHIR servers; clients SHOULD be robust to servers that ignore `_typeFilter`.

*Note for client developers*: Because both `_typeFilter` and `_since` can restrict the results returned, the interaction of these parameters may be surprising. Think carefully through the implications when constructing a query with both of these parameters.

###### Example Request with `_typeFilter`

The following is an export request for `MedicationRequest` resources and `Condition` resources, where the client would further like to restrict `MedicationRequests` to requests that are `active`, or else `completed` after July 1 2018. This can be accomplished with two subqueries, joined together with a comma for a logical "or":

* `MedicationRequest?status=active`
* `MedicationRequest?status=completed&date=gt2018-07-01T00:00:00Z`

To create a `_typeFilter` parameter, a client should URL encode these two subqueries and join them with `,`.
Newlines and spaces have been added for clarity, and would not be included in a real request:

```
$export?
  _type=
    MedicationRequest,
    Condition&
  _typeFilter=
    MedicationRequest%3Fstatus%3Dactive,
    MedicationRequest%3Fstatus%3Dcompleted%26date%3Dgt2018-07-01T00%3A00%3A00Z
```

Note: the `Condition` resource is included in `_type` but omitted from `_typeFilter` because the client intends to request all `Condition` resources without any filters.

#### Response - Success

- HTTP Status Code of ```202 Accepted``` 
- ```Content-Location``` header with an absolute URI for subsequent status requests
- Optionally a FHIR OperationOutcome in the body

#### Response - Error (eg. unsupported search parameter)

- HTTP Status Code of ```4XX``` or ```5XX```
- The body MUST be a FHIR OperationOutcome in JSON format

If a server wants to prevent a client from beginning a new export before an in-progress export is completed, it SHOULD respond with a `429 Too Many Requests` status and a Retry-After header, following the rate-limiting advice for "Bulk Data Status Request" below.

---
### Bulk Data Delete Request

After a bulk data request has been started, a client MAY send a delete request to the URI provided in the ```Content-Location``` header to cancel the request.    

#### Endpoint 

```DELETE [polling content location]```

#### Response - Success

- HTTP Status Code of ```202 Accepted```
- Optionally a FHIR OperationOutcome in the body

#### Response - Error Status

- HTTP status code of ```4XX``` or ```5XX```
- The body MUST be a FHIR OperationOutcome in JSON format

---
### Bulk Data Status Request

After a bulk data request has been started, the client MAY poll the URI provided in the ```Content-Location``` header.  

Note: Clients SHOULD follow an [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff) approach when polling for status. Servers SHOULD supply a [Retry-After header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After) with a http date or a delay time in seconds. When provided, clients SHOULD use this information to inform the timing of future polling requests. Servers SHOULD keep an accounting of status queries received from a given client, and if a client is polling too frequently, the server SHOULD respond with a `429 Too Many Requests` status code in addition to a Retry-After header, and optionally a FHIR OperationOutcome resource with further explanation.  If excessively frequent status queries persist, the server MAY return a `429 Too Many Requests` status code and terminate the session.  

Note: When requesting status, the client SHOULD use an ```Accept``` header for indicating content type  ```application/json```. In the case that errors prevent the export from completing, the server SHOULD respond with  a JSON-encoded FHIR OperationOutcome resource. 

#### Endpoint 

```GET [polling content location]```

#### Response - In-Progress Status

- HTTP Status Code of ```102 Processing```
- Optionally, the server MAY return an ```X-Progress``` header with a text description of the status of the request that's less than 100 characters. The format of this description is at the server's discretion and may be a percentage complete value or a more general status such as "in progress". The client MAY parse the description, display it to the user, or log it.

#### Response - Error Status

- HTTP status code of ```5XX```
- The server MUST return a FHIR OperationOutcome resource in JSON format
- Even if some of the requested resources cannot successfully be exported, the overall export operation MAY still succeed. In this case, the `Response.error` array of the completion response MUST be populated (see below) with one or more files in ndjson format containing FHIR `OperationOutcome` resources to indicate what went wrong.

#### Response - Complete Status

- HTTP status of ```200 OK```
- ```Content-Type header``` of ```application/json```
- The server MAY return an ```Expires``` header indicating when the files listed will no longer be available.
- A body containing a json object providing metadata and links to the generated bulk data files.  Note that the files-for-download MAY be served by a file server other than a FHIR-specific server.

  Required Fields:
  - ```transactionTime``` - a FHIR instant type that indicates the server's time when the query is run. The response SHOULD NOT include any resources modified after this instant, and SHALL include any matching resources modified up to (and including) this instant. Note: to properly meet these constraints, a FHIR Server might need to wait for any pending transactions to resolve in its database, before starting the export process. 
  - ```request``` - the full URI of the original bulk data kick-off request
  - ```requiresAccessToken``` - boolean value of ```true``` or ```false``` indicating whether downloading the generated files requires an access token. Value MUST be ```true``` for all RESTful downloads (e.g., SMART).   Value MAY be ```false``` for non-RESTful downloads, such as downloads from Amazon S3 bucket URIs or verifiable file servers within an organization's firewall.
  - ```output``` - array of bulk data file items with one entry for each generated file. Note: If no resources are returned from the kick-off request, the server SHOULD return an empty array. 
  - ```error``` - array of error file items following the same structure as the `output` array. Note: If no errors occurred, the server SHOULD return an empty array.  Note: Only the `OperationOutcome` resource type is currently supported, so a server MUST generate files in the same format as the bulk data output files that contain `OperationOutcome` resources.
  
  Each file item SHOULD contain the following fields:
   - ```type``` - the FHIR resource type that is contained in the file. Note: Each file MUST contain resources of only one type, but a server MAY create more than one file for each resources type returned. The number of resources contained in a file is MAY  vary between servers. If no data are found for a resource, the server SHOULD NOT return an output item for that resource in the response.
   - ```url``` - the path to the file. The format of the file SHOULD reflect that requested in the ```_outputFormat``` parameter of the initial kick-off request.
   
  Each file item MAY optionally contain the following field:
   - ```count``` - the number of resources in the file, represented as a JSON number.
    

	Example response body:
    
	```json
    {
      "transactionTime": "[instant]",
      "request" : "[base]/Patient/$export?_type=Patient,Observation", 
      "requiresAccessToken" : true,
      "output" : [{
        "type" : "Patient",
        "url" : "http://serverpath2/patient_file_1.ndjson"
      },{
        "type" : "Patient",
        "url" : "http://serverpath2/patient_file_2.ndjson"
      },{
        "type" : "Observation",
        "url" : "http://serverpath2/observation_file_1.ndjson"
      }],
      "error" : [{
        "type" : "OperationOutcome",
        "url" : "http://serverpath2/err_file_1.ndjson"
      }]
    }
    ```

---
### File Request

Using the URIs supplied by the FHIR server in the Complete Status response body, a client MAY download the generated bulk data files (one or more per resource type) within the specified ```Expires``` time period. If the ```requiresAccessToken``` field in the Complete Status body is set to ```true```, the request MUST include a valid access token.  See the Security Considerations section above.  

#### Endpoint 

```GET [url from status request output field]```

#### Headers

- ```Accept``` (optional, defaults to ```application/fhir+ndjson```)

Specifies the format of the file being requested. 

#### Response - Success

- HTTP status of ```200 OK```
- ```Content-Type``` header that matches the file format being delivered.  For files in ndjson format, MUST be ```application/fhir+ndjson```
- Body of FHIR resources in newline delimited json - [ndjson](http://ndjson.org/) or other requested format

#### Response - Error

- HTTP Status Code of ```4XX``` or ```5XX```



---
## Change Log:

#### 1/27/2018 (Draft v0.1)

- Moved output links from ```Link``` header to body and defined an output JSON format
- Added ```output-format``` parameter to differentiate between the format of the response to the request and the format of the linked output files

#### 1/28/2018 (Draft v0.2)

- Renamed ```output-format``` parameter to ```_outputFormat``` since it can apply to any async request
- Added ```DELETE``` method to status endpoint
- Changed the operation name to ```$export``` (under discussion)
- Changed ```start``` parameter to ```_since``` to align with existing FHIR semantics

#### 2/1/2018 (Draft v0.2.1)

- Added recommendation around use of ```Retry-After``` header

#### 5/30/2018 (Draft v0.2.2)

- Added recommendation around use of ```Accept``` header in status request

#### 5/30/2018 (Draft v0.3.0)

- Added system wide `$export`
- Renamed `secure` to `requiresAuthorizationToken`
- Added `error`  and `count` properties to copmletion response

#### 6/21/2018 (Draft 0.4.0)
- Renamed `requiresAuthorizationToken` to `requiresAccessToken`

### 9/1/2018 (Draft 0.4.1)
- Require OperationOutcome in error conditions

### 9/29/2018 (Draft 0.4.2)
- Restrict Content-location to an absolute URL
