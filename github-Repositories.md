---
# title: Projects
layout: page
exclude_from_nav: true
---

# openphacts github Repositories

[visit/edit this page at github](https://github.com/openphacts/openphacts.github.io/blob/master/github-Repositories.md)



----

Assembly of Open PHACTS Projects
---------------

----


## [ops-docker](http://github.com/openphacts/ops-docker)

A [Docker Compose](https://docs.docker.com/compose/) configuration to bring to life the Open PHACTS platform on Docker.

Most of the docker images are in https://hub.docker.com/u/openphacts/ from other GitHub repositores (See below) - while others are specialized within this repository (e.g. mysql).

* Slides: http://slides.com/soilandreyes/2016-07-06-openphacts#/
* [Webinar](https://www.big-data-europe.eu/the-open-phacts-pilot-second-hangout-for-the-health-societal-challenge/) and [Video](https://www.youtube.com/watch?v=L6sF_0rgocA)
* has Dockerfile - Uses Docker Compose
* Does not include 3-scale (https + api key authentication)
* includes stack from virtuoso up to explorer2
* not included: CRS (runs on Windows), ConceptWiki (shared instance)

Q: What does ops-docker depend on?


## [ops-platform-setup](http://github.com/openphacts/ops-platform-setup)

This is a "meta" repository that contains notes and a few bits-and-bobs scripts and config files. It can't be "run" by itself.

has 2.0.0 branch, no 2.1.

platform w/o data


----

Core Open PHACTS Components
---------------
----

## [OPS_LinkedDataApi](http://github.com/openphacts/OPS_LinkedDataApi)

PHP implemenation of the Open PHACTS API.

Directory `api-config-files` contains Puelia templates.

PHP Dependencies:

- Puelia
- Moriarity
- ARC

Web Service Dependencies:

- queryExpander
- IMS


## [queryExpander](http://github.com/openphacts/queryExpander)

Java.  Last edited 2016/04.

The Standalone Query Expander.

Depends on repos:

* [BridgeDb](https://github.com/openphacts/BridgeDb)
* [Validator](https://github.com/openphacts/Validator)
* [IdentityMappingService](https://github.com/openphacts/IdentityMappingService)

[Query Expander Demo Page](http://ops2.few.vu.nl/QueryExpander)


## [IdentityMappingService](http://github.com/openphacts/IdentityMappingService)

Java application and web service to obtain URI mappings.

Last commit: 2016/04


## [explorer2](http://github.com/openphacts/explorer2)

>The Open PHACTS Explorer is an HTML5 & CSS3 application for chemical information discovery and browsing. It is used to search for chemical compound and target information using a web  search interface.

Last updated: 2015/11

Has Dockerfile. Has Docker image [openphacts/explorer2](https://registry.hub.docker.com/u/openphacts/explorer2/).

Uses / depends-on:

* Ruby on Rails
* Ember JS
* [ops.js repo](http://github.com/openphacts/ops.js)
* [OPS_LinkedDataApi repo](http://github.com/openphacts/OPS_LinkedDataApi) (Linked Data API)


## [ops-crs](http://github.com/openphacts/ops-crs)

The sources of the Royal Society of Chemistry's (RSC) Chemical Registration Service (CRS).

The CRS is a top-level component of the Open PHACTS platform.  However, it is not part of the Open
PHACTS distribution.  Rather, it's functionality is provided via web apps running at
http://www.rsc.org.

The `/structure` commands of the Open PHACTS API are forwarded to the CRS web app.  Thus those
commands will fail if you are not connected to the internet.

The CRS is a Microsoft .NET and C# web application.

[Chemical Structure Search](https://dev.openphacts.org/docs/2.1#!/Chemical_structure_search)

Last updated: 2016/07

Currently "broken" (2016/10).


----

Projects using Open PHACTS
---------------

----

## [ops-search](http://github.com/openphacts/ops-search)

ElasticSearch-based search service for Open PHACTS, previously called "IRS2".  Under development.

has Dockerfile


## [ops.js](http://github.com/openphacts/ops.js)

> Javascript based API with methods used to access the Open PHACTS Discovery Platform and parse
> the responses

JavaScript, last edited 2016/01.

has Dockerfile for testing

* Used by `explorer2` and `ops-html-widgets` and http://www.biojs.io/d/openphacts-vis-compoundinfo
* Published on https://www.npmjs.com/package/ops.js
* Can be executed standalone as it includes comprehensive tests of API (using node.js)

Available as an npm package (for 1.5): <https://www.npmjs.com/package/ops.js>


## [OPS-Knime](http://github.com/openphacts/OPS-Knime)


## [ops_gems](http://github.com/openphacts/ops_gems)

Ruby gems.


## [JavaLDAClient](http://github.com/openphacts/JavaLDAClient)

A simple Scala + Java test app that wraps a few example calls to the Linked Data API and pulls content out of the returned JSON.

Scala, Java, 2013


----

Data and Linksets
---------------

----

## [ops-surechembl-linksets](http://github.com/openphacts/ops-surechembl-linksets)

## [ops-rsc-linksets](http://github.com/openphacts/ops-rsc-linksets)

## [ops-hgnc-linkset](http://github.com/openphacts/ops-hgnc-linkset)

## [ops-uniprot-linksets](http://github.com/openphacts/ops-uniprot-linksets)

## [ops-hmdb-linksets](http://github.com/openphacts/ops-hmdb-linksets)

## [ops-chembl-linksets](http://github.com/openphacts/ops-chembl-linksets)

## [ops-ensembl-linksets](http://github.com/openphacts/ops-ensembl-linksets)


## [ops-linkset-testing](http://github.com/openphacts/ops-linkset-testing)

> Testing of IMS linksets in Open PHACTS


## [ops-rsc-surechembl-dataset](http://github.com/openphacts/ops-rsc-surechembl-dataset)

## [ops-rsc-wikipathways-dataset](http://github.com/openphacts/ops-rsc-wikipathways-dataset)

## [ops-chembl-rdf](http://github.com/openphacts/ops-chembl-rdf)


## [chembl.rdf](http://github.com/openphacts/chembl.rdf)

**forked** from egonw/chembl.rdf
2013


## [ops-ncats-opdsr-rdf](http://github.com/openphacts/ops-ncats-opdsr-rdf)


----

The Rest
---------------

----


## [conceptwiki-docker](http://github.com/openphacts/conceptwiki-docker)

August 2015


## [QAWorkflows](http://github.com/openphacts/QAWorkflows)

**Private**

> Workflows for the testing the data and filters from the Open PHACTS API


## [ld-r](http://github.com/openphacts/ld-r)

Linked Data Reactor.  **Forked** from ali1k/ld-r


## [Void-Editor2](http://github.com/openphacts/Void-Editor2)

css
2016.05


## [Void-Editor](http://github.com/openphacts/Void-Editor)

2012


## [Validator](http://github.com/openphacts/Validator)

> The Void Validator, stripped out of the BridgeDb project, exposed via a web service.


## [openphacts-vis-compoundinfo](http://github.com/openphacts/openphacts-vis-compoundinfo)

> Displays the information available in the Open PHACTS Linked Data cache about a compound

## [ops-html-widgets](http://github.com/openphacts/ops-html-widgets)


## [deployment](http://github.com/openphacts/deployment)


## [coreGUI](http://github.com/openphacts/coreGUI)

> This is the repo for the central GUI API and GUI components code.


## [BridgeDb_Old](http://github.com/openphacts/BridgeDb_Old)

**forked** from amarillion/BridgeDb, updated Dec 2014


## [biojs](http://github.com/openphacts/biojs)

> A library of JavaScript components to represent biological data

**forked** from biojs/biojs.  2014


## [jqudt](http://github.com/openphacts/jqudt)

**forked** from egonw/jqudt

2014


## [PM](http://github.com/openphacts/PM)

Issues

**Private** 2014


## [Hosting](http://github.com/openphacts/Hosting)

> GitHub Issue Tracker for hosting provider.

**Private**

2014


## [DATA](http://github.com/openphacts/DATA)

Issues (1)
2016


## [GLOBAL](http://github.com/openphacts/GLOBAL)

Issues
**Private**
2016/09


## [Documentation](http://github.com/openphacts/Documentation)

2014.02


## [ApiDocsGenerator](http://github.com/openphacts/ApiDocsGenerator)

**forked** from fundatureanu-sever/DocsGenerator

> A repository hosting a Java helper application to generate response template descriptions for
> API methods, with color-coded markup in HTML.

2013


## [OpsPlatform](http://github.com/openphacts/OpsPlatform)

> ops platform and larkc plugins

**Private**
2012


## [OPS_widgets](http://github.com/openphacts/OPS_widgets)

> Repo for all general widgets in the project.

**Private**
2012


## [GitHubTest](http://github.com/openphacts/GitHubTest)

> Purely for testing Github. DO NOT save anything here

2011


================================================================
