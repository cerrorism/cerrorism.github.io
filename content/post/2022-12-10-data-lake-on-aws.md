---
title: Create a simple data lake on AWS (1)

date: 2022-12-10
---

While the web industry is generating more and more data, the need for data
storage and processing is increasing. The data volume is growing rapidly, which
makes it difficult to store and process data in a single machine. In this case,
we need to use a distributed storage system to store and process data, which is
usually called a data lake.

There are many data lake solutions on the market, such as Databricks, Snowflake,
and Cloudera. Once you purchase a data lake solution, you don't need to worry
about the underlying storage and processing system, unless you are the unlucky
one who is part of DevOps and needs to maintain the data lake system. But in
case your company doesn't have a data lake solution, and are using AWS for
other scenarios already, you can use AWS services to build your own data
lake. It will of course less powerful than a data lake solution I mentioned
above, but it's still a good solution for small and medium-sized companies.

## The overall architecture

A best practice for building a data lake is to split the data storage and
the data processing. Most of the data processing systems are designed to
work with a "native" storage system, such as HDFS for Hadoop. But you never
know what is the best storage system for your data lake until the data
arrives to you one day. And you never know what kind of processing you need
for your data. So it's better to separate the storage and processing, and
let different systems work on their own strengths.

Another reason to decouple the storage and processing is that you can easily
consume data from different sources. For example, you can read logs from a
plain text storage, and read structured data from a relational database. For
some realtime queries, you can also read data from the production database
or even from a message queue.

For our simple data lake, we will use AWS S3 as the storage system, and use
Glue to store the metadata of the data. We can use Athena, EMR, Notebook, or
Redshift to query the data, depending on the complexity of the query.

#### The storage: S3

Why S3? Because it's cheap, and it's easy to use. You can use S3 to store
TB-level data for a few dollars per month. And you can use S3 to store any
kind of data, including structured data, unstructured data, and even binary
data. Of course, if you don't use S3 in the right way, you can easily run into
issues that are hard to debug.

As the name suggested (S3 = Simple Storage Service), S3 is a simple
key-value storage, which won't provide you any advanced features like
database or file system. But it's still a good choice for a data lake, since
we usually don't need a lot of advanced features at all for our storage.
Still, there are something we need to keep in mind when using S3.

1. Don't store a lot of small objects. When you store millions of small
   objects under a single prefix, it will slow down the performance of S3.
   That is because each time S3 needs to read the object, the client will
   send out an HTTP request to S3, which will slow down the performance
   dramatically. It will be even worse if we want to list all the objects
   under a prefix. So it's better to store a lot of small objects under a
   better structure, such as a date-based structure.
2. Please note the read-write consistency. S3 is not a file system, and its
   write operation is so-called "eventual consistency". That means if you
   write a new object to S3, you may not be able to read it immediately or
   show that file in a list operation. (This is outdated since S3 improved
   that to be strong consistent in 2021.
   Check [this news](https://aws.amazon.com/s3/consistency/) for more details.)
3. Don't use S3 as a file system. S3 is not a file system, and it's not
   designed to be a file system. S3 kindly shows you a file system-like
   interface, and structures the objects in a folder-like web-interface. For
   example, if the key of the object is `a/b/c/d.txt`, you can see the
   folders `a`, `b`, and `c` in the frontend, and the file `d.txt` in the
   `c` folder. We can also use the prefix `a/b/c/` to list all the objects
   under that folder. But S3 is not a file system after all. So we can't add
   properties to those folders, or move the files to another folder. Because
   the concept `folder` doesn't exist in S3. It's just a prefix of the
   object keys. The performance of S3 folder operations is also not good.

#### The metadata: Glue

Although it is easy to use S3, but logically it is just a key-value storage,
with a little or no metadata. We can't use S3 to query the data with a
complex criteria. For data processing, the metadata is very important. For
example, if we store a CSV file in S3, we need to know the schema of the
file in order to query the data. We can use Glue to store the metadata of
the files in S3. Glue will either crawl the files in S3, or we can manually
add the metadata to it. Glue will also expose a SQL-like interface to query
the data.

Moreover, Glue can also connect to other data sources, such as relational
databases. In that case, we can join the data from S3 and the data from
other data source and query them together.

#### The processing: Athena, EMR, Notebook, and Redshift

EMR is a managed Hadoop cluster, which can be used to process the data in S3.
IF we want to use Spark to process the data, we can declare a Spark
dependency when creating the EMR cluster. We don't need to worry about the
maintenance of the Spark cluster, since AWS will take care of it. Moreover,
if we store the metadata of the data in Glue, we can use Athena or Redshift
to query the data as well.

## The implementation

We will follow the best practice of "Infrastructure as Code" to implement
the data lake. We will use Terraform to create the infrastructure. The
resources will be created one by one, and the dependencies between the
resources will be managed by Terraform.

We will take care of the security of the data lake as well. If you don't
care about the security (in the beginning, at least), you can skip most of
the content in the following sections.
