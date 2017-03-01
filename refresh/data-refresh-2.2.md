---
layout: page
exclude_from_nav: true
title: Dataset Refresh v2.2
---

# Plans for refreshing Datasets for version 2.2. 

## chembl

Download new versions of RDF files form ChEMBL FTP site.

FTP site: [ftp://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBL-RDF/22.1/](
ftp://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBL-RDF/22.1/)

Files to download:

```
cco.ttl.gz
chembl_22.1_activity.ttl.gz
chembl_22.1_assay.ttl.gz
chembl_22.1_bindingsite.ttl.gz
chembl_22.1_biocmpt.ttl.gz
chembl_22.1_cellline.ttl.gz
chembl_22.1_document.ttl.gz
chembl_22.1_indication.ttl.gz
chembl_22.1_journal.ttl.gz
chembl_22.1_moa.ttl.gz
chembl_22.1_molecule.ttl.gz
chembl_22.1_molhierarchy.ttl.gz
chembl_22.1_protclass.ttl.gz
chembl_22.1_source.ttl.gz
chembl_22.1_target.ttl.gz
chembl_22.1_targetcmpt.ttl.gz
chembl_22.1_targetrel.ttl.gz
chembl_22.1_unichem.ttl.gz
void.ttl.gz
```

Put into RDF Graph:  `<http://www.ebi.ac.uk/chembl>`

ChEMBL Linkset files can be downloaded from the same FTP directory.  Those file names end with "_ls.ttl.gz".


## uniprot

To obtain UniProt data for Open PHACTS, issue 3 queries to UniProt REST service:

```
curl "http://www.uniprot.org/uniprot/?query=reviewed:yes&format=rdf&compress=yes" -o swissprot.rdf.gz
curl "http://www.uniprot.org/uniparc/?query=reviewed:yes&format=rdf&compress=yes" -o uniparc.rdf.gz
curl "http://www.uniprot.org/uniref/?query=reviewed:yes&format=rdf&compress=yes" -o uniref.rdf.gz
```


## enzyme

Download new version of RDF file form UniProt FTP site.  The `enzyme` data is part of UniProt
distribution.  So if UniProt is refreshed, then 'enzyme' should also be refreshed to the same
version to keep it in sync.

FTP site: [ftp://ftp.uniprot.org/pub/databases/uniprot/current_release/rdf/enzyme.rdf.xz](
ftp://ftp.uniprot.org/pub/databases/uniprot/current_release/rdf/enzyme.rdf.xz)

Put into RDF Graph:  `<http://purl.uniprot.org/enzyme>`


## wikipathways

Egon & crew will create new versions.  When ready, download the following files:

```
http://data.wikipathways.org/20170210/rdf/voidGpml.ttl
http://data.wikipathways.org/20170210/rdf/voidWp.ttl
http://data.wikipathways.org/20170210/rdf/wikipathways-20170210-rdf-gpml.zip
http://data.wikipathways.org/20170210/rdf/wikipathways-20170210-rdf-wp.zip
```

The `20170210` part is the version number.  The data appears to be updated monthly.

The '*.zip' files will need to be _unpacked_ before they can be loaded.  Each 'zip' file
contains many RDF files.

Put into RDF Graph:  `<http://www.wikipathways.org>`


## ocrs

Open PHACTS Chemistry Registration Service.

New versions will be created by Valery.  Creation of the OCRS data depends on ChEMBL data and
other chemistry datasets (which ones?).

Put into RDF Graph:  `<http://ops.rsc.org>`


# Other Datasets

For all other Open PHACTS datasets, the current plan (as of 2017-02-26) is not to refresh them,
but to re-use the data used for version 2.1.
