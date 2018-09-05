# FHIR Bulk Data Export API Proposal

## Request Flow

### Authorization

Bulk data servers should implement the OAuth based [SMART backend services](./authorization.md) authorization process. On the requests outlined below, clients should include an ```Authorization``` header containing the bearer token received from the authorization flow. If the server responds to a request with a ```401 Unauthorized``` header, the client should follow the authorization flow to obtain a new token.

---
### Bulk Data Kick-off Request

This FHIR Operation initiates the asynchronous generation of data files for all patients, a group of patients, or all available data contained in a FHIR server.

Note: Only data the client application has authorization to access and that the relevant business agreements allow should be returned.

#### Endpoint - All Patients

```GET [fhir base]/Patient/$export```

#### Endpoint - Group of Patients

```GET [fhir base]/Group/[id]/$export```

FHIR Operation to obtain data on all patients listed in a single [FHIR Group Resource](https://www.hl7.org/fhir/group.html). Note: How these groups are defined will be implementation specific for each clinical system. For example, a payer may send a healthcare institution a roster file that can be imported into their EHR to create or update a FHIR group. Group membership could be based upon explicit attributes of the patient, such as: age, sex or a particular condition such as PTSD or Chronic Opioid use, or on more complex attributes, such as a recent inpatient discharge or membership in the population used to calculate a quality measure. Although, FHIR based group management is out of scope for the bulk data project, it would be a valuable project.

#### Endpoint - System Level Export

```GET [fhir base]/$export```

Export data from a FHIR server whether or not it is associated with a patient. This supports use cases like backing up a server or exporting terminology data by restricting the resources returned using the ```_type``` parameter.

#### Headers

- ```Accept``` (required)

  Specifies the format of the optional OperationOutcome response to the kick-off request. Currently, only application/fhir+json is supported.

- ```Prefer``` (required)

  Specifies whether the response is immediate or asynchronous. Currently must be set to ```respond-async```.

#### Query Parameters

- ```_outputFormat``` (string, optional, defaults to ```application/fhir+ndjson```)

  The format for the generated bulk data files. Currently, [ndjson](http://ndjson.org/) must be supported, though servers may choose to also support other output formats. Servers should support the full content type of ```application/fhir+ndjson``` as well as abbreviated representations including ```application/ndjson``` and ```ndjson```.

- ```_since``` (FHIR instant type, optional)  

  Resources updated after this period will be included in the response

  Note: This parameter was named ```start``` in an earlier version of this proposal

- ```_type``` (string of comma-delimited FHIR resource types, optional)

  Only resources of the specified resource types(s) will be included in the response. If this parameter is omitted, the server should return all supported resources that the client has authorization to access and that the relevant business agreements allow. The [Patient Compartment](https://www.hl7.org/fhir/compartmentdefinition-patient.html) should act as a point of reference for recommended resources to be returned as well as other resources outside of the patient compartment that are helpful in interpreting the patient data such as Organization and Practitioner.

  Note: Some implementations may limit the resources returned to specific subsets of FHIR like those defined in the [Argonaut Implementation Guide](http://www.fhir.org/guides/argonaut/r2/)

##### Experimental Query Parameters

As a community, we've identified use cases for finer-grained, client-specified filtering. For example, some clients may want to retrieve only active prescriptions (rather than historical prescriptions), or only laboratory observations (rather than all observations). We have considered several approaches to finer-grained filtering, including FHIR's `GraphDefinition`, the Clinical Query Language, and FHIR's REST API search parameters. We expect this will be an area of active exploration, so for the time being we're defining an experimental syntax based on search parameters that works side-by-side with our coarse-grained `_type`-based filtering.

To request finer-grained filtering, a client can supply a `_typeFilter` parameter alongside the `_type` parameter. The value of the `_typeFilter` parameter is a comma-separated list of FHIR REST API queries that further restrict the results of the query. Understanding `_typeFilter` is optional for servers; clients should be robust to servers that ignore `_typeFilter`.

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
- ```Content-Location``` header with a url for subsequent status requests
- Optionally a FHIR OperationOutcome in the body

#### Response - Error (eg. unsupported search parameter)

- HTTP Status Code of ```4XX``` or ```5XX```
- The body MUST be a FHIR OperationOutcome in JSON format

If a server wants to prevent a client from beginning a new export before an in-progress export is completed, it should respond with a `429` status and a Retry-After header, following the rate-limiting advice for "Bulk Data Status Request" below.

---
### Bulk Data Delete Request:

After a bulk data request has been started, clients can send a delete request to the url provided in the ```Content-Location``` header to cancel the request.

#### Endpoint 

```DELETE [polling content location]```

#### Response - Success

- HTTP Status Code of ```202 Accepted```
- Optionally a FHIR OperationOutcome in the body

#### Response - Error Status

- HTTP status code of ```4XX``` or ```5XX```
- The body MUST be a FHIR OperationOutcome in JSON format

---
### Bulk Data Status Request:

After a bulk data request has been started, clients can poll the url provided in the ```Content-Location``` header to obtain the status of the request. 

Note: Clients should follow an [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff) approach when polling for status. Servers may supply a [Retry-After header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After) with a http date or a delay time in seconds. When provided, clients should use this information to inform the timing of future polling requests. If a client is polling too frequently, the server should respond with a `429` status code in addition to a Retry-After header, and optionally an OperationOutcome with further explanation.

Note: The ```Accept``` header for this request should be ```application/json```. In the case that errors prevent the export from completing, the response will contain a JSON-encoded FHIR OperationOutcome resource. 

#### Endpoint 

```GET [polling content location]```

#### Response - In-Progress Status

- HTTP Status Code of ```202 Accepted```
- Optionally an ```X-Progress``` header with a text description of the status of the request that's less than 100 characters. The format of this description is at the server's discretion and may be a percentage complete value or a more general status such as "in progress". Clients can try to parse this value, display it to the user, or log it.

#### Response - Error Status

- HTTP status code of ```5XX```
- The body MUST be a FHIR OperationOutcome in JSON format
- Even if some resources cannot successfully be exported, the overall export operation may still succeed. In this case, the `Response.error` array of the completion Response must be populated (see below) with one or more files in ndjson format containing `OperationOutcome` resources to indicate what went wrong.

#### Response - Complete Status

- HTTP status of ```200 OK```
- ```Content-Type header``` of ```application/json```
-  Optionally an ```Expires``` header indicating when the files listed will no longer be available.
- A body containing a json object providing metadata and links to the generated bulk data files.

  Required Fields:
  - ```transactionTime``` - a FHIR instant type that indicates the server's time when the query is run. No resources that have a modified data after this instant should be in the response.
  - ```request``` - the full url of the original bulk data kick-off request
  - ```requiresAccessToken``` - boolean value indicating whether downloading the generated files will require an authentication token. Note: This may be false in the case of signed S3 urls or an internal file server within an organization's firewall.
  - ```output``` - array of bulk data file items with one entry for each generated file. Note: If no data is returned from the kick-off request, the server should return an empty array. 
  - ```error``` - array of error file items following the same structure as the `output` array. Note: If no errors occurred, the server should return an empty array.  Note: Only the `OperationOutcome` resource type is currently supported, so a server will generate ndjson files where each row is an `OperationOutcome` resource.
  
  Each file item should contain the following fields:
   - ```type``` - the FHIR resource type that is contained in the file. Note: Each file may only contain resources of one type, but a server may create more than one file for each resources type returned. The number of resources contained in a file is may vary between servers. If no data is found for a resource, the server should not return an output item for it in the response.
   - ```url``` - the path to the file. The format of the file should reflect that requested in the ```_outputFormat``` parameter of the initial kick-off request.
   
  Each file item may optionally contain the following field:
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
### File Requests:

Using the urls supplied in the completed status request body, clients can download the generated bulk data files (one or more per resource type). Note: These files may be served by a file server rather than a FHIR specific server. Also, if the ```requiresAccessToken``` field in the status body is set to ```true``` the request must include a valid access token in the ```Authorization``` header in these requests (i.e., `Authorization: Bearer {{token}}`).

#### Endpoint 

```GET [url from status request output field]```

#### Headers

- ```Accept``` (optional, defaults to ```application/fhir+ndjson```)

Specifies the format of the file being returned. Optional, but currently only application/fhir+ndjson is supported.

#### Response - Success

- HTTP status of ```200 OK```
- ```Content-Type``` header of ```application/fhir+ndjson```
- Body of FHIR resources in newline delimited json - [ndjson](http://ndjson.org/) format

#### Response - Error

- HTTP Status Code of ```4XX``` or ```5XX```

---
## Out of scope in v1 of this specification

- Legal framework for sharing data between partners - BAAs, SLAs, DUAs should continue to be negotiated out-of-band 
- Real-time data (although data loaded through bulk data can be supplemented at with synchronous FHIR REST API calls)
- Data transformation and transmission - different step of the ETL process
- Patient matching (although, it’s possible to include identifiers like subscriber number in FHIR resources)
- Management of FHIR groups within the clinical system - the bulk data operation will require a valid group id, but does not specify how FHIR Groups resources are created and maintained within a system

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
