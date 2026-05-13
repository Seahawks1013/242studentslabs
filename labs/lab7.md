# Lab 7: Observing Graphs and Graph Algorithms

**CPTR 242 — Sequential and Parallel Data Structures and Algorithms**
Walla Walla University

---

## Overview

In this lab you will **run complete programs and observe their behavior**. There is no coding required. Your job is to read the output carefully, record what you see, and reason about what it means.

By the end you should be able to:

- Explain the difference between adjacency list and adjacency matrix representations and identify which is preferable given graph density
- Trace BFS and DFS on the same graph and describe how the choice of frontier data structure determines traversal order
- Follow Dijkstra's algorithm step by step and explain why it fails on negative-weight edges
- Follow Bellman-Ford step by step and explain how it detects negative-weight cycles
- Compare the time and space complexity of BFS, DFS, Dijkstra, Bellman-Ford, topological sort, Prim's MST, and Floyd-Warshall across graph types and sizes

---

## Background: Vertices, Edges, and the Cost of Representation

Every data structure in this course so far has imposed a strict shape on its data — arrays are sequential, trees have a parent–child hierarchy. A **graph** imposes no shape at all. It is simply a set of **vertices** (nodes) and **edges** (connections between them). Edges may be directed or undirected, weighted or unweighted, and the same pair of vertices may be connected by multiple edges or none at all.

That freedom creates an immediate question: **how do you store a graph in memory?**

Two standard answers exist:

- An **adjacency list** stores, for each vertex, the list of vertices it connects to. Space is O(V + E) — proportional to what actually exists.
- An **adjacency matrix** stores a V × V grid where entry [i][j] holds the edge weight (or 0/∞ if no edge). Space is always O(V²), regardless of how many edges exist.

For a **sparse** graph (E ≪ V²) the list wastes far less memory. For a **dense** graph (E ≈ V²) the matrix offers O(1) edge lookup with no pointer chasing.

The central question: **does the choice of representation change what an algorithm can do, and how much does it cost?**

---

## Setup

All programs are self-contained. Each generates its own graph and prints results directly to the terminal.

**Compiler command for C++ programs:**
```bash
g++ -O0 -std=c++17 -o program program.cpp && ./program
```

> `-O0` disables compiler optimizations so you observe true algorithm behavior. `-std=c++17` is required for structured bindings.

> Some programs will also be in Python.

I encourage you to use a Jupyter notebook for the Python.
You can use regular Python files with the interpreter, but you have to import the inital 8-puzzle base code in every Python search algorithm.

---

## Model 1: Adjacency List vs. Adjacency Matrix — Representation and Memory

Before running any algorithm you need to understand what a graph looks like in memory. This program builds the same undirected weighted graph using both representations, prints each structure, and reports the memory each one occupies at several graph sizes.

### Program 1 — `graph_representations.cpp`

```cpp
#include <iostream>
#include <vector>
#include <iomanip>
using namespace std;

// -------- Adjacency list --------
struct AdjList {
    int V;
    vector<vector<pair<int,int>>> adj;  // adj[u] = {(v, weight), ...}
    AdjList(int v) : V(v), adj(v) {}
    void addEdge(int u, int v, int w) {
        adj[u].push_back({v, w});
        adj[v].push_back({u, w});   // undirected
    }
    void print() const {
        cout << "Adjacency List:\n";
        for (int u = 0; u < V; u++) {
            cout << "  " << u << ": ";
            for (auto [v, w] : adj[u]) cout << "(" << v << ",w=" << w << ") ";
            cout << "\n";
        }
    }
    // Each pair<int,int> = 8 bytes; vector metadata = 24 bytes per row
    size_t memBytes() const {
        size_t total = adj.size() * 24;          // vector headers
        for (auto& row : adj) total += row.size() * sizeof(pair<int,int>);
        return total;
    }
};

// -------- Adjacency matrix --------
struct AdjMatrix {
    int V;
    vector<vector<int>> mat;   // mat[u][v] = weight, 0 = no edge
    AdjMatrix(int v) : V(v), mat(v, vector<int>(v, 0)) {}
    void addEdge(int u, int v, int w) {
        mat[u][v] = w;
        mat[v][u] = w;   // undirected
    }
    void print() const {
        cout << "Adjacency Matrix:\n    ";
        for (int i = 0; i < V; i++) cout << setw(4) << i;
        cout << "\n    " << string(4*V, '-') << "\n";
        for (int u = 0; u < V; u++) {
            cout << setw(2) << u << " |";
            for (int v = 0; v < V; v++) cout << setw(4) << mat[u][v];
            cout << "\n";
        }
    }
    size_t memBytes() const { return (size_t)V * V * sizeof(int); }
};

// Build an identical graph in both representations
void buildGraph(AdjList& al, AdjMatrix& am,
                const vector<tuple<int,int,int>>& edges) {
    for (auto [u, v, w] : edges) {
        al.addEdge(u, v, w);
        am.addEdge(u, v, w);
    }
}

int main() {
    // Small example graph (6 vertices, 7 edges)
    int V = 6;
    vector<tuple<int,int,int>> edges = {
        {0,1,4}, {0,2,2}, {1,2,5}, {1,3,10},
        {2,4,3}, {3,5,7}, {4,5,1}
    };

    AdjList  al(V);
    AdjMatrix am(V);
    buildGraph(al, am, edges);

    al.print(); cout << "\n";
    am.print(); cout << "\n";

    // Memory comparison across graph sizes
    cout << "=== Memory Usage: Adjacency List vs Matrix ===\n\n";
    cout << setw(8)  << "V"
         << setw(10) << "E"
         << setw(22) << "List (bytes)"
         << setw(22) << "Matrix (bytes)"
         << setw(12) << "Ratio"
         << "\n" << string(74, '-') << "\n";

    // (V, E) pairs: sparse, medium, dense
    vector<pair<int,int>> configs = {
        {10,  12}, {10,  45},            // sparse vs dense, small
        {100, 150}, {100, 4950},         // sparse vs dense, medium
        {500, 600}, {500, 124750}        // sparse vs dense, large
    };

    for (auto [v, e] : configs) {
        AdjList  testAL(v);
        AdjMatrix testAM(v);
        // Distribute e edges evenly around a ring + random chords
        for (int i = 0; i < v; i++)
            testAL.addEdge(i, (i+1)%v, 1);
        // Fill up to e edges with stride-2 chords
        for (int added = v, s = 2; added < e && s < v; s++) {
            for (int i = 0; i < v && added < e; i++, added++)
                testAL.addEdge(i, (i+s)%v, 1);
        }
        // Mirror into matrix just for memory accounting
        for (int i = 0; i < v; i++)
            for (auto [nb, w] : testAL.adj[i])
                if (nb > i) testAM.addEdge(i, nb, w);

        size_t lm = testAL.memBytes();
        size_t mm = testAM.memBytes();
        cout << setw(8)  << v
             << setw(10) << e
             << setw(22) << lm
             << setw(22) << mm
             << setw(12) << fixed << setprecision(1)
             << (double)mm / lm << "x\n";
    }

    cout << "\nEdge lookup cost:\n";
    cout << "  Adjacency list:   O(degree(u))  — scan u's neighbor list\n";
    cout << "  Adjacency matrix: O(1)          — direct index mat[u][v]\n";
}
```

### Observation Table 1a — Small Graph Structure

Using the printed output for the 6-vertex graph, fill in the neighbor list for each vertex and the matrix entry at the specified positions.

| Vertex | Neighbors (v, weight) |
|---|---|
| 0 | |
| 1 | |
| 2 | |
| 3 | |
| 4 | |
| 5 | |

| Matrix position | Value | What it means |
|---|---|---|
| mat[0][1] | | |
| mat[1][0] | | |
| mat[0][3] | | |
| mat[3][5] | | |

### Observation Table 1b — Memory Usage

| V | E | List (bytes) | Matrix (bytes) | Ratio (matrix/list) |
|---|---|---|---|---|
| 10  | 12     | | | |
| 10  | 45     | | | |
| 100 | 150    | | | |
| 100 | 4,950  | | | |
| 500 | 600    | | | |
| 500 | 124,750 | | | |

---

### Critical Thinking Questions — Model 1

**Q1.** At V = 100 with E = 150 (sparse), which representation is smaller? At V = 100 with E = 4590, which is smaller? Identify the crossover condition in terms of E and V.

> Your answer:

---

## Model 2: Problem Formulation

### Description
This exercise demonstrates how to formulate a search problem by defining its five key components. Understanding problem formulation is the foundation for applying any search algorithm. We'll use the 8-puzzle as our example domain.

### Key Concepts
- **State**: A configuration of the environment (e.g., positions of tiles in the 8-puzzle)
- **Actions**: Possible moves from a given state (e.g., slide tile up, down, left, right)
- **Transition Model**: The result of applying an action to a state
- **Goal Test**: A function that determines if a state is the goal
- **Path Cost**: The cost of reaching a state (e.g., number of moves)

### Task
Run the code below and observe how a problem is formally defined. Pay attention to:
- How states are represented (what data structure is used?)
- How the `get_actions()` method determines valid moves
- What information the `result()` method returns
- How the goal test works

```python
class EightPuzzle:
    """Represents an 8-puzzle problem."""
    
    def __init__(self, initial_state, goal_state):
        self.initial = initial_state
        self.goal = goal_state
    
    def get_actions(self, state):
        """Returns list of valid actions from this state."""
        actions = []
        blank_pos = state.index(0)  # Find the blank tile
        row, col = blank_pos // 3, blank_pos % 3
        
        if row > 0: actions.append('UP')
        if row < 2: actions.append('DOWN')
        if col > 0: actions.append('LEFT')
        if col < 2: actions.append('RIGHT')
        
        return actions
    
    def result(self, state, action):
        """Returns the state that results from executing action."""
        new_state = list(state)
        blank_pos = state.index(0)
        row, col = blank_pos // 3, blank_pos % 3
        
        # Calculate new position based on action
        if action == 'UP':
            new_pos = (row - 1) * 3 + col
        elif action == 'DOWN':
            new_pos = (row + 1) * 3 + col
        elif action == 'LEFT':
            new_pos = row * 3 + (col - 1)
        elif action == 'RIGHT':
            new_pos = row * 3 + (col + 1)
        
        # Swap blank with tile in new position
        new_state[blank_pos], new_state[new_pos] = new_state[new_pos], new_state[blank_pos]
        return tuple(new_state)
    
    def is_goal(self, state):
        """Tests if state is the goal."""
        return state == self.goal
    
    def display_state(self, state):
        """Pretty print a state."""
        for i in range(0, 9, 3):
            print(f"  {state[i]} {state[i+1]} {state[i+2]}")

# Create a problem instance
initial = (1, 2, 3, 4, 0, 5, 6, 7, 8)  # 0 represents blank
goal = (0, 1, 2, 3, 4, 5, 6, 7, 8)

problem = EightPuzzle(initial, goal)

print("PROBLEM FORMULATION DEMONSTRATION")
print("=" * 50)
print("\nInitial State:")
problem.display_state(initial)

print("\nGoal State:")
problem.display_state(goal)

print("\nAvailable actions from initial state:", problem.get_actions(initial))

print("\nApplying action 'UP':")
new_state = problem.result(initial, 'UP')
problem.display_state(new_state)

print("\nIs new state the goal?", problem.is_goal(new_state))
print("Is goal state the goal?", problem.is_goal(goal))

print("\nPath cost: Each move costs 1 (implicit in our formulation)")
```

### Reflection Questions

**Q2:** In the 8-puzzle, we represent states as tuples of 9 numbers. What is one advantage of this representation compared to using a 2D array or other data structures? Consider computational efficiency, memory allocation, mutability and ease of use.

> Your answer:

---

## Model 3: Breadth-First Search (BFS)

### Description
This exercise demonstrates uninformed breadth-first search, which explores all nodes at depth *d* before exploring nodes at depth *d+1*. BFS uses a FIFO (first-in-first-out) queue to manage the frontier.

### Key Concepts
- **Uninformed Search**: Strategies that have no problem-specific knowledge beyond the problem definition
- **Frontier**: The set of nodes that have been generated but not yet expanded
- **FIFO Queue**: Data structure where the first element added is the first one removed
- **Completeness**: A search algorithm is complete if it always finds a solution when one exists
- **Optimality**: An algorithm is optimal if it finds a solution with the lowest path cost

### Task
Run the code and observe the search process. Focus on:
- The order in which nodes are explored (printed as "Exploring:")
- How the frontier size changes over time
- How many nodes are expanded before finding the goal
- The path returned by BFS

```python
from collections import deque

def bfs_search(problem):
    """Breadth-first search implementation."""
    
    # Node: (state, path_from_initial, cost)
    initial_node = (problem.initial, [], 0)
    
    if problem.is_goal(problem.initial):
        return []
    
    frontier = deque([initial_node])  # FIFO queue
    explored = set()
    nodes_expanded = 0
    max_frontier_size = len(frontier)
    
    print("BREADTH-FIRST SEARCH")
    print("=" * 50)
    
    while frontier:
        # Track max frontier size
        max_frontier_size = max(max_frontier_size, len(frontier))
        
        print(f"\nFrontier size: {len(frontier)}, Explored: {len(explored)}")
        
        # Remove first node from frontier (FIFO)
        state, path, cost = frontier.popleft()
        nodes_expanded += 1
        
        print(f"Exploring node {nodes_expanded} (depth {len(path)}):")
        problem.display_state(state)
        
        explored.add(state)
        
        # Expand the node
        for action in problem.get_actions(state):
            child_state = problem.result(state, action)
            
            if child_state not in explored and child_state not in [n[0] for n in frontier]:
                child_path = path + [action]
                child_cost = cost + 1
                
                if problem.is_goal(child_state):
                    print(f"\n✓ Goal found!")
                    print(f"Total nodes expanded: {nodes_expanded}")
                    print(f"Maximum frontier size: {max_frontier_size}") 
                    print(f"Solution path: {child_path}")
                    print(f"Path length: {len(child_path)}")
                    return child_path
                
                frontier.append((child_state, child_path, child_cost))
    
    print("\n✗ No solution found.")
    print(f"Total nodes expanded: {nodes_expanded}")
    print(f"Maximum frontier size: {max_frontier_size}")  
    return None


initial = (8, 6, 7, 2, 5, 4, 3, 0, 1)
goal    = (1, 2, 3, 4, 5, 6, 7, 8, 0)


problem = EightPuzzle(initial, goal)
solution = bfs_search(problem)
```

### Reflection Questions

**Q3:** BFS explores nodes level by level (all nodes at depth 1, then all at depth 2, etc.). How does this exploration strategy guarantee that BFS finds the optimal solution for problems where all actions have the same cost?

> Your answer:

**Q4:** Observe the frontier size as the search progresses. Why does the frontier grow so rapidly in BFS, and what does this tell you about the space complexity of this algorithm? How would this affect solving larger problems?

> Your answer:

---

## Model 4: Depth-First Search (DFS)

### Description
This exercise demonstrates depth-first search, which always expands the deepest node in the frontier using a LIFO (last-in-first-out) stack. Comparing DFS to BFS reveals important tradeoffs between different uninformed strategies.

### Key Concepts
- **LIFO Stack**: Data structure where the last element added is the first one removed
- **Depth-First Exploration**: Exploring as deeply as possible before backtracking
- **Space Complexity Advantage**: DFS stores fewer nodes in memory than BFS
- **Non-Optimal**: DFS may find a solution but not the shortest one
- **Depth Limiting**: Preventing infinite loops by limiting maximum depth

### Task
Run the code and compare DFS to BFS from Exercise 2. Notice:
- How the exploration order differs from BFS
- The maximum frontier size compared to BFS
- Whether the solution found is optimal (shortest)
- How the depth limit prevents infinite exploration

```python
def dfs_search(problem, max_depth=100):
    """Depth-first search with depth limiting."""
    
    initial_node = (problem.initial, [], 0)
    
    if problem.is_goal(problem.initial):
        return []
    
    frontier = [initial_node]  # LIFO stack (use list as stack)
    explored = set()
    nodes_expanded = 0
    max_frontier_size = 1
    
    print("DEPTH-FIRST SEARCH")
    print("=" * 50)
    
    while frontier:
        max_frontier_size = max(max_frontier_size, len(frontier))
        
        # Remove last node from frontier (LIFO)
        state, path, cost = frontier.pop()
        
        if len(path) > max_depth:
            continue  # Depth limit reached
        
        nodes_expanded += 1
        print(f"\nExploring node {nodes_expanded} (depth {len(path)}):")
        problem.display_state(state)
        
        if problem.is_goal(state):
            print(f"\n{'='*50}")
            print(f"✓ Goal found! Total nodes expanded: {nodes_expanded}")
            print(f"Maximum frontier size: {max_frontier_size}")
            print(f"\nGoal state reached:")
            problem.display_state(state)
            print(f"\nSolution path: {path}")
            print(f"Path length: {len(path)}")
            return path
        
        explored.add(state)
        
        # Show available actions
        actions = problem.get_actions(state)
        print(f"  Available actions: {actions}")
        
        # Add children to frontier (in reverse order for consistent behavior)
        for action in reversed(actions):
            child_state = problem.result(state, action)
            print(f"  Applying '{action}' →", end=" ")
            
            if child_state not in explored and child_state not in [n[0] for n in frontier]:
                child_path = path + [action]
                child_cost = cost + 1
                frontier.append((child_state, child_path, child_cost))
                print("added to frontier")
            else:
                if child_state in explored:
                    print("already explored")
                else:
                    print("already in frontier")
    
    return None

initial = (8, 6, 7, 2, 5, 4, 3, 0, 1)
goal    = (1, 2, 3, 4, 5, 6, 7, 8, 0)

problem = EightPuzzle(initial, goal)
solution = dfs_search(problem)
```

### Reflection Questions

**Q5:** Compare the maximum frontier size between DFS and BFS. Why does DFS use significantly less memory, and in what situations would this memory advantage be crucial for solving a problem?

> Your answer:

**Q6:** DFS found a solution, but was it the optimal (shortest) solution? Explain why DFS is not guaranteed to find the optimal solution even when one exists, and describe a scenario where DFS might find a very poor solution.

> Your answer:

**Q7:** We implemented a depth limit to prevent DFS from exploring infinitely deep paths. What problems could arise without this limit, and how does this relate to the concept of completeness in search algorithms?

> Your answer:

---

## Model 5: Uniform-Cost Search (UCS) also known as Dijkstra's Algorithm

### Description
This exercise demonstrates uniform-cost search, which expands nodes in order of their path cost. UCS uses a priority queue and is optimal for problems with varying action costs.

### Key Concepts
- **Priority Queue**: Data structure where elements are removed in order of priority (lowest cost first)
- **Path Cost**: The total cost of reaching a node from the initial state
- **Uniform-Cost Property**: Always expanding the lowest-cost node guarantees optimality
- **Action Cost Variation**: Different actions may have different costs
- **Completeness with Optimality**: UCS is both complete and optimal

### Task
Run the code and observe how UCS differs from BFS. Pay attention to:
- How nodes are ordered by path cost rather than depth
- The role of the priority queue in selecting which node to expand
- How UCS handles varying action costs
- Why the solution is guaranteed to be optimal

```python
import heapq

class GridWorld:
    """A grid world with varying terrain costs."""
    
    def __init__(self, grid, start, goal):
        self.grid = grid  # 2D list where values are terrain costs
        self.start = start
        self.goal = goal
        self.rows = len(grid)
        self.cols = len(grid[0])
    
    def get_actions(self, state):
        """Returns valid moves from state."""
        row, col = state
        actions = []
        
        if row > 0: actions.append(('UP', row - 1, col))
        if row < self.rows - 1: actions.append(('DOWN', row + 1, col))
        if col > 0: actions.append(('LEFT', row, col - 1))
        if col < self.cols - 1: actions.append(('RIGHT', row, col + 1))
        
        return actions
    
    def step_cost(self, state, action):
        """Returns cost of moving to new position."""
        _, new_row, new_col = action
        return self.grid[new_row][new_col]
    
    def is_goal(self, state):
        return state == self.goal

def ucs_search(problem):
    """Uniform-cost search implementation."""
    
    # Priority queue: (cost, counter, state, path)
    counter = 0  # For tie-breaking
    initial_node = (0, counter, problem.start, [])
    frontier = [initial_node]
    explored = set()
    nodes_expanded = 0
    
    print("UNIFORM-COST SEARCH")
    print("=" * 50)
    print("\nTerrain costs (lower is better):")
    for row in problem.grid:
        print(" ", row)
    print(f"\nStart: {problem.start}, Goal: {problem.goal}\n")
    
    while frontier:
        cost, _, state, path = heapq.heappop(frontier)  # Get lowest-cost node
        
        if state in explored:
            continue
        
        nodes_expanded += 1
        print(f"Expanding node {nodes_expanded}: {state}, Cost: {cost}")
        
        if problem.is_goal(state):
            print(f"\n✓ Goal found! Total nodes expanded: {nodes_expanded}")
            print(f"Optimal path: {path}")
            print(f"Total cost: {cost}")
            return path, cost
        
        explored.add(state)
        
        for action in problem.get_actions(state):
            action_name, new_row, new_col = action
            child_state = (new_row, new_col)
            
            if child_state not in explored:
                step_cost = problem.step_cost(state, action)
                child_cost = cost + step_cost
                child_path = path + [action_name]
                counter += 1
                
                heapq.heappush(frontier, (child_cost, counter, child_state, child_path))
    
    return None, float('inf')

# Grid where numbers represent terrain difficulty (cost to enter)
grid = [
    [1, 1, 1, 1, 1],
    [1, 5, 5, 5, 1],
    [1, 5, 1, 5, 1],
    [1, 1, 1, 5, 1],
    [1, 1, 1, 1, 1]
]

problem = GridWorld(grid, start=(0, 0), goal=(4, 4))
solution, cost = ucs_search(problem)
```

### Reflection Questions

**Q8:** How does UCS decide which node to expand next, and why does this strategy guarantee finding the optimal solution? Compare this to how BFS makes its expansion decisions.

> Your answer:

**Q9:** Observe the path found by UCS through the grid. Does it go directly toward the goal, or does it take a longer route? Explain why UCS chose this path in terms of path cost versus path length.

> Your answer:

**Q10:** In what types of problems would UCS perform identically to BFS? When would UCS clearly outperform BFS in terms of solution quality? Give specific examples of problem domains.

> Your answer:

---

## Model 6: Algorithm Comparison — BFS, DFS, Dijkstra, Bellman-Ford, Topological Sort, Prim's MST, Floyd-Warshall

This model benchmarks all seven graph algorithms discussed in the course across three graph types — sparse, medium, and dense — and reports wall-clock time, memory overhead, and the correctness conditions for each. No new algorithm is traced step-by-step here; the goal is to synthesize everything into a single comparative picture.

### Program 6 — `algorithm_comparison.cpp`

```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <stack>
#include <climits>
#include <chrono>
#include <iomanip>
#include <numeric>
#include <random>
#include <algorithm>
using namespace std;
using Clock = chrono::high_resolution_clock;

const int INF = INT_MAX / 2;
using WGraph = vector<vector<pair<int,int>>>;  // adj list (neighbor, weight)
using EdgeList = vector<tuple<int,int,int>>;   // (u, v, w)

// -------- BFS --------
void bfs(const vector<vector<int>>& adj, int src) {
    int V = adj.size();
    vector<bool> vis(V, false);
    queue<int> q;
    vis[src] = true; q.push(src);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) if (!vis[v]) { vis[v]=true; q.push(v); }
    }
}

// -------- DFS (iterative) --------
void dfs(const vector<vector<int>>& adj, int src) {
    int V = adj.size();
    vector<bool> vis(V, false);
    stack<int> s;
    s.push(src);
    while (!s.empty()) {
        int u = s.top(); s.pop();
        if (vis[u]) continue;
        vis[u] = true;
        for (int v : adj[u]) if (!vis[v]) s.push(v);
    }
}

// -------- Dijkstra --------
void dijkstra(const WGraph& adj, int src) {
    int V = adj.size();
    vector<int> dist(V, INF);
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    dist[src] = 0; pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;
        for (auto [v, w] : adj[u])
            if (dist[u]+w < dist[v]) { dist[v]=dist[u]+w; pq.push({dist[v],v}); }
    }
}

// -------- Bellman-Ford --------
void bellmanFord(int V, const EdgeList& edges, int src) {
    vector<int> dist(V, INF); dist[src] = 0;
    for (int i = 0; i < V-1; i++)
        for (auto [u, v, w] : edges)
            if (dist[u] < INF && dist[u]+w < dist[v]) dist[v] = dist[u]+w;
}

// -------- Topological sort (Kahn's algorithm) --------
void topoSort(const vector<vector<int>>& adj) {
    int V = adj.size();
    vector<int> indeg(V, 0);
    for (int u = 0; u < V; u++) for (int v : adj[u]) indeg[v]++;
    queue<int> q;
    for (int u = 0; u < V; u++) if (indeg[u]==0) q.push(u);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) if (--indeg[v]==0) q.push(v);
    }
}

// -------- Prim's MST --------
void prim(const WGraph& adj, int src) {
    int V = adj.size();
    vector<int> key(V, INF);
    vector<bool> inMST(V, false);
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    key[src] = 0; pq.push({0, src});
    while (!pq.empty()) {
        auto [k, u] = pq.top(); pq.pop();
        if (inMST[u]) continue;
        inMST[u] = true;
        for (auto [v, w] : adj[u])
            if (!inMST[v] && w < key[v]) { key[v]=w; pq.push({key[v],v}); }
    }
}

// -------- Floyd-Warshall --------
void floydWarshall(vector<vector<int>> dist) {
    int V = dist.size();
    for (int k = 0; k < V; k++)
        for (int i = 0; i < V; i++)
            for (int j = 0; j < V; j++)
                if (dist[i][k] < INF && dist[k][j] < INF)
                    dist[i][j] = min(dist[i][j], dist[i][k]+dist[k][j]);
}

// -------- Graph generators --------
struct TestGraph {
    int V, E;
    vector<vector<int>>    unweighted;
    WGraph                 weighted;
    EdgeList               edgeList;
    vector<vector<int>>    matrix;
};

TestGraph makeGraph(int V, int E, mt19937& rng) {
    TestGraph g;
    g.V = V; g.E = E;
    g.unweighted.assign(V, {});
    g.weighted.assign(V, {});
    g.matrix.assign(V, vector<int>(V, INF));
    for (int i = 0; i < V; i++) g.matrix[i][i] = 0;

    // Guarantee connectivity via a spanning chain
    for (int i = 0; i < V-1; i++) {
        int w = 1 + rng() % 20;
        g.unweighted[i].push_back(i+1);
        g.weighted[i].push_back({i+1, w});
        g.edgeList.push_back({i, i+1, w});
        g.matrix[i][i+1] = w;
    }
    // Fill remaining edges randomly
    uniform_int_distribution<int> vdist(0, V-1);
    int added = V-1;
    while (added < E) {
        int u = vdist(rng), v = vdist(rng);
        if (u == v) continue;
        int w = 1 + rng() % 20;
        g.unweighted[u].push_back(v);
        g.weighted[u].push_back({v, w});
        g.edgeList.push_back({u, v, w});
        if (g.matrix[u][v] == INF) g.matrix[u][v] = w;
        added++;
    }
    return g;
}

double bench(const string& algo, TestGraph& g) {
    auto t0 = Clock::now();
    if      (algo == "bfs")    bfs(g.unweighted, 0);
    else if (algo == "dfs")    dfs(g.unweighted, 0);
    else if (algo == "dijkstra") dijkstra(g.weighted, 0);
    else if (algo == "bellman")  bellmanFord(g.V, g.edgeList, 0);
    else if (algo == "topo")   topoSort(g.unweighted);
    else if (algo == "prim")   prim(g.weighted, 0);
    else if (algo == "floyd")  floydWarshall(g.matrix);
    return chrono::duration<double, milli>(Clock::now() - t0).count();
}

int main() {
    mt19937 rng(42);

    // (V, E) configs: sparse, medium, dense
    vector<tuple<int,int,string>> configs = {
        {500,  600,   "sparse (E≈V)"},
        {500,  5000,  "medium (E≈10V)"},
        {500,  50000, "dense  (E≈100V)"}
    };

    vector<string> algos = {
        "bfs","dfs","dijkstra","bellman","topo","prim","floyd"
    };
    vector<string> labels = {
        "BFS","DFS","Dijkstra","Bellman-Ford","Topo Sort","Prim MST","Floyd-Warshall"
    };

    for (auto& [V, E, desc] : configs) {
        TestGraph g = makeGraph(V, E, rng);
        cout << "=== " << desc << " (V=" << V << ", E=" << E << ") ===\n\n";
        cout << setw(18) << "Algorithm"
             << setw(14) << "Time (ms)"
             << "\n" << string(32, '-') << "\n";
        for (int i = 0; i < (int)algos.size(); i++) {
            // Floyd-Warshall only on small dense to keep runtime sane
            if (algos[i] == "floyd" && E > 10000) {
                cout << setw(18) << labels[i]
                     << setw(14) << "(skipped — O(V^3))" << "\n";
                continue;
            }
            double ms = bench(algos[i], g);
            cout << setw(18) << labels[i]
                 << setw(14) << fixed << setprecision(3) << ms << "\n";
        }
        cout << "\n";
    }

    // Complexity reference table
    cout << "=== Complexity Reference ===\n\n";
    cout << setw(18) << "Algorithm"
         << setw(18) << "Time"
         << setw(18) << "Space (extra)"
         << setw(16) << "Graph type"
         << setw(20) << "Handles neg. edges?"
         << "\n" << string(90, '-') << "\n";

    vector<array<string,5>> info = {
        {"BFS",           "O(V+E)",       "O(V)",      "unweighted",   "N/A"},
        {"DFS",           "O(V+E)",       "O(V)",      "any",          "N/A"},
        {"Dijkstra",      "O((V+E)logV)", "O(V)",      "non-neg wts",  "No"},
        {"Bellman-Ford",  "O(VE)",        "O(V)",      "any weights",  "Yes"},
        {"Topo Sort",     "O(V+E)",       "O(V)",      "DAG only",     "N/A"},
        {"Prim MST",      "O((V+E)logV)", "O(V)",      "undirected",   "No"},
        {"Floyd-Warshall","O(V^3)",       "O(V^2)",    "any weights",  "Yes"},
    };

    for (auto& row : info)
        cout << setw(18) << row[0] << setw(18) << row[1]
             << setw(18) << row[2] << setw(16) << row[3]
             << setw(20) << row[4] << "\n";
}
```

### Observation Table 6a — Benchmark Times (ms)

**Sparse (E ≈ V)**

| Algorithm | Time (ms) |
|---|---|
| BFS | |
| DFS | |
| Dijkstra | |
| Bellman-Ford | |
| Topo Sort | |
| Prim MST | |
| Floyd-Warshall | |

**Medium (E ≈ 10V)**

| Algorithm | Time (ms) |
|---|---|
| BFS | |
| DFS | |
| Dijkstra | |
| Bellman-Ford | |
| Topo Sort | |
| Prim MST | |
| Floyd-Warshall | |

**Dense (E ≈ 100V)**

| Algorithm | Time (ms) |
|---|---|
| BFS | |
| DFS | |
| Dijkstra | |
| Bellman-Ford | |
| Topo Sort | |
| Prim MST | |
| Floyd-Warshall | |

### Observation Table 6b — Complexity Reference

Fill in the table from the printed output.

| Algorithm | Time complexity | Extra space | Graph type required | Handles negative edges? |
|---|---|---|---|---|
| BFS | | | | |
| DFS | | | | |
| Dijkstra | | | | |
| Bellman-Ford | | | | |
| Topo Sort | | | | |
| Prim MST | | | | |
| Floyd-Warshall | | | | |

---

### Critical Thinking Questions — Model 6

**Q11.** From Table 6a, how does Bellman-Ford's time compare to Dijkstra's across the three densities? At what density does Bellman-Ford become significantly slower? Use the complexity formulas O(VE) vs O((V+E) log V) to explain what you observe.

> Your answer:

**Q12.** You are designing a navigation system (like Google Maps) for a city with 100,000 intersections and 300,000 road segments, all with positive travel times. You need to answer shortest-path queries from any source to any destination in under 50 ms. Which algorithm from Table 6b would you choose and why? Which would you rule out immediately?

> Your answer:

---


*Next lab: algorithm design — where the question shifts from "which algorithm" to "how do you prove one is correct and derive its complexity from scratch."*