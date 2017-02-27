---
layout: page
exclude_from_nav: false
---

# Build / Refresh Process of OPS Platform Components

The core components of the OPS platform are:

* Data Querying Service
* Identity Mapping Service
* Linked Data API Service


## Data Querying Service (Software)

Provides a SPARQL Endpoint for querying pharmacological & life sciences data.

Build & Deloy Process:

- Select Triple-Store product & version (e.g., Virtuoso)
- Obtain (download)
- Build and/or install
- Configure
- Integrate
- Run
- Expose/Shield
- Dockerize
- Publish
- Document
- Maintain


## Data Querying Service (Data)

Obtaining, preparing, and loading RDF data into the Data Service triple-store.

1. Dataset identification, inventory, selection, location

1. Pre-Download data file preparation

1. Download of source data files

1. Unpack archives

1. Transform non-RDF data into loadable RDF data files.

1. Assembly of all required files into a _staging_ area.

1. Add supplementary files.

1. Load the RDF data into the triple-store (Virtuoso)

1. Execute post-loading SPARQL Update queries.

1. SPARQL Data Service final steps

1. Dump loaded data for future "quick loading".


## Identity Mapping Service (Software)

Obtaining, building, configuring, running IMS - Identity Mapping Service

IMS consists of 4 software components:

- QueryExpander
- IdentityMappingService
- Validator
- BridgeDb.

Plus 2 database components:

- MySQL
- Derby

Build & Deloy Process:

- Obtain (download/clone from GitHub)
- Build and/or install
- Configure
- Integrate
- Run
- Expose/Shield
- Dockerize
- Publish
- Document
- Maintain


## Identity Mapping Service (Data)

Create, obtain, organize, integrate, and load the linkset data to populate the IMS.

Build process:

1. Linkset identification, inventory, selection, location

1. Pre-Download linkset preparation

1. Download Linkset files

1. Unpack linkset archives

1. Transform into IMS-ingestible format.

1. Generate linksets from software.

1. Assembly of all required Linkset files into a _staging_ area.

1. Add supplementary files

1. Load IMS

1. Post-loading IMS final steps.

1. Dump loaded data for future "quick loading".


## Linked Data API Service

Obtaining, building, configuring, and running the Linked Data API REST web service.
