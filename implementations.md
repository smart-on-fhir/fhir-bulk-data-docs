# FHIR Bulk Data Implementations

## Server

- SMART Reference Implementation - NodeJS (supports v0.2)

  https://github.com/smart-on-fhir/bulk-data-server (code)
  
  https://bulk-data.smarthealthit.org (online)
  
- HL7 (supports v0.1)
  
  https://test.fhir.org/r3
  
- Cerner (supports v0.2)
  
  https://fhir-open.stagingcerner.com/r4/a758f80e-aa74-4118-80aa-98cc75846c76/Patient/$export
  
- ONC (supports v0.1)
  
  http://52.70.192.201/open-fhir/fhir/ (open)
  
  http://52.70.192.201/secure-fhir/view/newuser.html (registration)
  
  http://52.70.192.201/secure-fhir/fhir/ (secure)

## Client

- SMART Reference Implementation - NodeJS (supports v0.2)
  
  https://github.com/smart-on-fhir/sample-apps-stu3/tree/master/fhir-downloader

- Python client application to stream data from a Bulk FHIR API into BigQuery, determining schema on the fly (supports v0.2)

  https://github.com/jmandel/fhir-bulk-data-to-bigquery

- Python client with auth support that converts a bulk data server into a generator for lightweight iteration through bulk resources returned (supports v0.2)

  https://github.com/plangthorne/python-fhir (code)
  
  https://github.com/plangthorne/python-fhir/blob/master/demo/BulkDataDemo.ipynb (demo notebook)

- Go client that fetches data and stores to local FS, google cloud storage and/or export to bigquery (supports v0.1)

  https://github.com/toby-hu/test/tree/master/client
