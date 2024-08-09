# AIM-Milvus-KB
Knowledge Base for Milvus


#### Features

***Docker Compose Requirements***

https://milvus.io/docs/prerequisite-docker.md

Make sure you have enough and fast enough RAM and disk.

In our document, the pre-requirement mentioned etcd ought to be deployed on NVMe SSD:


***Fast Insert***  by yhmo | Zilliz

There are two major interfaces that milvus ingests data:
insert()
This method accepts column-based or row-based data, and the data is passed through RPC channel from SDK client to Milvus server. Proxy node of the Milvus server receives the data, and passes the data into Pulsar/Kafka, then data nodes and query nodes consume the data from Pulsar/Kafka, and eventually persist the data into segments stored in S3.
The size of each insert request is limited to 64MB by default.

bulkinsert()
This method only accepts some relative paths of S3. Users provide the files with a certain format and upload the files to S3. Then call bulkinsert interface to pass the S3 paths to Milvus server. Milvus server tells data nodes to read the files from S3. (Note: the files must be uploaded into the bucket which the Milvus server can access)
The bulkinsert tasks in data nodes are asynchronously. Data node reads a file from S3, constructs segments from the data, and calls index node to build index for the new segments.

File format can be JSON, Numpy or Parquet. Parquet is recommended.
In python SDK, the method is utility.do_bulk_insert(). 
In Java SDK, the method is MilvusClient.bulkInsert().
The size of each data file is limited to 16GB. 

![image](https://github.com/user-attachments/assets/1c58efe5-9280-4703-b59a-f8876b909826)

https://milvus.io/docs/prepare-source-data.md


***Rerankers***

https://milvus.io/docs/rerankers-overview.md

Example:    https://github.com/run-llama/llama_index/blob/main/llama-index-integrations/vector_stores/llama-index-vector-stores-milvus/llama_index/vector_stores/milvus/base.py

https://docs.llamaindex.ai/en/stable/examples/vector_stores/MilvusHybridIndexDemo/

Cross Encoder

CrossEncoderRerankFunction` class. This functionality allows you to score the relevance of query-document pairs effectively.

https://milvus.io/docs/rerankers-cross-encoder.md



***Milvus Cloud Storage***

Azure:   Azure Blob Storage only.  ADLS1/2 not supported yet.

https://milvus.io/docs/abs.md


***Milvus Lite Note***

Indexes are not yet implemented.


***Milvus CLI***

milvus-cli doesn't support connecting milvus with TLS / SSL.

https://milvus.io/docs/install_cli.md


***Clean Restart***

Similar issue: https://github.com/milvus-io/milvus/issues/30845

I don't know what is the root cause. From the comment of this issue, looks like there are two options:
upgrade the milvus to >= 2.3.10, I think you can upgrade it to the latest version of v2.3. The latest version is v2.3.20

**delete WAL files.**

the directory tree of a standalone is like this:

The wal files are under the volumes/milvus/rdb_data and rdb_data_meta_kv. 

Stop the server, delete the two directories and restart the server.
Image



***Backup***

The official backup tool is milvus-backup: 

* https://milvus.io/docs/milvus_backup_overview.md
* https://github.com/zilliztech/milvus-backup/tree/main

The import interface is:
https://milvus.io/api-reference/pymilvus/v2.4.x/ORM/utility/do_bulk_insert.md

* It  supports JSON/Numpy/Parquet files with specified format.

Bulk Writer is a tool for generating Parquet files with correct format for milvus bulk insert interface.
https://milvus.io/api-reference/pymilvus/v2.4.x/DataImport/LocalBulkWriter/LocalBulkWriter.md


***Multi Vector Fields***

v2.4.x supports multi vector fields.
https://milvus.io/docs/glossary.md#Multi-Vector

The max number of vector fields in a collection is configured in the milvus.yaml, default value is 4, you can change  
it to no more than 10.
https://milvus.io/docs/configure_proxy.md#proxymaxVectorFieldNum

You can conduct hybrid search accross multi vector fields.
https://milvus.io/docs/multi-vector-search.md


#### Common Questions and Solutions

* Bug in Milvus 2.3, Solution is to upgrade
  https://github.com/milvus-io/milvus/issues/30845

* How to connect LangChain to Milvus
  https://github.com/milvus-io/milvus/discussions/35249

* Benchmark
  https://github.com/zilliztech/VectorDBBench
  
* Can milvus Standalone 2.3.0 be directly upgraded to 2.4.4 using docker? Whether to use migration data

If you are using docker-compose, 3 steps to upgrade:

* "docker-compose down" to remove the old containers. Milvus data will be kept in a folder "volumes" under the same path of the docker-compose.yaml

* edit the docker-compose.yaml, change the image name
standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.4.4
  
* "docker-compose up -d" to create new containers with 2.4.4. The new milvus container will reuse the data in the "volumes"


#### Settings

***Milvus Proxy Node***

* the maximum connections per a Milvus Proxy Node is configured by proxy.maxConnectionNum, default is 10000. 

* The connection TTL for the server is configured by proxy.connectionCheckIntervalSeconds, default is 120 seconds, but the client SDK will try reconnect transparently.


#### Tutorials

https://github.com/milvus-io/bootcamp/tree/master/bootcamp/tutorials/quickstart/apps/multimodal_rag_with_milvus

https://milvus.io/docs/integrate_with_memgpt.md



#### Milvus Under Construction

The Helm chart does not currently support specifying a secret to provide accessKey and secretKey. 


 It's a reasonable feature to return the scores (it preferable to use term 'score' instead of 'distance' in this case) for original sub search result. We filed a feature request here: https://github.com/milvus-io/milvus/issues/35062


In PyMilus for search, To support asyncIO, PyMilvus need to replace the underlying grpc channel to aio_channel, so currently there's no work around for pymilvus to fit in python asyncIO problem.

https://github.com/milvus-io/milvus/issues/34150



#### Tips


***Long Load Times***

TLDR:  Run Milvus 2.4 (most recent), have fast network between you and S3.   Know your collection size.


why does milvus take hours to load a collection into memory? We get frequest crashes on the milvus standalone container, logs are very unhelpful. Shouldnt loading a collection into memory take seconds if not minutes? Index files are already written on s3/minio correct?

After you call collection.load(), the query nodes will read index data from the S3/minio into memory. 

In the recent versions(v2.3 or v2.4), if the dataset is huge(100M ~ 1B), normally, the time to load the collection is less than 30 minutes.

In the old versions(v2.1, v2.2), for huge dataset, it might take hours to load the collection.

So, if your milvus is not an old version, it could be a bug that takes hours to load the collection. Another possibility is the network bandwidth between query nodes and s3 is poor.


***Multitenancy with Partition Keys***

````
# 2. Create a collection
schema = MilvusClient.create_schema(
    auto_id=False,
    enable_dynamic_field=True,
    partition_key_field="color",
    num_partitions=16 # Number of partitions. Defaults to 16.
)
````

You need to define a **partition_key_field** with the **num_partitions** being optional as it has a good default.


* https://milvus.io/docs/multi_tenancy.md

* https://milvus.io/docs/use-partition-key.md

* https://github.com/milvus-io/milvus/discussions/26320

* https://zilliz.com/blog/sharding-partitioning-segments-get-most-from-your-database

* https://docs.zilliz.com/docs/use-partition-key



***Dynamic Fields***

https://milvus.io/docs/enable-dynamic-field.md



***Multi-Vector Search with Ten Fields***

MaxVectorFieldNum probably 10.

https://github.com/milvus-io/milvus/blob/5037497929168510fc2c505a3678d361d4343fa3/pkg/util/paramtable/component_param.go#L1116

https://milvus.io/docs/multi-vector-search.md

Frank | Zilliz — 05/08/2024 2:58 PM

You can theoretically go above 10, but I guess you might run into performance issues. There is an upper limit to the number of fields in a collection schema as well via proxy.maxFieldNum. What are you looking to build? We might be able to find a workaround.


***Use the Latest Version***

Older version allowing query on deleted data.   Fixed.

https://github.com/milvus-io/milvus/issues/33247

***Import JSON***

https://milvus.io/docs/import-data.md#Import-data

You can also use  bulkinsert API, which can consume s3 json files directly

***Add your own extra fields with JSON, no schema***

milvus support dynamic field, so you can skip pre-define schema for scalar fields. but you need to define at lease two fields, one for primary key, another for embeddings. and you can reference the dynamic field usage

https://milvus.io/docs/enable-dynamic-field.md#Enable-Dynamic-Field


***How to Count Rows***

https://milvus.io/docs/get-and-scalar-query.md#Count-entities

***Processing Large Datasets***

If you are not concerned about search latency, you can employ the Iterator feature ( https://milvus.io/docs/with-iterators.md ) to process large datasets. For instance, to handle a dataset of 1 million records, you can set up an iterator that iterate for 1000 times with batch size of 1000 and limit of 1000,000.

****Obtain Recent Logs***

In the majority of instances, the script deployments/export-log/export-milvus-log.sh can be employed to obtain the most recent logs. The log entries associated with segment_loader provide insights into the segment loading process within the QueryNode. 


***Bandwidth Issues***

Should you encounter a bandwidth-related issue, the logs may indicate a prolonged download duration. To address this, it is recommended to consult with your storage service provider, such as Amazon S3, for tailored support and solutions.


***Add Client Connection to a Collection*** 

https://github.com/milvus-io/pymilvus/issues/2044


***File Lock Issue for some in Milvus Lite***

https://github.com/milvus-io/milvus-lite/issues/195

***Milvus Architecture***

Yes. Minio will be used. Minio acts as a mediator in between Milvus and S3. Minio keeps track of all the data in S3, across all its servers(if in case there are multiple miniò pods). So miniò is required for Milvus to store object data into S3

***Milvus Delete Note***

**delete()** is an async operation.

When you call **delete()**, a delete request is received by proxy node of milvus. The proxy node sends the request to Pulsar/Kafka. Then the data node and query node consume the request from Pulsar/Kafka asynchronously. The delete operation is complete only after the delete request is applied to the correct segment.

In **pymilvus**, there are two methods to get the row number of a collection.

**collection.num_entities**, this method gets the row number of each segment from Etcd. But this num_enties doesn't count deleted items. So, the num_entities doesn't change even you delete some items.

collection.query(expr=", output_fields=["count(*)"], this is a pure query action. It counts the deleted items. So, after a delete operation is completed, this method will return correct row number.

No need to call create_index again.

Once you have called create_index(), the index type is already specified. Milvus will build new index for new data. In fact, each segment has an independent index. Once a new segment is generated, milvus will automatically build index for the new segment.

If you want to change index type or index parameter, you need to release the collection and call drop_index() to drop the old index. Then call create_index() to create the new index.

No need to manually call compaction. Milvus will automatically compact the data by itself.


BinLog Import
https://github.com/milvus-io/pymilvus/pull/2222/files


***Commit/Rollback***

As of Milvus 2.4.4:  No, milvus doesn't have commit/rollback operations.


#### Resources

* https://python.langchain.com/v0.2/docs/integrations/vectorstores/milvus/#per-user-retrieval
* https://milvus.io/docs/performance_faq.md#How-to-set-nlist-and-nprobe-for-IVF-indexes
* https://milvus.io/api-reference/pymilvus/v2.4.x/ORM/utility/get_server_version.md
* https://github.com/milvus-io/bootcamp/blob/master/bootcamp/RAG/readthedocs_zilliz_langchain.ipynb
* https://milvus.io/docs/insert-update-delete.md#Upsert-entities
