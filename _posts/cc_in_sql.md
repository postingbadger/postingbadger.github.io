# Connected Components in BigQuery

Most people don’t think of BigQuery as a place to run graph algorithms. However, in theory,
graphs can be represented by adjacency lists, which are two-column tables (`src_node`,
`dst_node`) well suited for BigQuery. Moreover, due to its scaling capability, BigQuery is
great for large-scale graph analysis.
