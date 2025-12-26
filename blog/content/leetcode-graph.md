+++
title = "LeetCode Graph Question Summary"
date = 2024-11-15
draft = false
[taxonomies]
  tags = ["LeetCode", "C++", "Algorithm", "Graph"]
[extra]
  toc = true
  keywords = "LeetCode, C++, Algorithm, Graph"
+++

## Data Structure

Graph problems rely heavily on how the data is represented. While there are three standard academic ways, LeetCode focuses primarily on adjacency matrices and adjacency lists.

### Adjacency Matrix

This is a 2D array where `matrix[i][j]` represents an edge from node `i` to node `j`.

- **Pros:** Fast lookups ($O(1)$) to check if an edge exists.
- **Cons:** Consumes $O(V^2)$ space, which is inefficient for sparse graphs.
- **Usage:** Best when the number of vertices $V$ is small (e.g., $V \le 1000$).

```cpp
// Example: unweighted graph
// graph[i][j] = 1 means edge exists, 0 means no edge
vector<vector<int>> graph(n, vector<int>(n, 0));
```

### Adjacency List

Each vertex maintains a list of the vertices it connects to. This is the most common representation for interview questions.

- **Pros:** Efficient space complexity $O(V+E)$. Iterating neighbors is fast.
- **Cons:** Slightly slower lookup to check specific edge existence ($O(\text{degree of } V)$).

LeetCode often provides a flattened list of edges (e.g., `[[0,1], [1,2]]`). Convert that into an adjacency list first:

```cpp
// Conversion from edge list to Adjacency List
vector<vector<int>> buildGraph(int n, const vector<vector<int>>& edges) {
    vector<vector<int>> adj(n);
    for (const auto& edge : edges) {
        int u = edge[0];
        int v = edge[1];
        adj[u].push_back(v);
        // adj[v].push_back(u); // Uncomment for undirected graph
    }
    return adj;
}
```

### Incidence Matrix

Rows represent vertices and columns represent edges. This representation is rarely used in competitive programming or interviews due to complexity and space requirements.

## Graph Traversal

Traversing a graph involves visiting nodes while handling potential cycles (unless the graph is guaranteed to be a DAG). We typically use a visited array (or hash set) to track nodes we have already processed.

### Depth-First Search (DFS)

DFS goes as deep as possible before backtracking. It is implemented using recursion or an explicit stack.

```cpp
void dfs(int u, const vector<vector<int>>& adj, vector<bool>& visited) {
    visited[u] = true;
    // Process node u
    for (int v : adj[u]) {
        if (!visited[v]) {
            dfs(v, adj, visited);
        }
    }
}
```

**Common use cases:**

- Detecting cycles.
- Finding connected components (e.g., "Number of Islands").
- Pathfinding (not necessarily the shortest path).

### Breadth-First Search (BFS)

BFS explores neighbors layer by layer and is implemented with a queue.

```cpp
void bfs(int startNode, const vector<vector<int>>& adj, int n) {
    vector<bool> visited(n, false);
    queue<int> q;

    visited[startNode] = true;
    q.push(startNode);

    while (!q.empty()) {
        int u = q.front();
        q.pop();
        // Process node u

        for (int v : adj[u]) {
            if (!visited[v]) {
                visited[v] = true;
                q.push(v);
            }
        }
    }
}
```

**Common use cases:**

- Shortest path in unweighted graphs.
- Level-order traversal.

## Topological Sorting

Topological sorting is a linear ordering of vertices in a Directed Acyclic Graph (DAG) such that for every edge $u \to v$, $u$ appears before $v$. If the graph contains a cycle, topological sorting is impossible.

### Kahn's Algorithm (BFS approach)

Steps:

1. Calculate the in-degree (number of incoming edges) for every node.
2. Push all nodes with in-degree == 0 into a queue.
3. While the queue is not empty:
   - Pop a node `u` and add it to the sorted result.
   - For each neighbor `v` of `u`, decrement `v`'s in-degree; if it becomes 0, push `v` to the queue.
4. If the result size $\neq$ total nodes, the graph has a cycle.

```cpp
vector<int> topologicalSort(int n, const vector<vector<int>>& adj) {
    vector<int> inDegree(n, 0);
    for (int u = 0; u < n; u++) {
        for (int v : adj[u]) {
            inDegree[v]++;
        }
    }

    queue<int> q;
    for (int i = 0; i < n; i++) {
        if (inDegree[i] == 0) q.push(i);
    }

    vector<int> result;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        result.push_back(u);
        for (int v : adj[u]) {
            inDegree[v]--;
            if (inDegree[v] == 0) q.push(v);
        }
    }

    if (result.size() != (size_t)n) return {};
    return result;
}
```

**Classic problem:** 207. Course Schedule

## Union-Find (Disjoint Set Union)

Union-Find tracks elements partitioned into disjoint sets. It's efficient for detecting cycles in undirected graphs, connecting components (e.g., Kruskal's MST), and "Number of Provinces" problems.

Core operations:

- `find(x)`: Returns the representative (root) of the set containing `x` (path compression).
- `union(x, y)`: Merges the sets containing `x` and `y` (union by rank).

```cpp
class UnionFind {
    vector<int> parent;
    vector<int> rank;
public:
    UnionFind(int n) {
        parent.resize(n);
        rank.resize(n, 0);
        iota(parent.begin(), parent.end(), 0);
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    void unite(int x, int y) {
        int rootX = find(x), rootY = find(y);
        if (rootX == rootY) return;
        if (rank[rootX] > rank[rootY]) parent[rootY] = rootX;
        else if (rank[rootX] < rank[rootY]) parent[rootX] = rootY;
        else { parent[rootY] = rootX; rank[rootX]++; }
    }

    bool isConnected(int x, int y) { return find(x) == find(y); }
};
```
