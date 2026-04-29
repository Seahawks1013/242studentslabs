# Lab 5: Observing Hash Tables

**CPTR 242 — Sequential and Parallel Data Structures and Algorithms**
Walla Walla University

---

## Overview

In this lab you will **run complete programs and observe their behavior**. There is no coding required. Your job is to read the output carefully, record what you see, and reason about what it means.

By the end you should be able to:

- Trace a hash table through a sequence of insertions and look up the expected slot for any key
- Explain how chaining handles collisions using a linked list per bucket
- Describe how linear probing finds an open slot and why deletion requires tombstones
- Compare hash table performance against a sorted array and reason about when O(1) average breaks down
- Recognize the load-factor threshold that triggers resizing and understand why rehashing is necessary

---

## Background: From Search to O(1) Lookup

Every data structure you have studied so far — arrays, linked lists, trees — locates a value by *comparing* keys. Binary search on a sorted array takes O(log n); a balanced BST takes O(log n). A hash table sidesteps comparison entirely: it computes an array index from the key and accesses that slot directly. When the hash function distributes keys uniformly and the table is not too full, every lookup, insertion, and deletion takes O(1) average time.

The central question: **where does that O(1) come from, and when does it break?**

---

## Setup

All programs are self-contained. Each generates its own test data and prints results directly to the terminal.

**Compiler command for all programs:**
```bash
g++ -O0 -o program program.cpp && ./program
```

> `-O0` disables compiler optimizations so you observe true algorithm behavior.

---

## Model 1: Hash Function Behavior and Slot Distribution

Before examining a full hash table, you need to understand what a hash function actually does to a set of keys. This program hashes a fixed list of strings using a polynomial rolling hash and prints the raw hash value, the slot index (`hash % TABLE_SIZE`), and the final slot assignment for each key.

### Program 1 — `hash_distribution.cpp`

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <iomanip>
using namespace std;

const int TABLE_SIZE = 11;   // prime

int hashFn(const string& s) {
    int h = 0;
    for (char c : s)
        h = h * 31 + c;
    return abs(h) % TABLE_SIZE;
}

int rawHash(const string& s) {
    int h = 0;
    for (char c : s)
        h = h * 31 + c;
    return abs(h);
}

int main() {
    vector<string> keys = {
        "alice", "bob", "carol", "dave", "eve",
        "frank", "grace", "heidi", "ivan", "judy"
    };

    cout << "TABLE_SIZE = " << TABLE_SIZE << "\n\n";
    cout << left
         << setw(10) << "Key"
         << setw(14) << "Raw hash"
         << setw(10) << "Slot"
         << "\n" << string(34, '-') << "\n";

    // Count collisions
    vector<int> slotCount(TABLE_SIZE, 0);
    for (const string& k : keys) {
        int raw  = rawHash(k);
        int slot = hashFn(k);
        slotCount[slot]++;
        cout << setw(10) << k
             << setw(14) << raw
             << setw(10) << slot << "\n";
    }

    cout << "\nSlot occupancy after all insertions:\n";
    cout << setw(8) << "Slot" << setw(8) << "Count" << "\n";
    cout << string(16, '-') << "\n";
    int collisions = 0;
    for (int i = 0; i < TABLE_SIZE; i++) {
        cout << setw(8) << i << setw(8) << slotCount[i];
        if (slotCount[i] > 1) { cout << "  ← collision"; collisions++; }
        cout << "\n";
    }
    cout << "\nSlots with collisions: " << collisions << "\n";
    cout << "Empty slots:           ";
    int empty = 0;
    for (int c : slotCount) if (c == 0) empty++;
    cout << empty << "\n";
    cout << "Load factor:           "
         << fixed << setprecision(2)
         << (double)keys.size() / TABLE_SIZE << "\n";
}
```

### Observation Table 1a — Hash Slot Assignments

Fill in the slot each key maps to:

| Key | Slot (0–10) |
|---|---|
| alice | |
| bob | |
| carol | |
| dave | |
| eve | |
| frank | |
| grace | |
| heidi | |
| ivan | |
| judy | |

### Observation Table 1b — Slot Occupancy

| Slot | Number of keys assigned |
|---|---|
| 0 | |
| 1 | |
| 2 | |
| 3 | |
| 4 | |
| 5 | |
| 6 | |
| 7 | |
| 8 | |
| 9 | |
| 10 | |

---

### Critical Thinking Questions — Model 1

**Q1.** How many slots are empty after inserting 10 keys into an 11-slot table? How many collisions occurred? Does the distribution look roughly uniform, or are keys clumped heavily in a few slots?

> Your answer:

**Q2.** The hash function multiplies by 31 before adding each character. What would happen if you simply summed the character values without multiplying? Give an example of two different strings that would collide under the summing approach but likely not under the polynomial approach.

> Your answer:

**Q3.** TABLE_SIZE is chosen to be prime (11). Suppose you changed it to 10 (not prime). Would you expect more or fewer collisions? What property of prime-sized tables makes them distribute keys more evenly?

> Your answer:

**Q4.** The load factor after 10 insertions is approximately 0.91. This is high. What are the consequences of a high load factor for (a) a chaining table and (b) an open-addressing table?

> Your answer:

---

## Model 2: Chaining — Insertion, Lookup, and Collision Handling

Chaining resolves collisions by storing a linked list at each bucket. This program builds a chaining hash table from a sequence of key–value pairs, prints the internal state of every bucket after each insertion, and then performs several lookups showing the probe path through the chain.

### Program 2 — `chaining.cpp`

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

const int TABLE_SIZE = 7;

int hashFn(const string& s) {
    int h = 0;
    for (char c : s) h = h * 31 + c;
    return abs(h) % TABLE_SIZE;
}

struct Entry {
    string key; int value; Entry* next;
    Entry(const string& k, int v) : key(k), value(v), next(nullptr) {}
};

class ChainingTable {
    Entry* table[TABLE_SIZE];
    int    count = 0;
public:
    ChainingTable() { fill(table, table + TABLE_SIZE, nullptr); }

    void insert(const string& key, int val) {
        int idx = hashFn(key);
        // Check for existing key (update)
        for (Entry* e = table[idx]; e; e = e->next) {
            if (e->key == key) { e->value = val; return; }
        }
        Entry* n = new Entry(key, val);
        n->next = table[idx];
        table[idx] = n;
        count++;
    }

    int search(const string& key) {
        int idx = hashFn(key);
        int probes = 0;
        cout << "  search(\"" << key << "\") → slot " << idx << " → probes: ";
        for (Entry* e = table[idx]; e; e = e->next) {
            probes++;
            cout << "\"" << e->key << "\"";
            if (e->key == key) {
                cout << " ✓  (" << probes << " probe(s))\n";
                return e->value;
            }
            cout << " → ";
        }
        cout << "null  NOT FOUND (" << probes << " probe(s))\n";
        return -1;
    }

    void printTable(const string& label = "") const {
        if (!label.empty()) cout << label << "\n";
        for (int i = 0; i < TABLE_SIZE; i++) {
            cout << "  [" << i << "]: ";
            for (Entry* e = table[i]; e; e = e->next)
                cout << "(\"" << e->key << "\"," << e->value << ") → ";
            cout << "null\n";
        }
        cout << "  Load factor: " << fixed << setprecision(2)
             << (double)count / TABLE_SIZE << "  (" << count << "/" << TABLE_SIZE << ")\n";
    }

    ~ChainingTable() {
        for (int i = 0; i < TABLE_SIZE; i++)
            for (Entry* e = table[i]; e; ) {
                Entry* t = e->next; delete e; e = t;
            }
    }
};

int main() {
    ChainingTable t;

    vector<pair<string,int>> data = {
        {"alice",91}, {"bob",85}, {"carol",78},
        {"dave",92},  {"eve",88}, {"frank",74},
        {"grace",95}, {"heidi",81}
    };

    cout << "=== Chaining Hash Table: Insertions ===\n\n";
    t.printTable("Initial state:");

    for (auto& [k, v] : data) {
        cout << "\ninsert(\"" << k << "\", " << v
             << ")  →  slot " << hashFn(k) << "\n";
        t.insert(k, v);
        t.printTable();
    }

    cout << "\n=== Lookups ===\n";
    t.search("alice");
    t.search("carol");
    t.search("grace");
    t.search("zara");   // not present
}
```

### Observation Table 2a — Table State After Each Insertion

For each insertion, record which slot the key lands in and whether a collision occurred (another key was already in that slot):

| Key inserted | Slot | Collision? (Y/N) | Chain length after insertion |
|---|---|---|---|
| alice | | | |
| bob | | | |
| carol | | | |
| dave | | | |
| eve | | | |
| frank | | | |
| grace | | | |
| heidi | | | |

### Observation Table 2b — Lookup Probe Counts

| Key searched | Slot | Probes needed | Found? |
|---|---|---|---|
| alice | | | |
| carol | | | |
| grace | | | |
| zara | | | |

---

### Critical Thinking Questions — Model 2

**Q5.** When two keys land in the same slot, the new key is **prepended** to the chain (inserted at the front), not appended to the back. Does this affect correctness? Does it affect performance? Would you ever prefer to append instead?

> Your answer:

**Q6.** The `search("zara")` call walks an entire chain and finds nothing. How many nodes did it examine? In terms of load factor λ, what is the expected number of probes for an unsuccessful search in a chaining table?

> Your answer:

**Q7.** After all 8 insertions into a 7-slot table the load factor exceeds 1.0. Chaining still works. Open addressing would not — why not? What invariant does open addressing require that chaining does not?

> Your answer:

**Q8.** Look at the final table printout. Some slots have chains of length 2 or more; others are empty. Even with a good hash function, perfect uniformity is not guaranteed. What is the theoretical expected maximum chain length for n keys in a table of size n, under uniform hashing?

> Your answer:

---

## Model 3: Linear Probing — Insertion, Clustering, and Tombstones

Linear probing stores everything in the array itself — no linked lists, no extra allocation. On collision it steps forward by 1 until an empty slot is found. This program inserts a sequence of keys, prints the full array after each insertion (showing which slots are occupied and where keys "landed" after probing), then demonstrates the tombstone deletion problem.

### Program 3 — `linear_probing.cpp`

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <iomanip>
using namespace std;

const int TABLE_SIZE = 11;

int hashFn(const string& s) {
    int h = 0;
    for (char c : s) h = h * 31 + c;
    return abs(h) % TABLE_SIZE;
}

enum State { EMPTY, OCCUPIED, DELETED };

struct Slot {
    string key;
    int    value = 0;
    State  state = EMPTY;
};

class LinearProbeTable {
    Slot table[TABLE_SIZE];
    int  count = 0;
public:
    void insert(const string& key, int val) {
        int start = hashFn(key);
        int idx   = start;
        int probes = 0;
        while (table[idx].state == OCCUPIED && table[idx].key != key) {
            idx = (idx + 1) % TABLE_SIZE;
            probes++;
            if (probes == TABLE_SIZE) { cout << "  TABLE FULL\n"; return; }
        }
        if (table[idx].state != OCCUPIED) count++;
        table[idx] = {key, val, OCCUPIED};
        cout << "  insert(\"" << key << "\") home=" << start;
        if (probes == 0) cout << "  placed at " << start << "  (0 extra probes)\n";
        else             cout << "  placed at " << idx
                              << "  (" << probes << " extra probe(s), cluster!)\n";
    }

    void remove(const string& key) {
        int start = hashFn(key);
        int idx   = start;
        int probes = 0;
        while (table[idx].state != EMPTY) {
            if (table[idx].state == OCCUPIED && table[idx].key == key) {
                table[idx].state = DELETED;
                count--;
                cout << "  remove(\"" << key << "\") → slot " << idx
                     << " marked TOMBSTONE\n";
                return;
            }
            idx = (idx + 1) % TABLE_SIZE;
            if (++probes == TABLE_SIZE) break;
        }
        cout << "  remove(\"" << key << "\") → NOT FOUND\n";
    }

    int search(const string& key) {
        int start = hashFn(key);
        int idx   = start;
        int probes = 0;
        cout << "  search(\"" << key << "\") home=" << start << "  path: ";
        while (table[idx].state != EMPTY) {
            string label = (table[idx].state == DELETED) ? "TOMB"
                         : "\"" + table[idx].key + "\"";
            cout << "[" << idx << "]=" << label;
            if (table[idx].state == OCCUPIED && table[idx].key == key) {
                cout << " ✓  (" << probes+1 << " probe(s))\n";
                return table[idx].value;
            }
            idx = (idx + 1) % TABLE_SIZE;
            probes++;
            if (probes == TABLE_SIZE) break;
            cout << " → ";
        }
        cout << "[" << idx << "]=EMPTY  NOT FOUND (" << probes << " probe(s))\n";
        return -1;
    }

    void printTable(const string& label = "") const {
        if (!label.empty()) cout << label << "\n";
        for (int i = 0; i < TABLE_SIZE; i++) {
            cout << "  [" << setw(2) << i << "] ";
            if      (table[i].state == EMPTY)    cout << ".\n";
            else if (table[i].state == DELETED)  cout << "<TOMBSTONE>\n";
            else cout << "\"" << table[i].key << "\" = " << table[i].value << "\n";
        }
        cout << "  Load factor: " << fixed << setprecision(2)
             << (double)count / TABLE_SIZE << "  (" << count << "/" << TABLE_SIZE << ")\n";
    }
};

int main() {
    LinearProbeTable t;

    cout << "=== Linear Probing: Insertions ===\n\n";
    t.printTable("Initial state:");

    vector<pair<string,int>> data = {
        {"alice",91}, {"bob",85}, {"carol",78},
        {"dave",92},  {"eve",88}, {"frank",74},
        {"grace",95}
    };

    for (auto& [k, v] : data) {
        t.insert(k, v);
        t.printTable();
    }

    cout << "\n=== Deletion and Tombstones ===\n\n";
    t.printTable("Before deletion:");

    // Search succeeds before deletion
    t.search("carol");

    // Delete a key in the middle of a potential probe chain
    t.remove("bob");
    t.printTable("After removing \"bob\":");

    // Search after deletion — must traverse tombstone
    t.search("carol");

    // Demonstrate danger: search WITHOUT tombstone support
    // (we simulate by asking about a key whose path goes through the deleted slot)
    cout << "\n=== Probe path comparison ===\n";
    cout << "Note: carol's home slot and bob's slot — observe how the\n"
         << "search for carol must cross bob's tombstone to succeed.\n\n";
    t.search("alice");
    t.search("grace");
    t.search("zara");   // not present
}
```

### Observation Table 3a — Slot Placement After Each Insertion

Record where each key ended up (its final slot index) and how many extra probes were needed beyond the home slot:

| Key | Home slot (`hash % 11`) | Final slot | Extra probes | Caused by cluster? |
|---|---|---|---|---|
| alice | | | | |
| bob | | | | |
| carol | | | | |
| dave | | | | |
| eve | | | | |
| frank | | | | |
| grace | | | | |

### Observation Table 3b — Search Probe Paths After Tombstone Insertion

After `bob` is deleted (tombstone), record the slots visited for each search:

| Key searched | Slots visited (in order) | Probes | Result |
|---|---|---|---|
| carol (before delete) | | | |
| carol (after delete) | | | |
| alice | | | |
| grace | | | |
| zara | | | |

---

### Critical Thinking Questions — Model 3

**Q9.** Look at the extra-probe column in Table 3a. Several keys required probing past their home slot. This is **primary clustering** — runs of occupied slots that grow longer over time. Describe in your own words why a new key hashing *anywhere into* an existing cluster makes the cluster grow longer.

> Your answer:

**Q10.** After `bob` is deleted, the search for `carol` must cross the tombstone. What would happen if the tombstone were replaced with an EMPTY marker instead — would the search for `carol` succeed or fail? Explain why.

> Your answer:

**Q11.** Tombstones accumulate over time. A slot marked DELETED counts as "occupied" for searches (you must keep probing past it) but as "available" for insertions (you can place a new key there). What happens to average probe length as the fraction of tombstone slots grows? How does rehashing fix this?

> Your answer:

**Q12.** Linear probing requires λ < 1 (the table can never be completely full). Chaining from Model 2 allowed λ > 1. Why does open addressing impose this strict limit, while chaining does not?

> Your answer:

---

## Model 4: Collision Strategy Comparison and Load Factor Effects

Linear probing degrades quickly as the table fills. Double hashing avoids clustering by using a second hash function to compute the step size. This program inserts the same keys into three tables — chaining, linear probing, and double hashing — and reports the average probe count for both successful and unsuccessful searches at several load factors.

### Program 4 — `collision_comparison.cpp`

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <cmath>
#include <iomanip>
#include <functional>
using namespace std;

// Prime table sizes for each load factor target
const vector<int> TABLE_SIZES = {101, 67, 43, 29, 17};
// Corresponding approximate load factors when filled with 50% of size keys

int h1(const string& s, int sz) {
    int h = 0; for (char c : s) h = h * 31 + c; return abs(h) % sz;
}
int h2(const string& s, int sz) {
    int h = 0; for (char c : s) h = h * 37 + c;
    return 1 + (abs(h) % (sz - 1));
}

// ---------- Chaining ----------
struct CEntry { string key; CEntry* next; };

double chainAvgProbes(const vector<string>& keys, int sz) {
    vector<CEntry*> table(sz, nullptr);
    for (auto& k : keys) {
        int idx = h1(k, sz);
        table[idx] = new CEntry{k, table[idx]};
    }
    long long total = 0;
    for (auto& k : keys) {
        int idx = h1(k, sz); int p = 0;
        for (CEntry* e = table[idx]; e; e = e->next) {
            p++;
            if (e->key == k) break;
        }
        total += p;
    }
    for (int i = 0; i < sz; i++)
        for (CEntry* e = table[i]; e; ) { CEntry* t = e->next; delete e; e = t; }
    return (double)total / keys.size();
}

// ---------- Linear Probing ----------
double linearAvgProbes(const vector<string>& keys, int sz) {
    vector<string> table(sz, "");
    vector<bool>   used(sz, false);
    for (auto& k : keys) {
        int idx = h1(k, sz);
        while (used[idx]) idx = (idx + 1) % sz;
        table[idx] = k; used[idx] = true;
    }
    long long total = 0;
    for (auto& k : keys) {
        int idx = h1(k, sz); int p = 0;
        while (used[idx] && table[idx] != k) { idx = (idx + 1) % sz; p++; }
        total += p + 1;
    }
    return (double)total / keys.size();
}

// ---------- Double Hashing ----------
double doubleAvgProbes(const vector<string>& keys, int sz) {
    vector<string> table(sz, "");
    vector<bool>   used(sz, false);
    for (auto& k : keys) {
        int idx  = h1(k, sz);
        int step = h2(k, sz);
        while (used[idx]) idx = (idx + step) % sz;
        table[idx] = k; used[idx] = true;
    }
    long long total = 0;
    for (auto& k : keys) {
        int idx  = h1(k, sz);
        int step = h2(k, sz);
        int p = 0;
        while (used[idx] && table[idx] != k) { idx = (idx + step) % sz; p++; }
        total += p + 1;
    }
    return (double)total / keys.size();
}

// Generate n unique synthetic keys
vector<string> makeKeys(int n) {
    vector<string> v;
    for (int i = 0; i < n; i++) {
        string s = "";
        int x = i;
        do { s += (char)('a' + x % 26); x /= 26; } while (x > 0);
        v.push_back(s);
    }
    return v;
}

int main() {
    // Test across several load factors using varying n/sz ratios
    vector<pair<int,int>> tests = {
        {10,  101},   // λ ≈ 0.10
        {25,  101},   // λ ≈ 0.25
        {50,  101},   // λ ≈ 0.50
        {70,  101},   // λ ≈ 0.69
        {90,  101},   // λ ≈ 0.89
    };

    cout << "=== Average Successful Probe Count by Strategy and Load Factor ===\n\n";
    cout << fixed << setprecision(3);
    cout << setw(12) << "Load (λ)"
         << setw(16) << "Chaining"
         << setw(16) << "Linear probe"
         << setw(16) << "Double hash"
         << "\n" << string(60, '-') << "\n";

    for (auto& [n, sz] : tests) {
        auto keys = makeKeys(n);
        double lam   = (double)n / sz;
        double chain = chainAvgProbes(keys, sz);
        double lin   = linearAvgProbes(keys, sz);
        double dbl   = doubleAvgProbes(keys, sz);
        cout << setw(12) << lam
             << setw(16) << chain
             << setw(16) << lin
             << setw(16) << dbl << "\n";
    }

    // Theoretical values for reference
    cout << "\n=== Theoretical Predictions (for comparison) ===\n\n";
    cout << setw(12) << "Load (λ)"
         << setw(20) << "Linear (theory)"
         << setw(20) << "Double (theory)"
         << "\n" << string(52, '-') << "\n";

    vector<double> lambdas = {0.10, 0.25, 0.50, 0.69, 0.89};
    for (double lam : lambdas) {
        double linTheory = 0.5 * (1.0 + 1.0 / (1.0 - lam));
        double dblTheory = (1.0 / lam) * log(1.0 / (1.0 - lam));
        cout << setw(12) << lam
             << setw(20) << linTheory
             << setw(20) << dblTheory << "\n";
    }
}
```

### Observation Table 4a — Measured Average Probe Counts

| Load factor (λ) | Chaining | Linear probing | Double hashing |
|---|---|---|---|
| ~0.10 | | | |
| ~0.25 | | | |
| ~0.50 | | | |
| ~0.69 | | | |
| ~0.89 | | | |

### Observation Table 4b — Theoretical Linear vs. Double Hashing Predictions

| Load factor (λ) | Linear (theory) | Double (theory) |
|---|---|---|
| 0.10 | | |
| 0.25 | | |
| 0.50 | | |
| 0.69 | | |
| 0.89 | | |

---

### Critical Thinking Questions — Model 4

**Q13.** At λ = 0.50 (half full), linear probing already takes noticeably more probes than double hashing. At λ ≈ 0.89 the gap widens dramatically. In your own words, explain *why* double hashing avoids the primary clustering that hurts linear probing.

> Your answer:

**Q14.** Chaining's probe count grows slowly and almost linearly with λ — and it still works above λ = 1.0. Open-addressing schemes blow up as λ → 1. From the table, at what load factor does linear probing exceed 3 probes on average? What does that tell you about a safe upper bound for λ in a linearly-probed table?

> Your answer:

**Q15.** Both theoretical formulas (linear and double hashing) are derived assuming **uniform random hashing** — every key equally likely in any slot. Your measured values are based on a specific deterministic hash function on synthetic keys. Are the measured values close to the theoretical predictions? What might cause discrepancies?

> Your answer:

---


*Next lab: trees — where we trade O(1) average for O(log n) guaranteed and gain the ability to iterate keys in sorted order.*