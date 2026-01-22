# Connected Components in BigQuery

Most people don’t think of BigQuery as a place to run graph algorithms. However, in theory,
graphs can be represented by adjacency lists, which are two-column tables (`src_node`,
`dst_node`) well suited for BigQuery. Moreover, due to its scaling capability, BigQuery is
great for large-scale graph analysis.

In this post, we will go through how to compute connected components in an **undirected
graph** and assign connected component IDs, all using BigQuery. The methodology could
also work for any SQL engine that supports iterations.
