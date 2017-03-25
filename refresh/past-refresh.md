---
layout: page
exclude_from_nav: true
title: Refreshing Past Versions
---

# Notes from past refresh efforts.

README from OPS 2.0 data refresh of data sources, from `ops-platform-setup` github project.

At the end is a count of number of triples from each dataset.

- [Data Loaded by the platform (v2.0)](
https://github.com/openphacts/ops-platform-setup/blob/2.0.0/data-sources/README.md)


Changes made in each data refresh (thru v1.4).  From wiki.  Includes sections for 1.5 and 2.0
updating, but the information does not appear to reflect any changes after v1.4 in August, 2013.

- [OPS datasets update planning (2013)](https://wiki.openphacts.org/index.php/OPS_datasets)


Artifactory repository of all version 2.1 RDF data files loaded into the Open PHACTS triple-store
(except SureChEMBL and DataBank).

- [v2.1 RDF data files](https://data.openphacts.org/free/2.1/rdf/)


Artifactory repository of all version 2.1 Linksets.

- [v2.1 Linkset files](https://data.openphacts.org/free/2.1/ims/)


Count of number of triples in Virtuoso from each dataset for version 2.1. November, 2016.

- [v2.1 triple counts by dataset](
https://wiki.openphacts.org/index.php/2.1_testing_aws#Graph_Comparison_between_beta_2.1_and_AWS)


2013 technical paper on various aspects of datasets and linksets, with emphasis on Void dataset
descriptions and predicates used in linksets.

- Paper: [Dataset Descriptions for the Open Pharmacological Space](
  http://www.openphacts.org/specs/2013/WD-datadesc-20130912/)


Open PHACTS wiki page planning 2.0 Linkset refresh.

- [IMS 2.0 reload](https://wiki.openphacts.org/index.php/IMS_2.0_reload)


From IdentityMappingService GitHub project:

- [Linksets To-Do table for v1.5.1](
https://github.com/openphacts/IdentityMappingService/blob/master/doc/ops-1.5.1/ims.csv)


Notes from May, 2015 on updating of Linksets.

- [IMS linksets 20150518](https://wiki.openphacts.org/index.php/IMS_linksets_20150518)


14 *.ttl files from `ops-platform-setup` GitHub project to load into Open PHACTS.

Q: Is this still true?  Should these be loaded for version 2.2, or superceded by other
files or processes?

- [Linkset files to load](
https://github.com/openphacts/ops-platform-setup/tree/2.0.0/linksets_dev)


Linkset file collections created within Open PHACTS project:

- uniprot


Linkset file collections obtained from external organizations (or uncertain):

- aers
- conceptwiki
- disgenet
- drugbank
- nextprot
- wikipathways


Linkset file collections downloaded using Maven projects in GitHub.

- [ops-chembl](https://github.com/openphacts/ops-chembl-linksets)
- [ops-ensembl-homosapiens](https://github.com/openphacts/ops-ensembl-linksets/tree/master/homosapiens)
- [ops-ensembl-musmusculus](https://github.com/openphacts/ops-ensembl-linksets/tree/master/musmusculus)
- [ops-hgnc](https://github.com/openphacts/ops-hgnc-linkset)
- [ops-hmdb](https://github.com/openphacts/ops-hmdb-linksets)
- [ops-rsc](https://github.com/openphacts/ops-rsc-linksets)
- [ops-rsc-surechembl](https://github.com/openphacts/ops-surechembl-linksets)
- [ops-surechembl](https://github.com/openphacts/ops-surechembl-linksets)


## BridgeDb

BridgeDb page, referencing 5 locations of Linkset data.

- [Identifier Mapping Databases](http://www.bridgedb.org/mapping-databases/)

README for the GitHub project for creating the BridgeDb Linksets.

- [Create BridgeDb Identity Mapping files](
  https://github.com/egonw/create-bridgedb-hmdb/blob/master/README.md)
