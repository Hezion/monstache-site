---
title: Options
weight: 20
---

Options can be specified in your TOML config file or be passed into monstache as Go program arguments on the command line.
Options specified as program arguments take precedance over the same option in the TOML config file.

## print-config

### boolean (default false)

When print-config is true monstache will print its configuration and then exit

## stats

### boolean (default false)

When stats is true monstache will periodically print statistics accumulated by the indexer

## stats-duration

### string (default 30s)

Sets the duration after which statistics are printed if stats is enabled

## index-stats

### boolean (default false)

When index-stats is true monstache will write statistics about its indexing progress in
Elasticsearch.  The indexes used to store the statistics are time stamped by day and 
prefixed `monstache.stats.`. E.g. monstache.stats.2017-07-01 and so on. 

As these indexes will accrue over time your can use a tool like [curator](https://github.com/elastic/curator)
to prune them with a Delete Indices action and an age filter.

## gzip

### boolean (default false)

When gzip is true, monstache will compress requests to elasticsearch to increase performance. 
If you enable gzip in monstache and are using elasticsearch prior to version 5 you will also 
need to update the elasticsearch config file to set http.compression: true. In elasticsearch 
version 5 and above http.compression is enabled by default. Enabling gzip is recommended 
especially if you enable the index-files setting.

## fail-fast

### boolean (default false)

When fail-fast is true, if monstache receives a failed bulk indexing response from Elasticsearch, monstache
will log the request that produced the response as an ERROR and then exit immediately with an error status. 
Normally, monstache just logs the error and continues processing events.

If monstache has been configured with elasticsearch-retry true, a failed request will be
retried before being considered a failure.

## index-oplog-time

### boolean (default false)

If this option is set to true monstache will include 2 automatic fields in the source document indexed into
Elasticsearch.  The first is _oplog_ts which is the timestamp for the event copied directly from the MongoDB
oplog. The second is _oplog_date which is an Elasticsearch date field corresponding to the time of the same
event.

This information is generally useful in Elasticsearch giving the notion of last updated.  However, it's also
valuable information to have for failed indexing requests since it gives one the information to replay from 
a failure point.  See the option resume-from-timestamp for information on how to replay oplog events since 
a given event occurred. 

For data read via the direct read feature the oplog time will only be available if the id of the MongoDB
document is an ObjectID.  If the id of the MongoDB document is not an ObjectID and the document source is
a direct read query then the oplog time with not be available.

## resume

### boolean (default false)

When resume is true, monstache writes the timestamp of mongodb operations it has successfully synced to elasticsearch
to the collection monstache.monstache.  It also reads the timestamp from that collection when it starts in order to replay
events which it might have missed because monstache was stopped. If monstache is started with the cluster-name option
set then resume is automatically turned on.  

## resume-name

### string (default "default")

monstache uses the value of resume-name as an id when storing and retrieving timestamps
to and from the mongodb collection monstache.monstache. The default value for this option is default.
However, there are some exceptions.  If monstache is started with the cluster-name option set then the
name of the cluster becomes the resume-name.  This is to ensure that any process in the cluster is able to resume
from the last timestamp successfully processed.  The other exception occurs when resume-name is not given but
worker-name is.  In that cause the worker name becomes the resume-name.

## resume-from-timestamp

### int64 (default 0)

When resume-from-timestamp (a 64 bit timestamp where the high 32 bytes represent the number of seconds since epoch and the low 32 bits
represent an offset within a second) is given, monstache will sync events starting immediately after the timestamp.  This is useful if you have 
a specific timestamp from the oplog and would like to start syncing from after this event. 

## replay

### boolean (default false)

When replay is true, monstache replays all events from the beginning of the mongodb oplog and syncs them to elasticsearch.

When resume and replay are both true, monstache replays all events from the beginning of the mongodb oplog, syncs them
to elasticsearch and also writes the timestamp of processed events to monstache.monstache. 

When neither resume nor replay are true, monstache reads the last timestamp in the oplog and starts listening for events
occurring after this timestamp.  Timestamps are not written to monstache.monstache.  This is the default behavior. 

## resume-write-unsafe

### boolean (default false)

When resume-write-unsafe is true monstache sets the safety mode of the mongodb session such that writes are fire and forget.
This speeds up writing of timestamps used to resume synching in a subsequent run of monstache.  This speed up comes at the cost
of no error checking on the write of the timestamp.  Since errors writing the last synched timestamp are only logged by monstache
and do not stop execution it's not unreasonable to set this to true to get a speedup.

## direct-read-namespaces

### []string (default nil)

At times even being able to replay from the beginning of the oplog is not enough to sync all of your mongodb data.
The oplog is a capped collection and may only contain a subset of the data.  In this case you can perform a direct
sync of mongodb to elasticsearch.  To do this, set direct-read-namespaces to an array of namespaces that you would 
like to copy.  Monstache will perform reads directly from the given set of db.collection and sync them to elasticsearch.

This option may be passed on the command line as ./monstache --direct-read-namespace test.foo --direct-read-namespace test.bar

For maximum indexing performance when doing alot of a direct reads you will want to adjust the refresh interval during indexing on the
destination Elasticsearch indices.  The refresh interval can be set at a global level in elasticsearch.yml or on a per
index basis by using the Index Settings or Index Template APIs.  For more information see [Update Indices Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-update-settings.html).

By default, Elasticsearch refreshes every second.  You will want to increase this value or turn off refresh completely during the indexing
phase by setting the refresh_interval to -1.  Remember to reset the refresh_interval to a positive value and do a force merge after the indexing 
phase has completed if you decide to temporarily turn off refresh, otherwise you will not be able to see the new documents in queries.

By default, Monstache maps a MongoDB collection named `foo` in a database named `test` to the `test.foo` index in Elasticsearch.

## direct-read-limit

### int (default 5000)

The maximum number of documents to retreive in each direct read query

## direct-read-batch-size

### int (default 500)

The batch size to set on direct read queries

## direct-readers-per-col

### int (default 10)

The number of concurrent readers per collection for direct reads

## exit-after-direct-reads

### boolean (default false)

The direct-read-namespaces option gives you a way to do a full sync on multiple collections.  At times you may want
to perform a full sync via the direct-read-namespaces option and then quit monstache.  Set this option to true and
monstache will exit after syncing the direct read collections instead of continuing to tail the oplog. This is useful
if you would like to run monstache to run a full sync on a set of collections via a cron job.

## namespace-regex

### regexp (default "")

When namespace-regex is given this regex is tested against the namespace, database.collection, of the event. If
the regex matches monstache continues processing event filters, otherwise it drops the event. By default monstache
processes events in all databases and all collections with the exception of the reserved database monstache, any
collections suffixed with .chunks, and the system collections. For more information see the section [Namespaces](/namespaces/).

## namespace-exclude-regex

### regex (default "")

When namespace-exclude-regex is given this regex is tested against the namespace, database.collection, of the event. If
the regex matches monstache ignores the event, otherwise it continues processing event filters. By default monstache
processes events in all databases and all collections with the exception of the reserved database monstache, any
collections suffixed with .chunks, and the system collections. For more information see the section [Namespaces](/namespaces/).

## mongo-url

### string (default localhost)

The URL to connect to MongoDB which must follow the [Standard Connection String Format](https://docs.mongodb.com/v3.0/reference/connection-string/#standard-connection-string-format)

## mongo-pem-file

### string (default "")

When mongo-pem-file is given monstache will use the given file path to add a local certificate to x509 cert
pool when connecting to mongodb. This should only be used when mongodb is configured with SSL enabled.

## mongo-validate-pem

### boolean (default true)

When mongo-validate-pem-file is false TLS will be configured to skip verification

## mongo-oplog-database-name

### string (default local)

When mongo-oplog-database-name is given monstache will look for the mongodb oplog in the supplied database

## mongo-oplog-collection-name

### string (default $oplog.main)

When mongo-oplog-collection-name is given monstache will look for the mongodb oplog in the supplied collection

## mongo-cursor-timeout

### string (default 100s)

When mongo-cursor-timeout is given monstache will time out and re-query the oplog after the supplied duration.
Duration values are expected in the form 50s.

## mongo-dial-settings

### TOML table (default nil)

The following mongodb dial properties are available

### ssl

#### bool (default false)

Set to true to establish a connection using TLS.

### timeout

#### int (default 10)

Seconds to wait when establishing a connection to mongodb before giving up

## mongo-session-settings

### TOML table (default nil)

The following mongodb session properties are available

### socket-timeout

#### int (default 60)

Seconds to wait for a non-responding socket before it is forcefully closed

### sync-timeout

#### int (default 60)

Amount of time in seconds an operation will wait before returning an error in case a connection to a usable server can't be established.
Set it to zero to wait forever.

## gtm-settings

### TOML table (default nil)

The following gtm configuration properties are available.  See [gtm](https://github.com/rwynn/gtm) for details

### channel-size

#### int (default 512)

Controls the size of the go channels created for processing events.  When many events
are processed at once a larger channel size may prevent blocking in gtm.

### buffer-size

#### int (default 32)

Determines how many documents are buffered by a gtm worker go routine before they are batch fetched from
mongodb.  When many documents are inserted or updated at once it is better to fetch them together.

### buffer-duration

#### string (default 750ms)

A string representation of a golang duration.  Determines the maximum time a buffer is held before it is 
fetched in batch from mongodb and flushed for indexing.

## index-files

### boolean (default false)

When index-files is true monstache will index the raw content of files stored in GridFS into elasticsearch as an attachment type.
By default index-files is false meaning that monstache will only index metadata associated with files stored in GridFS.
In order for index-files to index the raw content of files stored in GridFS you must install a plugin for elasticsearch.
For versions of elasticsearch prior to version 5, you should install the [mapper-attachments](https://www.elastic.co/guide/en/elasticsearch/plugins/2.3/mapper-attachments.html) plugin.  In version 5 or greater
of elasticsearch the mapper-attachment plugin is deprecated and you should install the [ingest-attachment](https://www.elastic.co/guide/en/elasticsearch/plugins/master/ingest-attachment.html) plugin instead.
For further information on how to configure monstache to index content from GridFS, see the section [Indexing Gridfs Files](/gridfs/).

## max-file-size

### int (default 0)

When max-file-size is greater than 0 monstache will not index the content of GridFS files that exceed this limit in bytes.

## file-namespaces

### []string (default nil)

The file-namespaces config must be set when index-files is enabled.  file-namespaces must be set to an array of mongodb
namespace strings.  Files uploaded through gridfs to any of the namespaces in file-namespaces will be retrieved and their
raw content indexed into elasticsearch via either the mapper-attachments or ingest-attachment plugin. 

This option may be passed on the command line as ./monstache --file-namespace test.foo --file-namespace test.bar

## file-highlighting

### boolean (default false)

When file-highlighting is true monstache will enable the ability to return highlighted keywords in the extracted text of files
for queries on files which were indexed in elasticsearch from gridfs.

## verbose

### boolean (default false)

When verbose is true monstache with enable debug logging including a trace of requests to elasticsearch

## elasticsearch-user

### string (default "")

Optional Elasticsearch username for basic auth

## elasticsearch-password

### string (default "")

Optional Elasticsearch password for basic auth

## elasticsearch-urls

### []string (default `[ "http://localhost:9200" ]`)

An array of URLs to connect to the Elasticsearch REST Interface

This option may be passed on the command line as ./monstache --elasticsearch-url URL1 --elasticsearch-url URL2 

## elasticsearch-version

### string (by default determined by connecting to the server)

When elasticsearch-version is provided monstache will parse the given server version to determine how to interact with
the elasticsearch API.  This is normally not recommended because monstache will connect to elasticsearch to find out
which version is being used.  This option is provided for cases where connecting to the base URL of the elasticsearch REST
API to get the version is not possible or desired.

## elasticsearch-max-conns

### int (default 10)

The size of the Elasticsearch HTTP connection pool. This determines the concurrency of bulk indexing requests to Elasticsearch.
If you increase this value you may begin to see bulk indexing failures if the bulk index queue gets overloaded.  The default size
of this queue is 50.  To increase the size of the bulk indexing queue you can update the Elasticsearch config file:

	thread_pool:
	    bulk:
		queue_size: 200

For more information see [Thread Pool](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html).

## elasticsearch-retry

### boolean (default false)

When elasticseach-retry is true a failed request to elasticsearch will be retried with an exponential backoff policy. The policy
is set with an initial timeout of 50 ms, an exponential factor of 2, and a max wait of 20 seconds. For more information on how 
this works see [Back Off Strategy](https://github.com/olivere/elastic/blob/release-branch.v5/backoff.go)

## elasticsearch-client-timeout

### int (default 60)

The number of seconds before a request to elasticsearch times out

## elasticsearch-max-docs

### int (default 1000)

When elasticsearch-max-docs is given a bulk index request to elasticsearch will be forced when the buffer reaches the given number of documents

## elasticsearch-max-bytes

### int (default 16384)

When elasticsearch-max-bytes is given a bulk index request to elasticsearch will be forced when the buffer reaches the given number of bytes

## elasticsearch-max-seconds

### int (default 1)

When elasticsearch-max-seconds is given a bulk index request to elasticsearch will be forced when a request has not been made in the given number of seconds

## elasticsearch-pem-file

### string (default "")

When elasticsearch-pem-file is given monstache will use the given file path to add a local certificate to x509 cert
pool when connecting to elasticsearch. This should only be used when elasticsearch is configured with SSL enabled.

## elasticsearch-validate-pem

### boolean (default true)

When elasticsearch-validate-pem-file is false TLS will be configured to skip verification

## dropped-databases

### boolean (default true)

When dropped-databases is false monstache will not delete the mapped indexes in elasticsearch if a mongodb database is dropped

## dropped-collections

### boolean (default true)

When dropped-collections is false monstache will not delete the mapped index in elasticsearch if a mongodb collection is dropped

## worker

### string (default "")

When worker is given monstache will enter multi-worker mode and will require you to also provide the config option workers.  Use this mode to run
multiple monstache processes and distribute the work between them.  In this mode monstache will ensure that each mongo document id always goes to the
same worker and none of the other workers. See the section [workers](/workers/) for more information.

## workers

### []string (default nil)

An array of worker names to be used in conjunction with the worker option. 

This option may be passed on the command line as ./monstache --workers w1 --workers w2 

## enable-patches

### boolean (default false) 

Set to true to enable storing [rfc7396](https://tools.ietf.org/html/rfc7396) patches in your elasticsearch documents

## patch-namespaces

### []string (default nil)

An array of mongodb namespaces that you would like to enable rfc7396 patches on

This option may be passed on the command line as ./monstache --patch-namespace test.foo --patch-namespace test.bar 

## merge-patch-attribute

### string (default "json-merge-patches") 

Customize the name of the property under which merge patches are stored

## cluster-name

### string (default "")

When cluster-name is given monstache will enter a high availablity mode. Processes with cluster name set to the same value will coordinate.  Only one of the
processes in a cluster will be sync changes.  The other process will be in a paused state.  If the process which is syncing changes goes down for some reason
one of the processes in paused state will take control and start syncing.  See the section [high availability](/high-availability/) for more information.

## script

### [] array of TOML table (default nil)

When script is given monstache will pass the mongodb document into the script before indexing into elasticsearch.  See the section [Transform and Filter](/transform-filter/)
for more information.

## logs

### TOML table (default nil)

Allows writing logs to a file using a rolling appender instead of stdout.  Supply a file path for each type of log you would like to send to a file.

### info

#### string (default "")

The file path to write info level logs to

### error

#### string (default "")

The file path to write error level logs to

### trace

#### string (default "")

The file path to write trace level logs to. Trace logs are enabled via the verbose option.

### stats

#### string (default "")

The file path to write indexing statistics to. Stats logs are enabled via the stats option.

## enable-http-server

### boolean (default false)

Add this flag to enable an embedded HTTP server at localhost:8080

## http-server-addr

### string (default ":8080")

The address to bind the embedded HTTP server on if enabled

---
