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


#### Tips

***Processing Large Datasets***

If you are not concerned about search latency, you can employ the Iterator feature ( https://milvus.io/docs/with-iterators.md ) to process large datasets. For instance, to handle a dataset of 1 million records, you can set up an iterator that iterate for 1000 times with batch size of 1000 and limit of 1000,000.

****Obtain Recent Logs***

In the majority of instances, the script deployments/export-log/export-milvus-log.sh can be employed to obtain the most recent logs. The log entries associated with segment_loader provide insights into the segment loading process within the QueryNode. 


***Bandwidth Issues***

Should you encounter a bandwidth-related issue, the logs may indicate a prolonged download duration. To address this, it is recommended to consult with your storage service provider, such as Amazon S3, for tailored support and solutions.


***Add Client Connection to a Collection*** 

https://github.com/milvus-io/pymilvus/issues/2044



