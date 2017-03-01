---
layout: page
exclude_from_nav: true
title: External Componenets
---


## BridgeDB

[in GitHub](https://github.com/bridgedb/BridgeDb)

Java project.  For mapping identifiers (in databases) to each other.  Used by IMS
([Identity Mapping Service](https://github.com/openphacts/IdentityMappingService)).


## IRS (Identity Resolution Service)

Conceptual conponent that helps users find a proper URI identifier for some entity they want to
use in Open PHACTS.

Input = a String from the user
Output = a list of candidate URI identifiers along with the basic type (e.g., Compound, Protein, Pathway) of that entity.

Scenario:  User wants to call an Open PHACTS query about Aspirin.  However, to use most API
calls, the user needs to know a URI for Aspirin.  They will then enter the string "Aspirin" into
IRS and get back a list of URIs associated with the value "Aspirin".  The user will browse the
list of results and select a URI to use in future API calls.

The initial implementation of IRS is based on ConceptWiki.

The second implementation of IRS is based on ElasticSearch.  See the "ops-search" github
project:  [ops-search](http://github.com/openphacts/ops-search).


## ConceptWiki

System for associating terms with identifiers and categories, finding synonyms.

One of the core Open PHACTS datasets plus provides a web service endpoint for some
Linked Data API commands.

Used in the initial implementation of Open PHACTS IRS.

From NBIC. Project is mostly dormant.  Not updated since 2013-2014.

List of semantic types used to tag Open PHACTS entities.

[ConceptWiki semantic types](
  http://support.openphacts.org/support/solutions/articles/169690-conceptwiki-uuids-for-most-frequent-semantic-tags)


## ElasticSearch

[www](https://www.elastic.co/products)

For free-text searching.  Based on Lucene.  Technology upon with new version of Open PHACTS's
IRS (Identity Resolution Service) is based.  See
[ops-search](http://github.com/openphacts/ops-search).


## Puelia-php

An implementation of the "Linked Data API".  Used by the Open PHACTS Linked Data API (github
project = "OPS_LinkedDataApi").

[Puelia at github](https://github.com/kwijibo/puelia)

[Puelia at google-code](https://code.google.com/archive/p/puelia-php/)

> It is an application that handles incoming requests by reading rdf/turtle configuration files (in /api-config-files/) and converting those requests into SPARQL queries which are used to retrieve RDF data from SPARQL endpoints declared in the configuration files.

> The RDF data is then served up in a number of format options, including turtle, rdf/xml and "simple" json and xml formats.

> To use, save your configuration files as turtle with a .ttl extension in "api-config-files/"

Uses Moriarity to make requests to SPARQL endpoints. And cache the responses it gets.


## Linked Data API

[in GitHub](https://github.com/UKGovLD/linked-data-api/blob/wiki/Specification.md)


## Moriarity

[in GitHub](https://github.com/iand/moriarty)

Moriarty is a simple PHP library for accessing the Talis Platform. And working with SPARQL and RDF.


## CRS

The RSC "Chemical Registration Service".

CRS = ChemSpider + CVSP + Compounds (?)

Q: How much of the RSC software (CRS, CVSP, etc.) required by Open PHACTS covered by the "ops-crs" project?

[ops-crs GitHub project](https://github.com/openphacts/ops-crs/)

Open PHACTS at RSC: [http://ops.rsc.org](http://ops.rsc.org)


## ChemSpider

[www](http://www.chemspider.com)

A database and web service providing information on chemicals, with an emphasis on chemical
structure.  An aggregator.  Created and maintained by RSC (Royal Society of Chemistry).

Millions of compounds.  Combines hundreds of datasets.  Difficult to manually curate that much
data.

ChemSpider core compound identifier = InChI-Key.


## CVSP

Chemical Validation and Standardization Platform.

From Royal Society of Chemistry. Associated with ChemSpider.

[www](http://cvsp.chemspider.com)

[paper](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4494041/)

For validating chemical records.  ~100 rules (can add your own).  Finds errors and inconsitencies.
Used to improve the quality of *compound* data in CRS and ChemSpider.  Difficult to manually
curate the quantity of chemical data in Compound database.


## OPCS

Open PHACTS Chemistry Service (or is it Chemical Structure?).

A legacy name given to the CRS web app that the Open PHACTS API calls to execute chemistry structure
queries.


## OPCR

Open PHACTS Chemistry Registry (or is it Chemical Registry, or Registraction?).

A legacy name given to the CRS web app used to validate and update the RSC chemistry
data. Essentially an alias for CVSP.


## CRS Compounds

The CRS database where its chemistry data is stored. Validated and updated by CVSP.  At runtime,
Open PHACTS API chemistry structure commands sent to CRS (the OPCS part of CRS) will access the
data stored in the "Compounds" database.
