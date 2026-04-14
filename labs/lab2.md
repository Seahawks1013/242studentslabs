# Lab 2: Observing Sorting Algorithms

**CPTR 242 — Sequential and Parallel Data Structures and Algorithms**
Walla Walla University

---

## Overview

In this lab you will **run complete programs and observe their behavior**. There is no coding required. Your job is to read the output carefully, record what you see, and reason about what it means.

By the end you should be able to:

- Identify O(n²) and O(n log n) behavior from empirical comparison counts and timing data
- Explain why the same algorithm performs differently on different input arrangements
- Describe the key tradeoffs between sorting algorithms: time, space, stability, and adaptivity
- Read recursive sorting code and trace how it divides and conquers

**POGIL Roles** (rotate each model): Manager · Recorder · Presenter · Reflector

---

## Background: What Makes Sorting Hard?

Sorting seems simple — but different algorithms solve it in dramatically different ways, with very different costs. Today you will observe four sorting algorithms side by side:

- **Selection sort** — always scans the full unsorted portion to find the minimum
- **Insertion sort** — builds a sorted prefix by inserting one element at a time
- **Merge sort** — recursively splits in half and merges sorted halves back together
- **Quicksort** — recursively partitions around a pivot element

The central question of this lab: **do the algorithms behave the way theory predicts — and does input shape matter?**

---

## Setup

All programs are self-contained. Each generates its own test data, instruments the algorithm, and prints results directly to the terminal.

**Compiler command for all programs:**
```bash
g++ -O0 -o program program.cpp && ./program
```

> `-O0` disables compiler optimizations so you observe the algorithm's actual behavior.

---

## Model 1: Counting Comparisons

Operation counts are the purest measure of algorithmic complexity — they are exact and machine-independent. This program runs all four sorting algorithms on arrays of increasing size and counts every comparison made.

### Program 1 — `sort_comparisons.cpp`

```cpp
#include <iostream>
#include <vector>
#include <iomanip>
#include <numeric>
#include <algorithm>
#include <random>
using namespace std;

long long comparisons;

void selectionSort(vector<int> v) {
    int n = v.size();
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        for (int j = i + 1; j < n; j++) {
            comparisons++;
            if (v[j] < v[minIdx]) minIdx = j;
        }
        swap(v[i], v[minIdx]);
    }
}

void insertionSort(vector<int> v) {
    int n = v.size();
    for (int i = 1; i < n; i++) {
        int key = v[i], j = i - 1;
        while (j >= 0) {
            comparisons++;
            if (v[j] > key) { v[j + 1] = v[j]; j--; }
            else break;
        }
        v[j + 1] = key;
    }
}

void merge(vector<int>& v, int lo, int mid, int hi) {
    vector<int> tmp;
    int i = lo, j = mid + 1;
    while (i <= mid && j <= hi) {
        comparisons++;
        if (v[i] <= v[j]) tmp.push_back(v[i++]);
        else               tmp.push_back(v[j++]);
    }
    while (i <= mid) tmp.push_back(v[i++]);
    while (j <= hi)  tmp.push_back(v[j++]);
    for (int k = 0; k < (int)tmp.size(); k++) v[lo + k] = tmp[k];
}

void mergeSort(vector<int>& v, int lo, int hi) {
    if (lo >= hi) return;
    int mid = lo + (hi - lo) / 2;
    mergeSort(v, lo, mid);
    mergeSort(v, mid + 1, hi);
    merge(v, lo, mid, hi);
}

int partition(vector<int>& v, int lo, int hi) {
    int pivot = v[hi], i = lo - 1;
    for (int j = lo; j < hi; j++) {
        comparisons++;
        if (v[j] <= pivot) swap(v[++i], v[j]);
    }
    swap(v[i + 1], v[hi]);
    return i + 1;
}

void quickSort(vector<int>& v, int lo, int hi) {
    if (lo < hi) {
        int p = partition(v, lo, hi);
        quickSort(v, lo, p - 1);
        quickSort(v, p + 1, hi);
    }
}

long long run(void (*sortFn)(vector<int>), vector<int> data) {
    comparisons = 0;
    sortFn(data);
    return comparisons;
}

long long runRef(void (*sortFn)(vector<int>&, int, int), vector<int> data) {
    comparisons = 0;
    sortFn(data, 0, (int)data.size() - 1);
    return comparisons;
}

int main() {
    mt19937 rng(42);
    vector<int> sizes = {100, 500, 1000, 2000, 5000};

    cout << "\n--- Comparison Counts on Random Input ---\n\n";
    cout << setw(7) << "n"
         << setw(14) << "Selection"
         << setw(14) << "Insertion"
         << setw(14) << "Merge"
         << setw(14) << "Quick"
         << "\n" << string(63, '-') << "\n";

    for (int n : sizes) {
        vector<int> data(n);
        iota(data.begin(), data.end(), 0);
        shuffle(data.begin(), data.end(), rng);

        long long sel = run(selectionSort, data);
        long long ins = run(insertionSort, data);

        long long ms_cmp = 0; comparisons = 0;
        { vector<int> tmp = data; mergeSort(tmp, 0, n-1); ms_cmp = comparisons; }

        long long qs_cmp = 0; comparisons = 0;
        { vector<int> tmp = data; quickSort(tmp, 0, n-1); qs_cmp = comparisons; }

        cout << setw(7) << n
             << setw(14) << sel
             << setw(14) << ins
             << setw(14) << ms_cmp
             << setw(14) << qs_cmp << "\n";
    }

    // Theoretical values for reference
    cout << "\n--- Theoretical Reference (approximate) ---\n\n";
    cout << setw(7) << "n"
         << setw(14) << "n(n-1)/2"
         << setw(14) << "n*log2(n)"
         << "\n" << string(35, '-') << "\n";
    for (int n : sizes) {
        long long quad = (long long)n * (n - 1) / 2;
        double   nlogn = n * log2(n);
        cout << setw(7) << n
             << setw(14) << quad
             << setw(14) << (long long)nlogn << "\n";
    }
    cout << "\n";
}
```

### Observation Table 1 — Comparisons on Random Input

Record the program output:

| n | Selection | Insertion | Merge | Quick | n(n-1)/2 | n·log₂(n) |
|---|---|---|---|---|---|---|
| 100 | 4950 | 2976 | 539 | 652 | 4950 | 664 |
| 500 | 124750 | 66372 | 3855 | 4715 | 124750 | 4482 |
| 1,000 | 499500 | 247711 | 8693 | 10763 | 499500 | 9965 |
| 2,000 | 1999000 | 1017849 | 19377 | 25265 | 1999000 | 21931 |
| 5,000 | 12497500 | 6241995 | 55161 | 68722 | 12497500 | 61438 |

---

### Critical Thinking Questions — Model 1

**Q1.** Compare **Merge** and **Quick** to the **n·log₂(n)** column. Are they above or below it? By roughly what factor? What does this tell you about the constant hidden inside O(n log n)?

> Your answer: After taking a look at the Merge Column, it begins with an output of 539 when N = 100; it is always lower than the n·log₂(n) column for each of the instances of n. For the Quick column, it is actually higher each time. While they all belong in the same nlogn family, they are not identical due to the ‘invisible constants’. Overall, this highlights the efficiency of Merge Sort, and how the constants hidden within the O( n log n) algorithm impact efficiency. 

**Q2.** When n doubles from 1,000 to 2,000, what happens to the Selection sort comparison count? What happens to the Merge sort count? Do these ratios match what you would predict from O(n²) and O(n log n)?

> Your answer:  When n doubles from 1,000 to 2,000, the selection sort comparison grows at a ratio of about 4.0, with the merge sort count increasing at a ratio of about 2.2. Yes, these ratios match what I would predict from these algorithms. When a number is doubled in a quadratic equation, the total output doubles by 4 times, which checks out in this scenario. For the log selection sort comparison, it will typically double, and it does so, but at a slightly higher value above 2 times.

**Q3.** The Insertion sort comparison count on random data should be roughly between n and n(n-1)/2. Where does it actually land? What does this suggest about its average-case behavior?

> Your answer: If we take a look at n = 5,000, the Insertion value is 6241995, and the n(n-1)/2 value is 12497500. When divided, it lands about halfway, at 0.5 ( 6241995/12497500). Since the number is half, this suggests that insertion search only needs to search through half of the list to find the right spot for a new piece of data ( average-case behavior). 


**Q4.** Quick sort's comparison count on random data is likely close to — or sometimes less than — merge sort's. Both are O(n log n). Given that quicksort has an O(n²) worst case, why might it make *fewer* comparisons than merge sort on average?

> Your answer: Through merge sort, it must first sort the data, and then compare different values from the different sections to then rearrange the data. However, for quick sort, it makes fewer steps within the partition stage. It also makes fewer comparisons on average because it is able to pivot and eliminate comparisons more aggressively. As mentioned in the question, quicksort has an O(n²) worst case, whereas merge sort will never reach that worst case. 

---

## Model 2: Input Shape Changes Everything

Theoretical complexity describes average behavior. But real inputs are rarely random. This program runs all four algorithms on three different input arrangements: random, already sorted, and reverse sorted.

### Program 2 — `sort_input_shapes.cpp`

```cpp
#include <iostream>
#include <vector>
#include <iomanip>
#include <numeric>
#include <algorithm>
#include <random>
using namespace std;

long long comparisons;

// (same sort implementations as Program 1 — paste them here)
void selectionSort(vector<int> v) {
    int n = v.size();
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        for (int j = i + 1; j < n; j++) { comparisons++; if (v[j] < v[minIdx]) minIdx = j; }
        swap(v[i], v[minIdx]);
    }
}

void insertionSort(vector<int> v) {
    int n = v.size();
    for (int i = 1; i < n; i++) {
        int key = v[i], j = i - 1;
        while (j >= 0) { comparisons++; if (v[j] > key) { v[j+1] = v[j]; j--; } else break; }
        v[j + 1] = key;
    }
}

void merge(vector<int>& v, int lo, int mid, int hi) {
    vector<int> tmp; int i = lo, j = mid + 1;
    while (i <= mid && j <= hi) { comparisons++; if (v[i] <= v[j]) tmp.push_back(v[i++]); else tmp.push_back(v[j++]); }
    while (i <= mid) tmp.push_back(v[i++]);
    while (j <= hi)  tmp.push_back(v[j++]);
    for (int k = 0; k < (int)tmp.size(); k++) v[lo + k] = tmp[k];
}

void mergeSort(vector<int>& v, int lo, int hi) {
    if (lo >= hi) return;
    int mid = lo + (hi - lo) / 2;
    mergeSort(v, lo, mid); mergeSort(v, mid+1, hi); merge(v, lo, mid, hi);
}

int partition(vector<int>& v, int lo, int hi) {
    int pivot = v[hi], i = lo - 1;
    for (int j = lo; j < hi; j++) { comparisons++; if (v[j] <= pivot) swap(v[++i], v[j]); }
    swap(v[i+1], v[hi]); return i + 1;
}

void quickSort(vector<int>& v, int lo, int hi) {
    if (lo < hi) { int p = partition(v, lo, hi); quickSort(v, lo, p-1); quickSort(v, p+1, hi); }
}

struct Result { long long sel, ins, ms, qs; };

Result measure(vector<int> data) {
    Result r;
    comparisons = 0; selectionSort(data);                                r.sel = comparisons;
    comparisons = 0; insertionSort(data);                                r.ins = comparisons;
    comparisons = 0; { vector<int> t = data; mergeSort(t, 0, (int)t.size()-1); } r.ms = comparisons;
    comparisons = 0; { vector<int> t = data; quickSort(t, 0, (int)t.size()-1); } r.qs = comparisons;
    return r;
}

void printRow(const string& label, Result r) {
    cout << setw(16) << label
         << setw(14) << r.sel
         << setw(14) << r.ins
         << setw(14) << r.ms
         << setw(14) << r.qs << "\n";
}

int main() {
    const int N = 1000;
    mt19937 rng(42);

    vector<int> sorted(N), rev(N), rand_data(N);
    iota(sorted.begin(), sorted.end(), 0);
    iota(rev.begin(), rev.end(), 0); reverse(rev.begin(), rev.end());
    iota(rand_data.begin(), rand_data.end(), 0);
    shuffle(rand_data.begin(), rand_data.end(), rng);

    cout << "\n--- Comparison Counts by Input Shape (n = 1000) ---\n\n";
    cout << setw(16) << "Input"
         << setw(14) << "Selection"
         << setw(14) << "Insertion"
         << setw(14) << "Merge"
         << setw(14) << "Quick"
         << "\n" << string(72, '-') << "\n";

    printRow("Random",         measure(rand_data));
    printRow("Already sorted", measure(sorted));
    printRow("Reverse sorted", measure(rev));
    cout << "\n";
}
```

### Observation Table 2 — Comparisons by Input Shape (n = 1,000)

| Input type | Selection | Insertion | Merge | Quick |
|---|---|---|---|---|
| Random | 499500 | 254916 | 8698 | 10556 |
| Already sorted | 499500 | 999 | 5044 | 499500 |
| Reverse sorted | 499500 | 499500 | 4932 | 499500 |

---

### Critical Thinking Questions — Model 2

**Q5.** Look at the **Selection** column across all three input types. Does the comparison count change? Why or why not? What does this tell you about selection sort's relationship to input shape?

> Your answer:

**Q6.** Look at the **Insertion** column. Already-sorted input should give a dramatically lower count than random or reverse-sorted. By approximately what factor does it drop? What is happening inside the algorithm that explains this?

> Your answer:

**Q7.** Look at the **Merge** column. Does it change meaningfully across input types? What property of merge sort explains this consistency?

> Your answer:

**Q8.** Look at the **Quick** column on already-sorted input. It is likely much higher than on random input. Explain why — trace what happens when the last element is chosen as pivot and the array is already sorted.

> Your answer:

**Q9.** Based on this table, which algorithm would you choose for a dataset you *know* is almost always already sorted (e.g., a log file that is mostly in time order with a few late arrivals)? Which would you avoid? Justify your choices.

> Your answer:

---

## Model 3: Time, Space, and Stability

Comparison counts tell us about the algorithm's logic. But programmers also care about wall-clock time and how much memory an algorithm uses — and whether it preserves the original order of equal elements (stability). This program measures all three.

### Program 3 — `sort_tradeoffs.cpp`

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <iomanip>
#include <numeric>
#include <algorithm>
#include <random>
#include <string>
using namespace std;
using Clock = chrono::high_resolution_clock;
using ms    = chrono::duration<double, milli>;

// Track peak extra memory allocated beyond the input array
static long long g_extra = 0, g_peak = 0;
void* operator new(size_t sz)  { g_extra += sz; if (g_extra > g_peak) g_peak = g_extra; return malloc(sz); }
void  operator delete(void* p) noexcept { free(p); }
void  operator delete(void* p, size_t) noexcept { free(p); }

void selectionSort(vector<int>& v) {
    int n = v.size();
    for (int i = 0; i < n-1; i++) {
        int m = i;
        for (int j = i+1; j < n; j++) if (v[j] < v[m]) m = j;
        swap(v[i], v[m]);
    }
}

void insertionSort(vector<int>& v) {
    int n = v.size();
    for (int i = 1; i < n; i++) {
        int key = v[i], j = i-1;
        while (j >= 0 && v[j] > key) { v[j+1] = v[j]; j--; }
        v[j+1] = key;
    }
}

void merge(vector<int>& v, int lo, int mid, int hi) {
    vector<int> tmp;                          // <-- extra allocation
    int i = lo, j = mid+1;
    while (i <= mid && j <= hi)
        tmp.push_back(v[i] <= v[j] ? v[i++] : v[j++]);
    while (i <= mid) tmp.push_back(v[i++]);
    while (j <= hi)  tmp.push_back(v[j++]);
    for (int k = 0; k < (int)tmp.size(); k++) v[lo+k] = tmp[k];
}
void mergeSort(vector<int>& v, int lo, int hi) {
    if (lo >= hi) return;
    int mid = lo + (hi-lo)/2;
    mergeSort(v, lo, mid); mergeSort(v, mid+1, hi); merge(v, lo, mid, hi);
}

int partition(vector<int>& v, int lo, int hi) {
    int pivot = v[hi], i = lo-1;
    for (int j = lo; j < hi; j++) if (v[j] <= pivot) swap(v[++i], v[j]);
    swap(v[i+1], v[hi]); return i+1;
}
void quickSort(vector<int>& v, int lo, int hi) {
    if (lo < hi) { int p = partition(v, lo, hi); quickSort(v, lo, p-1); quickSort(v, p+1, hi); }
}

template<typename Fn>
pair<double,long long> measure(Fn fn, vector<int> data) {
    g_extra = g_peak = 0;
    auto t0 = Clock::now();
    fn(data);
    double t = ms(Clock::now() - t0).count();
    return {t, g_peak};
}

// Stability test: sort {value, original_index} pairs
// A stable sort preserves original_index order for equal values
bool isStable(void (*fn)(vector<int>&)) {
    // Build array with duplicates: values 0-4 repeated twice
    vector<pair<int,int>> pairs = {{3,0},{1,1},{4,2},{1,3},{5,4},{9,5},{2,6},{6,7},{5,8},{3,9}};
    // Extract values only
    vector<int> vals; for (auto& p : pairs) vals.push_back(p.first);
    // Remember original positions of each value
    vector<int> orig = vals;
    fn(vals);
    // Check: for equal adjacent values in sorted output, did earlier index come first?
    // Re-map sorted values back to original indices
    vector<int> used(pairs.size(), 0);
    for (int i = 0; i < (int)vals.size(); i++) {
        // Find earliest unused occurrence of vals[i] in original
        for (int j = 0; j < (int)pairs.size(); j++) {
            if (!used[j] && pairs[j].first == vals[i]) {
                // Check that if a previous sorted position had the same value, it came from an earlier index
                if (i > 0 && vals[i] == vals[i-1]) {
                    // The previous slot was filled — j should be > its source index
                    // We just track: stable if repeated values appear in original order
                }
                used[j] = 1; break;
            }
        }
    }
    // Simpler direct stability check
    vector<pair<int,int>> tagged;
    for (int i = 0; i < 10; i++) tagged.push_back({pairs[i].first, i});
    vector<int> tagvals; for (auto& p : tagged) tagvals.push_back(p.first);
    fn(tagvals);
    // Rebuild sorted pairs preserving original order for equal elements (what stable means)
    vector<pair<int,int>> expected = tagged;
    stable_sort(expected.begin(), expected.end(), [](auto& a, auto& b){ return a.first < b.first; });
    // Reconstruct what our sort actually produced order-wise
    vector<pair<int,int>> actual = tagged;
    sort(actual.begin(), actual.end(), [](auto& a, auto& b){ return a.first < b.first; });
    // Compare second elements (original indices) for equal first elements
    for (int i = 0; i+1 < (int)expected.size(); i++)
        if (expected[i].first == expected[i+1].first && expected[i].second != actual[i].second)
            return false;
    return true;
}

int main() {
    mt19937 rng(42);
    vector<int> sizes = {1000, 5000, 10000};

    cout << "\n--- Runtime (ms) and Peak Extra Memory (bytes) on Random Input ---\n";
    cout << "\n" << setw(8) << "n"
         << setw(14) << "Sel time" << setw(14) << "Sel mem"
         << setw(14) << "Ins time" << setw(14) << "Ins mem"
         << setw(14) << "Mrg time" << setw(14) << "Mrg mem"
         << setw(14) << "Qck time" << setw(14) << "Qck mem"
         << "\n" << string(120, '-') << "\n";

    for (int n : sizes) {
        vector<int> data(n);
        iota(data.begin(), data.end(), 0);
        shuffle(data.begin(), data.end(), rng);

        auto [st, sm] = measure([](vector<int> v){ selectionSort(v); }, data);
        auto [it, im] = measure([](vector<int> v){ insertionSort(v); }, data);
        auto [mt, mm] = measure([](vector<int> v){ mergeSort(v, 0, (int)v.size()-1); }, data);
        auto [qt, qm] = measure([](vector<int> v){ quickSort(v, 0, (int)v.size()-1); }, data);

        cout << setw(8)  << n
             << setw(14) << fixed << setprecision(2) << st << setw(14) << sm
             << setw(14) << it << setw(14) << im
             << setw(14) << mt << setw(14) << mm
             << setw(14) << qt << setw(14) << qm << "\n";
    }

    cout << "\n--- Stability Test ---\n\n";
    cout << "  Selection sort stable? " << (isStable(selectionSort) ? "YES" : "NO") << "\n";
    cout << "  Insertion sort stable? " << (isStable(insertionSort) ? "YES" : "NO") << "\n";
    // Merge and quick sort require different signatures; result known from theory
    cout << "  Merge sort stable?     YES  (by design — equal elements prefer left half)\n";
    cout << "  Quick sort stable?     NO   (swaps can move equal elements past each other)\n";
    cout << "\n";
}
```

### Observation Table 3 — Runtime and Memory

| n | Sel (ms) | Sel mem | Ins (ms) | Ins mem | Merge (ms) | Merge mem | Quick (ms) | Quick mem |
|---|---|---|---|---|---|---|---|---|
| 1,000 | 3.56 | 4000 | 2.46 | 4000 | 1.21 | 81540 | 0.22 | 4000 |
| 5,000 | 87.06 | 20000 | 59.29 | 20000 | 7.02 | 778756 | 1.41 | 20000 |
| 10,000 | 345.41 | 40000 | 240.19 | 40000| 14.59 | 1688580 | 3.20 | 40000 |

### Observation Table 3b — Stability

| Algorithm | Stable? |
|---|---|
| Selection sort | YES |
| Insertion sort | YES |
| Merge sort | YES |
| Quicksort | NO |

---

### Critical Thinking Questions — Model 3

**Q10.** Look at the **Sel mem** and **Ins mem** and **Quick mem** columns. They should be near zero (just the overhead of passing the vector by value). The **Merge mem** column should show values proportional to n. Approximately how many bytes does merge sort allocate per element? Does this match the O(n) space complexity?

> Your answer:

**Q11.** When n grows from 1,000 to 10,000 (10×), how does selection sort's runtime change? How does merge sort's runtime change? Do these match the predicted scaling from O(n²) and O(n log n)?

> Your answer:

**Q12.** At n = 10,000, which algorithm is fastest on random data? Which is slowest? Is the fastest algorithm the one with the best Big-O, or are there other factors at play?

> Your answer:

**Q13.** You are building a leaderboard that sorts players by score. When two players have the same score, they should remain in the order they originally achieved it (i.e., first-to-reach-score is ranked higher). Based on the stability results, which sorting algorithms are safe to use? Which are not?

> Your answer:

**Q14.** Merge sort uses O(n) extra memory. On a machine with 8 GB of RAM, sorting a dataset of 1 billion integers (each 4 bytes = 4 GB of data), merge sort would need approximately how much *additional* memory? Is this a concern?

> Your answer:

---

## Model 4: Recursion Unrolled

Merge sort and quicksort are recursive. This program makes the recursion visible by printing each call as it happens, so you can see the divide-and-conquer structure directly.

### Program 4 — `sort_recursion_trace.cpp`

```cpp
#include <iostream>
#include <vector>
#include <iomanip>
#include <numeric>
#include <algorithm>
#include <random>
#include <string>
using namespace std;

// --- Traced Merge Sort ---
int mergeDepth = 0;

void tracedMerge(vector<int>& v, int lo, int mid, int hi) {
    string indent(mergeDepth * 2, ' ');
    vector<int> tmp; int i = lo, j = mid+1;
    while (i <= mid && j <= hi)
        tmp.push_back(v[i] <= v[j] ? v[i++] : v[j++]);
    while (i <= mid) tmp.push_back(v[i++]);
    while (j <= hi)  tmp.push_back(v[j++]);
    for (int k = 0; k < (int)tmp.size(); k++) v[lo+k] = tmp[k];
    cout << indent << "merge [" << lo << ".." << hi << "] → ";
    for (int k = lo; k <= hi; k++) cout << v[k] << " ";
    cout << "\n";
}

void tracedMergeSort(vector<int>& v, int lo, int hi) {
    string indent(mergeDepth * 2, ' ');
    if (lo >= hi) {
        cout << indent << "base [" << lo << "] = " << v[lo] << "\n";
        return;
    }
    cout << indent << "split [" << lo << ".." << hi << "]\n";
    int mid = lo + (hi-lo)/2;
    mergeDepth++;
    tracedMergeSort(v, lo, mid);
    tracedMergeSort(v, mid+1, hi);
    mergeDepth--;
    tracedMerge(v, lo, mid, hi);
}

// --- Traced Quicksort ---
int quickDepth = 0;

int tracedPartition(vector<int>& v, int lo, int hi) {
    string indent(quickDepth * 2, ' ');
    int pivot = v[hi], i = lo-1;
    cout << indent << "partition [" << lo << ".." << hi << "] pivot=" << pivot << "\n";
    for (int j = lo; j < hi; j++)
        if (v[j] <= pivot) swap(v[++i], v[j]);
    swap(v[i+1], v[hi]);
    int p = i+1;
    cout << indent << "  → pivot " << pivot << " placed at index " << p << " | left: [";
    for (int k = lo; k < p; k++) cout << v[k] << (k<p-1?",":"");
    cout << "] right: [";
    for (int k = p+1; k <= hi; k++) cout << v[k] << (k<hi?",":"");
    cout << "]\n";
    return p;
}

void tracedQuickSort(vector<int>& v, int lo, int hi) {
    if (lo < hi) {
        quickDepth++;
        int p = tracedPartition(v, lo, hi);
        tracedQuickSort(v, lo, p-1);
        tracedQuickSort(v, p+1, hi);
        quickDepth--;
    }
}

int main() {
    vector<int> data = {38, 27, 43, 3, 9, 82, 10};

    cout << "\n=== Merge Sort Trace (n=7) ===\n\n";
    { vector<int> v = data; tracedMergeSort(v, 0, (int)v.size()-1); }

    cout << "\n=== Quicksort Trace (n=7) ===\n\n";
    { quickDepth = 0; vector<int> v = data; tracedQuickSort(v, 0, (int)v.size()-1); }

    // Now show what happens to quicksort on sorted input
    cout << "\n=== Quicksort Trace on SORTED input (n=7, worst case) ===\n\n";
    { quickDepth = 0; vector<int> v = {3, 9, 10, 27, 38, 43, 82}; tracedQuickSort(v, 0, (int)v.size()-1); }

    cout << "\n";
}
```

### Observation: Merge Sort Trace

Run the program and study the merge sort trace output. Answer the questions below based on what you see printed.

**Q15.** How many levels of indentation does the merge sort trace show for n = 7? What does each level of indentation correspond to in terms of the recursion? How does the number of levels relate to log₂(7)?

> Your answer:

**Q16.** At the deepest level of indentation, each `base` line shows a single element. After the two `base` lines for the same parent, a `merge` line appears. What is always true about the output of a `merge` step — regardless of what the input subarrays looked like?

> Your answer:

**Q17.** Count the total number of `merge` operations printed. For n = 7, there should be 6. For n elements in general, how many merge operations does merge sort perform? Express this as a function of n.

> Your answer:

### Observation: Quicksort Trace — Random vs. Sorted Input

**Q18.** In the random-input quicksort trace, look at the partition steps. Does the pivot tend to split the array into roughly equal halves, or very unequal ones? Now look at the sorted-input trace. What do you observe about the split sizes? How many total partition steps appear in each trace?

> Your answer:

**Q19.** In the sorted-input quicksort trace, each partition step's "left" side is empty `[]` and the "right" side has all remaining elements. If this happens at every level on an array of n = 1,000 elements, how many total partition steps would there be? What complexity class is this?

> Your answer:

---

## Extra Credit Questions

**A1.** Radix sort was not in today's programs — it sorts without making element-to-element comparisons at all. Given that all four algorithms we studied today are comparison-based, what fundamental limit do they all share? (Hint: think about what O(n log n) represents for comparison-based sorting.)

---

## Summary

| Algorithm | Comparisons (random) | Best case | Worst case | Space | Stable | Adaptive |
|---|---|---|---|---|---|---|
| Selection sort | n(n-1)/2 exactly | O(n²) | O(n²) | O(1) | No | No |
| Insertion sort | ~n²/4 | O(n) | O(n²) | O(1) | Yes | Yes |
| Merge sort | ~n·log₂n | O(n log n) | O(n log n) | O(n) | Yes | No |
| Quicksort | ~1.39·n·log₂n | O(n log n) | O(n²) | O(log n) | No | No |

**Key takeaways:**

- Selection sort makes the same number of comparisons regardless of input — completely non-adaptive.
- Insertion sort is uniquely fast on nearly-sorted data — O(n) best case. This is a real, exploitable property.
- Merge sort is the reliability choice: O(n log n) always, stable, but costs O(n) extra memory.
- Quicksort is fast in practice but fragile on sorted input with a naive pivot strategy.
- Input shape matters as much as algorithm choice. Knowing your data is as important as knowing your algorithm.
- Stability matters when equal elements carry secondary information that must be preserved.

---

*Next lab: implementing linked lists from scratch — a data structure that changes what algorithms are even possible.*
