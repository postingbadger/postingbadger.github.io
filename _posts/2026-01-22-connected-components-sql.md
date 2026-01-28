# Connected Components in BigQuery

Most people don’t think of BigQuery as a place to run graph algorithms. However, in theory,
graphs can be represented by adjacency lists, which are two-column tables (`src_node`,
`dst_node`) well suited for BigQuery. Moreover, due to its scaling capability, BigQuery is
great for large-scale graph analysis.

In this post, we will go through how to compute connected components in an **undirected
graph** and assign connected component ID to each node, all using BigQuery. The methodology could
also work for any SQL engine that supports iterations.

---

## What Is a Connected Component?

A connected component is a maximum set of nodes in a graph where every node is
reachable from every other node in that group.

Visually, in an undirected graph, they look like “isolated islands,” illustrated below. Here, nodes A, B, C, D, E belong to the same blue connected component, and node A
and F are not in the same connected component.

![Figure 1: Connected component illustration](/resources/connected_component_sql/connected%20component%20illustration.png)

---

## Intuition

Consider the example graph below with 8 nodes. The goal is to assign each node to one of
the 3 connected components:

- Nodes `{1,2,3,4,5}`
- Nodes `{6,7}`
- Node `{8}`

One way to achieve this is by updating each node’s label iteratively.

At each iteration, every node updates its label to be the minimum value of its neighboring
nodes’ labels. Therefore, labels only ever decrease, and they converge to the minimum ID
in each connected component.

![Figure 2: Connected Component Example](/resources/connected_component_sql/cc_example_iter1to3.gif)

### Iterations

| Node | Iter 0 | Iter 1 | Iter 2 |
|----|----|----|----|
| 1 | 1 | 1 | 1 |
| 2 | 2 | 1 | 1 |
| 3 | 3 | 1 | 1 |
| 4 | 4 | 3 | 1 |
| 5 | 5 | 3 | 1 |
| 6 | 6 | 6 | 6 |
| 7 | 7 | 6 | 6 |
| 8 | 8 | 8 | 8 |

As a result, we find 3 distinct connected component IDs: **1, 6, and 8**.  
A mapping from node → connected component ID is also available.

---

## Implementation in BigQuery

### Step 1: Represent the Graph as an Adjacency List

Here’s how the adjacency list looks for the example graph:

| src_node | dst_node |
|--------|--------|
| 1 | 2 |
| 2 | 1 |
| 1 | 3 |
| 3 | 1 |
| 2 | 3 |
| 3 | 2 |
| … | … |

Note that the adjacency list implicitly represents direction via `src_node` (source) and
`dst_node` (destination). Therefore, an undirected edge is represented bidirectionally
(e.g., `(1,2)` and `(2,1)`).

We refer to this table as `bi_edge`.

---

## Full BigQuery Script

```sql
DECLARE unconverged BOOL DEFAULT TRUE;
DECLARE iter INT64 DEFAULT 0;
DECLARE max_iter INT64 DEFAULT 100;

-- Create bidirectional edge table
CREATE OR REPLACE TEMP TABLE bi_edge AS
WITH edge_table AS (
  SELECT 1 AS src_node, 2 AS dst_node UNION ALL
  SELECT 1 AS src_node, 3 AS dst_node UNION ALL
  SELECT 2 AS src_node, 3 AS dst_node UNION ALL
  SELECT 3 AS src_node, 5 AS dst_node UNION ALL
  SELECT 3 AS src_node, 4 AS dst_node UNION ALL
  SELECT 4 AS src_node, 5 AS dst_node UNION ALL
  SELECT 6 AS src_node, 7 AS dst_node
)
SELECT src_node, dst_node FROM edge_table
UNION ALL
SELECT dst_node, src_node FROM edge_table;

-- Initialize node labels
CREATE OR REPLACE TEMP TABLE node_cc AS
SELECT n AS node_id, n AS label
FROM UNNEST([1,2,3,4,5,6,7,8]) AS n;

-- Iterative label propagation
WHILE unconverged AND iter < max_iter DO
  SET iter = iter + 1;

  CREATE OR REPLACE TEMP TABLE new_node_cc AS
  SELECT
    nc.node_id,
    MIN(
      LEAST(
        nc.label,
        -- nc1.label is NULL when the node has no neighbors
        IFNULL(nc1.label, nc.label)
      )
    ) AS label
  FROM node_cc nc
  LEFT JOIN bi_edge be
    ON nc.node_id = be.src_node
  LEFT JOIN node_cc nc1
    ON be.dst_node = nc1.node_id
  GROUP BY nc.node_id;

  SET unconverged = EXISTS (
    SELECT 1
    FROM node_cc nc
    JOIN new_node_cc nnc
      USING (node_id)
    WHERE nc.label != nnc.label
  );

  CREATE OR REPLACE TEMP TABLE node_cc AS
  SELECT * FROM new_node_cc;

END WHILE;
