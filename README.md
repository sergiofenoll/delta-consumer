# delta-consumer

Extendable consumer to sync data from external sources based on diff files generated by a producer. You can find an example
producer [here](http://github.com/lblod/mandatendatabank-mandatarissen-producer).

It does two things:
- Initial sync by getting dump files to ingest, and happens on service startup. (This occurs  only once if enabled.)
- Delta sync at regular intervals where the consumer checks for new diff files and loads the retrieved data.

By default, the delta-producer will use the fetched information to maintain an ingest graph. It implements the same behaviour as [delta-consumer-single-graph-maintainer](https://github.com/lblod/delta-consumer-single-graph-maintainer), which is deprecated by now.

However, custom ingestion rules are perfectly possible. Read along if you want to know how you can achieve this.
'Triples-dispatching' is the term we will use when moving triples to the correct place.

## Tutorials
### Add the service to a stack, with default behaviour.
The default behaviour fetches the information from the producer and maintains a single ingest graph.
To add this behaviour to your stack:

Add the following to your `docker-compose.yml`:

```yaml
consumer:
  image: lblod/delta-consumer
  environment:
    DCR_SERVICE_NAME: 'your-custom-consumer-identifier' # replace with the desired consumer identifier
    DCR_SYNC_BASE_URL: 'http://base-sync-url # replace with link the application hosting the producer server
    DCR_SYNC_DATASET_SUBJECT: "http://data.lblod.info/datasets/delta-producer/dumps/CacheGraphDump"
    DCR_INITIAL_SYNC_JOB_OPERATION: "http://redpencil.data.gift/id/jobs/concept/JobOperation/deltas/consumer/xyzInitialSync"
    DCR_DELTA_SYNC_JOB_OPERATION: "http://redpencil.data.gift/id/jobs/concept/JobOperation/deltas/consumer/xyzDeltaFileSyncing"
    DCR_JOB_CREATOR_URI: "http://data.lblod.info/services/id/consumer"
    INGEST_GRAPH: 'http://uri/of/the/graph/to/ingest/the/information'
```

### Add the service to a stack with custom behaviour.
This service assumes hooks, where you can inject custom code.

For your convenience, we've added an example custom hook in `./triples-dispatching/example-custom-dispatching`.
1. Copy the folder `example-custom-dispatching` into `config/consumer/`
2. Add the following to your `docker-compose.yml`:
  ```yaml
  consumer:
    image: lblod/delta-consumer
    environment:
      DCR_SERVICE_NAME: 'your-custom-consumer-identifier' # replace with the desired consumer identifier
      DCR_SYNC_BASE_URL: 'http://base-sync-url # replace with link the application hosting the producer server
      DCR_SYNC_DATASET_SUBJECT: "http://data.lblod.info/datasets/delta-producer/dumps/CacheGraphDump"
      DCR_INITIAL_SYNC_JOB_OPERATION: "http://redpencil.data.gift/id/jobs/concept/JobOperation/deltas/consumer/xyzInitialSync"
      DCR_DELTA_SYNC_JOB_OPERATION: "http://redpencil.data.gift/id/jobs/concept/JobOperation/deltas/consumer/xyzDeltaFileSyncing"
      DCR_JOB_CREATOR_URI: "http://data.lblod.info/services/id/consumer"
    volumes:
      - ./config/consumer/example-custom-dispatching:/config/triples-dispatching/custom-dispatching
  ```
3. Start the stack. The console will print the fetched information from the producer.

Please read further to find out more about the API of the hooks.

## Configuration
### Environment variables
#### What's with the weird variables names?
When accessing `process.env`, we distinguish between core service environment variables and triples-dispatching variables.

Variables prefixed with `DCR_` belong to the core. `DCR` could be an abbreviation for `delta-consumer`.
Custom logic for triples-dispatching should not access these directly, at the risk of breaking if the service evolves.
If you want to extend the variables in the core, make sure to respect the convention.

#### Core
The following environment variables are required:

- `DCR_SERVICE_NAME`: consumer identifier. important as it is used to ensure persistence. The identifier should be unique within the project. [REQUIRED]
- `DCR_SYNC_BASE_URL`: Base URL of the stack hosting the producer API [REQUIRED]
- `DCR_JOB_CREATOR_URI`: URL of the creator of the sync jobs [REQUIRED]
- `DCR_DELTA_SYNC_JOB_OPERATION`: Job operation of the delta sync job, used to describe the created jobs [REQUIRED]
- `DCR_SYNC_DATASET_SUBJECT`: subject used when fetching the dataset [REQUIRED BY DEFAULT]
- `DCR_INITIAL_SYNC_JOB_OPERATION`: Job operation of the initial sync job, used to describe the created jobs [REQUIRED BY DEFAULT]

To overrule the last two default required settings, and thus just ingest delta files, set `DCR_WAIT_FOR_INITIAL_SYNC: false` and `DCR_DISABLE_INITIAL_SYNC: true`.

The following environment variables are optional:

- `DCR_SYNC_FILES_PATH (default: /sync/files)`: relative path to the endpoint to retrieve the meta-data from the diff-files. Note: often, you will need to change this one.
- `DCR_DOWNLOAD_FILES_PATH (default: /files/:id/download)`: relative path to the endpoint to download a diff file
  from. `: id` will be replaced with the UUID of the file.
- `DCR_CRON_PATTERN_DELTA_SYNC (default: 0 * * * * *)`: cron pattern at which the consumer needs to sync data automatically.
- `DCR_START_FROM_DELTA_TIMESTAMP (ISO DateTime)`: timestamp to start sync data from (e.g. "2020-07-05T13:57:36.344Z") Only required when initial ingest hasn't run.
- `DCR_DISABLE_INITIAL_SYNC (default: false)`: flag to disable initial sync
- `DCR_DISABLE_DELTA_INGEST (default: false)`: flag to disable data ingestion, for example, while initializing the sync
- `DCR_WAIT_FOR_INITIAL_SYNC (default: true)`: flag to not wait for initial ingestion (meant for debugging)
- `DCR_KEEP_DELTA_FILES (default: false)`: if you want to keep the downloaded delta-files (ease of troubleshooting)

#### Triples dispatching: single graph ingestion (default behaviour)
- `INGEST_GRAPH (default: http://mu.semte.ch/graphs/public)`: graph in which all insert changesets are ingested
- `BATCH_SIZE (default: 100)`: Size of the batches to ingest in DB
- `BYPASS_MU_AUTH_FOR_EXPENSIVE_QUERIES (default: false)`: (see code where it is called) This has repercussions! Know what you do!
- `DIRECT_DATABASE_ENDPOINT (default: http://virtuoso:8890/sparql)`: only used when BYPASS_MU_AUTH_FOR_EXPENSIVE_QUERIES is set to true
- `MU_CALL_SCOPE_ID_INITIAL_SYNC (default: 'http://redpencil.data.gift/id/concept/muScope/deltas/consumer/initialSync)'`: A scope that can be set to refine dispatching rules of the (internal) deltanotifier. This variable is relevant during the initial sync.
- `MAX_DB_RETRY_ATTEMPTS (defaut: 5)`: Max DB retries in case of issues.
- `SLEEP_BETWEEN_BATCHES (default: 1000 ms)`: To not overload the system, every batch is paused.
- `SLEEP_TIME_AFTER_FAILED_DB_OPERATION (default: 60000 ms)`: In case of failure during a DB operation, execution between retries is paused for a while.
``
### API
There is a little debugger API available. Please check `app.js` to see how it works.

### Model
#### prefixes
```
  PREFIX mu: <http://mu.semte.ch/vocabularies/core/>
  PREFIX task: <http://redpencil.data.gift/vocabularies/tasks/>
  PREFIX dct: <http://purl.org/dc/terms/>
  PREFIX prov: <http://www.w3.org/ns/prov#>
  PREFIX nie: <http://www.semanticdesktop.org/ontologies/2007/01/19/nie#>
  PREFIX ext: <http://mu.semte.ch/vocabularies/ext/>
  PREFIX oslc: <http://open-services.net/ns/core#>
  PREFIX cogs: <http://vocab.deri.ie/cogs#>
  PREFIX adms: <http://www.w3.org/ns/adms#>
```

#### Job
The instance of a process or group of processes (workflow).

##### class
`cogs:Job`

##### properties

Name | Predicate | Range | Definition
--- | --- | --- | ---
uuid |mu:uuid | xsd:string
creator | dct:creator | rdfs:Resource
status | adms:status | adms:Status
created | dct:created | xsd:dateTime
modified | dct:modified | xsd:dateTime
jobType | task:operation | skos:Concept
error | task:error | oslc:Error

#### Task
Subclass of `cogs:Job`

##### class
`task:Task`

##### properties

Name | Predicate | Range | Definition
--- | --- | --- | ---
uuid |mu:uuid | xsd:string
status | adms:status | adms:Status
created | dct:created | xsd:dateTime
modified | dct:modified | xsd:dateTime
operation | task:operation | skos:Concept
index | task:index | xsd:string | May be used for orderering. E.g. : '1', '2.1', '2.2', '3'
error | task:error | oslc:Error
parentTask| cogs:dependsOn | task:Task
job | dct:isPartOf | rdfs:Resource | Refer to the parent job
resultsContainer | task:resultsContainer | nfo:DataContainer | An generic type, optional
inputContainer | task:inputContainer | nfo:DataContainer | An generic type, optional


#### DataContainer
A generic container gathering information about what has been processed. The consumer needs to determine how to handle it.
The extensions created by this service are rather at hoc, i.e. `ext:` namespace
See also: [job-controller-service](https://github.com/lblod/job-controller-service) for a more standardized use.

##### class
`nfo:DataContainer`

##### properties

Name | Predicate | Range | Definition
--- | --- | --- | ---
uuid |mu:uuid | xsd:string
subject | dct:subject | skos:Concept | Provides some information about the content
hasDeltafileTimestamp | ext:hasDeltafileTimestamp | timestamp from the processed deltafile
hasDeltafileId |ext:hasDeltafileId | id from the processed deltafile
hasDeltafileName |ext:hasDeltafileName | Name on disk about the processed deltafile

#### Error

##### class
`oslc:Error`

##### properties
Name | Predicate | Range | Definition
--- | --- | --- | ---
uuid |mu:uuid | xsd:string
message | oslc:message | xsd:string

### Data flow
#### Initial sync
Finds the latest dcat:Dataset a sync point to ingest. Once done, it proceeds in delta-sync mode.
See also [delta-producer-dump-file-publisher](https://github.com/lblod/delta-producer-dump-file-publisher).

#### Delta-sync
At regular intervals, the service will schedule a sync task. Execution of a task consists of the following steps:

1. Retrieve the timestamp to start the sync from
2. Query the producer service for all diff files since that specific timestamp
3. Download the content of each diff file
4. Process each diff file in order

During the processing of a diff file, the insert and delete changesets are processed.
The behaviour depends on the 'triples-dispatching'-logic, by default we have:

**Delete changeset**
Apply a delete query triple per triple in the graph `INGEST_GRAPH`.

**Insert changeset**
Ingest the changeset in the graph `INGEST_GRAPH`.

If the ingestion of one file fails, the service will block the queued files. The service must process the files in order of publication.

The service makes two core assumptions that must be respected at all times:

1. At any moment, we know that the latest `ext:hasDeltafileTimestamp` timestamp on the resultsContainer of a task OR if not found -because initial sync has been disabled- provided from `DCR_START_FROM_DELTA_TIMESTAMP`
   This reflects the timestamp of the latest delta file that has been completely and successfully consumed.
2. Maximum 1 sync task is running at any moment in time

### Migrating from  [delta-consumer-single-graph-maintainer](https://github.com/lblod/delta-consumer-single-graph-maintainer) to this service
The model to keep track of the processed data changed.
It is only required to provide `DCR_START_FROM_DELTA_TIMESTAMP` as a correct starting point.

Migrating is not required but advised. The following options are:

#### Cleaning up previous tasks
In case it doesn't really make sense to keep this information.

```
PREFIX ext: <http://mu.semte.ch/vocabularies/ext/>
DELETE {
  GRAPH ?g {
    ?s ?p ?o.
  }
}
WHERE {
  ?s a ext:SyncTask.
  GRAPH ?g {
    ?s ?p ?o.
  }
}
```
#### Migrate ext:SyncTask to cogs:Job
TODO...

## Adding custom triples-dispatching
### flow
By default, the service will look first for custom triples-dispachting, and if not found, load the default behaviour.
### file and folder names
Refer to `./triples-dispatching/example-custom-dispatching` for the naming convention of the files.
A folder `/config/custom-dispatching` should be mounted
### API
#### initial sync
A function with signature `dispatch(lib, data)` should be exported. The documentation states:
```
 * @param { mu, muAuthSudo } lib - The provided libraries from the host service.
 * @param { termObjects } data - The fetched quad information, which objects of serialized Terms
 *          [ {
 *              graph: "<http://foo>",
 *              subject: "<http://bar>",
 *              predicate: "<http://baz>",
 *              object: "<http://boom>^^<http://datatype>"
 *            }
 *         ]
 * @return {void} Nothing
```
#### delta sync
A function with signature `dispatch(lib, data)` should be exported. The documentation states:
```
 * @param { mu, muAuthSudo } lib - The provided libraries from the host service.
 * @param { termObjectChangeSets: { deletes, inserts } } data - The fetched changes sets, which objects of serialized Terms
 *          [ {
 *              graph: "<http://foo>",
 *              subject: "<http://bar>",
 *              predicate: "<http://baz>",
 *              object: "<http://boom>^^<http://datatype>"
 *            }
 *         ]
 * @return {void} Nothing
 */
```
#### Extra notes
- The API is deliberately limited. We provide a minimal toolset to CRUD the database, which limits the chances we don't regret our choices later and break existing implementations.
Hence, only `mu, muAuthSudo ` are provided for now. Adding libraries should be done under careful consideration. (It is still extendable)

- Custom triples-dispatching allow their environment variables. Make sure to respect the convention, to differentiate core from custom.
As an inspiration, check `single-graph-dispatching` for complex dispatching rules.

- Currently, `import` statements don't work in custom triples-dispatching. Hence you will have to stay in the `require` world.
