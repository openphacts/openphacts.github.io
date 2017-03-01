---
layout: page
exclude_from_nav: true
title: System Architecture
---


This diagram shows top-level components of the core Open PHACTS platform.

Many of the component names are dated, or convey the role of that module rather then the name of
the app filling that role.  Details available below the diagram.


![Diagram of docker components.](/images/ops-arch-old-names.png)


### IMS

The Open PHACTS Identity Mapping Service.  Implemented by a combination of Query Exapander,
IdentityMappingService, and BridgeDb projects/apps.

Handles URI identifier mapping. Given a URI identifier as input, returns a list of URIs that
represent the same entity as the input URI. Essentially, returns all URIs that should have a
'owl:sameAs' or 'skos:exactMatch' relationship to the input URI.

In OPS, the IMS module performs *query expansion*.  Behind every API command, there is a SPARQL
query that is filled in with the URI identifier provided as an input parameter. *Query
exppansion* will then enrich that initial SPARQL query with all of a URI's aliases/synonyms
before executing that enriched query in the Open PHACTS Database.

**NOTE**:  The naming of "IMS" components is rather confusing. The acronym "IMS" stands for
"Identity Mapping Service".  IMS functionality is provided as a web app implemented by three
primary Java applications in the following GitHub projects:

* [queryExpander](https://github.com/openphacts/queryExpander)
* [IdentityMappingService](https://github.com/openphacts/IdentityMappingService)
* [BridgeDb](https://github.com/bridgedb/BridgeDb)

It is most correct to say that the OPS "IMS" *service* is provided by the *queryExpander*
project, rather than the *IdentityMappingService* project.  The *queryExpander* project has a
dependency on *IdentityMappingService* which in turn has a dependency on *BridgeDb*. The
URI-to-URI mapping data underlying *IdentityMappingService* and *BridgeDb* is stored in a MySQL
database.

See the following pages for naming of services (in Docker) and the projects/apps that implement
them:

* ['ops-docker' Docker structure](/ops-docker-components)
* [Docker images at hub.docker.com](/Docker-Hub-Images)
* [Project dependencies](/Repo-Dependencies)

The Docker service/container is named "ims", backed by the Docker image named
"identitymappingservice", which is derived from the GitHub project/app named "queryExpander".

A Maven repository of the IMS, QueryExpander, and Validator source code can be found here:

* [Maven software repository at Manchester](
http://repository.mygrid.org.uk/artifactory/ops/uk/ac/manchester/cs/)

### IRS

Identity Resolution Service.  A tool to aid the user in finding the proper URI identifier to use
in an API command.  The user can submit keywords or free text in a UI and get back a list of
candidate URI identifiers that might correspond to the text input.  For example, the user wants
to issue an API command about "Aspirin" or "Tuberculosis".  But in order to do so, they must
know a valid URI.  The IRS module is there to help them discover the right URI.

Currently, this functionality is provided by ConceptWiki. The plan is to create a new version
based on Elastic Search.  This is addressed in the GitHub project called [ops-search](https://github.com/openphacts/ops-search).


### OPCS

Open PHACTS Chemistry Service.

The RSC (Royal Society of Chemistry) module implements the API's "Chemistry structure search"
commands --- those starting with `/structure`.


### OPCR

Open PHACTS Chemistry Registry.

The RSC module used to validate and update the data in the "Compound" database that the OPCS
module accesses to implement the OPS API chemistry commands.  This functionality is provided by
**CVSP**.

This module is not used at runtime.  It is used to improve the chemistry data that OPCS uses at runtime.


### Open PHACTS Database

The RDF triplestore and SPARQL endpoint for executing the queries that implement most API commnds.

