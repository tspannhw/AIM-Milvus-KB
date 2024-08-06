# AIM-Milvus-KB
Knowledge Base for Milvus


#### Features

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


#### Milvus Under Construction

The Helm chart does not currently support specifying a secret to provide accessKey and secretKey. 




