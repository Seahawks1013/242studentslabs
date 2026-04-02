# Lab 1: Observing Algorithm Efficiency

**CPTR 242 — Sequential and Parallel Data Structures and Algorithms**
Walla Walla University

---

## Overview

In this lab you will **run complete programs and observe their behavior**. There is no coding required. Your job is to read the output carefully, record what you see, and reason about what it means.

By the end you should be able to:

- Explain what runtime and memory usage tell you about an algorithm
- Describe the difference between best-case, worst-case, and average-case performance
- Recognize O(1), O(log n), O(n), and O(n²) behavior from empirical data
- Explain why theoretical complexity and real-world measurements sometimes differ


---

## Background: What Are We Measuring?

When we say an algorithm is "efficient," we mean two things:

- **Time complexity** — how does the number of operations grow as input size `n` grows?
- **Space complexity** — how does memory usage grow as `n` grows?

We also care about *when* we measure:

- **Best case** — the luckiest possible input (e.g., searching for an item that's in position 0)
- **Worst case** — the hardest possible input (e.g., searching for an item that isn't there)
- **Average case** — a typical random input

Today you will observe all three using two classic algorithms: **linear search** and **binary search**.

---

## Setup

I suggest running the following commands from your base user directory:

```bash
mkdir cs242 
cd cs242
```

All programs in this lab are self-contained. Each one:
- Generates its own test data
- Runs the algorithm multiple times to average out noise
- Prints results directly to the terminal

Compile and run each program exactly as shown. Record the output in the tables provided.

**Compiler command for all programs:**
```bash
g++ -O0 -o program program.cpp && ./program
```

> The `-O0` flag disables compiler optimizations so you observe the algorithm's true behavior, not the compiler's cleverness.

---

## Model 1: Counting Operations

Before we look at real time, let's look at something more precise: **operation counts**.

The following program runs linear search and binary search on arrays of increasing size, and **counts exactly how many comparisons each algorithm makes**.

### Program 1 — `operation_counts.cpp`

```cpp
#include <iostream>
#include <vector>
#include <iomanip>
#include <numeric>
using namespace std;

// Returns the number of comparisons made to find target.
// Returns -1 if not found (count still recorded).
int linearSearch(const vector<int>& v, int target, long long& comparisons) {
    comparisons = 0;
    for (int i = 0; i < (int)v.size(); i++) {
        comparisons++;
        if (v[i] == target) return i;
    }
    return -1;
}

int binarySearch(const vector<int>& v, int target, long long& comparisons) {
    comparisons = 0;
    int lo = 0, hi = (int)v.size() - 1;

    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;

        comparisons++; // == comparison
        if (v[mid] == target) return mid;

        comparisons++; // < comparison
        if (v[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

void printRow(int n, long long linBest, long long linWorst, long long linAvg,
                      long long binBest, long long binWorst, long long binAvg) {
    cout << setw(10) << n
         << setw(12) << linBest  << setw(12) << linWorst << setw(12) << linAvg
         << setw(12) << binBest  << setw(12) << binWorst << setw(12) << binAvg
         << "\n";
}

int main() {
    vector<int> sizes = {10, 100, 1000, 10000, 100000};

    cout << "\n--- Operation Counts: Linear Search vs. Binary Search ---\n\n";
    cout << setw(10) << "n"
         << setw(12) << "Lin Best" << setw(12) << "Lin Worst" << setw(12) << "Lin Avg"
         << setw(12) << "Bin Best" << setw(12) << "Bin Worst" << setw(12) << "Bin Avg"
         << "\n";
    cout << string(82, '-') << "\n";

    for (int n : sizes) {
        // Sorted array: [0, 1, 2, ..., n-1]
        vector<int> arr(n);
        iota(arr.begin(), arr.end(), 0);

        long long comparisons;

        // Best case: target is the first element (linear) or exact midpoint (binary)
        linearSearch(arr, 0, comparisons);
        long long linBest = comparisons;

        binarySearch(arr, arr[(n - 1) / 2], comparisons);
        long long binBest = comparisons;

        // Worst case: target is not in the array
        linearSearch(arr, n, comparisons);   // n is one past the end
        long long linWorst = comparisons;

        binarySearch(arr, n, comparisons);
        long long binWorst = comparisons;

        // Average case: search for each element once, take the mean
        long long totalLin = 0, totalBin = 0;
        int sampleStep = max(1, n / 200);    // sample up to 200 values
        int samples = 0;
        for (int i = 0; i < n; i += sampleStep) {
            linearSearch(arr, arr[i], comparisons);
            totalLin += comparisons;
            binarySearch(arr, arr[i], comparisons);
            totalBin += comparisons;
            samples++;
        }
        long long linAvg = totalLin / samples;
        long long binAvg = totalBin / samples;

        printRow(n, linBest, linWorst, linAvg, binBest, binWorst, binAvg);
    }

    cout << "\n";
    return 0;
}
```

### Observation Table 1 — Record Your Output

Run the program and fill in the table:

| n | Lin Best | Lin Worst | Lin Avg | Bin Best | Bin Worst | Bin Avg |
|---|---|---|---|---|---|---|
| 10 | 1 | 10 | 5 | 1  | 8 | 4 |
| 100 | 1 | 100 | 50 | 1 | 14 | 10 |
| 1,000 | 1 | 1000 | 498 | 1 | 20 | 17 |
| 10,000 | 1 | 10000 | 4976 | 1 | 28 | 23 |
| 100,000 | 1 | 100000 | 49751 | 1 | 34 | 30 |

---

### Critical Thinking Questions — Model 1

**Q1.** Look at the **Lin Worst** column. Each time `n` grows by 10×, what happens to the number of comparisons? Write a formula relating Lin Worst to `n`.

> Your answer:

**Q2.** Look at the **Bin Worst** column. Each time `n` grows by 10×, what happens? Is the growth the same as linear search? Describe the pattern in words.

> Your answer:

**Q3.** The **Bin Best** column should show `1` for all values of `n`. Why? What is the best-case scenario for binary search, and why does the array size not affect it?

> Your answer:

**Q4.** Compare **Lin Avg** to **Lin Worst**. What is the approximate ratio between them across all values of `n`? What does this tell you about where a typical search ends up?

> Your answer:

**Q5.** For `n = 100,000`, binary search makes far fewer comparisons than linear search in the worst case. But for `n = 10`, the difference is small. Does this mean binary search is *always* the right choice? What would make you prefer linear search?

> Your answer:

---

## Model 2: Measuring Real Time

Operation counts are exact and machine-independent. But programmers also care about **wall-clock time** — how long the user actually waits.

### Program 2 — `runtime_measurement.cpp`

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <numeric>
#include <iomanip>
#include <cmath>
using namespace std;
using Clock = chrono::high_resolution_clock;
using us    = chrono::duration<double, micro>;

int linearSearch(const vector<int>& v, int target) {
    for (int i = 0; i < (int)v.size(); i++)
        if (v[i] == target) return i;
    return -1;
}

int binarySearch(const vector<int>& v, int target) {
    int lo = 0, hi = (int)v.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if      (v[mid] == target) return mid;
        else if (v[mid] <  target) lo = mid + 1;
        else                       hi = mid - 1;
    }
    return -1;
}

// Run fn() REPS times and return average microseconds.
template<typename Fn>
double timeIt(Fn fn, int reps = 500) {
    auto t0 = Clock::now();
    for (int i = 0; i < reps; i++) fn();
    auto t1 = Clock::now();
    return us(t1 - t0).count() / reps;
}

int main() {
    vector<int> sizes = {1000, 10000, 100000, 1000000, 10000000};

    cout << "\n--- Runtime Measurement (microseconds, worst-case search) ---\n\n";
    cout << setw(12) << "n"
         << setw(18) << "Linear (us)"
         << setw(18) << "Binary (us)"
         << setw(18) << "Speedup"
         << "\n";
    cout << string(66, '-') << "\n";

    for (int n : sizes) {
        vector<int> arr(n);
        iota(arr.begin(), arr.end(), 0);

        int target = n;   // not in array — worst case for both

        double linTime = timeIt([&]{ linearSearch(arr, target); });
        double binTime = timeIt([&]{ binarySearch(arr, target); });

        cout << setw(12) << n
             << setw(18) << fixed << setprecision(2) << linTime
             << setw(18) << binTime
             << setw(18) << setprecision(1) << (linTime / binTime) << "x"
             << "\n";
    }

    cout << "\n--- Runtime Measurement (microseconds, best-case search) ---\n\n";
    cout << setw(12) << "n"
         << setw(18) << "Linear (us)"
         << setw(18) << "Binary (us)"
         << "\n";
    cout << string(48, '-') << "\n";

    for (int n : sizes) {
        vector<int> arr(n);
        iota(arr.begin(), arr.end(), 0);

        double linTime = timeIt([&]{ linearSearch(arr, 0); });      // first element
        double binTime = timeIt([&]{ binarySearch(arr, arr[n/2]); }); // midpoint

        cout << setw(12) << n
             << setw(18) << fixed << setprecision(2) << linTime
             << setw(18) << binTime
             << "\n";
    }

    cout << "\n";
    return 0;
}
```

### Observation Table 2a — Worst-Case Runtime

| n | Linear (μs) | Binary (μs) | Speedup |
|---|---|---|---|
| 1,000 | 13.96  | 0.26 | 54.7x |
| 10,000 | 142.13 | 0.29 | 491.9x |
| 100,000 | 1255.13 | 0.25 | 5110.2x |
| 1,000,000 | 6290.82 | 0.15 | 40818.2x |
| 10,000,000 | 48625.98 | 0.17 | 290974.9x |

### Observation Table 2b — Best-Case Runtime

| n | Linear (μs) | Binary (μs) |
|---|---|---|
| 1,000 | 0.01 | 0.07 | 
| 10,000 | 0.01 | 0.09 |
| 100,000 | 0.01 | 0.14 |
| 1,000,000 | 0.01 | 0.14 |
| 10,000,000 | 0.01 | 0.17 |

---

### Critical Thinking Questions — Model 2

**Q6.** In Table 2a, when `n` grows from 1,000 to 10,000 (10×), approximately how much does linear search time increase? Does this match what you predicted from your answer to Q1?

> Your answer:

**Q7.** Look at the **Speedup** column in Table 2a. Does the speedup stay constant, grow, or shrink as `n` increases? Explain why, using the operation counts from Model 1 as evidence.

> Your answer:

**Q8.** In Table 2b (best case), what happens to linear search time as `n` grows? What about binary search? Why are these results so different from the worst case?

> Your answer:

---

## Model 3: Memory Usage

Runtime is only half the picture. Algorithms also consume **memory**. This program measures how much heap memory each approach allocates as `n` grows.

### Program 3 — `memory_usage.cpp`

```cpp
#include <iostream>
#include <vector>
#include <iomanip>
#include <numeric>
using namespace std;

// A simple allocator that tracks total bytes allocated on the heap.
static long long g_allocated = 0;

void* operator new(size_t size) {
    g_allocated += size;
    return malloc(size);
}
void operator delete(void* ptr) noexcept { free(ptr); }
void operator delete(void* ptr, size_t) noexcept { free(ptr); }

long long measure(int n) {
    g_allocated = 0;
    {
        vector<int> arr(n);
        iota(arr.begin(), arr.end(), 0);
        // Searching doesn't allocate — all work is done in-place.
        // This measures the cost of *storing* the data structure.
    }
    return g_allocated;
}

// Recursive binary search — allocates a stack frame per call.
long long recursiveBinSearch(const vector<int>& v, int target, int lo, int hi) {
    if (lo > hi) return 0;
    int mid = lo + (hi - lo) / 2;
    if      (v[mid] == target) return 0;
    else if (v[mid] <  target) return 1 + recursiveBinSearch(v, target, mid+1, hi);
    else                       return 1 + recursiveBinSearch(v, target, lo, mid-1);
}

int main() {
    vector<int> sizes = {1000, 10000, 100000, 1000000};

    cout << "\n--- Memory Usage: Storing the Array ---\n\n";
    cout << setw(10) << "n"
         << setw(20) << "Heap bytes (array)"
         << setw(22) << "Bytes per element"
         << "\n";
    cout << string(52, '-') << "\n";

    for (int n : sizes) {
        long long bytes = measure(n);
        cout << setw(10) << n
             << setw(20) << bytes
             << setw(22) << fixed << setprecision(1) << (double)bytes / n
             << "\n";
    }

    cout << "\n--- Call Stack Depth: Iterative vs. Recursive Binary Search ---\n\n";
    cout << setw(10) << "n"
         << setw(22) << "Iter stack frames"
         << setw(24) << "Recur stack frames (worst)"
         << "\n";
    cout << string(56, '-') << "\n";

    for (int n : sizes) {
        vector<int> arr(n);
        iota(arr.begin(), arr.end(), 0);

        // Iterative: always 1 stack frame (the function itself)
        long long iterFrames = 1;

        // Recursive: depth = number of recursive calls until base case
        long long recurFrames = 1 + recursiveBinSearch(arr, n, 0, n - 1);

        cout << setw(10) << n
             << setw(22) << iterFrames
             << setw(24) << recurFrames
             << "\n";
    }

    cout << "\n";
    return 0;
}
```

### Observation Table 3a — Array Memory

| n | Heap bytes | Bytes per element |
|---|---|---|
| 1,000 | 4000 | 4.0 |
| 10,000 | 40000 | 4.0 |
| 100,000 | 400000| 4.0 |
| 1,000,000 | 4000000 | 4.0 |

### Observation Table 3b — Call Stack Depth

| n | Iterative frames | Recursive frames (worst) |
|---|---|---|
| 1,000 | 1 | 11 |
| 10,000 | 1 | 15 |
| 100,000 | 1 | 18 |
| 1,000,000 | 1 | 21 |

---

### Critical Thinking Questions — Model 3


**Q9.** When `n` doubles, what happens to total heap memory? What complexity class describes the memory usage of storing an array of `n` integers?

> Your answer:

**Q10.** Look at Table 3b. The iterative version always uses 1 stack frame. The recursive version uses more. How does the number of recursive frames grow as `n` grows? Does this match O(log n)?

> Your answer:


**Q11.** Both the iterative and recursive binary search are O(log n) in *time* complexity. But their *space* complexity is different. State the space complexity of each and explain why they differ.

> Your answer:

---

## Model 4: Putting It Together — The Full Picture

This final program runs an O(n²) algorithm alongside the O(n) and O(log n) algorithms, so you can see all three growth rates in the same table.

### Program 4 — `complexity_comparison.cpp`

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <numeric>
#include <iomanip>
#include <algorithm>
using namespace std;
using Clock = chrono::high_resolution_clock;
using ms    = chrono::duration<double, milli>;

// O(n²) — selection sort (we only care about time, not the sorted result)
void selectionSort(vector<int> v) {   // pass by value intentionally
    int n = v.size();
    for (int i = 0; i < n - 1; i++) {
        int minIdx = i;
        for (int j = i + 1; j < n; j++)
            if (v[j] < v[minIdx]) minIdx = j;
        swap(v[i], v[minIdx]);
    }
}

// O(n) — linear scan
void linearScan(const vector<int>& v) {
    volatile long long sum = 0;    // volatile prevents the compiler optimizing this away
    for (int x : v) sum += x;
}

// O(log n) — binary search, worst case
int binarySearch(const vector<int>& v, int target) {
    int lo = 0, hi = (int)v.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if      (v[mid] == target) return mid;
        else if (v[mid] <  target) lo = mid + 1;
        else                       hi = mid - 1;
    }
    return -1;
}

template<typename Fn>
double timeMs(Fn fn, int reps = 5) {
    auto t0 = Clock::now();
    for (int i = 0; i < reps; i++) fn();
    return ms(Clock::now() - t0).count() / reps;
}

int main() {
    // Selection sort is slow — keep n small for it
    vector<int> sortSizes = {500, 1000, 2000, 4000, 8000};

    cout << "\n--- Complexity Comparison (milliseconds) ---\n\n";
    cout << setw(8)  << "n"
         << setw(18) << "O(n²) sort"
         << setw(18) << "O(n) scan"
         << setw(18) << "O(log n) search"
         << "\n";
    cout << string(62, '-') << "\n";

    for (int n : sortSizes) {
        vector<int> arr(n);
        iota(arr.begin(), arr.end(), 0);

        double t_sort = timeMs([&]{ selectionSort(arr); });
        double t_scan = timeMs([&]{ linearScan(arr); }, 50);
        double t_bin  = timeMs([&]{ binarySearch(arr, n); }, 50000);

        cout << setw(8)  << n
             << setw(18) << fixed << setprecision(3) << t_sort
             << setw(18) << t_scan
             << setw(18) << t_bin
             << "\n";
    }

    cout << "\n--- Predicted vs. Observed Scaling (O(n²) sort) ---\n";
    cout << "(Each row doubles n — O(n²) should multiply time by ~4)\n\n";
    cout << setw(8) << "n" << setw(18) << "Time (ms)" << setw(20) << "Ratio vs prev n" << "\n";
    cout << string(46, '-') << "\n";

    double prev = -1;
    for (int n : sortSizes) {
        vector<int> arr(n);
        iota(arr.begin(), arr.end(), 0);
        double t = timeMs([&]{ selectionSort(arr); });
        string ratio = (prev < 0) ? "—" : to_string(t / prev).substr(0, 4) + "x";
        cout << setw(8) << n << setw(18) << fixed << setprecision(3) << t
             << setw(20) << ratio << "\n";
        prev = t;
    }

    cout << "\n";
    return 0;
}
```

### Observation Table 4a — Three Growth Rates Side by Side

| n | O(n²) sort (ms) | O(n) scan (ms) | O(log n) search (ms) |
|---|---|---|---|
| 500 | 1.940 | 0.015 | 0.000|
| 1,000 | 7.865 | 0.031 | 0.000|
| 2,000 | 30.229 | 0.061 | 0.000 |
| 4,000 | 99.302 | 0.077 | 0.000 |
| 8,000 | 245.008 | 0.107 | 0.000 |

### Observation Table 4b — O(n²) Scaling

| n | Time (ms) | Ratio vs. prev |
|---|---|---|
| 500 | 0.881 | — |
| 1,000 | 3.488 | 3.96x |
| 2,000 | 14.044 | 4.02x |
| 4,000 | 55.044 | 3.91x |
| 8,000 | 212.299 | 3.85x |

---

### Critical Thinking Questions — Model 4

**Q12.** In Table 4a, the O(log n) search times are probably so small they appear as `0.000` ms. This doesn't mean the algorithm takes zero time — it means our millisecond timer isn't precise enough. What does this tell you about using wall-clock time for benchmarking very fast algorithms?

> Your answer:

**Q13.** At `n = 8,000`, how much longer does the O(n²) sort take compared to the O(n) scan? Now imagine `n = 1,000,000`. Using the complexity ratios, estimate how much longer the O(n²) algorithm would take relative to O(n) at that scale. Show your reasoning.

> Your answer:

**Q14.** The O(n) scan uses `volatile` on the accumulator variable. Remove the `volatile` keyword, recompile with `-O2` instead of `-O0`, and run again. What happens to the O(n) scan time? Why? What does this reveal about the relationship between benchmarking and compiler optimizations?

> Your answer:

**Q15.** Suppose you are building an app that needs to sort a list of items every time a user opens it. The list is currently 1,000 items and your O(n²) sort takes 1 ms — fast enough. Your product grows and the list becomes 100,000 items. Using your observed scaling ratio, estimate the new sort time. Is it still acceptable? What would you do?

> Your answer:

---

**Key takeaways:**

- Operation counts are the purest measure of complexity — they are machine-independent.
- Wall-clock time is useful but noisy. Compiler optimizations, caching, and hardware all affect it.
- Best and worst case can differ dramatically. Always ask: what input triggers the worst case?
- Space complexity is a second dimension of cost. A faster algorithm is not always worth it if it requires significantly more memory.
- O(n²) algorithms that feel fast on small inputs become unacceptably slow at scale.

---

## Submission Instructions

Create a new **public** Github Repository called `cs242`, upload your local `cs242` folder there including all answers from this lab.

Each answer should be 2-5 sentences that demonstrate your understanding of the concepts through the lens of the exercises you ran.

Email the GitHub repository web link to me at `chike.abuah@wallawalla.edu`

*If you're concerned about privacy* 

You can make a **private** Github Repo and add me as a collaborator, my username is `abuach`.

Congrats, you're done with the first lab!

---
