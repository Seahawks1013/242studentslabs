# CPTR 242 — Lab 11: Balanced Trees and B-Trees

**Estimated time:** 75–90 minutes
**Submission:** Upload your completed answers to all observation and reflection questions.

---

## Overview

This lab has five parts. You will observe a plain BST degenerating under sorted input, watch the four AVL rotation cases fire in isolation, trace balance factors through a growing AVL tree, step through 2-3-4 tree insertions and splits, and then measure the performance consequences of all these design choices on real timing data. No implementation is required — compile, run, observe, and reason.

---

## Setup

Save the starter code below as `lab11.cpp` and compile with:

```bash
g++ -std=c++17 -O2 -o lab11 lab11.cpp && ./lab11
```

**Starter code — `lab11.cpp`:**

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>
#include <random>
#include <cmath>
#include <chrono>
#include <map>
using namespace std;

// ── Plain BST ─────────────────────────────────────────────────────────────────
struct BNode {
    int val; BNode *left=nullptr, *right=nullptr;
    BNode(int v) : val(v) {}
};
BNode* b_insert(BNode* n, int k) {
    if (!n) return new BNode(k);
    if (k < n->val) n->left  = b_insert(n->left,  k);
    else if (k > n->val) n->right = b_insert(n->right, k);
    return n;
}
bool b_search(BNode* n, int k) {
    while (n) { if (k==n->val) return true; n = k<n->val ? n->left : n->right; }
    return false;
}
int b_height(BNode* n) {
    if (!n) return -1;
    return 1 + max(b_height(n->left), b_height(n->right));
}
void b_destroy(BNode* n) { if (!n) return; b_destroy(n->left); b_destroy(n->right); delete n; }

// ── AVL tree ──────────────────────────────────────────────────────────────────
struct ANode {
    int val, ht=1; ANode *left=nullptr, *right=nullptr;
    ANode(int v) : val(v) {}
};
int ah(ANode* n)  { return n ? n->ht : 0; }
int abf(ANode* n) { return n ? ah(n->right) - ah(n->left) : 0; }
void aupd(ANode* n) { if (n) n->ht = 1 + max(ah(n->left), ah(n->right)); }

ANode* arotL(ANode* x) {
    ANode* y = x->right; x->right = y->left; y->left = x;
    aupd(x); aupd(y); return y;
}
ANode* arotR(ANode* y) {
    ANode* x = y->left; y->left = x->right; x->right = y;
    aupd(y); aupd(x); return x;
}
ANode* afix(ANode* n) {
    aupd(n);
    int bf = abf(n);
    if (bf < -1) { if (abf(n->left) > 0) n->left = arotL(n->left); return arotR(n); }
    if (bf >  1) { if (abf(n->right) < 0) n->right = arotR(n->right); return arotL(n); }
    return n;
}
ANode* a_insert(ANode* n, int k) {
    if (!n) return new ANode(k);
    if (k < n->val) n->left  = a_insert(n->left,  k);
    else if (k > n->val) n->right = a_insert(n->right, k);
    return afix(n);
}
bool a_search(ANode* n, int k) {
    while (n) { if (k==n->val) return true; n = k<n->val ? n->left : n->right; }
    return false;
}
void a_destroy(ANode* n) { if (!n) return; a_destroy(n->left); a_destroy(n->right); delete n; }

// ── 2-3-4 tree ────────────────────────────────────────────────────────────────
struct TFNode {
    int keys[3]; int nkeys=0;
    TFNode* ch[4]={}; bool isLeaf=true;
};

TFNode* tf_root = nullptr;

void tf_splitChild(TFNode* par, int i) {
    TFNode* full = par->ch[i];
    TFNode* newN = new TFNode();
    newN->isLeaf = full->isLeaf;
    newN->nkeys = 1; newN->keys[0] = full->keys[2];
    if (!full->isLeaf) { newN->ch[0]=full->ch[2]; newN->ch[1]=full->ch[3]; }
    int mid = full->keys[1]; full->nkeys = 1; full->ch[2]=full->ch[3]=nullptr;
    for (int j=par->nkeys; j>i; j--) par->ch[j+1]=par->ch[j];
    par->ch[i+1] = newN;
    for (int j=par->nkeys-1; j>=i; j--) par->keys[j+1]=par->keys[j];
    par->keys[i] = mid; par->nkeys++;
}

void tf_insertNF(TFNode* n, int k) {
    int i = n->nkeys - 1;
    if (n->isLeaf) {
        while (i>=0 && k<n->keys[i]) { n->keys[i+1]=n->keys[i]; i--; }
        n->keys[i+1] = k; n->nkeys++;
    } else {
        while (i>=0 && k<n->keys[i]) i--; i++;
        if (n->ch[i]->nkeys == 3) { tf_splitChild(n,i); if (k>n->keys[i]) i++; }
        tf_insertNF(n->ch[i], k);
    }
}

void tf_insert(int k) {
    if (!tf_root) { tf_root=new TFNode(); tf_root->keys[0]=k; tf_root->nkeys=1; return; }
    if (tf_root->nkeys == 3) {
        TFNode* s = new TFNode(); s->isLeaf=false; s->ch[0]=tf_root;
        tf_splitChild(s, 0); tf_root=s;
    }
    tf_insertNF(tf_root, k);
}

void tf_print(TFNode* n, int depth=0) {
    if (!n) return;
    cout << string(depth*4,' ') << "[";
    for (int i=0; i<n->nkeys; i++) { if (i) cout<<"|"; cout<<n->keys[i]; }
    cout << "]\n";
    if (!n->isLeaf) for (int i=0; i<=n->nkeys; i++) tf_print(n->ch[i], depth+1);
}

bool tf_search(TFNode* n, int k) {
    if (!n) return false;
    int i=0; while (i<n->nkeys && k>n->keys[i]) i++;
    if (i<n->nkeys && k==n->keys[i]) return true;
    if (n->isLeaf) return false;
    return tf_search(n->ch[i], k);
}

int tf_height(TFNode* n) { if (!n||n->isLeaf) return 0; return 1+tf_height(n->ch[0]); }

// ── Timing helpers ────────────────────────────────────────────────────────────
using HRC = chrono::high_resolution_clock;
long us(HRC::time_point a, HRC::time_point b) {
    return chrono::duration_cast<chrono::microseconds>(b-a).count();
}

// ─────────────────────────────────────────────────────────────────────────────

int main() {

    // ── Part A: sorted insertion ──────────────────────────────────────────────
    cout << "=== Part A: Sorted Insertion — BST vs AVL ===\n";
    vector<int> sv = {5,10,15,20,25,30,35};
    BNode* bst=nullptr; ANode* avl=nullptr;
    cout << "Inserting in sorted order: ";
    for (int k : sv) { cout<<k<<" "; bst=b_insert(bst,k); avl=a_insert(avl,k); }
    cout << "\n";
    cout << "BST height: " << b_height(bst) << "\n";
    cout << "AVL height: " << avl->ht-1 << "\n";
    cout << "floor(log2(7)) = " << (int)log2(7) << "\n";
    b_destroy(bst); a_destroy(avl);

    // ── Part B: four rotation cases ───────────────────────────────────────────
    cout << "\n=== Part B: Rotation Cases ===\n";
    struct Case { string name; vector<int> seq; };
    vector<Case> cases = {
        {"LL — insert 30,20,10", {30,20,10}},
        {"RR — insert 10,20,30", {10,20,30}},
        {"LR — insert 30,10,20", {30,10,20}},
        {"RL — insert 10,30,20", {10,30,20}}
    };
    for (auto& c : cases) {
        ANode* t=nullptr;
        for (int k : c.seq) t = a_insert(t, k);
        cout << c.name << ":\n";
        cout << "  root=" << t->val
             << "  left="  << (t->left  ? to_string(t->left->val)  : "null")
             << "  right=" << (t->right ? to_string(t->right->val) : "null")
             << "  height=" << t->ht-1 << "\n";
        a_destroy(t);
    }

    // ── Part C: balance factor trace ──────────────────────────────────────────
    cout << "\n=== Part C: Balance Factor Trace ===\n";
    ANode* tr=nullptr;
    for (int k : {50,30,70,20,40,60,80,10,5}) {
        tr = a_insert(tr, k);
        cout << "insert(" << k << "):  root=" << tr->val
             << "  root_bf=" << abf(tr)
             << "  height=" << tr->ht-1 << "\n";
    }
    a_destroy(tr);

    // ── Part D: 2-3-4 tree ────────────────────────────────────────────────────
    cout << "\n=== Part D: 2-3-4 Tree Insertions ===\n";
    tf_root = nullptr;
    for (int k : {10,20,30,40,50,60,70,80}) {
        tf_insert(k);
        cout << "After insert(" << k << "):\n";
        tf_print(tf_root);
    }
    cout << "Search 40: " << tf_search(tf_root,40) << "\n";
    cout << "Search 45: " << tf_search(tf_root,45) << "\n";
    cout << "Height: "    << tf_height(tf_root) << "\n";

    // ── Part E: timing comparison ─────────────────────────────────────────────
    cout << "\n=== Part E: Timing ===\n";
    const int N=20000, REPS=20;
    vector<int> sdata(N), rdata(N);
    for (int i=0; i<N; i++) sdata[i]=rdata[i]=i+1;
    mt19937 rng(42); shuffle(rdata.begin(), rdata.end(), rng);

    auto t1=HRC::now();
    BNode* bs=nullptr; for (int v:sdata) bs=b_insert(bs,v);
    auto t2=HRC::now();
    ANode* as=nullptr; for (int v:sdata) as=a_insert(as,v);
    auto t3=HRC::now();
    map<int,int> ms; for (int v:sdata) ms[v]=v;
    auto t4=HRC::now();
    cout << "Sorted insert (" << N << " items):\n";
    cout << "  Plain BST: " << us(t1,t2) << " us  height=" << b_height(bs) << "\n";
    cout << "  AVL:       " << us(t2,t3) << " us  height=" << as->ht-1 << "\n";
    cout << "  std::map:  " << us(t3,t4) << " us\n";

    t1=HRC::now();
    BNode* br=nullptr; for (int v:rdata) br=b_insert(br,v);
    t2=HRC::now();
    ANode* ar=nullptr; for (int v:rdata) ar=a_insert(ar,v);
    t3=HRC::now();
    map<int,int> mr; for (int v:rdata) mr[v]=v;
    t4=HRC::now();
    cout << "Random insert (" << N << " items):\n";
    cout << "  Plain BST: " << us(t1,t2) << " us  height=" << b_height(br) << "\n";
    cout << "  AVL:       " << us(t2,t3) << " us  height=" << ar->ht-1 << "\n";
    cout << "  std::map:  " << us(t3,t4) << " us\n";

    volatile bool sink=false;
    t1=HRC::now();
    for (int r=0;r<REPS;r++) for (int v:sdata) sink=b_search(bs,v);
    t2=HRC::now();
    for (int r=0;r<REPS;r++) for (int v:sdata) sink=a_search(as,v);
    t3=HRC::now();
    for (int r=0;r<REPS;r++) for (int v:sdata) sink=ms.count(v);
    t4=HRC::now();
    cout << "Sorted-built search (" << N*REPS << " lookups):\n";
    cout << "  Plain BST: " << us(t1,t2) << " us\n";
    cout << "  AVL:       " << us(t2,t3) << " us\n";
    cout << "  std::map:  " << us(t3,t4) << " us\n";

    b_destroy(bs); a_destroy(as);
    b_destroy(br); a_destroy(ar);
    return 0;
}
```

---

## Part A — Sorted Insertion: BST vs AVL

Run the program and observe the Part A output.

**[OBSERVE 1]**
Record the three printed values: BST height, AVL height, and ⌊log₂(7)⌋. What is the ratio of BST height to AVL height? What would the ratio be for 1,000 elements inserted in sorted order?

---

## Part B — The Four Rotation Cases

Run the program and observe the Part B output.

**[OBSERVE 2]**
All four cases — LL, RR, LR, RL — produce `root=20, left=10, right=30`. Why does every three-node rotation case end up with the median value at the root?

---

## Part C — Balance Factor Trace

Run the program and observe the Part C output.

**[OBSERVE 3]**
Copy the full output for Part C. For the first seven insertions (`50` through `80`), the root never changes and the root balance factor stays in {−1, 0}. Why? What property of the insertion sequence ensures no rotation is needed during these seven inserts?

**[OBSERVE 4]**
After inserting `10`, the root balance factor becomes −1 and height becomes 3. After inserting `5`, the root is still `50`, balance factor is still −1, and height is still 3.

If no rotation fires when inserting 5, where does the rotation happen? Add this line after inserting 5 and re-run to check:

```cpp
cout << "  left child of root: " << tr->left->val
     << "  bf=" << abf(tr->left) << "\n";
cout << "  left-left child:    " << tr->left->left->val
     << "  bf=" << abf(tr->left->left) << "\n";
```

Record what you see. Which node had balance factor ±2 before `afix` corrected it?

---

## Part D — 2-3-4 Tree Insertions

Run the program and observe the Part D output.

**[OBSERVE 5]**
The sequence `10, 20, 30` is inserted into a single root node. When `40` is inserted, the root was a 4-node (`[10|20|30]`) and had to be split before the insert could proceed.

Trace what happens during the split:
1. What value moves **up** to become the new root?
2. What two nodes remain as children?
3. Where does `40` finally land?

**[OBSERVE 6]**
The tree is built from a purely sorted sequence `10, 20, ..., 80` — exactly the input that destroys a plain BST. Yet the printed height after all 8 insertions is **1**.

Why doesn't sorted input degenerate a 2-3-4 tree the way it degenerates a plain BST? What structural property prevents it?

---

## Part E — Timing: BST vs AVL vs `std::map`

Run the program and observe the Part E output.

**[OBSERVE 7]**
The plain BST sorted insert takes roughly **1,000× longer** than the AVL insert for the same input. Both are O(h) per operation, but their heights differ by a factor of ~1,400 (height 19,999 vs 14). Yet the timing ratio is only ~1,000, not ~1,400.

What would explain the timing ratio being smaller than the height ratio? (Hint: think about what operations besides comparisons are happening, and how cache behavior changes with tree depth.)

**[OBSERVE 8]**
For **random** input, all three structures are within a small constant factor of each other in insert time, and the BST height (~32) is within 2× of the AVL height (~16).

What does this tell you about when the plain BST is an acceptable choice? What kind of guarantee does it still fail to provide even for random input?

**[OBSERVE 9]**
The sorted-built search for `std::map` is slower than AVL in this run, even though both are O(log n) with similar heights. One reason: `std::map` uses a Red-Black tree, which is slightly less balanced than AVL (height ≤ 2 log n vs ≤ 1.44 log n).

But there is another reason related to **memory layout**. A `std::map` node is heap-allocated separately and contains extra pointers (parent, color flag). An `ANode` in this lab also heap-allocates, but what does the `std::map` node contain that the lab's `ANode` does not, and how might that affect cache performance?

**[OBSERVE 10]**
You have now seen three balanced structures — AVL tree, Red-Black tree (`std::map`), and 2-3-4 tree — and timed two of them.

Suppose you are building a database index that:
- Is stored on disk (each node access is a disk read costing ~5ms)
- Has 10 million keys
- Needs to support search and range queries

Which of the three structures from this lab would you choose, and why? Be specific about height, disk reads per operation, and the cost of the alternative.
