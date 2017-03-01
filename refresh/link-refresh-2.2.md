---
layout: page
exclude_from_nav: true

title: Linkset Refresh v2.2
---


## chembl

Linkset files for ChEMBL are provided directly by the ChEMBL curators and available for download
along with the standard ChEMBL RDF datasets.  The following Linkset files are provided for
ChEMBL version 22.1:

- chembl_22.1_complextarget_targetcmpt_ls.ttl.gz
- chembl_22.1_grouptarget_targetcmpt_ls.ttl.gz
- chembl_22.1_molecule_chebi_ls.ttl.gz
    - chembl compound -> chebi
- chembl_22.1_singletarget_targetcmpt_ls.ttl.gz
    - chembl target -> chembl target component
- chembl_22.1_targetcmpt_uniprot_ls.ttl.gz
    - chembl target component -> uniprot TrEMBL


## uniprot

The GitHub project 'ops-uniprot-linksets' contains 13 *.sparql query files to create Linkset files
from UniProt data.  The 'data' directory appears to contain 12 *.ttl.gz files resulting from
executing the 13 *.sparql queries.  The `load.xml` contains the 12 files in the 'data'
directory.  The one *.sparql file that does not seem to be included in the data is called
'uniprot_ipi.sparql'.

- GitHub Project: [ops-uniprot-linksets](https://github.com/openphacts/ops-uniprot-linksets)

## wikipathways

New Linksets related to WikiPathways will be created by Egon W. and the folks at Maastricht.

They will also be creating some (or all?) of the linksets for the following:

* chebi
* goa
* hgnc
* Ensembl

## chebi

## goa

## hgnc

## Ensembl

Ensembl linksets for Human and Mouse, Sept, 2015.

- [Ensembl Linksets data for Human & Mouse](http://bridgedb.org/data/linksets/)
