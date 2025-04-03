# **RFC0012 for Presto**

## CLP connector

Proposers

* Rui Wang (@wraymo), YScope
* Devin Gibson (@gibber9809), YScope
* Xiaochong Wei (@anlowee), YScope
* Yu(Jack) Luo (@jackluo923), YScope
* Kirk Rodrigues (@kirkrodrigues), YScope

## Related Issues

* [y-scope/clp#78](https://github.com/y-scope/clp/issues/780)
* [yscope-clp Zulip chat](https://yscope-clp.zulipchat.com/#narrow/channel/364020-feature-requests/topic/Trino.20integration)

## Summary

We propose adding a connector and one or more UDFs to query specially-compressed schemaless log data directly through Presto and Prestissimo via a schema-oriented interface. Several users generate petabytes of log data per day and wish to store and query it efficiently. [CLP] is an open-source log management system that can significantly compress this log data and efficiently query it without full decompression. A Presto connector for CLP would allow Presto users to efficiently query their log data and correlate it with other data sources.

## Background

CLP is an open-source log management system that can significantly compress log data and efficiently query it without full decompression. Today’s internet-scale companies (e.g., [eBay][ebay-logging], [Uber][clp-osdi24-paper], etc.) generate petabytes of log data per day, which is useful for a variety of tasks including debugging, security auditing, trend analysis, and so on. Log data here refers to a collection of log records, where each log record is a collection of key-value pairs (kv-pairs)—e.g., a JSON object. We define a record’s schema as the set of key and value-type pairs in the record. Compared to other records, log records typically have dynamic schemas, wherein each record may have a different set of keys, and the value mapped to a particular key may change its type between records (i.e., values can have polymorphic types). As CLP’s [research (OSDI) paper][clp-osdi24-paper] explains, storing these records in conventional log management or database systems is challenging due to the volume of data and the records’ dynamic schemas. For instance, most databases (and formats like Parquet) require a stable schema, so they can only store such records using two sets of columns—a set of columns for key & value-type pairs that exist in all records, and a JSON column for all other key & value-type pairs. Since log records have dynamic schemas, the JSON column often results in high storage overhead and non-ideal query performance. Even log management tools like Elasticsearch incur high storage costs and can’t handle values with polymorphic types, leading to significant [management overhead][uber-clickhouse-logging-blog]. In contrast, as shown in the research paper, CLP can store these records with minimal management overhead, it achieves significantly lower storage overhead, and it can efficiently query the compressed records.

Although log records have dynamic schemas, CLP still exposes a table-like interface to users. In a typical deployment, each service/application’s logs are collected, aggregated, and then compressed into what CLP calls a dataset (akin to a relational database table).CLP archives. Users can then query log records in the dataset using a [flavour][clp-kql-docs] of the Kibana Query Language (KQL). A KQL query is a combination of conditions (predicates) where:

- each predicate filters for matching kv-pairs;
- keys are specified relative to the root of the record; and
- the values to filter for are primitives.

For example, consider the log records in Table 1 and the KQL query (level: "INFO" OR level <= 3) AND attr.service: "*DonorService*". The query has three predicates joined by boolean operators. Notice how one of the predicates is for a nested kv-pair, attr.service, where the key is specified relative to the root of the record. Also notice that query filters for primitives values (as opposed to filtering for objects or arrays). This query would match records 1 & 3 in Table 1. Note that unlike a relational database table, CLP supports values with polymorphic types, so queries can include the same key with different value types (e.g., level in the example query).

| **#** | **Log Record**                                                      |
|-------|---------------------------------------------------------------------|
| 1     | `{"level": "INFO", "attr": {"service": "SharedSplitDonorService"}}` |
| 2     | `{"level": "DEBUG", "msg": "Ran initializers"}`                     |
| 3     | `{"level": 3, "attr": {"service": "TenantMigrationDonorService"}}`  |

**Table 1:** Example log records.

For each dataset, CLP stores the compressed log files in a series of archives and queries them using a typical scatter-gather execution model. Within each archive, CLP stores records using columnar storage, where each column corresponds to a particular key (relative to the root of the record) and primitive value-type pair. Each archive is independent, making it the ideal unit of parallelism for searches. Thus, when a user submits a query, CLP will distribute metadata about each archive (e.g., its path) and the query to the search workers. As a worker executes the query, it will send its results to some output. Since a dataset may have hundreds of thousands of archives, CLP also uses a database to store metadata about the records in each archive, and uses this metadata to narrow the set of archives to search (archive pruning) before distributing them to the search workers. For instance, the metadata store may contain the time range of the records in each archive. More details about CLP’s design and implementation are available in the [research paper][clp-osdi24-paper].

Note that in practice, different users may have their own implementation of a CLP metadata database, with different granularities of metadata. For instance, one user may use MySQL with a table to store metadata about each archive—i.e., archive-level metadata—including the time range of records within each archive. Another user may use Pinot with a table to store metadata about each compressed log file—i.e., file-level metadata—including the time range of records within each file, and the archive that contains each file. The former implementation can only be used to prune irrelevant archives, but the latter, being more fine-grained, can also be used to perform filtering within the archive.

The main benefit of a CLP connector is to allow Presto users to query CLP-compressed schemaless log data and to correlate it with their other data sources through a schema-oriented interface. In addition, the proposed implementation would allow users to use Presto as the engine for querying CLP log data, eliminating the need to run a separate CLP search cluster. As the referenced issues demonstrate and from private discussions with users, we’ve seen significant interest for such a connector.

> [!NOTE]
>  For readers who are interested, CLP is an umbrella term that encompasses a subsystem for compressing structured logs (clp-json/clp-s) and another for compressing unstructured logs (clp-text/clp). This proposal is primarily concerned with the first subsystem, but in time, both subsystems will be merged into one.

### [Optional] Goals

### [Optional] Non-goals

## Proposed Implementation

How do you intend to implement the feature? This section can be as detailed as possible with large subsections of its own, or may be a few sentences depending on the scope of the feature proposed. Explain the design in enough detail for existing users/contributors to understand. Design should include all the corner cases you can think of. Feel free to include any new SPI method signatures, class hierarchies or system contracts here. It is recommended to mention any methods, variables, classes, or SQL language additions which you think are needed to provide a broader view of the code changes. Please mention/describe the below on a high level -

1. What modules are involved
2. Any new terminologies/concepts/SQL language additions
3. Method/class/interface contracts which you deem fit for implementation.
4. Code flow using bullet points or pseudo code as applicable
5. Any new user facing metrics that can be shown on CLI or UI.

## [Optional] Metrics

How can we measure the impact of this feature?

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Test Plan

How do we ensure the feature works as expected? Mention if any functional tests/integration tests are needed. Special mention for product-test changes. If any PoC has been done already, please mention the relevant test results here that you think will bolster your case of getting this RFC approved.

[CLP]: https://github.com/y-scope/clp
[clp-osdi24-paper]: https://www.usenix.org/system/files/osdi24-wang-rui.pdf
[clp-kql-docs]: https://docs.yscope.com/clp/main/user-guide/reference-json-search-syntax.html
[ebay-logging]: https://www.elastic.co/elasticon/conf/2018/sf/monitoring-anything-and-everything-with-beats-at-ebay
[uber-clickhouse-logging-blog]: https://www.uber.com/blog/logging/
