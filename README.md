# AIM-Milvus-KB
Knowledge Base for Milvus

#### Tips

````
In the milvus.yaml, add this configuration item:
quotaAndLimits:
  limits:
    maxOutputSize: 104857600

The default is 104857600 bytes, you can change it to a larger value.
````
#### How to use dates

````

https://github.com/milvus-io/milvus/discussions/35939

````

#### Dynamic vs Predefined Fields

````

Is predefined fields more optimized than dynamic fields?


yhmo
A predefined field is stored as column-based files. The dynamic field is stored as a list of JSON strings. It is easier to read/parse column-based file than JSON strings.

Currently, Milvus supports scalar index for the predefined fields, but doesn't support index for dynamic field.

In practice, if you want to do filtering-search on a property, you can define the property as a scalar field so that the filtering-search can use scalar index to improve search performance.
````


#### FP32

````
JeremyZhu — Yesterday at 11:57 PM
Thank you for your insightful observation and question. Let me explain the reasoning behind this and address your concerns:

Data Source Precision:
The OpenAI embedding models indeed generate vectors with float64 (double) precision. This is why you're seeing the double type in the train.parquet files. You can find this information in OpenAI's official documentation: https://platform.openai.com/docs/guides/embeddings/what-are-embeddings

Storage and Processing Considerations:
While the original data is in float64, many vector databases and ANN (Approximate Nearest Neighbor) search libraries typically use float32. This is done for performance and storage efficiency reasons. For instance, Milvus supports float32 as its highest precision.
Precision Conversion:
When these vectors are converted from float64 to float32 for database storage, there is indeed some loss of precision.
Impact Assessment:
Although there is a loss in precision, this loss is generally acceptable for most ANN search applications. float32 provides about 6-9 decimal digits of precision, which is usually sufficient for most similarity search tasks.
Performance Trade-off:
Using float32 instead of float64 can offer significant performance advantages:
   Halved memory usage
   Faster computation speeds
   Better cache efficiency
Practical Application:
For OpenAI's 1536-dimensional embeddings, using float32 for storage typically doesn't significantly impact search quality in most use cases.
````
![image](https://github.com/user-attachments/assets/92a0fee2-4aca-4652-b241-4db6dd4ec435)


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


***LLama-Index Tips***
````
index = VectorStoreIndex.from_documents(documents)    <- for llama-index this uses their in-memory only Milvus that gets flushed everytime.   all data removed.

```

* https://discord.com/channels/1160323594396635310/1276169672869417071
* https://docs.llamaindex.ai/en/stable/module_guides/indexing/vector_store_index/
* https://docs.llamaindex.ai/en/stable/examples/vector_stores/MilvusIndexDemo/
* https://docs.llamaindex.ai/en/stable/module_guides/indexing/vector_store_guide/.


***Milvus Cloud Storage***

Azure:   Azure Blob Storage only.  ADLS1/2 not supported yet.

https://milvus.io/docs/abs.md


***Milvus Lite Note***

Indexes are not yet implemented.

***Milvus to Minio***

Opens a new connection and performs action when needed, no connection kept open


***Milvus CLI***

milvus-cli doesn't support connecting milvus with TLS / SSL.

https://milvus.io/docs/install_cli.md

***ATTU Hint***

The milvus server is remote, you should add a MILVUS_URL to specify the remote milvus host.
 I think the SERVER_NAME should match the CommonName configured in the certificate.

````
sudo docker run -p 8080:3000 
    -e MILVUS_URL={milvus server IP}:19530
    ......
````

***Milvus Row Count***

The 'Approx Entity Count' is equal to the 'collection.num_entities' in pymilvus. This method quickly pick the number from Etcd. 
As we know, when you insert data, the data firstly is passed to the Pulsar(or local message queue) as write-ahead-log. And the query nodes and data nodes consume the data from the message queue. Data node accumulates data in an in-memory buffer, once the buffer size exceeds a threshold, data node flushes the buffer to be sealed segment.
Only sealed segments are recorded in Etcd. This method only counts the number of rows in sealed segments. Some data maight still in message queue or in the buffer of data node. So, this method is inaccurate. And, this method doesn't count the deleted items.

The accurate way is: https://milvus.io/docs/get-and-scalar-query.md#Advanced-operators
In pymilvus:
````
res = client.query(collection_name=name, filter="", output_fields=["count(*)"])
print(f'row count: {res[0]["count(*)"]}')
```
It is a real query to iterate all the segments including the in-memory buffer to sum up the row count, and deleted items are skipped.

In Attu, if you click load button to load the collection, it will call query(count(*)) to get the accurate row count and show the number in the 'Approx Entity Count' field.

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


***MMAP***

https://milvus.io/docs/manage-collections.md#Set-MMAP


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


***ID Orders***

````
If a primary key is auto-id, when you call the insert() interface, it returns a list of IDs and the order of the IDs is consistent with the order of the vectors.
For example, I insert 3 rows:
data = [
  {"vector": vector_1},
  {"vector": vector_2},
  {"vector": vector_3},
]
ids = client.insert(collection_name=collection_name, data=data)
print(ids)

The returned object that show you the ID list:
{'insert_count': 3, 'ids': [451792818590572750, 451792818590572751, 451792818590572752]}

451792818590572750 is the ID of the vector_1
451792818590572751 is the ID of the vector_2
451792818590572752 is the ID of the vector_3

````
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


#### Issues to Watch

https://github.com/milvus-io/milvus/issues/35389


#### Resources

* https://python.langchain.com/v0.2/docs/integrations/vectorstores/milvus/#per-user-retrieval
* https://milvus.io/docs/performance_faq.md#How-to-set-nlist-and-nprobe-for-IVF-indexes
* https://milvus.io/api-reference/pymilvus/v2.4.x/ORM/utility/get_server_version.md
* https://github.com/milvus-io/bootcamp/blob/master/bootcamp/RAG/readthedocs_zilliz_langchain.ipynb
* https://milvus.io/docs/insert-update-delete.md#Upsert-entities
