---
layout: page
exclude: true
exclude_from_nav: true
---

# Tutorial: Loading additional data in Open PHACTS

> Q: How can I update RDF datasets in Open PHACT's RDF store?

We'll assume you have already [installed Open PHACTS](https://github.com/openphacts/ops-docker) using Docker, and that you have [Virtuoso](http://virtuoso.openlinksw.com/) running as the Docker container `ops-virtuoso`, exposed on http://localhost:3003/sparql (or equivalent hostname). Similarly you will have `ops-api` exposed on http://localhost:3002/ and `ops-ims` on http://localhost:3004/QueryExpander/

> **Warning**: In this tutorial we'll use an `ops-docker` installation on http://heater.cs.man.ac.uk:3002/ - please note that this service is only for demo purposes and might not remain available or in the same state. It should not be accessed for production use as it runs on low-spec hardware. 


The RDF data loaded in Open PHACTS come from [different sources](https://github.com/openphacts/ops-platform-setup/blob/2.0.0/data-sources/README.md) - some of which require manual download or transformation.

Each dataset is kept in a different [named graphs](https://github.com/openphacts/ops-platform-setup/blob/2.0.0/data-sources/README.md#how-many-triples) in Virtuoso, meaning we can query them separately. For instance, on http://localhost:3003/sparql try this query to count triples in the Uniprot graph:

```sparql
SELECT COUNT(*) WHERE {
    GRAPH <http://purl.uniprot.org> {
      ?s ?p ?o.
    }
}
```

| count?     |
| ---------- |
| 1131186434 |

The query might take some seconds to execute the first time if the Virtuoso server has recently been restarted.

## Overview

In short, updating a data source consists of:

1. Download/generate the updated RDF data
2. Drop the old GRAPH (if replacing)
3. Parse/load RDF into Virtuoso  (NOTE: this could take several hours or even days)
4. Download/generate any updated linksets
5. Load updated linksets to Identity Mapping Service
6. If the RDF has changed in structure (e.g. vocabulary changes), update affected queries in the API
7. Test that API calls that worked before still work as expected (allowing for actual data change)

As a running example this page uses [Uniprot](http://www.uniprot.org/), but the process will be similar for each of the data sources.


## What data source?

So let's say we want to update Uniprot, which we find in the [Open PHACTS 2.0 data sources](https://github.com/openphacts/ops-platform-setup/blob/2.0.0/data-sources/README.md#uniprot-on-20151116-release-2015_11) as being version `2015_11`, however [latest version](ftp://ftp.uniprot.org/pub/databases/uniprot/relnotes.txt) is `2016_07`.

Note that Uniprot is a special case for Open PHACTS, because although you can download all of the [Uniprot RDF](ftp://ftp.uniprot.org/pub/databases/uniprot/current_release/rdf/), that is 128 GB - compressed as `rdf.xz`, which mean it would require something like 2 TB just for loading. The queries used by Open PHACTS luckily only require a much smaller subset of this, about 7.9 GB as `rdf.gz`.

Still, just parsing may take many hours, so for demonstration purposes this tutorial downloads a small subset.

## Download data

The [Uniprot data source](https://github.com/openphacts/ops-platform-setup/blob/2.0.0/data-sources/README.md#uniprot-on-20151116-release-2015_11) is described as:

```
curl -d 'query=reviewed%3ayes&force=yes&format=rdf' http://www.uniprot.org/uniprot/ > swissprot_20151116.rdf.xml
curl -d 'query=reviewed%3ayes&force=yes&format=rdf' http://www.uniprot.org/uniparc/ > uniparc_20151116.rdf.xml
curl -d 'sort=&desc=&query=reviewed%3ayes&fil=&format=rdf&force=yes' http://www.uniprot.org/uniref/ > uniref_20151116.rdf.xml
```

That is three separate queries against [Uniprot API](http://www.uniprot.org/help/programmatic_access), which return the result as RDF based on the latest release.   So if we run those queries again today we should get the latest Uniprot data.

We can improve the download speed and disk space requirement to compress with `gzip` by adding `compress=true` and run all three downloads at once:

```bash
date=`date -I`  # e.g. 2016-07-27
curl -d 'query=reviewed%3ayes&force=yes&format=rdf&compress=true' http://www.uniprot.org/uniprot/ > swissprot_$date.rdf.gz &
curl -d 'query=reviewed%3ayes&force=yes&format=rdf&compress=true' http://www.uniprot.org/uniparc/ > uniparc_$date.rdf.gz &
curl -d 'query=reviewed%3ayes&force=yes&format=rdf&compress=true' http://www.uniprot.org/uniref/ > uniref_$date.rdf.gz &
wait
```

Note that executing the above query download can take multiple hours, so in this tutorial we will cheat a bit to make a small subset.  The Uniprot query can be accessed in the browser for each of those three sources:

* http://www.uniprot.org/uniprot/
* http://www.uniprot.org/uniparc/
* http://www.uniprot.org/uniref/

You may notice that all three queries filter crucially by `reviewed%3ayes` - that is `reviewed:yes` in the search box.

* http://www.uniprot.org/uniprot/?query=reviewed%3Ayes&sort=score
* http://www.uniprot.org/uniparc/?query=reviewed%3Ayes&sort=score
* http://www.uniprot.org/uniref/?query=reviewed%3Ayes&sort=score

This filters to only include the the Swiss-prot manually reviewed entries of Uniprot, which as you see drastically reduces the result size:

| Data source | Reviewed | Unreviewed | Total |
| ----------- | -------- | ---------- | ----- |
| [uniprot](http://www.uniprot.org/uniprot/) |  [551,705](http://www.uniprot.org/uniprot/?query=*&fil=reviewed%3Ayes) | [65,378,749 ](http://www.uniprot.org/uniprot/?query=*&fil=reviewed%3Ano) | [65,930,454](http://www.uniprot.org/uniprot/?query=*&fil=)
| [uniparc](http://www.uniprot.org/uniparc/) |  [503,157](http://www.uniprot.org/uniparc/?query=*&fil=reviewed%3Ayes) | [123,735,183](http://www.uniprot.org/uniparc/?query=*&fil=reviewed%3Ano) | [ 124,238,340](http://www.uniprot.org/uniparc/?query=*&fil=)
| [uniref](http://www.uniprot.org/uniref/) |  [952,465](http://www.uniprot.org/uniref/?query=*&fil=reviewed%3Ayes) | [143,351,546](http://www.uniprot.org/uniref/?query=*&fil=reviewed%3Ano) | [144,304,011](http://www.uniprot.org/uniref/?query=*&fil=)

For each of these queries you will see there's a **Download** button, from where you select:

* Download all (551705)
* Format: RDF/XML
* Compressed


For the purpose of this tutorial we will modify the query to download only the entries created in the last month. You can similarly modify the queries to only load a particular species, etc.

* http://www.uniprot.org/uniprot/?query=reviewed%3Ayes+AND+created%3A%5B20160601+TO+*%5D
* http://www.uniprot.org/uniparc/?query=reviewed%3Ayes+AND+created%3A%5B20160601+TO+*%5D
* http://www.uniprot.org/uniref/?query=reviewed%3Ayes+published%3A%5B20160601+TO+*%5D+count%3A%5B100+TO+*%5D+AND+identity%3A1.0


Note, as `uniref` is quite large (it contains the protein sequences), the above query is artificially limited to `reviewed:yes published:[20160601 TO *] count:[100 TO *] AND identity:1.0` resulting in just `2815` entries.

Now you should have three files:

```
-rw-rw-r--  1 stain stain  18K Jul 25 14:12 uniparc-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf.gz
-rw-rw-r--  1 stain stain 3.1M Jul 25 14:12 uniprot-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf.gz
-rw-rw-r--  1 stain stain  13M Jul 25 15:26 uniref-reviewed%3Ayes+published%3A%5B20160601+TO+-%5D+count%3A%5B100+TO+-%5D+AND--.rdf.gz
```

Don't let the small compressed sizes deceive you:

```
stain@biggiebuntu:~/Downloads/uniprot$ gunzip *

stain@biggiebuntu:~/Downloads/uniprot$ ls -alh
total 675M
-rw-rw-r-- 1 stain stain  78K Jul 25 14:12 uniparc-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf
-rw-rw-r-- 1 stain stain  23M Jul 25 14:12 uniprot-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf
-rw-rw-r-- 1 stain stain 213M Jul 25 15:26 uniref-reviewed%3Ayes+published%3A%5B20160601+TO+-%5D+count%3A%5B100+TO+-%5D+AND--.rdf
```


Note that you **do not need** to `gunzip` these files as Virtuoso can uncompress `.gz` directly (and newer versions even `.xz`).

Note that we used the filtering only for demonstration purposes, for instance the above do not include pre-existing entries that have since been modified, and also we only go one month back.  The full update with the [queries from the wiki page](https://github.com/openphacts/ops-platform-setup/blob/2.0.0/data-sources/README.md#uniprot-on-20151116-release-2015_11) should eventually end up with something like:

```
-rw-rw-r-- 4 stain stain 2.2G Nov 16  2015 swissprot_20151116.rdf.xml.gz
-rw-rw-r-- 4 stain stain 4.5G Nov 16  2015 uniparc_20151116.rdf.xml.gz
-rw-rw-r-- 4 stain stain 1.3G Nov 16  2015 uniref_20151116.rdf.xml.gz
```



## Dropping the old graph

If we are replacing an existing graph, then we would generally want to [DROP GRAPH](http://virtuoso.openlinksw.com/dataspace/doc/dav/wiki/Main/VirtTipsAndTricksDROPSilentGraph) to remove the old named graph and keep the other graphs which you are not replaced, as doing a full reload of all the sources from RDF can be quite time consuming.

Note that for large graphs as in Open PHACTS, dropping a graph in Virtuoso can also be time consuming (sometimes slower than loading the new graph!) and require a large amount of memory. The Virtuoso FAQ [How can I delete graphs containing large numbers of triples from the Virtuoso Quad Store?](http://virtuoso.openlinksw.com/dataspace/doc/dav/wiki/Main/VirtTipsAndTricksGuideDeleteLargeGraphs) recommends to do this outside a transaction, e.g. with `isql`:

```sql
log_enable(3,1);
SPARQL CLEAR GRAPH  <http://g1.example.com>;
```

For the largest graphs (e.g. chembl, uniprot, goa, surechembl) we recommend you do the `DELETE FROM rdf_quad` trick  to speed up:

```sql
log_enable(3,1);
DELETE FROM rdf_quad WHERE g = iri_to_id ('http://g1.example.com');
SPARQL CLEAR GRAPH <http://purl.uniprot.org>;
```


In this tutorial we will later update the Uniprot graph in-line and so we actually do NOT want to clear that graph, but we'll have a go dropping one of the other graphs as you would normally need to do that to update a single source completely.

Virtuoso is running in a separate _Docker container_, so we use `docker exec -it` to start a second `isql` process within that container:

```
stain@heater:~/ops-docker$ docker exec -it ops-virtuoso isql
OpenLink Interactive SQL (Virtuoso), version 0.9849b.
Type HELP; for help and EXIT; to exit.
SQL>
```

(Tip: We need `docker exec -it` to get a terminal binding for the interactive isql shell. Similarly `docker exec -it ops-virtuoso bash` would give you a bash shell)


So let's say we find the [bioassay datasource](https://github.com/openphacts/ops-platform-setup/blob/2.0.0/data-sources/README.md) to have graph `http://www.bioassayontology.org` (remember to strip the trailing `/` in Open PHACTS graph names).

We'll check first how much we'll be deleting (to be sure we got the right graph name):

```sparql
SQL> SPARQL SELECT COUNT(*) WHERE { GRAPH <http://www.bioassayontology.org> { ?s ?p ?o } };
Type the rest of statement, end with a semicolon (;)> ;
callret-0
INTEGER
_______________________________________________________________________________

10360
```

The number should roughly match the data sources table. Next we'll delete it:

```sparql
SQL> log_enable(3,1);
Connected to OpenLink Virtuoso
Driver: 07.20.3215 OpenLink Virtuoso ODBC Driver

Done. -- 1 msec.
SQL> SPARQL CLEAR GRAPH <http://www.bioassayontology.org>;

```

Afterwards that particular graph should be empty:

```sparql
SQL> SPARQL SELECT COUNT(*) WHERE { GRAPH <http://www.bioassayontology.org> { ?s ?p ?o } };
callret-0
INTEGER
_______________________________________________________________________________

0

1 Rows. -- 2 msec.
```

Note that `checkpoint;` is not needed here because we disabled transactions with `log_enable(3,1)`;

For demonstration purposes the above deletes one of the smaller graph, for a graph like Uniprot this could actually take an hour.


## Staging the data

Now we need to move the data to be available from within Virtuoso's Docker container. If you have downloaded the files on a different computer, you need to first transfer them to your Open PHACTS server:

```shell
    cd ~/Downloads
    mkdir uniprot    
    ls -al uni*gz  # Should just be 3 files made just now

    ssh heater.cs.man.ac.uk mkdir -p data
    scp -r uniprot/ heater.cs.man.ac.uk:data/
```

The easiest way to load data into Virtuoso with Docker is to use the [staging script](https://github.com/stain/virtuoso-docker#staging) - which automates using Virtuoso [RDF Bulk loading](http://virtuoso.openlinksw.com/dataspace/doc/dav/wiki/Main/VirtBulkRDFLoader) feature (`ld_dir` and `rdf_loader_run`).

This script assumes the data to be loaded is in the [Docker volume](https://docs.docker.com/engine/tutorials/dockervolumes/) `/staging` (mapped from the local file system), and that the Virtuoso store is in `/virtuoso`. A file `staging.sql` should be in the top-level of the mapped `/staging` folder.

So in our case the data to load is already in `/home/stain/data` in the `uniprot` subfolder:

```
stain@heater:~/data$ ls -al uniprot/
total 51204
drwxrwxr-x 2 stain stain     4096 Jul 25 14:36 .
drwxrwxr-x 3 stain stain     4096 Jul 25 14:37 ..
-rw-rw-r-- 1 stain stain    17547 Jul 25 14:36 uniparc-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf.gz
-rw-rw-r-- 1 stain stain  3156208 Jul 25 14:36 uniprot-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf.gz
-rw-rw-r-- 1 stain stain 49242276 Jul 25 14:36 uniref-reviewed%3Ayes+AND+published%3A%5B20160706+TO+-%5D.rdf.gz
```

We'll create our `data/staging.sql` file by modifying the relevant lines from
[ops-docker's staging.sql](https://github.com/openphacts/ops-docker/blob/master/openphacts-rdf/staging.sql)

```shell
stain@heater:~/data$ vi staging.sql
```

```sql
ld_dir('/staging/uniprot' , '*.rdf.gz' , 'http://purl.uniprot.org' );
```

Note that we changed the filename pattern `'*.nq.gz'` (which is used within the https://data.openphacts.org/ downloads) to ` '*.rdf.gz'` to match our filenames.  (If your data is not compressed, don't include `.gz` here)

Remember that the first parameter is the path within the Docker container, and must start with `/staging`, as our `/home/stain/data` folder will appear as `/staging` within the container.

**Tip**: Be careful about filename case as Linux is case-sensitive - keeping it lowercase is easiest.

**Note:** note that most of the graph names used in Open PHACTS do NOT include the trailing `/`.

### Parsing the RDF

Now that our data is ready to be loaded by Virtuoso, we are going to:

* Shut down `ops-virtuoso`
* Run the bulk loading script from our mapped `/staging`
* Restart `ops-virtuoso`
* Inspect by SPARQ to verify the load was complete
* Verify the API still works

**Important**: You **must shutdown** the running `ops-virtuoso` instance before we run the staging script. The reason is that the script runs in a separate Docker container and needs full exclusive access to its `/virtuoso`.

```
stain@heater:~/data$ sudo docker stop ops-virtuoso
```

Next we'll start the bulk loading [staging script](https://github.com/stain/virtuoso-docker#staging), using the `ops-virtuosodata` data volume, in addition to the volume `/home/stain/data` mapped as `/staging` (read-only).

```
stain@heater:~/data$ docker run -v /home/stain/data:/staging:ro --volumes-from ops-virtuosodata -it stain/virtuoso staging.sh
 * Starting Virtuoso Open Source Edition 7.2  virtuoso-opensource-7                                                                                                                                                                                                                       
```

Note that starting Virtuoso might take some time before you should see:

```
Configuring SPARQL
Populating from /staging/staging.sql
Starting 7 rdf_loader_runs
Starting RDF loader 1
Starting RDF loader 2
Starting RDF loader 3
Starting RDF loader 4
Starting RDF loader 5
Starting RDF loader 6
Starting RDF loader 7
Starting RDF loader 8
```

The number of loader threads depend on the number of CPU cores, which generally significantly speed up loading, but in this case there's only 3 files, so in effect only three loader threads will be active.   If you are preparing another RDF data source for loading with Virtuoso, then it's generally advisable to have many smaller RDF files (e.g. 10 MB each) than a single large file.


While this is loading, we can peek inside to have a look at what's going on in a second window:

```
stain@heater:~$ docker ps
CONTAINER ID        IMAGE                                 COMMAND                  CREATED             STATUS              PORTS                              NAMES
d3587a04eba5        stain/virtuoso                        "/docker-entrypoint.s"   4 minutes ago       Up 4 minutes        1111/tcp, 8890/tcp                 modest_lichterman

root@d3587a04eba5:/virtuoso# ls -alh
total 111G
drwxr-xr-x  2 root root  36K Jul 25 13:49 .
drwxr-xr-x 50 root root 4.0K Jul 25 13:49 ..
-rw-r--r--  1 root root    0 Jun 16 15:13 .loaded
-rw-r--r--  1 root root 6.0M Jul 25 13:47 virtuoso-temp.db
-rw-r--r--  1 root root 111G Jul 25 13:49 virtuoso.db
-rw-r--r--  1 root root   12 Jul 25 13:47 virtuoso.lck
-rw-r--r--  1 root root  69K Jul 25 13:52 virtuoso.log
-rw-r--r--  1 root root    0 Jul  4 12:59 virtuoso.pxa
-rw-r--r--  1 root root 1022 Jul 25 13:49 virtuoso.trx
```

We'll inspect the loader status in the [DB.DBA.load_list](http://virtuoso.openlinksw.com/dataspace/doc/dav/wiki/Main/VirtBulkRDFLoader) table:

```
root@d3587a04eba5:~# isql
OpenLink Interactive SQL (Virtuoso), version 0.9849b.
Type HELP; for help and EXIT; to exit.

SQL> select * from DB.DBA.load_list WHERE ll_file LIKE '/staging/uniprot/u%';
ll_file                                                                           ll_graph                                                                          ll_state    ll_started           ll_done              ll_host     ll_work_time  ll_error
VARCHAR NOT NULL                                                                  VARCHAR                                                                           INTEGER     TIMESTAMP            TIMESTAMP            INTEGER     INTEGER     VARCHAR
_______________________________________________________________________________

/staging/uniprot/uniparc-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf.gz  http://purl.uniprot.org                                                           2           2016.7.25 13:49.46 802045000  2016.7.25 13:50.45 238954000  0           NULL        NULL
/staging/uniprot/uniprot-reviewed%3Ayes+AND+created%3A%5B20160601+TO+-%5D.rdf.gz  http://purl.uniprot.org                                                           1           2016.7.25 13:49.46 802045000  NULL                 0           NULL        NULL
/staging/uniprot/uniref-reviewed%3Ayes+AND+published%3A%5B20160706+TO+-%5D.rdf.gz  http://purl.uniprot.org                                                           1           2016.7.25 13:49.46 802591000  NULL                 0           NULL        NULL

3 Rows. -- 5 msec.

```

> The `ll_state` field can have three values:
> * `0` indicating the data set is to be loaded
> * `1` the data set load is in progress
> * `2` the data set load is complete

_adapted from http://virtuoso.openlinksw.com/dataspace/doc/dav/wiki/Main/VirtBulkRDFLoader_

This table is interesting if you have many larger files, as it shows how far you got, however not how far Virtuoso got by loading a particular file.

If you are bored loading a large set, you can have a look at the triple count, but remember this can also take some time to calculate:

```
SQL> SPARQL SELECT COUNT (*) WHERE { GRAPH <http://purl.uniprot.org> { ?s ?p ?o . } } ;
_______________________________________________________________________________
1131508309
```

(`321875` more than what we started with)

Note that when data loading finishes, the `staging.sh` script will force a `commit`, then shutdown the Virtuoso server, meaning your `isql` shell will die:

```
Checkpointing
Staging finished, total triples: 2934990243
 * Stopping Virtuoso Open Source Edition 7.2 virtuoso-opensource-7   
```

(Note that _total triples_ refers to triples across all graphs, including pre-existing data)


# Checking the loaded data

Now we that staging has completed we can restart `ops-virtuoso`:

```
stain@heater:~/data$ sudo docker restart ops-virtuoso
```

We'll check the logs to ensure Virtuoso has started again for normal operations:

```
stain@heater:~/data$ docker logs --follow --tail=20  ops-virtuoso
14:22:54   SUCCESS plugin 3: loaded from /usr/lib/virtuoso-opensource-7/hosting/creolewiki.so }
14:22:54 { Loading plugin 4: Type `plain', file `im' in `/usr/lib/virtuoso-opensource-7/hosting'
14:22:54   IM version 0.6 from OpenLink Software
14:22:54   Support functions for Image Magick 6.7.7
14:22:54   SUCCESS plugin 4: loaded from /usr/lib/virtuoso-opensource-7/hosting/im.so }
14:22:54 OpenLink Virtuoso Universal Server
14:22:54 Version 07.20.3215-pthreads for Linux as of Feb 11 2016
14:22:54 uses parts of OpenSSL, PCRE, Html Tidy
14:22:56 Database version 3126
14:22:56 SQL Optimizer enabled (max 1000 layouts)
14:22:57 Compiler unit is timed at 0.000194 msec
14:23:38 Roll forward started
14:24:13     88 transactions, 440200 bytes replayed (100 %)
14:24:13 Roll forward complete
14:24:22 PL LOG: Can't get list of vad packages in /usr/share/virtuoso-opensource-7/vad/
14:24:24 Checkpoint started
14:24:27 Checkpoint finished, log reused
14:24:28 HTTP/WebDAV server online at 8890
14:24:28 Server online at 1111 (pid 1)
```


Once you see `Server online` the startup is complete, press `Ctrl-C` to abort `docker logs`

Now the equivalent of http://heater.cs.man.ac.uk:3003/sparql should be back online, and we can repeat the `COUNT` query:

```sparql
SELECT COUNT(*) WHERE {
    GRAPH <http://purl.uniprot.org> {
      ?s ?p ?o.
    }
}
```

| count?     |
| ---------- |
| 1132342349 |

You did write down the old number, right?  Now the number should be slightly larger, e.g. in our case we have added `1155915` triples (just 0.10% of Uniprot)

Now as this time we added only new entries, we should be able to query for them. In our [filtered Uniprot search](http://www.uniprot.org/uniprot/?query=reviewed%3Ayes+AND+created%3A%5B20160601+TO+*%5D) we find for instance the newcomer http://www.uniprot.org/uniprot/E9PX95, so let's ask about that.

Note the above URL from the browser using `www.uniprot.org` must be rewritten to `purl.uniprot.org` to match the RDF identifiers instead of the web URIs - this mapping is normally done by the Identity Mapping Service.

As we are not sure, we inspect the RDF files manually as [N-Triples](https://www.w3.org/TR/n-triples/) using [Apache Jena riot](https://jena.apache.org/documentation/io/#command-line-tools) to avoid looking directly at RDF/XML without protective eyewear.

```
stain@heater:~/data/uniprot$ docker run --volume /home/stain/data/uniprot:/rdf  stain/jena riot uniref*rdf.gz | head -n 200 | tail
```

```turtle
<http://purl.uniprot.org/uniparc/UPI0000124E24> <http://purl.uniprot.org/core/sequenceFor> <http://purl.uniprot.org/uniprot/Q53Z42> .
<http://purl.uniprot.org/uniparc/UPI0000124E24> <http://purl.uniprot.org/core/memberOf> <http://purl.uniprot.org/uniref/UniRef90_P01892> .
<http://purl.uniprot.org/uniparc/UPI0000124E24> <http://purl.uniprot.org/core/memberOf> <http://purl.uniprot.org/uniref/UniRef50_P01892> .
<http://purl.uniprot.org/uniprot/P01892> <http://www.w3.org/2000/01/rdf-schema#label> "HLA class I histocompatibility antigen, A-2 alpha chain" .
<http://purl.uniprot.org/uniprot/P01892> <http://purl.uniprot.org/core/representativeFor> <http://purl.uniprot.org/uniref/UniRef100_P01892> .
<http://purl.uniprot.org/uniprot/P01892> <http://purl.uniprot.org/core/seedFor> <http://purl.uniprot.org/uniref/UniRef100_P01892> .
<http://purl.uniprot.org/uniprot/P01892> <http://purl.uniprot.org/core/reviewed> "true"^^<http://www.w3.org/2001/XMLSchema#boolean> .
<http://purl.uniprot.org/uniprot/P01892> <http://purl.uniprot.org/core/mnemonic> "1A02_HUMAN" .
<http://purl.uniprot.org/uniprot/P01892> <http://purl.uniprot.org/core/organism> <http://purl.uniprot.org/taxonomy/9606> .
<http://purl.uniprot.org/uniprot/Q53Z42> <http://www.w3.org/2000/01/rdf-schema#label> "HLA class I antigen" .
```

(Notice the use of `head` and `tail` above to avoid parsing the whole file)

So let's go a head and ask our SPARQL endpoint on the equivalent of http://heater.cs.man.ac.uk:3003/sparql


```sparql
DESCRIBE <http://purl.uniprot.org/uniprot/E9PX95>
```

Which seems to return a lot of (new) data, in a slightly confusing RDF Turtle syntax:

```turtle
@prefix rdf:	<http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix ns1:	<http://purl.uniprot.org/uniprot/> .
@prefix ns2:	<http://purl.uniprot.org/core/> .
ns1:E9PX95	rdf:type	ns2:Protein .
@prefix rdfs:	<http://www.w3.org/2000/01/rdf-schema#> .
ns1:E9PX95	rdfs:label	"ATP-binding cassette sub-family A member 17" .
@prefix ns4:	<http://purl.uniprot.org/pfam/> .
ns1:E9PX95	rdfs:seeAlso	ns4:PF00005 .
@prefix ns5:	<http://purl.uniprot.org/paxdb/> .
ns1:E9PX95	rdfs:seeAlso	ns5:Q4H4D7 ,
		<http://purl.uniprot.org/unigene/Mm.328842> .
@prefix ns6:	<http://purl.uniprot.org/ensembl/> .
ns1:E9PX95	rdfs:seeAlso	ns6:ENSMUST00000039324 .
@prefix ns7:	<http://purl.uniprot.org/panther/> .
ns1:E9PX95	rdfs:seeAlso	ns7:PTHR19229 .
@prefix ns8:	<http://purl.uniprot.org/interpro/> .
ns1:E9PX95	rdfs:seeAlso	ns8:IPR026082 ,
		ns6:ENSMUST00000121226 ,
		<http://purl.uniprot.org/gene3d/3.40.50.300> ,
		ns8:IPR027417 .
@prefix ns9:	<http://purl.uniprot.org/eggnog/> .
ns1:E9PX95	rdfs:seeAlso	ns9:COG1131 ,
		<http://purl.uniprot.org/refseq/NP_001026792.2> .
@prefix ns10:	<http://purl.uniprot.org/genetree/> .
ns1:E9PX95	rdfs:seeAlso	ns10:ENSGT00760000118965 ,
		ns9:KOG0059 ,
		<http://purl.uniprot.org/ucsc/uc008avk.1> .
@prefix ns11:	<http://purl.uniprot.org/supfam/> .
ns1:E9PX95	rdfs:seeAlso	ns11:SSF52540 ,
		<http://purl.uniprot.org/embl-cds/BAD97416.1> .
@prefix ns12:	<http://purl.uniprot.org/ko/> .
ns1:E9PX95	rdfs:seeAlso	ns12:K05643 ,
		<http://purl.uniprot.org/geneid/381072> .
@prefix ns13:	<http://purl.uniprot.org/prosite/> .
ns1:E9PX95	rdfs:seeAlso	ns13:PS00211 .
@prefix ns14:	<http://purl.uniprot.org/oma/> .
ns1:E9PX95	rdfs:seeAlso	ns14:FICVELI ,
		<http://purl.uniprot.org/string/10090.ENSMUSP00000046218> ,
		ns13:PS50893 .
@prefix ns15:	<http://purl.uniprot.org/smart/> .
ns1:E9PX95	rdfs:seeAlso	ns15:SM00382 .
@prefix ns16:	<http://purl.uniprot.org/treefam/> .
ns1:E9PX95	rdfs:seeAlso	ns16:TF105191 .
@prefix ns17:	<http://purl.uniprot.org/orthodb/> .
ns1:E9PX95	rdfs:seeAlso	ns17:EOG7RNJZ8 .
@prefix ns18:	<http://purl.uniprot.org/hogenom/> .
ns1:E9PX95	rdfs:seeAlso	ns18:HOG000006613 .
@prefix ns19:	<http://purl.uniprot.org/hovergen/> .
ns1:E9PX95	rdfs:seeAlso	ns19:HBG050435 ,
		<http://purl.uniprot.org/mgi/3625331> ,
		<http://purl.uniprot.org/ctd/381072> ,
		ns8:IPR017871 ,
		<http://purl.uniprot.org/kegg/mmu:381072> .
@prefix ns20:	<#> .
ns1:E9PX95	rdfs:seeAlso	ns20:_45395058393500E ,
		<http://purl.uniprot.org/refseq/XP_006524540.1> ,
		<http://purl.uniprot.org/refseq/XP_006524541.1> ,
		<http://purl.uniprot.org/refseq/XP_006524538.1> ,
		<http://purl.uniprot.org/refseq/XP_006524539.1> ,
		<http://purl.uniprot.org/refseq/XP_006524542.1> ,
		ns8:IPR003593 ,
		ns8:IPR003439 ;
	ns2:replaces	ns1:Q4H4D7 ;
	ns2:annotation	<http://purl.uniprot.org/SHA-384/1EF01A6A442E9595B3253D464C35B1E5A308CD965F00365ECBE549FF14108CDEFF5EE33C3466F6EEC8B260BCC4BC1F26> ,
		<http://purl.uniprot.org/SHA-384/8AC5DB683B03113DDFEDC528EC7CB07491CEBFFA1FD936CC5A1BD5C523AEB42AF2EEBC204E71E119E4603FF71EB25398> ,
		<http://purl.uniprot.org/SHA-384/6E85C69C8372CC5D5705961B42357C74A5080422982E2E5D74941D52D3CE44536E85176A9263F909557E197F65F6CE67> ,
		<http://purl.uniprot.org/SHA-384/8A2648D8D1E802D380815784832E2639EA8E91E91EA323BD1B35F9B2707A6F3E57AE0D75812C225C4B47ADF0B87BE8FE> .
@prefix ns21:	<http://purl.uniprot.org/SHA-384/> .
ns1:E9PX95	ns2:annotation	ns21:FBDFE044E4F64AB6FEB1F4C8C5B751E3396F4D2A0F8B260D4332989932A07CE1ADC9F6BF51278EC26974EB29095CEEB7 ,
		<http://purl.uniprot.org/SHA-384/9B37DE38DA91F7E22E506E081D97CAE7A7BBD676C32E39A830F313E1C9EE3CA356DB2137621DF8D70EC8AB0EC388E546> ,
		ns21:B04E2A55A2961641EB5B02A3FDC0F38B5BBD393E53AD1B6018861E13A095D7642BA052397637CB897468D193D6CCA6A9 ,
		ns21:AEFE0F039262F337FF92A1B514DE1B4E30BFB4561CC893A75BDAB8AEBD8BFEB03C64D3BB599BA2CF124B485A068307FB ,
		<http://purl.uniprot.org/SHA-384/0D7A4F5DAF5F3E6919DEA5A2E50369D75F42C25922F011B919B0B77C9A9810853CCEB07465495FB817913D5337BBE6E9> ,
		<http://purl.uniprot.org/SHA-384/99159609F2CF54CCFC3A07FCAA47B5909195FC3E8C64ED469463B0120FD54288405007E88AE03045EE0F4F3019AC3D66> ,
		ns21:FAC1A58A17EBE51847B3E99E778C4F0E4C19F163209B39362D2D18123175A6116F1AD274598AD23CBE0F9479C31D7965 ,
		<http://purl.uniprot.org/SHA-384/0CED70F30FBD6C24022F6F4C92008010A55D685FA25B5541384C70C7F021C445D3BA6DDC3657498CDA7DBC2611D7D75E> ,
		<http://purl.uniprot.org/SHA-384/2B5F5D51B4AE9BD67B6EFE97C0A5BB67D3FCAA8C94990A5B3D89997C0917FA3E62D28460D5D9AA7787C3A34A0DA88CDC> ,
		<http://purl.uniprot.org/SHA-384/8DDC8F5ECAF867556C38B3C8C38A04FE69FB4029DA10E28D74DD2561B40125BE90036600AA6FFFAD9517A7CE574F93EC> ,
		<http://purl.uniprot.org/SHA-384/01A606883E27E085EA43FC8BF27534B5AFC1DBC633AB3C3A7236A6F0BC8EECC47C9307125776ECAB948B93256EEBEF32> ,
		<http://purl.uniprot.org/SHA-384/71E1FF1916540B06F83136968E03D9FA4A550C1D57A880E20CDC96C9D6F7B7ACBB78268C16910BB0CFE3383B23D9B644> ,
		<http://purl.uniprot.org/SHA-384/004470679AAABD8BC28CB43F7713BCA526C86F83C01B02360838DE6E55FB4796422472263790851E68EDFDD6F0FE5E28> ,
		<http://purl.uniprot.org/SHA-384/1173E0298228525CDDCC8442691EC5F2C72A565C935361B3EA2CECF4AD8226E504A626ABE31F65423521B9AF425D3C20> ,
		<http://purl.uniprot.org/SHA-384/280E844129D081124EC8256456EE4EF895BD0FDA38AC75CA36444A8CFAC0F6A610D4051EF806F287DB26B910DA6436BA> ,
		<http://purl.uniprot.org/SHA-384/65A16701EC757C6C46202EA9ACE9FD9BF56BAA2AAF78FC0108200F34C1A68A9848B1D69BACD01680CF66EFBD36694001> .
@prefix ns22:	<http://purl.uniprot.org/annotation/> .
ns1:E9PX95	ns2:annotation	ns22:PRO_0000436476 ,
		<http://purl.uniprot.org/SHA-384/073E4E53E70B58BE582F5F709D3ED2EBB379C2FBC8227E55F4018725637646394B4FF412C99F9CE989F3B9DEAAD0321C> ,
		<http://purl.uniprot.org/SHA-384/7664062C3FD4DE33DFAAC9D5F8CADBA60149B0336F56366A6B7D4C45E8647E542D7B279E6E493C3431D438CF91045F8B> ,
		<http://purl.uniprot.org/SHA-384/4FB358D64731FEACEB66925BF25E1C9638A853CE4DC9BBAD779191D132D03977363334FAD640921925E21A5164FF763A> ,
		<http://purl.uniprot.org/SHA-384/62C7D9BE775C586A8838ABF643949A973C85673FF035E20284927BE23DF81EE1D6B6A5EC0D66544366DBA7F63EEA3C59> ,
		<http://purl.uniprot.org/SHA-384/4358EBCFFCFC0FBF4E6CF4F979A5A5CD0D4D9B5AB404F65A2A201A34900536DDAEE639DA594E6787A251B9B371523AF0> ,
		<http://purl.uniprot.org/SHA-384/82863BAB25BEBA565BEF25A5E348DF9DC5A982A5E320947EBBB699A6F437C95858DB3E495EB414CCA3ABE4253540BE0F> ,
		<http://purl.uniprot.org/SHA-384/92DAF86165D76B7F0E1CF94E2E92E1CE33A5E683393038D65BF9F3A9C340E1217888DAC545D3E9960795688200C6953D> ,
		<http://purl.uniprot.org/SHA-384/04932E9BD714A120725C5576BFE1AD346705BC1C5BD5C5F2134245CA086E05DD991A5BCDF85339114DA2E9FB5D461C12> ,
		<http://purl.uniprot.org/SHA-384/1B39897A268868F18689A1088C8146658CCE43ECAF62A026A5F52972CFB966494ADB0948F9DF1F32D934B12306B27E43> ,
		<http://purl.uniprot.org/SHA-384/3092A4B26B4B8F532F79825F89BAC2D6E523A7ED4BD7CF28E492711CD0AF9897756B0230F223D46716E3FA6E51A70895> ,
		ns21:CEFB3BCB86FFC6478DB3F354544552FB50A322FDC6C17C4EACBB6809A66D379E1BD77D3D57F00F4CE12E9C73EA8A495D ,
		<http://purl.uniprot.org/SHA-384/91C6F1ED8F46769EC05CE21058F9ADFBDDA317A9345BD8E6C974C10F4B216039474CF33C66A229AC912E3E59F1D81947> ,
		ns21:ADDEC2EEB89B3187A0AFE96939491721D17131C509DC384CD7EBFEE4C0B50034F5219DEE0577804A1986FC73993A41C4 ,
		ns21:E07E9BD306C58D567AE23DB96F60703D2A30F2A20EA2CB73DBCF411EFDE1621737444AF820FD8F146ED6BC5C39934C94 ,
		<http://purl.uniprot.org/SHA-384/9806948C467C3AB7A827AA78527D408647DABBBDAF016A3DB0DF13CB1D9A21B42C30BB4095AE5E5326D10C75FB30999B> ,
		<http://purl.uniprot.org/SHA-384/205FE99ECCFEAFF8EB465F24D3A743E81017C0B20AADF51BF29565BAB0C98BE311171D2096E28F257932C43012810EE1> ,
		ns21:C031404D95B96975DD6B3426762B4D576E3831693143F8C3B89218EE1D0C177430A0D532D238024618B96D4A17437D51 ,
		ns21:ABC429598016C103E25EAF3C7FC7D08C215652F0CBF1C2519D94EA109A23FCC0E3183BFCDA23C3B8DCDF0E2D7C792EEF ,
		<http://purl.uniprot.org/SHA-384/96CB8367EFD0DA607B806B5E51A8DC681C80C8572E50C0F6619F082500BE7AF6855CD0CF61C06A9433545518F5CBAC29> ;
	ns2:attribution	ns20:_4539505839350019 ,
		ns20:_453950583935009C ,
		ns20:_453950583935009A ,
		ns20:_4539505839350098 ,
		ns20:_453950583935006F ,
		ns20:_453950583935004 ,
		ns20:_4539505839350034 ,
		ns20:_4539505839350031 ,
		ns20:_4539505839350021 ,
		ns20:_4539505839350020 ,
		ns20:_4539505839350026 ,
		ns20:_45395058393500A2 ,
		ns20:_45395058393500A0 ,
		ns20:_45395058393500A ,
		ns20:_453950583935009E ,
		ns20:_45395058393500A4 ;
	ns2:citation	<http://purl.uniprot.org/citations/19468303> ,
		<http://purl.uniprot.org/citations/22237709> ,
		<http://purl.uniprot.org/citations/21183079> ,
		<http://purl.uniprot.org/citations/15810880> ;
	ns2:classifiedWith	<http://purl.uniprot.org/keywords/256> .
@prefix ns23:	<http://purl.obolibrary.org/obo/> .
ns1:E9PX95	ns2:classifiedWith	ns23:GO_0005524 ,
		<http://purl.uniprot.org/keywords/443> ,
		ns23:GO_0005783 ,
		ns23:GO_0005789 ,
		ns23:GO_0006638 ,
		<http://purl.uniprot.org/keywords/67> ,
		<http://purl.uniprot.org/keywords/677> ,
		<http://purl.uniprot.org/keywords/445> ,
		<http://purl.uniprot.org/keywords/325> ,
		<http://purl.uniprot.org/keywords/1133> ,
		<http://purl.uniprot.org/keywords/1185> ,
		ns23:GO_0016021 ,
		<http://purl.uniprot.org/keywords/963> ,
		ns23:GO_0042626 ,
		ns23:GO_0006869 ;
	ns2:conflictingSequence	ns21:D730E6F2413EC2F62297D8E5F6C0BB29DED8A8DA907D804BF35A8EE2A9401D213627728F03BD127825A93C46286565B1 .
@prefix xsd:	<http://www.w3.org/2001/XMLSchema#> .
ns1:E9PX95	ns2:created	"2016-06-08"^^xsd:date ;
	ns2:encodedBy	ns20:_453950583935001C ;
	ns2:existence	ns2:Evidence_at_Protein_Level_Existence ;
	ns2:isolatedFrom	<http://purl.uniprot.org/tissues/1030> ;
	ns2:mnemonic	"ABCAH_MOUSE" ;
	ns2:modified	"2016-07-06"^^xsd:date ;
	ns2:organism	<http://purl.uniprot.org/taxonomy/10090> ;
	ns2:proteome	<http://purl.uniprot.org/proteomes/UP000000589#Chromosome%2017> ;
	ns2:recommendedName	ns21:FE0BBEED5918A288AB704738089BC2BA1E23AA084909CA65009B3763FB14561C5FB038863522AC41245314F9E9059E75 ;
	ns2:reviewed	"true"^^xsd:boolean .
@prefix ns25:	<http://purl.uniprot.org/isoforms/> .
ns1:E9PX95	ns2:sequence	ns25:E9PX95-1 ;
	ns2:version	"49"^^xsd:int .
@prefix ns26:	<http://purl.uniprot.org/uniref/> .
ns1:E9PX95	ns2:representativeFor	ns26:UniRef100_E9PX95 ;
	ns2:seedFor	ns26:UniRef100_E9PX95 .
ns20:_4539505531370017	ns2:source	ns1:E9PX95 .
ns20:_453950583935001	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350010	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350011	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350012	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350013	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350014	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350015	rdf:subject	ns1:E9PX95 .
ns20:_453950583935001A	rdf:subject	ns1:E9PX95 .
ns20:_453950583935001B	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350037	rdf:subject	ns1:E9PX95 .
ns20:_453950583935003A	rdf:subject	ns1:E9PX95 .
ns20:_453950583935003D	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350040	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350043	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350046	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350049	rdf:subject	ns1:E9PX95 .
ns20:_453950583935004C	rdf:subject	ns1:E9PX95 .
ns20:_453950583935004F	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350052	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350055	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350058	rdf:subject	ns1:E9PX95 .
ns20:_453950583935005B	rdf:subject	ns1:E9PX95 .
ns20:_453950583935005E	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350061	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350064	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350067	rdf:subject	ns1:E9PX95 .
ns20:_453950583935006A	rdf:subject	ns1:E9PX95 .
ns20:_453950583935006D	rdf:subject	ns1:E9PX95 .
ns20:_453950583935007	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350071	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350074	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350077	rdf:subject	ns1:E9PX95 .
ns20:_453950583935007A	rdf:subject	ns1:E9PX95 .
ns20:_453950583935007D	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350080	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350083	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350086	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350089	rdf:subject	ns1:E9PX95 .
ns20:_453950583935008C	rdf:subject	ns1:E9PX95 .
ns20:_453950583935008F	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350092	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350095	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350097	rdf:subject	ns1:E9PX95 .
ns20:_4539505839350099	rdf:subject	ns1:E9PX95 .
ns20:_453950583935009B	rdf:subject	ns1:E9PX95 .
ns20:_453950583935009D	rdf:subject	ns1:E9PX95 .
ns20:_453950583935009F	rdf:subject	ns1:E9PX95 .
ns20:_45395058393500A1	rdf:subject	ns1:E9PX95 .
ns20:_45395058393500A3	rdf:subject	ns1:E9PX95 .
ns20:_45395058393500B	rdf:subject	ns1:E9PX95 .
ns20:_45395058393500D	rdf:subject	ns1:E9PX95 .
ns20:_45395058393500F	rdf:subject	ns1:E9PX95 .
@prefix ns27:	<http://purl.uniprot.org/uniparc/> .
ns27:UPI0000607A3F	ns2:sequenceFor	ns1:E9PX95 .
```

## Testing the API

So now that we know the new URIs are loaded, let's see if the API can recognize it.  Just in case you have already tried these in the API before, let's first flush the API's `memcache`:

```
stain@heater:~/data/uniprot$ docker restart ops-memcached
ops-memcached
```

Now we think the data loaded should be targets, so let's ask the official [Open PHACTS `/target` API](https://dev.openphacts.org/docs/2.1) first:

https://beta.openphacts.org/2.1/target?uri=http%3A%2F%2Fpurl.uniprot.org%2Funiprot%2FE9PX95&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_format=xml

> Page Not Found


So we'll rewrite that URL to use our new install:

http://heater.cs.man.ac.uk:3002/target?uri=http%3A%2F%2Fpurl.uniprot.org%2Funiprot%2FE9PX95&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_format=xml

The first time you run the query it might be slow, but it should eventually return:

```xml
<result format="linked-data-api" version="2.0" href="http://heater.cs.man.ac.uk:3002/target?uri=http%3A%2F%2Fpurl.uniprot.org%2Funiprot%2FE9PX95&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c">
<primaryTopic href="http://purl.uniprot.org/uniprot/E9PX95">
<exactMatch href="http://purl.uniprot.org/uniprot/E9PX95"/>
<molecularWeight datatype="int">196020</molecularWeight>
<inDataset href="http://purl.uniprot.org"/>
<sequence>
MEVLKKLKLLLWKNFILKRRKTLITLLEMLMPLLFCAIVLYLRLNSMPRKKSSTNYPAVDVSLLPVYFYNYPLKSKFQLAYIPSKSETLKAVTEVVEQTFAVDFEVLGFPSVPLFEDYIIKDPKSFYILVGIIFHHDFNSSNEPLPLVVKYDLRFSYVQRNFVSPPRHLFFQEEIEGWCTAFLYPPNLSQAPREFSYADGGNPGYNKEGFLAIQHAVDKAIMRHHAPKAALNMFKDLHVLVQRFPFGPHIQDPFLVILQNEFPLLLMLSFICVELIITNSVLSEKERKQKEYMSMMGVESWLHWVAWFITFFISVSITVSVMTVLFCTKINRVAVFRNSNPTLIFIFLMCFAIATIFFAFMMSTFFQRAHVGTVIGGTVFFFTYLPYMYITFSYHQRTYTQKILSCLFSNVAMATGVRFISLFEAEGTGIQWRNIGSVWGDFSFAQVLGMLLLDSFLYCLIAFLVESLFPRKFGIPKSWYIFAKKPVPEIPPLLNIGDPEKPSKGNFMQDEPTNQMNTIEIQHLYKVFYSGRSKRTAIRDLSMNLYKGQVTVLLGHNGAGKTTVCSVLTGLITPSKGHAYIHGCEISKDMVQIRKSLGWCPQHDILFDNFTVTDHLYFYGQLKGLSPQDCHEQTQEMLHLLGLKDKWNSRSKFLSGGMKRKLSIGIALIAGSKVLILDEPTSGLDSPSRRAIWDLLQQQKGDRTVLLTTHFMDEADLLGDRIAILAKGELQCCGSPSFLKQKYGAGYYMTIIKTPLCDTSKLSEVIYHHIPNAVLESNIGEEMIVTLPKKTIHRFEALFNDLELRQTELGISTFATSVTTMEEVFIRVCKLADPSTNVLTEKRHSLHPLPRHHRVPVDRIKCLHSGTFPVSTEQPMRLNTGFCLLCQQFYAMLLKKITYSRRNWMLVLSVQVLLPLAIIMLSLTFFNFKLRKLDNVPLELTLQTYGQTIVPFFIAENSHLDPQLSDDFVKMLVAAGQVPLRIQGSVEDFLLKKAKEAPEGFDKLYVVAASFEDVNNHTTVKALFNNQAYHSPSLALTLVDNLLFKLLSGANASITTTNYPQPQTAIEVSESILYQGPKGHYLVVNFLFGIAFLSSSFSILTVGEKSVKSKSLQFVSGVSTAVFWLSALLWDLISFLVPTLLLVLVFLWYKEEAFAHHESIPAVVLIMMLYGWAVIPLVYTVSFSFNTPGSACVKLVVMLTFLSISPVVLVTVTSEKDLGYTELSDSLDHIFLILPGHCLGMALSNLYYNFELKKFCSAKNLSDIDCNDVLEGYVVQENIYAWESLGIGKYLTALAVLGPVYITMLFLTEANAFYVLKSRLSGFFPSFWKEKSGMIFDVAEPEDEDVLEEAETIKRYLETLVKKNPLVVKEVSKVYKDKVPLLAVNKVSFVVKEEECFGLLGLNGAGKTSIFNMLTSEQPITSGDAFVKGFNIKSDIAKVRQWIGYCPEFDALLNFMTGREMLVMYARIRGIPECHIKACVDLILENLLMCVCADKLVKTYSGGNKRMLSTGIALVGEPAVILLDEPSTGMDPVARRLLWDTVERVRESGKTIVITSHSMEECEALCTRLAIMVQGQFKCLGSPQHLKSKFGISYSLQAKVRRKWQQQMLEEFKAFVDLTFPGSNLEDEHQNMLQYYLPGPNLSWAKVFSIMEQAKKDYMLEDYSISQLSLEDIFLNFTRPESSTKEQIQQEQAVLASPSPPSNSRPISSPPSRLSSPTPKPLPSPPPSEPILL
</sequence>
<mass datatype="int">196020</mass>
<classifiedWith>
<item href="http://purl.obolibrary.org/obo/GO_0016021"/>
<item href="http://purl.uniprot.org/keywords/325"/>
<item href="http://purl.uniprot.org/keywords/67"/>
<item href="http://purl.uniprot.org/keywords/677"/>
<item href="http://purl.uniprot.org/keywords/445"/>
<item href="http://purl.uniprot.org/keywords/256"/>
<item href="http://purl.uniprot.org/keywords/443"/>
<item href="http://purl.obolibrary.org/obo/GO_0042626"/>
<item href="http://purl.uniprot.org/keywords/963"/>
<item href="http://purl.obolibrary.org/obo/GO_0006638"/>
<item href="http://purl.obolibrary.org/obo/GO_0005783"/>
<item href="http://purl.obolibrary.org/obo/GO_0005789"/>
<item href="http://purl.uniprot.org/keywords/1133"/>
<item href="http://purl.obolibrary.org/obo/GO_0005524"/>
<item href="http://purl.uniprot.org/keywords/1185"/>
<item href="http://purl.obolibrary.org/obo/GO_0006869"/>
</classifiedWith>
<Function_Annotation>
Promotes cholesterol efflux from sperm which renders sperm capable of fertilization (PubMed:22237709). Has also been shown to decrease levels of intracellular esterified neutral lipids including cholesteryl esters, fatty acid esters and triacylglycerols (PubMed:15810880).
</Function_Annotation>
</primaryTopic>
<activeLens>Default</activeLens>
<linkPredicate href="http://www.w3.org/2004/02/skos/core#exactMatch"/>
<extendedMetadataVersion href="http://heater.cs.man.ac.uk:3002/target?uri=http%3A%2F%2Fpurl.uniprot.org%2Funiprot%2FE9PX95&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_metadata=all%2Cviews%2Cformats%2Cexecution%2Cbindings%2Csite"/>
<definition href="http://heater.cs.man.ac.uk:3002/api-config"/>
</result>
```

## Identity mapping


When we add new data to Open PHACTS, that will usually come with new URIs. One great advantage of the Open PHACTS platform is that it performs identity mapping, so if we [ask the Identity Mapping Service](http://heater.cs.man.ac.uk:3004/QueryExpander/mapUri?Uri=http%3A%2F%2Fwww.uniprot.org%2Funiprot%2FP01189&lensUri=http%3A%2F%2Fopenphacts.org%2Fspecs%2F%2FLens%2FDefault&Pattern+Filter=&overridePredicateURI=&format=text%2Fhtml) (IMS) about http://www.uniprot.org/uniprot/P01189 - we get also related identifiers including:

* http://identifiers.org/drugbankv4.target/BE0000947
* http://identifiers.org/ensembl/ENSG00000115138
* http://bio2rdf.org/genbank:AAA35799
* http://identifiers.org/nextprot/NX_P01189

This mapping means that any of these URIs can be used instead of the original uniprot URL in the Open PHACTS API calls, e.g. asking `/target` about [genbank:AAA35799](https://beta.openphacts.org/2.1/target?uri=http%3A%2F%2Fbio2rdf.org%2Fgenbank%3AAAA35799&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_format=xml) includes the Uniprot information:

```xml
<exactMatch href="http://purl.uniprot.org/uniprot/P01189">
<molecularWeight datatype="int">29424</molecularWeight>
<inDataset href="http://purl.uniprot.org"/>
<sequence>
MPRSCCSRSGALLLALLLQASMEVRGWCLESSQCQDLTTESNLLECIRACKPDLSAETPMFPGNGDEQPLTENPRKYVMGHFRWDRFGRRNSSSSGSSGAGQKREDVSAGEDCGPLPEGGPEPRSDGAKPGPREGKRSYSMEHFRWGKPVGKKRRPVKVYPNGAEDESAEAFPLEFKRELTGQRLREGDGPDGPADDGAGAQADLEHSLLVAAEKKDEGPYRMEHFRWGSPPKDKRYGGFMTSEKSQTPLVTLFKNAIIKNAYKKGE
</sequence>
<organism href="http://purl.uniprot.org/taxonomy/9606"/>
<!-- .. -->
```

But in order for IMS to do this, it needs to be loaded with [linksets](http://ops2.few.vu.nl/QueryExpander/SourceInfos?lensUri=All).  Now we have updated the Uniprot RDF data in Virtuoso, but we have not provided new linksets to IMS - let's try with the new identifier http://purl.uniprot.org/uniprot/E9PX95 - [IMS only shows self-mappings](http://ops2.few.vu.nl/QueryExpander/mapUri?Uri=http%3A%2F%2Fpurl.uniprot.org%2Funiprot%2FE9PX95&lensUri=http%3A%2F%2Fopenphacts.org%2Fspecs%2F%2FLens%2FDefault&Pattern+Filter=&overridePredicateURI=&format=text%2Fhtml):

> UniProt Isoform: [E9PX95](http://purl.uniprot.org/uniprot/E9PX95)
> * http://purl.uniprot.org/uniprot/E9PX95
> * http://www.uniprot.org/uniparc/?query=E9PX95
> * http://identifiers.org/uniprot.isoform/E9PX95
> * http://info.identifiers.org/uniprot.isoform/E9PX95
> * http://www.uniprot.org/uniprot/E9PX95
> * This is an automatic mapping to alternative equals URIs



However we can see in from the earlier `DESCRIBE` that the Uniprot data include database cross-references, e.g. to [Unigene](http://purl.uniprot.org/unigene/Mm.328842) and [Ensembl](http://purl.uniprot.org/ensembl/ENSMUST00000039324):

```turtle
ns1:E9PX95	rdfs:seeAlso
		<http://purl.uniprot.org/unigene/Mm.328842> .

@prefix ns6:	<http://purl.uniprot.org/ensembl/> .
ns1:E9PX95	rdfs:seeAlso	ns6:ENSMUST00000039324 .

```


Different linksets are downloaded or created in different ways, for instance the [mapping from Uniprot to UniGene](http://ops2.few.vu.nl/QueryExpander/mappingSet?sourceCode=S&targetCode=U) is loaded from [uniprot_unigene.ttl.gz](https://data.openphacts.org/free/2.1/ims/linksets/uniprot/uniprot_unigene.ttl.gz).

Many of our [linksets](https://github.com/openphacts/?utf8=%E2%9C%93&query=linkset) have a corresponding GitHub repository, in this case https://github.com/openphacts/ops-uniprot-linksets which has a series of [sparql queries](https://github.com/openphacts/ops-uniprot-linksets/tree/master/sparql) together with the resulting linksets [void metadata](https://github.com/openphacts/ops-uniprot-linksets/tree/master/void).

Here we will check this out with git, but you could also use the _Download ZIP_ button.

```shell
git clone https://github.com/openphacts/ops-uniprot-linksets.git
```

In this case we can regenerate the unigene linkset by running the [uniprot unigene query](https://github.com/openphacts/ops-uniprot-linksets/blob/master/sparql/uniprot_unigene.sparql) against our updated SPARQL endpoint on the equivalent of http://heater.cs.man.ac.uk:3003/sparql

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX uniprot: <http://purl.uniprot.org/core/>
CONSTRUCT {
	?uniprot_uri rdfs:seeAlso ?unigene_uri
} WHERE {
	GRAPH <http://purl.uniprot.org> {
		?uniprot_uri rdfs:seeAlso ?unigene_uri .
		?unigene_uri uniprot:database <http://purl.uniprot.org/database/UniGene> .
	}
}
```

We'll save the result into our `ops-uniprot-linksets/data` folder, deleting the old `uniprot_unigene.ttl.gz`.

We'll need to compress it to `gzip`.

Next we should be good metadata curators and update the [corresponding VoID](https://github.com/openphacts/ops-uniprot-linksets/blob/master/void/uniprot_unigene_ls_void.ttl) - the changes should be something like

```turtle
pav:createdBy <http://orcid.org/0000-0002-1303-2189> ;
pav:createdOn "2015-11-17T17:00:23Z"^^xsd:dateTime ;
pav:version "2016_07";
dcterms:description "Linkset relating Uniprot entries to UniGene, generated by querying Uniprot KB (2016_07)" ;
void:triples "104567"^^xsd:integer .
```

The most important statements in here for loading is `dul:expresses` and `void:linkPredicate` - but unless we
change the query we should not need to modify these.

To update `void:triples` we need to count them in the linkset. One way is to rewrite the query to a `COUNT(*)`:


```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX uniprot: <http://purl.uniprot.org/core/>

SELECT COUNT(*)
WHERE {
	GRAPH <http://purl.uniprot.org> {
		?uniprot_uri rdfs:seeAlso ?unigene_uri .
		?unigene_uri uniprot:database <http://purl.uniprot.org/database/UniGene> .
	}
}
```

To simplify loading, the void info can be added to the top or bottom of the Turtle file,
we'll use the `merge.sh` shell script to do that.

```
stain@BIGGIE:/mnt/d/src/ops-uniprot-linksets$ sh merge.sh
gzip: data/uniprot_ipi.ttl.gz: No such file or directory
gzip: data/uniprot_unigene.ttl.gz: No such file or directory
```

Uups! Oh, what went wrong? We don't care about the `ipi` linkset,
but `unigene` is ours.. let's see:

```
stain@BIGGIE:/mnt/d/src/ops-uniprot-linksets$ ls data
uniprot_ensembl.ttl.gz  uniprot_mgi.ttl.gz   uniprot_refseq.ttl.gz  uniprot-unigene.ttl.gz
```

Ah, we got the filename wrong..

```shell
cd data
mv uniprot-unigene.ttl.gz uniprot_unigene.ttl.gz
```

Now

```
stain@BIGGIE:/mnt/d/src/ops-uniprot-linksets$ ./merge.sh
gzip: data/uniprot_ipi.ttl.gz: No such file or directory

stain@BIGGIE:/mnt/d/src/ops-uniprot-linksets$ ls -al merged/*unigene*
-rwxrwxrwx 1 root root 817521 Jul 28 12:49 merged/uniprot_unigene.ttl.gz
```

So we'll move it to the server:

```
stain@BIGGIE:/mnt/d/src/ops-uniprot-linksets$ scp merged/uniprot_unigene.ttl.gz heater.cs.man.ac.uk:data
stain@heater.cs.man.ac.uk's password:
uniprot_unigene.ttl.gz                                                                               100%  798KB 798.4KB/s   00:00
```

OK, next we'll do the data loading for the `ops-ims` Docker image. This is actually backed by the
`ops-mysql` database, which can be updated with a
[IMS data loading script](https://github.com/openphacts/queryExpander/tree/master/docker#data-loading)

Note that we do not need to shutdown `ops-ims` while adding to the database.

While the loader script can load over `http`, e.g. from `http://data.openphacts.org/` for the production
linksets, in this case we want to load it from `file:///`.

In our `data/` folder, make a new file `load.xml` containing:

```xml
<loadSteps>
  <linkset>file:///staging/linksets/uniprot_unigene.ttl</linkset>
</loadSteps>
```

Now we'll prepare the `linksets/` folder. Note that the loader script does not support `.gz`
when loading locally, so we'll have to unpack the `.ttl.gz`:

```shell
stain@heater:~$ mkdir -p data/linksets
stain@heater:~/data/linksets$ cd data/linksets

stain@heater:~/data/linksets$ mv ../uniprot_unigene.ttl.gz .

stain@heater:~/data/linksets$ gunzip uniprot_unigene.ttl.gz

stain@heater:~/data/linksets$ ls -al
total 6860
drwxrwxr-x 2 stain stain    4096 Jul 25 17:01 .
drwxrwxr-x 5 stain stain    4096 Jul 25 17:00 ..
-rw-rw-r-- 1 stain stain 7015594 Jul 25 16:58 uniprot_unigene.ttl
```

Now we should be ready to load according to the [queryExpander docker image](https://github.com/openphacts/queryExpander/tree/master/docker#data-loading) instructions, substituting `ops-mysql` and the equivalent of `/home/stain/data`.  Note that we have to say `file:///staging/load.xml` as `/home/stain/data/load.xml` will appear under the `/staging` folder within the Docker container.

```
docker run --link ops-mysql:mysql -v /home/stain/data:/staging openphacts/identitymappingservice loader file:///staging/load.xml
```

This will output a fair bit of log messages while the URL patterns in the mySQL database is updated (if needed):

```
(..)
FullName mismatch for M BridgeDb has MGI while miriam uses Mouse Genome Database
Regex patterns do not match for http://www.ebi.ac.uk/miriam/main/collections/MIR:00000039 was ^((AC|AP|NC|NG|NM|NP|NR|NT|NW|XM|XP|XR|YP|ZP)_\d+|(NZ\_[A-Z]{4}\d+))(\.\d+)?$ but BridgeBD has ^(NC|AC|NG|NT|NW|NZ|NM|NR|XM|XR|NP|AP|XP|ZP)_\d+$
FullName mismatch for Tb BridgeDb has TubercuList while miriam uses MycoBrowser tuberculosis
Alternative mismatch for Tb BridgeDb has TubercuList while miriam uses MycoBrowser tuberculosis
Regex patterns do not match for http://www.ebi.ac.uk/miriam/main/collections/MIR:00000216 was ^Rv\d{4}(A|B|c)?$ but BridgeBD has Rv\d{4}(A|B|c|\.\d)?
FullName mismatch for Gg BridgeDb has Gramene Genes DB while miriam uses Gramene genes
Alternative mismatch for Gg BridgeDb has Gramene Genes while miriam uses Gramene genes
(..)
Loaded file local.properties using CATALINA_HOME from /usr/local/tomcat/conf/BridgeDb/local.properties
```

And then finally:

```
Load 1 linksets plus their transdatives
```

Now that the linkset has been loaded, let's have a look again at the [UniGene mappings](http://heater.cs.man.ac.uk:3004/QueryExpander/SourceTargetInfos?sourceCode=U&lensUri=All) - which now include [a new mapping set](http://heater.cs.man.ac.uk:3004/QueryExpander/mappingSet/240) from file:///staging/linksets/uniprot_unigene.ttl

> **Tip:** Obviously this `file:///staging` source URL won't work outside the IMS Docker container - to fix this you can put the linksets on your own http server or GitHub and use `http://` urls in `load.xml` instead)


And now let's see if http://purl.uniprot.org/unigene/Mm.328842 [includes our new Uniprot identifier](http://heater.cs.man.ac.uk:3004/QueryExpander/mapUri?Uri=http%3A%2F%2Fpurl.uniprot.org%2Funigene%2FMm.328842&lensUri=http%3A%2F%2Fopenphacts.org%2Fspecs%2F%2FLens%2FDefault&Pattern+Filter=&overridePredicateURI=&format=text%2Fhtml) `E9PX95`:


> Ensembl ENSMUSG00000035435
> * http://identifiers.org/ensembl/ENSMUSG00000035435
> Entrez Gene 381072
> ..
> UniGene Mm.328842
> * http://www.ncbi.nlm.nih.gov/unigene?term=Mm.328842
> * http://purl.uniprot.org/unigene/Mm.328842UniGene
> Uniprot-TrEMB E9PX95
> * http://identifiers.org/uniprot/E9PX95
> ..

This mean that we should now be able to use UniGene `Mm.328842` as an identifier to `/target`, which does not work with
the 2.1 API:

https://beta.openphacts.org/2.1/target?uri=http%3A%2F%2Fpurl.uniprot.org%2Funigene%2FMm.328842&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_format=xml

> 404 Page not found

But which our updated API now can find Uniprot properties for:

http://heater.cs.man.ac.uk:3002/target?uri=http%3A%2F%2Fpurl.uniprot.org%2Funigene%2FMm.328842&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_format=xml

```xml
<primaryTopic href="http://purl.uniprot.org/unigene/Mm.328842">
<exactMatch href="http://purl.uniprot.org/uniprot/E9PX95">
<molecularWeight datatype="int">196020</molecularWeight>
<inDataset href="http://purl.uniprot.org"/>
<sequence>
MEVLKKLKLLLWKNFILKRRKTLITLLEMLMPLLFCAIVLYLRLNSMPRKKSSTNYPAVDVSLLPVYFYNYPLKSKFQLAYIPSKSETLKAVTEVVEQTFAVDFEVLGFPSVPLFEDYIIKDPKSFYILVGIIFHHDFNSSNEPLPLVVKYDLRFSYVQRNFVSPPRHLFFQEEIEGWCTAFLYPPNLSQAPREFSYADGGNPGYNKEGFLAIQHAVDKAIMRHHAPKAALNMFKDLHVLVQRFPFGPHIQDPFLVILQNEFPLLLMLSFICVELIITNSVLSEKERKQKEYMSMMGVESWLHWVAWFITFFISVSITVSVMTVLFCTKINRVAVFRNSNPTLIFIFLMCFAIATIFFAFMMSTFFQRAHVGTVIGGTVFFFTYLPYMYITFSYHQRTYTQKILSCLFSNVAMATGVRFISLFEAEGTGIQWRNIGSVWGDFSFAQVLGMLLLDSFLYCLIAFLVESLFPRKFGIPKSWYIFAKKPVPEIPPLLNIGDPEKPSKGNFMQDEPTNQMNTIEIQHLYKVFYSGRSKRTAIRDLSMNLYKGQVTVLLGHNGAGKTTVCSVLTGLITPSKGHAYIHGCEISKDMVQIRKSLGWCPQHDILFDNFTVTDHLYFYGQLKGLSPQDCHEQTQEMLHLLGLKDKWNSRSKFLSGGMKRKLSIGIALIAGSKVLILDEPTSGLDSPSRRAIWDLLQQQKGDRTVLLTTHFMDEADLLGDRIAILAKGELQCCGSPSFLKQKYGAGYYMTIIKTPLCDTSKLSEVIYHHIPNAVLESNIGEEMIVTLPKKTIHRFEALFNDLELRQTELGISTFATSVTTMEEVFIRVCKLADPSTNVLTEKRHSLHPLPRHHRVPVDRIKCLHSGTFPVSTEQPMRLNTGFCLLCQQFYAMLLKKITYSRRNWMLVLSVQVLLPLAIIMLSLTFFNFKLRKLDNVPLELTLQTYGQTIVPFFIAENSHLDPQLSDDFVKMLVAAGQVPLRIQGSVEDFLLKKAKEAPEGFDKLYVVAASFEDVNNHTTVKALFNNQAYHSPSLALTLVDNLLFKLLSGANASITTTNYPQPQTAIEVSESILYQGPKGHYLVVNFLFGIAFLSSSFSILTVGEKSVKSKSLQFVSGVSTAVFWLSALLWDLISFLVPTLLLVLVFLWYKEEAFAHHESIPAVVLIMMLYGWAVIPLVYTVSFSFNTPGSACVKLVVMLTFLSISPVVLVTVTSEKDLGYTELSDSLDHIFLILPGHCLGMALSNLYYNFELKKFCSAKNLSDIDCNDVLEGYVVQENIYAWESLGIGKYLTALAVLGPVYITMLFLTEANAFYVLKSRLSGFFPSFWKEKSGMIFDVAEPEDEDVLEEAETIKRYLETLVKKNPLVVKEVSKVYKDKVPLLAVNKVSFVVKEEECFGLLGLNGAGKTSIFNMLTSEQPITSGDAFVKGFNIKSDIAKVRQWIGYCPEFDALLNFMTGREMLVMYARIRGIPECHIKACVDLILENLLMCVCADKLVKTYSGGNKRMLSTGIALVGEPAVILLDEPSTGMDPVARRLLWDTVERVRESGKTIVITSHSMEECEALCTRLAIMVQGQFKCLGSPQHLKSKFGISYSLQAKVRRKWQQQMLEEFKAFVDLTFPGSNLEDEHQNMLQYYLPGPNLSWAKVFSIMEQAKKDYMLEDYSISQLSLEDIFLNFTRPESSTKEQIQQEQAVLASPSPPSNSRPISSPPSRLSSPTPKPLPSPPPSEPILL
</sequence>
<mass datatype="int">196020</mass>
<!-- .. -->
```

BTW, we saw in the IMS mapping there was also the ensembl `ENSMUSG00000035435` ID, but have a look at

http://heater.cs.man.ac.uk:3004/QueryExpander/mapUri?Uri=http%3A%2F%2Fidentifiers.org%2Fensembl%2FENSMUSG00000035435&lensUri=http%3A%2F%2Fopenphacts.org%2Fspecs%2F%2FLens%2FDefault&Pattern+Filter=&overridePredicateURI=&format=text%2Fhtml

> Uniprot-TrEMBL E9PX95
> http://identifiers.org/uniprot/E9PX95
> http://www.ncbi.nlm.nih.gov/protein/E9PX95
> http://purl.uniprot.org/uniprot/E9PX95
> http://www.ebi.uniprot.org/entry/E9PX95
> http://info.identifiers.org/uniprot/E9PX95
> http://bio2rdf.org/uniprot:E9PX95
> http://www.uniprot.org/uniprot/E9PX95
> Found in [mappingSet/113](http://heater.cs.man.ac.uk:3004/QueryExpander/mappingSet/113)

That [mappingSet 113](http://heater.cs.man.ac.uk:3004/QueryExpander/mappingSet/113) is not the new one we just
added, but `Ensembl_Mm_uniprot.inferred_from_translation.LS.ttl.gz` - so this means by loading the RDF data we
have activated a "dormant" identity mapping from Ensembl:


https://beta.openphacts.org/2.1/target?uri=http%3A%2F%2Fidentifiers.org%2Fensembl%2FENSMUSG00000035435&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_format=xml

> 404 Not Found

Compared to:

http://heater.cs.man.ac.uk:3002/target?uri=http%3A%2F%2Fidentifiers.org%2Fensembl%2FENSMUSG00000035435&app_id=161aeb7d&app_key=333c09ae195d777b68a117bb42f29b1c&_format=xml


## What's next?

As you see there are still many manual steps in a data update.  Some of these can be automated, but things like updating queries and checking linksets still requires manual work. However it would be good to have this process
more reproducible and 

We have been working towards an automated way to update a dataset for Open PHACTS, using Apache Maven and a [data-maven-plugin](https://github.com/stain/data-maven-plugin) - which we use to populate a Maven repository on [data.openphacts.org](https://data.openphacts.org/maven/org/openphacts/data/).

Let's have a quick look on how the Chembl dataset can be updated to Chembl 21.0:

```shell
git clone https://github.com/openphacts/ops-chembl-rdf.git
cd ops-chembl-rdf
git checkout data-plugin
mvn clean install
```

```
[INFO] --- wagon-maven-plugin:1.0:download (download-rdf) @ chembl-rdf ---
[INFO] Scanning remote file system: http://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBL-RDF/20.1 ...
[INFO] Downloading http://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBL-RDF/20.1/cco.ttl.gz to D:\src\ops-chembl-rdf\target\classes\data\chembl-rdf\cco.ttl.gz ...
[INFO] Downloading http://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBL-RDF/20.1/chembl_20.1_activity.ttl.gz to D:\src\ops-chembl-rdf\target\classes\data\chembl-rdf\chembl_20.1_activity.ttl.gz ...
(..)
```

So this takes care of the download and packaging for us. Updating this is usually quite easy, just edit `pom.xml` to change the version numbers:

```xml
<url>http://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBL-RDF/${chembl.version}</url>
```

Looking at http://ftp.ebi.ac.uk/pub/databases/chembl/ChEMBL-RDF/ we see that `21.0` is the latest version. We'll change the `chembl.version` property:

```xml
<chembl.version>21.0</chembl.version>
```

As well as the version of the packaged data archive:

```
  <groupId>org.openphacts.data</groupId>
  <artifactId>chembl-rdf</artifactId>
  <version>21.0.0-SNAPSHOT</version>
```

> **Tip:** We can increase the third _patch_ digit `.0` if we are improving the packaging of an existing upstream version.

We working towards making the data archives be valid [Research Objects](http://www.researchobject.org/) for propagation of provenance metadata, and to automate downloading and loading of a set of such data archives into the Open PHACTS platform. 
