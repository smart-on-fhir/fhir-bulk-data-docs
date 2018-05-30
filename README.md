# FHIR Bulk Data Proposal

## Overview

Providers and organizations accountable for managing the health of populations often need to efficiently access large volumes of information on a group of individuals. For example, a health system may want to periodically retrieve updated clinical data from an EHR to a research database, a provider may want to send clinical data on a roster of patients to their ACO to calculate quality measures, or an EHR may want to access claims data to close gaps in care. Currently, this is often done by exporting a specific subset of fields from the source system into a file format like CSV, transmitting these files to a server, and then importing the files into the target system. The effort involved in manually mapping the data fields to read and write these extracts raises the cost of integration projects and can act as a counter incentive to data liquidity.

Existing FHIR APIs work well for accessing small amounts of data, but large exports can require hundreds of thousands of requests. These draft specifications outline a standardized, FHIR based approach to bulk data export.

## Draft Specifications
 - [Authorization](./authorization.md)
 - [Export Operation](./export.md)

## Resources
 - [Overview Presentation](https://docs.google.com/presentation/d/14ZHmam9hwz6-SsCG1YqUIQnJ56bvSqEatebltgEVR6c/edit?usp=sharing)
 - [Client and Server Implementations](./implementations.md)
 - [Discussion Group (FHIR Zulip "Bulk Data" Track)](https://chat.fhir.org/#narrow/stream/bulk.20data)