These scripts and documentation will help you migrate from a A4C 2.2.X / ES 1.7 to a A4C 3.0.X / ES 6.6.2 setup.

We consider that you have a running A4C 2.2.X with a remote Elasticsearch cluster 1.7.
Your need is to migrate to A4C 3.0.X with a remote Elasticsearch cluster in version 6.6.2.

You can use the same infrastructure for this migration for both A4C and ES machines (no parallel run).

Regarding the amount of data you have and the quality of your network, these operations can take some time. As an example, a test instance having ~ 1000 applications took 5 minutes to dump and 10 minute to load.

# Limitations

Plugins and their configuration won't be migrated:

* Plugins provided by A4C distribution will be new ones (3.0.X).
* If you have custom plugins, please ensure they build when referring to [3.0.X A4C branch](https://github.com/alien4cloud/alien4cloud/tree/3.0.x). You'll need to install them after migration (or put them in the `init` directory of your 3.0.X A4C distribution).

# Pre-requisites

You need a machine with sufficient disk space to store your whole dataset with network access to both elastic clusters.

You need to install [elasticdump](https://www.npmjs.com/package/elasticdump) on this machine.

```
npm install elasticdump -g
elasticdump
```

# Dump data

**Step 1**: Stop the old 2.2.X A4C instance.

**Step 2**: Ensure the following settings are set in the file `env.sh` :
* `es_source_url`: url of the source elasticsearch from which data will be dump, for example `https://34.244.42.130:9200`.
* `data_folder`: relative path to a local folder where data will be dump (will be recursively created if not exist).
* `tsl_enabled`: should be set to true if your ES need TLS authentication, comment if not.
* `client_cert_path`: client certificate file path (only needed if `tsl_enabled=true`).
* `client_key_path`: private key file path (only needed if `tsl_enabled=true`).

The dump script is not destructive, it just read data from the remote ES and dump it into local file.

You can ensure your config is correct by launching the following command :

```
source env.sh && curl -k --key "$client_key_path" --cert "$client_cert_path"  "$es_source_url/_cat/indices?v"
```

or, if you don't use TLS authentication :

```
source env.sh && curl -k "$es_source_url/_cat/indices?v"
```

**Step 3**: Launch the dump script.

This script will parse the `indexes.txt` file that contains the index list. All indexes except `plugin` and `pluginconfiguration` will be dump into local files.

If you want to keep traces of the dump, you can redirect std output in a file :

```
./dumpData.sh >> dumpData.log & tail -f dumpData.log
```

Dump is done when the message `End of dump !` appears.

**Step 4**: You can stop the old 1.7 Elasticsearch cluster.

# Installation

Install your new [6.6.2 Elasticsearch cluster](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/install-elasticsearch.html).

**Step 5**: Start the new 6.6.2 Elasticsearch cluster.

Install your new 3.0.X A4C instance. Ensure it target your new ES cluster. **Don't start it yet.**

:warning: Some properties have been added in `alien4cloud-config.yml` that you should add if you reuse your legacy config from your 2.2.X setup.

Ensure the following section is present n your config file :

```
ttl:
  period: 86400
```

In newer version of Elasticsearch, there is no more TTL for indexes. We have added a daemon in A4C that check TTL for few indexes (logs, audit, events). This period is the period in seconds between each run of this TTL job.

# Fileset migration

**Step 6**: Migrate fileset.

Alien4cloud stores some data in the local filesystem. The configuration property `directories.alien` (in `alien4cloud-config.yml` config file) contains the path where local data is stored on the filesystem. Let's call it the *data folder*.

This folder must be shared between the legacy 2.2.X instance and the new 3.0.X instance.

You have two strategy for the migration of this *data folder*:
1. Setup your new instance to target the same data folder. You can use this strategy if the new 3.0.X A4C instance is installed on the same machine than the legacy 2.2.X instance.
2. Copy the data folder from the legacy 2.2.X A4C instance to the new 3.0.X A4C instance. You may use this strategy if your new 3.0.X A4C instance is hosted on a new machine.

Whatever the strategy you adopt, you must exclude the `work` directory of the *data folder*. This directory contains the plugins content and we want the new plugins to be used instead of the legacy ones.

1. If you choose to setup your new 3.0.X A4C instance to target the legacy *data folder*, just backup the `work` directory (in case of rollback) :

```
mv data/work data/work_v2
```

2. If you choose to export/import your legacy *data folder*, just omit the `work` directory :

```
tar -czf data.tar.gz --exclude=*/work data
```

# Index initialization

**Step 7**: Start the new 3.0.X A4C instance. This operation will initialize the indexes on the new Elasticsearch cluster.
Wait for the full startup, then shutdown A4C.

**Step 8**: Shutdown the new 3.0.X A4C instance.

# Load data

**Step 9**: Ensure the following settings are set in the file `env.sh`:
* `es_dest_url`: url of the target elasticsearch into which data will be loaded.

If you use TLS authentication but still use the same keys and certificates for the new Elasticsearch cluster, you can leave the same configuration for `client_cert_path` and `client_key_path`. Otherwise, adapt these properties.

If you have securized your ES cluster following the [documentation](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/configuring-tls.html) and have generated PK12 file using `elasticsearch-certutil`, you'll be able to generate two .pem files containing respectively the key and the certificate using the following commands :

```
openssl pkcs12 -in elastic-client-certificates.p12 -out key.pem -nocerts -nodes
openssl pkcs12 -in elastic-client-certificates.p12 -out cert.pem -clcerts -nokeys
```

You can ensure your config is correct by launching the following command :

```
source env.sh && curl -k --key "$client_key_path" --cert "$client_cert_path"  "$es_dest_url/_cat/indices?v"
```

or, if you don't use TLS authentication :

```
source env.sh && curl -k "$es_dest_url/_cat/indices?v"
```

**Step 10**: Launch the load script that will load data file content into the new 6.6.2 ES cluster :

```
./loadData.sh >> loadData.log & tail -f loadData.log
```

Load is done when the message `End of load !` appears.

**Step 11**: Start the new 3.0.X A4C instance.
