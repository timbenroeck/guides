# Snowflake Data Loading Guide

## Introduction

This document provides guidance on efficiently loading data into Snowflake. While Snowflake offers various data loading methods, this guide focuses on optimizing performance and cost-effectiveness, primarily using the `COPY INTO <table>` command with files staged in Amazon S3, Azure Blob Storage, or Google Cloud Storage (GCS).

For detailed information on specific commands and features, please refer to the official Snowflake documentation.

## Data Loading Methods

Snowflake supports several methods for data ingestion:

  * **`COPY INTO <table>` Command:** This is a highly versatile and performant method for bulk loading data from files already staged in an internal (Snowflake) or external (Azure, GCS, S3) stage. It allows for massively parallel processing.
      * *Further Reading:* [COPY INTO \<table\> Documentation](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table)
  * **Snowpipe:** Designed for automated and continuous micro-batch loading of data from external stages (GCS, S3, Azure) as new files arrive.
      * *Further Reading:* [Introduction to Snowpipe](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro)
  * **Kafka Connector:** Enables direct data streaming from Apache Kafka topics into Snowflake tables.
      * *Further Reading:* [Snowflake Connector for Kafka Overview](https://docs.snowflake.com/en/user-guide/kafka-connector-overview)
  * **Snowpipe Streaming API:** Provides a low-latency, row-based data ingestion mechanism for near real-time continuous loading.
      * *Further Reading:* [Snowpipe Streaming Overview](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-streaming-overview)

All these methods are engineered to leverage Snowflake's massively parallel processing (MPP) architecture.

## Optimizing Data Loading Performance with the `COPY INTO <table>` Command

The efficiency of the `COPY INTO <table>` command is significantly influenced by factors such as file size, the number of files, and the size of the virtual warehouse used for the operation.

### Key Principle: Parallelism and vCPUs

A fundamental concept to understand is that **Snowflake typically utilizes one virtual CPU (vCPU) to load one data file at a time.** The warehouse size determines the number of available vCPUs:

  * **XS (Extra Small) Warehouse:** 1 node, 8 vCPUs
  * **S (Small) Warehouse:** 2 nodes, 16 vCPUs
  * **M (Medium) Warehouse:** 4 nodes, 32 vCPUs
  * **L (Large) Warehouse:** 8 nodes, 64 vCPUs
  * **XL (Extra Large) Warehouse:** 16 nodes, 128 vCPUs
  * **2XL Warehouse:** 32 nodes, 256 vCPUs
  * **3XL Warehouse:** 64 nodes, 512 vCPUs
  * **4XL Warehouse:** 128 nodes, 1024 vCPUs

**Example Scenario:** Consider a scenario where approximately 22,000 compressed ORC files (average size \~84MB each) needed to be loaded. Using a 4XL warehouse, this volume of data was processed in approximately 12 minutes.

### File Sizing and Splitting Strategy

  * **Single Large Files:** Loading a single, very large file (e.g., 10 GB) will not leverage the full parallelism of a larger warehouse. An XS warehouse (8 vCPUs) will load this single file as quickly as a 4XL warehouse because only one vCPU will be engaged in reading that file. The remaining vCPUs on the larger warehouse would remain idle, leading to inefficient resource utilization.
  * **Optimal File Size:** To maximize parallelism and loading speed, it is recommended to split large datasets into multiple smaller files. A general guideline is to aim for file sizes between **50MB and 100MB** (compressed).
  * **Benefits of Splitting:** If a 10GB dataset is split into one hundred 100MB files, these files can be loaded in parallel. This can result in significantly faster load times compared to loading a single 10GB file. For instance, with 100 files, an XL warehouse (128 vCPUs) could theoretically load all files concurrently, dramatically reducing the overall load time.

### Balancing Performance and Cost: Files per vCPU

While maximum parallelism is achieved by having one file per vCPU, this may not always be the most cost-effective approach. A single file may not fully saturate a vCPU due to I/O wait times.

  * **Recommended Ratio:** For optimal balance between performance and cost, aim for **4-10 files per vCPU**, depending on the file size.
      * For smaller files (e.g., a few megabytes), a higher number of files per vCPU is generally better.
      * For larger files (e.g., \~80-100MB), a lower number of files per vCPU is more efficient.
  * **Node Saturation:** This translates to roughly **30-80 files per node** in a warehouse (since each node has 8 vCPUs).
      * **Example (84MB files):** For files around 84MB, an ideal performance vs. cost ratio would be achieved with 3-4 files per vCPU, which corresponds to 24-32 files per node.

### Warehouse Sizing Considerations

  * **Sufficient vCPUs:** Choose a warehouse size that provides enough vCPUs to process your files in parallel according to your desired file-to-vCPU ratio.
  * **Avoiding Underutilization:** If loading 100 files, an XL warehouse (128 vCPUs) would provide one vCPU per file. However, this would leave 28 vCPUs idle, which is not the most cost-effective strategy.
  * **Scaling Out with Multiple `COPY INTO <table>` Statements:** If a single large warehouse isn't achieving the desired speed or if you wish to further optimize, consider splitting the file set and running multiple `COPY INTO <table>` commands in parallel using separate and appropriately sized warehouses.
      * **Example:** In the scenario with \~22,000 files (resulting in \~21 files per vCPU on a 4XL) taking 12 minutes:
          * Dividing the files between two `COPY INTO <table>` jobs running in parallel on two 4XL warehouses would result in \~11 files per vCPU, potentially reducing the load time to \~6 minutes at the same cost.
          * Using three 4XL warehouses in parallel could further reduce the time to \~3 minutes all at the same cost.

### The "Long Pole" Effect

The total time for a `COPY INTO <table>` job to complete is determined by the vCPU that takes the longest to process its assigned files.

  * **Uneven Distribution:** If you have a warehouse with many vCPUs (e.g., a 4XL with 1024 vCPUs) and you load a number of files slightly exceeding the vCPU count (e.g., 1025 files), one vCPU will be assigned two files while others get one.
  * **Cost Implication:** If these files are large (e.g., 1GB each), the initial 1024 files might finish loading relatively quickly. However, the entire warehouse (and its associated costs) will remain active while waiting for the last vCPU to finish processing its additional file. This can lead to significant costs for idle resources.

## General Recommendations

  * **File Size:** Aim for compressed file sizes between **50MB and 100MB**.
  * **Files per Node:** Target **30-40 files per node** for a good balance of performance and cost-efficiency.
      * Ideally, the number of files should be a multiple of 8 (the number of vCPUs per node, e.g., 32 files per node) to ensure even distribution.
  * **Consistent File Sizes:** Strive for relatively uniform file sizes to ensure a more predictable and even workload distribution across vCPUs.

## Conclusion

By understanding the relationship between file size, file count, warehouse size, and the underlying vCPU architecture, data loading operations in Snowflake using `COPY INTO <table>` and other methods can be significantly optimized for both speed and cost. Strategic file preparation and warehouse selection are key to leveraging Snowflake's parallel processing capabilities effectively.

We hope this guidance proves beneficial for your data loading tasks. For more in-depth information, always refer to the official [Snowflake Documentation](https://docs.snowflake.com/).
