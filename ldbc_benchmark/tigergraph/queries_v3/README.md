# LDBC SNB GSQL Benchmark

## Table of Contents
* [Overview](#Overview)
* [Pre-requisite](#Pre-requisite)
* [Download](#Donwload-LDBC-SNB-Data)
* [Load-data](#Load-data)
* [Run](#run)

## Overview
This follows [LDBC Social Network Benchmark v.0.4.0](https://github.com/ldbc/ldbc_snb_docs). BI 4 and 17 are modified based on discussion with Garbor. 
```
BI4: Find the most popular *Forums* by *Country*, where the popularity of a *Forum*
  is measured by the number of members that *Forum* has from a given *Country*.
  Calculate the top 100 most popular *Forums*.
  If a *Forum* is popular in multiple countries, it should only included in the top 100 once.
  In case of a tie, the *Forum*(s) with the smaller id value(s) should be selected.
```
* [`quereis`](./queries) directory contains the BI queries in GSQL.
* [`driver.py`](./driver.py) is the utility script to do anything related to the benchmark in this implementation.
* [`schema.gsql`](./schema.gsql) defines the LDBC SNB schema and loading job in GSQL.
* [`cypher`](./cypher) directory contains benchmark using Neo4j.

Related links
* [JIRA page](https://graphsql.atlassian.net/wiki/spaces/GRAP/pages/2352251355/LDBC-SNB)
* [LDBC_BI by Garbor](https://github.com/ldbc/ldbc_snb_bi) contains the benchmark using Cypher and postgres
* [Google Sheet](https://docs.google.com/spreadsheets/d/1NVdrOQtYBZl3g2B_jxYozo2pV-8B0Zzf50XDVw0JzTg/edit?ts=60b84592#gid=1034343597) contains internal benchmark results

## Pre-requisite 
* `TigerGraph` (at least 3.1.0) must be installed. I used 3.2.0. 
* `Python` (at least 3.6) must be installed to use the driver script. 
* Python library `requests` is required.

```sh
sudo yum install wget git tar python3 sshpass zstd 
python3 -m pip install requests
```
If zstd is not available. Download and compile the source from their github.
```sh
git clone https://github.com/facebook/zstd
cd zstd 
make && sudo make install
cd ..
```

Install tigergraph-3.1.3 Or find the latest build from (http://192.168.11.192/download.html). 
```sh
wget https://dl.tigergraph.com/enterprise-edition/tigergraph-3.1.3-offline.tar.gz
tar -xf tigergraph-3.1.3-offline
cd tigergraph-3.1.3-offline/
./install.sh
# following the instruction to install
su tigergraph
gadmin status
# check if TigerGraph is running or not.
```

## Config Tigergraph
To increase loading intensive, we can increase the number of loading handler based on the number of cpus in the machine. To see the cpu info, use `lscpu`. 
The timeout is only needed for large scale factors.
```sh
gadmin config group RESTPP-LOADER
# change FileLoader.Factory.HandlerCount from 4 to 40

gadmin config group timeout 
# Change FileLoader.Factory.DefaultQueryTimeoutSec: 16 -> 6000
# Change KafkaLoader.Factory.DefaultQueryTimeoutSec: 16 -> 6000
# Change RESTPP.Factory.DefaultQueryTimeoutSec: 16 -> 6000

gadmin config apply -y
gadmin restart all -y
```

## Donwload LDBC SNB Data 
LDBC data are available for scale factor [1](https://surfdrive.surf.nl/files/index.php/s/xM6ujh448lnJxXX/download), [3](https://surfdrive.surf.nl/files/index.php/s/fY7YocVgsJhmqdT/download), [10](https://surfdrive.surf.nl/files/index.php/s/SY6lRzEzDvvESfJ/download), [30](https://surfdrive.surf.nl/files/index.php/s/dtkgN7ZDT37vOnm/download), [100](https://surfdrive.surf.nl/files/index.php/s/gxNeHFKWVwO0WRm/download). To download data of scale factor 1,

```sh
wget -O sf1-composite-projected-fk.tar.zst https://surfdrive.surf.nl/files/index.php/s/xM6ujh448lnJxXX/download 
zstd -d sf1-composite-projected-fk.tar.zst 
tar -xvf sf1-composite-projected-fk.tar
```

## Load data
Checkout ldbc branch of the current repository
```sh
git clone --branch ldbc https://github.com/tigergraph/ecosys.git
cd ecosys/ldbc_benchmark/tigergraph/queries_v3
```
Load schema, data, and query. Usage of `driver.py` can be `-h` option. For example, to check how to load the data use `./driver.py load data -h`. The data directory should contain 31 folders in name of the vertex and edge type names. The CSV files inside these folders are loaded. 
```sh
./driver.py load schema
./driver.py load data ~/sf1/sf1/csv/bi/composite-projected-fk/
./driver.py load query
```
THis is equivalent to
```sh
./driver.py load all ~/initial_snapshot
```

The directory can include the machine. If you want to load data that is distributed on all the machines, use
```sh
./driver.py load data ALL:~/initial_snapshot 
``` 
After loading, you can use the following GSQL script to check the number of vertices and edges. For SF1, there are 1116485 Comment vertices in the initial snapshot.

```sh
gsql stat.gsql
```

## Run
Usage of `./driver.py` can be found using `./driver.py run -h`. The basic usage is `./driver.py run -q [queries] -n [number of runs] -p [parameter file]`. The default parameter file is `parameters/sf1/2012-09-13.json`.
```sh
./driver.py run  -n 3
```


```sh
# Query bi19 is expensive, we recomment to run without bi19 for 3 times
./driver.py run -q not:19 -n 3
# To run all the queries
./driver.py run 
```
## Compare the results of initial state

The starting time of LDBC SNB graph is 2012-09-13. The documented GSQL results are in `results_sf[scale factor]/initial`. The documented  Cypher results are in `cypher/results_sf[scale factor]/initial`. To compare the results with the documented GSQL results.
```sh
 ./driver.py compare -s results -t results_sf1/initial
```
The script can be also used to compare the GSQL and cypher results. 
```sh
# this is also the default parameter setting
 ./driver.py compare -s results -t cypher/results 
```

## Considerations in writing queries
There are many ways to write the query and here what we present is the one with the best performance. 
I may create a folder to discuss. The query is usually faster if:
* if you know the degree of the edges, and use SumAccum to store the information instead of SetAccum or MapAccum.
* if you start from a smaller vertex set 

## refreshes

Then run the refresh workloads. The results and timelog are output to `results/`. 
```sh
./driver.py refresh ~/sf1/csv/bi/composite-projected-fk/ 
```
After runnning neo4j benchmark, you can compare the results
```sh
# after running neo4j, compare thje
./driver.py compare 
```

For large scale factors, I prefer to run in background. 
```sh
nohup python3 -u ./driver.py all ~/sf1/csv/bi/composite-projected-fk/ > foo.out 2>&1 & 
```

## How does the driver.py work
./driver.py load query 
- gsql schema.gsql

./driver.py load query 
- Copy paste user defined function `ExprFunctions.hpp` to tigergraph. 
- `gsql [queries/*.gsql, delete/*.gsql, stat.gsql, parameter/gen_para.gsql]`
- `gsql -g ldbc_snb 'install query [all queries]'` 

./driver.py load data
- remove _SUCCESS files in sf1, `find sf1 -name _SUCCESS -type f -delete`
- `gsql -g ldbc_snb 'run loading job load_static using FILENAMES'`
- `gsql -g ldbc_snb 'run loading job load_dynamic using FILENAMES'`
