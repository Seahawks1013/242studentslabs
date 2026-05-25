# CPTR 242 — Lab 8: Binary Search Trees & Tries

---

## Overview

This lab has four parts. You will observe a working BST — including all three removal cases — and measure a Trie and a string BST head-to-head on 50,000 words. No implementation is required; the goal is understanding through careful observation and analysis.

---

## Setup

Save the starter code below as `lab8.cpp` and compile with:

```bash
g++ -std=c++17 -O2 -o lab8 lab8.cpp && ./lab8
```

**Starter code — `lab8.cpp`:**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <chrono>
#include <climits>
#include <cmath>
#include <set>
#include <algorithm>
using namespace std;

// ── Timing helpers ────────────────────────────────────────────────────────────

using HRC = chrono::high_resolution_clock;
using TP  = HRC::time_point;
TP   snap()              { return HRC::now(); }
long us(TP a, TP b)      { return chrono::duration_cast<chrono::microseconds>(b-a).count(); }

// ── Integer BST ───────────────────────────────────────────────────────────────

struct BSTNode {
    int val;
    BSTNode *left = nullptr, *right = nullptr;
    BSTNode(int v) : val(v) {}
};

BSTNode* bst_insert(BSTNode* n, int key) {
    if (!n) return new BSTNode(key);
    if (key < n->val) n->left  = bst_insert(n->left,  key);
    else if (key > n->val) n->right = bst_insert(n->right, key);
    return n;
}

bool bst_search(BSTNode* n, int key) {
    if (!n) return false;
    if (key == n->val) return true;
    return key < n->val ? bst_search(n->left, key) : bst_search(n->right, key);
}

int bst_height(BSTNode* n) {
    if (!n) return -1;
    return 1 + max(bst_height(n->left), bst_height(n->right));
}

void bst_inorder(BSTNode* n, vector<int>& out) {
    if (!n) return;
    bst_inorder(n->left, out);
    out.push_back(n->val);
    bst_inorder(n->right, out);
}

BSTNode* bst_minNode(BSTNode* n) {
    while (n->left) n = n->left;
    return n;
}

BSTNode* bst_remove(BSTNode* n, int key) {
    if (!n) return nullptr;
    if      (key < n->val) n->left  = bst_remove(n->left,  key);
    else if (key > n->val) n->right = bst_remove(n->right, key);
    else {
        // Case 1: leaf
        if (!n->left && !n->right) { delete n; return nullptr; }
        // Case 2: one child
        if (!n->left)  { BSTNode* t = n->right; delete n; return t; }
        if (!n->right) { BSTNode* t = n->left;  delete n; return t; }
        // Case 3: two children — replace with in-order successor
        BSTNode* s = bst_minNode(n->right);
        n->val = s->val;
        n->right = bst_remove(n->right, s->val);
    }
    return n;
}

void bst_destroy(BSTNode* n) {
    if (!n) return;
    bst_destroy(n->left); bst_destroy(n->right); delete n;
}

// ── String BST ────────────────────────────────────────────────────────────────

struct StrNode {
    string val;
    StrNode *left = nullptr, *right = nullptr;
    StrNode(string v) : val(move(v)) {}
};

StrNode* str_insert(StrNode* n, const string& key) {
    if (!n) return new StrNode(key);
    if (key < n->val) n->left  = str_insert(n->left,  key);
    else if (key > n->val) n->right = str_insert(n->right, key);
    return n;
}

bool str_search(StrNode* n, const string& key) {
    if (!n) return false;
    if (key == n->val) return true;
    return key < n->val ? str_search(n->left, key) : str_search(n->right, key);
}

int str_height(StrNode* n) {
    if (!n) return -1;
    return 1 + max(str_height(n->left), str_height(n->right));
}

void str_destroy(StrNode* n) {
    if (!n) return;
    str_destroy(n->left); str_destroy(n->right); delete n;
}

// ── Trie ─────────────────────────────────────────────────────────────────────

struct TrieNode {
    TrieNode* ch[26] = {};
    bool isEnd = false;
};

void trie_insert(TrieNode* root, const string& word) {
    TrieNode* cur = root;
    for (char c : word) {
        if (!cur->ch[c-'a']) cur->ch[c-'a'] = new TrieNode();
        cur = cur->ch[c-'a'];
    }
    cur->isEnd = true;
}

bool trie_search(TrieNode* root, const string& word) {
    TrieNode* cur = root;
    for (char c : word) {
        if (!cur->ch[c-'a']) return false;
        cur = cur->ch[c-'a'];
    }
    return cur->isEnd;
}

bool trie_hasPrefix(TrieNode* root, const string& prefix) {
    TrieNode* cur = root;
    for (char c : prefix) {
        if (!cur->ch[c-'a']) return false;
        cur = cur->ch[c-'a'];
    }
    return true;
}

// ── Word generator ────────────────────────────────────────────────────────────

vector<string> makeWords(int n, unsigned seed = 42) {
    srand(seed);
    const string alpha = "abcdefghijklmnopqrstuvwxyz";
    set<string> seen;
    vector<string> words;
    words.reserve(n);
    while ((int)words.size() < n) {
        int len = 4 + rand() % 7;
        string w;
        for (int j = 0; j < len; j++) w += alpha[rand() % 26];
        if (seen.insert(w).second) words.push_back(w);
    }
    return words;
}

// ── Minimax: stone-taking game ────────────────────────────────────────────────
// Two players alternate. Each turn: take 1 or 2 stones.
// The player who takes the last stone wins.
// Returns +1 if current player wins with perfect play, -1 if they lose.

int minimax(int stones, bool isMax) {
    if (stones == 0) return isMax ? -1 : 1;
    int best = isMax ? INT_MIN : INT_MAX;
    for (int take : {1, 2}) {
        if (take > stones) continue;
        int score = minimax(stones - take, !isMax);
        best = isMax ? max(best, score) : min(best, score);
    }
    return best;
}

// ─────────────────────────────────────────────────────────────────────────────

int main() {

    // ── Part A: Insertion order and height ───────────────────────────────────
    cout << "=== Part A: BST Height by Insertion Order ===\n";

    vector<int> sorted_keys   = {1,2,3,4,5,6,7,8,9,10,11,12,13,14,15};
    vector<int> shuffled_keys = {8,4,12,2,6,10,14,1,3,5,7,9,11,13,15};

    BSTNode* sorted_bst   = nullptr;
    BSTNode* shuffled_bst = nullptr;
    for (int k : sorted_keys)   sorted_bst   = bst_insert(sorted_bst,   k);
    for (int k : shuffled_keys) shuffled_bst = bst_insert(shuffled_bst, k);

    cout << "Sorted insertion   — height: " << bst_height(sorted_bst) << "\n";
    cout << "Shuffled insertion — height: " << bst_height(shuffled_bst) << "\n";

    vector<int> out;
    bst_inorder(shuffled_bst, out);
    cout << "In-order output:  ";
    for (int x : out) cout << x << " ";
    cout << "\n";

    bst_destroy(sorted_bst);
    bst_destroy(shuffled_bst);

    // ── Part B: Trie prefix search ───────────────────────────────────────────
    cout << "\n=== Part B: Trie Prefix Search ===\n";

    TrieNode* trie = new TrieNode();
    for (string w : {"apple","application","apply","apt",
                     "banana","band","bandana","bat",
                     "cat","card","care","car","carpet"})
        trie_insert(trie, w);

    for (string q : {"apple","app","car","carpet","bat","xyz","ap"}) {
        cout << "  \"" << q << "\"  search=" << trie_search(trie, q)
             << "  hasPrefix=" << trie_hasPrefix(trie, q) << "\n";
    }

    // ── Part C: BST removal — all three cases ────────────────────────────────
    cout << "\n=== Part C: BST Removal ===\n";

    BSTNode* rt = nullptr;
    for (int v : {50,30,70,20,40,60,80,35}) rt = bst_insert(rt, v);

    auto printTree = [&](const string& label) {
        out.clear(); bst_inorder(rt, out);
        cout << "  " << label;
        for (int x : out) cout << x << " ";
        cout << "  (height=" << bst_height(rt) << ")\n";
    };

    printTree("Initial:               ");
    rt = bst_remove(rt, 20);  printTree("After remove(20) [leaf]:       ");
    rt = bst_remove(rt, 30);  printTree("After remove(30) [one child]:  ");
    rt = bst_remove(rt, 50);  printTree("After remove(50) [two children]:");

    bst_destroy(rt);

    // ── Part D: Empirical comparison — Trie vs. string BST ───────────────────
    cout << "\n=== Part D: Trie vs. BST — 50,000 words ===\n";

    const int N    = 50000;
    const int REPS = 10;
    auto words = makeWords(N);

    TrieNode* big_trie = new TrieNode();
    StrNode*  str_root = nullptr;
    for (auto& w : words) {
        trie_insert(big_trie, w);
        str_root = str_insert(str_root, w);
    }

    cout << "BST height:    " << str_height(str_root) << "\n";
    cout << "log2(" << N << ") ≈ " << (int)log2(N)
         << "  (balanced ideal)\n\n";

    volatile bool sink = false;

    auto t1 = snap();
    for (int r = 0; r < REPS; r++)
        for (auto& w : words) sink = trie_search(big_trie, w);
    auto t2 = snap();
    for (int r = 0; r < REPS; r++)
        for (auto& w : words) sink = str_search(str_root, w);
    auto t3 = snap();

    cout << "Exact search (" << N*REPS << " lookups):\n";
    cout << "  Trie: " << us(t1,t2) << " μs\n";
    cout << "  BST:  " << us(t2,t3) << " μs\n\n";

    str_destroy(str_root);

    return 0;
}
```

---

## Part A — Insertion Order and BST Height

Run the program and observe the Part A output.

**[OBSERVE 1]**
Record both printed heights. The same 15 values are inserted in two different orders:

- **Sorted:** `1, 2, 3, 4, …, 15`
- **Shuffled:** `8, 4, 12, 2, 6, 10, 14, 1, 3, 5, 7, 9, 11, 13, 15`

What is the ratio of sorted height to shuffled height? What is log₂(15) rounded down?

> Answer: The sorted Insertion height is 14. The shuffled insertion height =3. The ratio would be 4.67 (14/3). Insertion creates an unbalanced tree, due to each number being bigger than the previous one, creating a long line on the right. Compared to the shuffled tree, which starts at the middle and creates a balanced tree. 


**[OBSERVE 2]**
Copy the in-order output from the shuffled BST. What do you notice? Is this output affected by which insertion order was used?

> Answer: The order is 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15. I notice that it is in ascending order. This output is affected by in-order traversal, which visits left -> root -> right nodes. No matter how the tree was built, it will always return in ascending order. Output is not affected by the insertion order. 


---

## Part B — Trie Prefix Search

Run the program and observe the Part B output.

The dictionary contains: `apple`, `application`, `apply`, `apt`, `banana`, `band`, `bandana`, `bat`, `cat`, `card`, `care`, `car`, `carpet`.

**[OBSERVE 3]**
`"car"` is in the dictionary. `"carpet"` is also in the dictionary. Both queries give `hasPrefix=1`. But their `search` results differ. Explain exactly what the trie node reached after traversing `c → a → r` looks like — specifically, its `isEnd` flag and its child pointers.

> Answer: The trie node reached after traversing c → a → r has isEnd = true (since "car" was inserted already as a complete word). It has an index pointer at index p, leading towards "carpet". 


**[OBSERVE 4]**
`"ap"` is **not** a word in the dictionary, but `hasPrefix` returns 1. Draw the trie path for `a → p` and identify where the traversal stops. Why does `trie_hasPrefix` return true while `trie_search` returns false?

> Answer: `a → p` lands on a valid trie node, since the path exists. `trie_hasPrefix` is true because the traversal finishes without hitting a null pointer. Since that node's isEnd flag is false, 'ap' was never inserted as a complete word, therefore returning false. 

**[OBSERVE 5]**
Look at the four words sharing the prefix `"app"`: `apple`, `application`, `apply`, `apt`. 

- How many trie nodes do all four share before their paths diverge?
- How many nodes would a string BST use to store the same four words?

> Answer: All four nodes share the 'ap' prefix, with three of them sharing the 'app' prefix. A string BST would store each word as its own, separate node. A string BST would use 4 full independent string nodes for the same four words. 

**[OBSERVE 6]**
A BST storing strings compares full strings at each node: O(L) per comparison, O(log n) comparisons → O(L log n) per lookup. A trie does O(L) total regardless of n.

But there is a storage scenario where the BST uses *less* memory than the trie. Describe it precisely. (Hint: think about what the trie's sharing advantage requires.)

> Answer: The BST would utilize less memory when those words share few or no common prefixes. A trie allocates one node per character, which gives it an advantage coming from prefix sharing. Even if a word is unique, the trie will allocate a separate node for every single character. The BST would store each word as a single node containing the whole string. This would favor the BST when short words and highly distinct starting characters are used. 

---

## Part C — BST Removal: All Three Cases

Run the program and observe the Part C output. The starting tree is:

```
        50
       /  \
     30    70
    /  \   / \
  20   40 60  80
        /
       35
```

**[OBSERVE 7]**
After every removal, is the BST property preserved? How does the in-order output confirm this without drawing the tree?

> Answer: Yes, the BST property is preserved after each removal. The in-order output remains sorted each time. Since the output is continuously sorted, this confirms that the BST was maintained throughout the removals. 

**[OBSERVE 8]**
`remove(30)` is a one-child case. After inserting `{50,30,70,20,40,60,80,35}` and removing 20, node 30 has only a right child (40), and 40 has a left child (35). 

Trace what `bst_remove` does step-by-step when called with key=30:
1. Which case fires?
2. What pointer is returned to replace node 30?
3. What happens to node 35?

> Answer: 1. The second case fires. Node 30 only has a right child (since node 20 was removed). 2. The pointer is returned to node 40. 3. Nothing happens to node 35. It stays as node 40s left child. 

**[OBSERVE 9]**
`remove(50)` is a two-children case. The code finds the **in-order successor** — the minimum of the right subtree.

After removing 20 and 30, what does the tree look like just before `remove(50)` is called? What is 50's in-order successor at that point?

> Answer: The tree has 50 at the root with the left subtree containing 40 -> 35 and right subtree 70 -> 60,80 (after removal of 20 and 30). Node 50 still has two children, so the code finds the in-order successor, which is the minimum node on the right subtree. Since the lowest node value on the right subtree is 60, it then takes node 50s spot. 


**[OBSERVE 10]**
The code always uses the in-order **successor** (smallest in right subtree) for Case 3. Using the in-order **predecessor** (largest in left subtree) would also produce a valid BST.

Would choosing one over the other affect correctness? Would it affect the height or balance of the resulting tree over many insertions and deletions? Describe a sequence of operations where the choice would matter practically.

> Answer: There isn't a definite 'correctness' in this example. However, it does affect balance over time. If the successor is always used, you would keep pulling values from the right subtree, resulting in the shrinking of the right side. An example would be inserting 100 values on the right side, and deleting the root using the successor strategy. Every deletion pulls from the right subtree, shrinking it faster than the left. When the tree becomes lopsided, performance will degrade. 

---

## Part D — Empirical Comparison: Trie vs. String BST

The program builds both structures from 50,000 distinct randomly generated words, then runs 500,000 lookups on each (10 passes × 50,000 words).

**[OBSERVE 11]**
Compute the speedup ratios: how many times faster is the Trie for exact search?

> Answer: From the output - Trie: 303,062 μs, and BST: 783,885 μs. Speedup = 783,885/303,062 = ~2.6x faster. The trie is faster because it does O(L) per lookup, compared to the BST, which does O(L x logn)

**[OBSERVE 12]**
The BST height is printed alongside log₂(50,000) ≈ 15. The actual BST height for random words should be around 35–40. Why is a randomly built BST not balanced, even with random data? What would need to be different about the insertion order to achieve height ≈ 15?

> Answer: Random words are not truly random alphabetically. Some letters in words occur more. An example would be the likelihood of having words starting with 'a' compared to 'x' - words starting with 'a' are much more likely to appear in this random generation. Therefore, certain subtrees grow more, which creates unequal leveling. The BST has no mechanism of self-correction. To achieve a height of 15, all 50,000 words would need to be alphabetically sorted, always insert the middle word first, and then do the middle of each half, and so on. 


