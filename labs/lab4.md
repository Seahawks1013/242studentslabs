# Lab 4: Observing Stacks and Queues

**CPTR 242 — Sequential and Parallel Data Structures and Algorithms**
Walla Walla University

---

## Overview

In this lab you will **run complete programs and observe their behavior**. There is no coding required. Your job is to read the output carefully, record what you see, and reason about what it means.

By the end you should be able to:

- Trace the state of a stack and queue through a sequence of push, pop, enqueue, and dequeue operations
- Explain how a linked-list stack and an array-based stack produce identical external behavior through different internal mechanisms
- Describe what goes wrong when a circular array queue wraps around and why the modulo trick fixes it
- Identify a real problem — bracket matching — that is naturally solved by a stack

---

## Background: One Interface, Many Implementations

A stack and a queue are both defined entirely by their ordering rules — LIFO and FIFO respectively. The ADT says nothing about internals. Today you will observe two implementations of each: one backed by a linked list and one backed by an array. The external behavior must be identical. The internal mechanics are completely different.

The central question: **do the two implementations always agree — and where do their differences become visible?**

---

## Setup

All programs are self-contained. Each generates its own test data and prints results directly to the terminal.

**Compiler command for all programs:**
```bash
g++ -O0 -o program program.cpp && ./program
```

> `-O0` disables compiler optimizations so you observe true algorithm behavior.

---

## Model 1: Stack State Tracing

Before measuring anything, you need to be able to predict and verify what a stack contains after a sequence of operations. This program executes a fixed sequence of push and pop operations on both a linked-list stack and an array-based stack, printing the full state after every step.

### Program 1 — `stack_trace.cpp`

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
using namespace std;

// ── Linked-list stack ──────────────────────────────────────────────────
struct Node { int data; Node* next; Node(int v):data(v),next(nullptr){} };

class LinkedStack {
    Node* head = nullptr;
    int   sz   = 0;
public:
    void push(int v) { Node* n=new Node(v); n->next=head; head=n; sz++; }
    int  pop()  {
        if (!head) throw underflow_error("empty");
        int v=head->data; Node* t=head; head=head->next; delete t; sz--;
        return v;
    }
    int  peek() const { if(!head) throw underflow_error("empty"); return head->data; }
    bool empty() const { return head==nullptr; }
    int  size()  const { return sz; }

    void print(const string& label) const {
        cout << label << " [top→] ";
        for (Node* c=head; c; c=c->next) cout << c->data << " ";
        cout << "(size=" << sz << ")\n";
    }
    ~LinkedStack() { while(head){ Node* t=head->next; delete head; head=t; } }
};

// ── Array-based stack ──────────────────────────────────────────────────
class ArrayStack {
    static const int CAP = 16;
    int  data[CAP];
    int  top = -1;
public:
    void push(int v) {
        if (top==CAP-1) throw overflow_error("full");
        data[++top]=v;
    }
    int  pop()  {
        if (top==-1) throw underflow_error("empty");
        return data[top--];
    }
    int  peek() const { return data[top]; }
    bool empty() const { return top==-1; }
    int  size()  const { return top+1; }

    void print(const string& label) const {
        cout << label << " [top→] ";
        for (int i=top; i>=0; i--) cout << data[i] << " ";
        cout << "(size=" << size() << ")\n";
    }
};

void doOp(LinkedStack& ls, ArrayStack& as, const string& op, int val=0) {
    cout << "\n── " << op;
    if (op=="push") { cout << "(" << val << ")\n"; ls.push(val); as.push(val); }
    else if (op=="pop") {
        int lv=ls.pop(), av=as.pop();
        cout << " → linked=" << lv << "  array=" << av << "\n";
    }
    else if (op=="peek") {
        cout << " → linked=" << ls.peek() << "  array=" << as.peek() << "\n";
    }
    ls.print("  Linked: ");
    as.print("  Array:  ");
}

int main() {
    LinkedStack ls;
    ArrayStack  as;

    cout << "=== Stack Operation Trace ===\n";
    ls.print("  Linked: "); as.print("  Array:  ");

    doOp(ls, as, "push", 10);
    doOp(ls, as, "push", 20);
    doOp(ls, as, "push", 30);
    doOp(ls, as, "peek");
    doOp(ls, as, "pop");
    doOp(ls, as, "push", 40);
    doOp(ls, as, "push", 50);
    doOp(ls, as, "pop");
    doOp(ls, as, "pop");
    doOp(ls, as, "pop");

    cout << "\nFinal state:\n";
    ls.print("  Linked: "); as.print("  Array:  ");
    cout << "  Both empty? " << (ls.empty() && as.empty() ? "yes" : "no") << "\n";
}
```

### Observation Table 1 — Stack State After Each Operation

Fill in both columns after running the program. Record the contents from top to bottom.

| Operation | Linked stack (top → bottom) | Array stack (top → bottom) |
|---|---|---|
| Initial | *(empty)* | *(empty)* |
| push(10) | | |
| push(20) | | |
| push(30) | | |
| peek | | |
| pop | | |
| push(40) | | |
| push(50) | | |
| pop | | |
| pop | | |
| pop | | |

---

### Critical Thinking Questions — Model 1

**Q1.** After every operation, the linked stack and array stack print the same contents in the same order. They are implemented completely differently internally. What does this tell you about the relationship between an ADT and its implementation?

> Your answer:

**Q2.** The linked stack prints from `head` forward. The array stack prints from `top` downward (index `top` to 0). Both show elements in top-to-bottom order. What does the array `top` index physically represent — and how does it change on push versus pop?

> Your answer:

**Q3.** The program does `push(10), push(20), push(30), pop, push(40), push(50), pop, pop, pop`. Before running the program, predict the return value of each `pop` in order. Check your prediction against the output.

> Your prediction (three values, in order):

**Q4.** The array stack has a compile-time capacity of 16. The linked stack has no capacity limit. If you pushed 17 items onto the array stack what would happen? What would happen to the linked stack? What is the practical consequence of choosing each implementation for an application where maximum stack depth is unknown?

> Your answer:

---

## Model 2: A Stack Solving a Real Problem

Bracket matching is a classic stack application — and a good test of whether you understand LIFO behavior at a deeper level than just "last in, first out." This program runs a bracket checker on several strings and prints its internal state after each character is processed.

### Program 2 — `bracket_matching.cpp`

```cpp
#include <iostream>
#include <string>
#include <stack>
using namespace std;

bool matches(char open, char close) {
    return (open=='(' && close==')') ||
           (open=='{' && close=='}') ||
           (open=='[' && close==']');
}

void check(const string& s) {
    stack<char> st;
    cout << "\nInput: \"" << s << "\"\n";
    cout << "  Step  Char  Action              Stack (top→)\n";
    cout << "  " << string(50,'-') << "\n";

    int step = 0;
    for (char c : s) {
        step++;
        string action;
        if (c=='(' || c=='{' || c=='[') {
            st.push(c);
            action = "push '" + string(1,c) + "'";
        } else if (c==')' || c=='}' || c==']') {
            if (st.empty()) {
                action = "ERROR: closer with empty stack";
                cout << "  " << step << "     '" << c << "'  " << action << "\n";
                cout << "  RESULT: INVALID\n";
                return;
            }
            char top = st.top(); st.pop();
            if (matches(top, c))
                action = "pop '" + string(1,top) + "' matched '" + string(1,c) + "'";
            else {
                action = "ERROR: '" + string(1,top) + "' does not match '" + string(1,c) + "'";
                cout << "  " << step << "     '" << c << "'  " << action << "\n";
                cout << "  RESULT: INVALID\n";
                return;
            }
        } else {
            action = "skip";
        }

        // Print stack contents
        cout << "  " << step << "     '" << c << "'  "
             << action;
        // Print stack (we have to copy it to read without destroying)
        string stack_str = "";
        stack<char> tmp = st;
        while (!tmp.empty()) { stack_str += tmp.top(); stack_str += ' '; tmp.pop(); }
        cout << string(max(0,(int)(22-action.size())), ' ')
             << "[" << stack_str << "]\n";
    }

    if (st.empty())
        cout << "  RESULT: VALID\n";
    else
        cout << "  RESULT: INVALID (unclosed: " << st.size() << " opener(s) remain)\n";
}

int main() {
    check("({[]})");
    check("({[}])");
    check("((())");
    check("hello(world[!])");
    check("");
}
```

### Observation Table 2 — Bracket Matching Results

Record the final RESULT printed for each input string:

| Input | Result | Reason (in your words) |
|---|---|---|
| `({[]})` | | |
| `({[}])` | | |
| `((())` | | |
| `hello(world[!])` | | |
| *(empty string)* | | |

---

### Critical Thinking Questions — Model 2

**Q5.** For input `({[]})`, trace the stack contents at each step in the table below before checking your answer against the program output:

| Char | Action | Stack after (top → bottom) |
|---|---|---|
| `(` | push | |
| `{` | push | |
| `[` | push | |
| `]` | pop/match | |
| `}` | pop/match | |
| `)` | pop/match | |

**Q6.** For input `({[}])` the program reports INVALID. At which character does it fail, and what exactly is wrong? Why is a queue not a useful data structure for bracket matching — what ordering property of the stack makes the matching work correctly?

> Your answer:

**Q7.** The input `hello(world[!])` contains letters, `!`, and brackets mixed together. The program still reports VALID. What does the program do with non-bracket characters? Why is it correct to ignore them?

> Your answer:

**Q8.** The empty string input reports VALID. Is this the right answer? Justify why an empty string is considered balanced.

> Your answer:

---

## Model 3: Queue State Tracing and the Circular Array

A queue has two active ends — front and back — which makes its state harder to visualize than a stack. This program traces both a linked-list queue and a circular-array queue through the same sequence of operations, printing front, back, and contents after each step.

### Program 3 — `queue_trace.cpp`

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
using namespace std;

// ── Linked-list queue ──────────────────────────────────────────────────
struct QNode { int data; QNode* next; QNode(int v):data(v),next(nullptr){} };

class LinkedQueue {
    QNode* head = nullptr;
    QNode* tail = nullptr;
    int    sz   = 0;
public:
    void enqueue(int v) {
        QNode* n = new QNode(v);
        if (!tail) { head=tail=n; }
        else       { tail->next=n; tail=n; }
        sz++;
    }
    int dequeue() {
        if (!head) throw underflow_error("empty");
        int v=head->data; QNode* t=head;
        head=head->next;
        if (!head) tail=nullptr;   // last element: reset tail too
        delete t; sz--;
        return v;
    }
    int  front() const { return head->data; }
    bool empty() const { return head==nullptr; }
    int  size()  const { return sz; }

    void print(const string& lbl) const {
        cout << lbl << "[front→] ";
        for (QNode* c=head; c; c=c->next) cout << c->data << " ";
        cout << "[←back]  (size=" << sz << ")\n";
    }
    ~LinkedQueue() { while(head){ QNode* t=head->next; delete head; head=t; } }
};

// ── Circular-array queue ───────────────────────────────────────────────
class ArrayQueue {
    static const int CAP = 8;
    int data[CAP];
    int front_idx = 0, back_idx = 0, cnt = 0;
public:
    void enqueue(int v) {
        if (cnt==CAP) throw overflow_error("full");
        data[back_idx] = v;
        back_idx = (back_idx+1) % CAP;
        cnt++;
    }
    int dequeue() {
        if (cnt==0) throw underflow_error("empty");
        int v = data[front_idx];
        front_idx = (front_idx+1) % CAP;
        cnt--;
        return v;
    }
    int  front() const { return data[front_idx]; }
    bool empty() const { return cnt==0; }
    int  size()  const { return cnt; }

    void print(const string& lbl) const {
        // Print raw array with front and back markers
        cout << lbl << "raw[";
        for (int i=0; i<CAP; i++) {
            if (cnt>0 && i==front_idx) cout << "F:";
            if (cnt>0 && i==(back_idx==0?CAP-1:back_idx-1)) cout << "B:";
            if (cnt==0) cout << ".";
            else {
                // Is this index logically in use?
                bool inUse = false;
                for (int j=0; j<cnt; j++)
                    if ((front_idx+j)%CAP == i) { inUse=true; break; }
                cout << (inUse ? to_string(data[i]) : ".");
            }
            if (i<CAP-1) cout << "|";
        }
        cout << "]  front=" << front_idx
             << " back=" << back_idx
             << " cnt=" << cnt << "\n";
    }
};

void doQOp(LinkedQueue& lq, ArrayQueue& aq, const string& op, int val=0) {
    cout << "\n── " << op;
    if (op=="enqueue") {
        cout << "(" << val << ")\n";
        lq.enqueue(val); aq.enqueue(val);
    } else if (op=="dequeue") {
        int lv=lq.dequeue(), av=aq.dequeue();
        cout << " → linked=" << lv << "  array=" << av << "\n";
    }
    lq.print("  Linked: ");
    aq.print("  Array:  ");
}

int main() {
    LinkedQueue lq; ArrayQueue aq;
    cout << "=== Queue Operation Trace ===\n";
    lq.print("  Linked: "); aq.print("  Array:  ");

    doQOp(lq, aq, "enqueue", 10);
    doQOp(lq, aq, "enqueue", 20);
    doQOp(lq, aq, "enqueue", 30);
    doQOp(lq, aq, "dequeue");
    doQOp(lq, aq, "enqueue", 40);
    doQOp(lq, aq, "dequeue");
    doQOp(lq, aq, "dequeue");
    doQOp(lq, aq, "enqueue", 50);
    doQOp(lq, aq, "enqueue", 60);
    doQOp(lq, aq, "enqueue", 70);
    doQOp(lq, aq, "dequeue");
    doQOp(lq, aq, "dequeue");
    doQOp(lq, aq, "dequeue");
    doQOp(lq, aq, "dequeue");

    cout << "\nFinal state:\n";
    lq.print("  Linked: "); aq.print("  Array:  ");
}
```

### Observation Table 3a — Queue Contents After Each Operation

Record front-to-back contents and dequeue return values:

| Operation | Queue contents (front → back) | Dequeue returned |
|---|---|---|
| Initial | *(empty)* | — |
| enqueue(10) | | — |
| enqueue(20) | | — |
| enqueue(30) | | — |
| dequeue | | |
| enqueue(40) | | — |
| dequeue | | |
| dequeue | | |
| enqueue(50) | | — |
| enqueue(60) | | — |
| enqueue(70) | | — |
| dequeue | | |
| dequeue | | |
| dequeue | | |
| dequeue | | |

### Observation Table 3b — Circular Array Internal State

Record the `front` index, `back` index, and `cnt` printed for the array queue after each operation:

| Operation | `front` index | `back` index | `cnt` |
|---|---|---|---|
| Initial | 0 | 0 | 0 |
| enqueue(10) | | | |
| enqueue(20) | | | |
| enqueue(30) | | | |
| dequeue | | | |
| enqueue(40) | | | |
| dequeue | | | |
| dequeue | | | |
| enqueue(50) | | | |
| enqueue(60) | | | |
| enqueue(70) | | | |
| dequeue | | | |
| dequeue | | | |
| dequeue | | | |
| dequeue | | | |

---

### Critical Thinking Questions — Model 3

**Q9.** Look at Table 3a. After every operation, do the linked queue and array queue always contain the same elements in the same order? What does this confirm about the two implementations?

> Your answer:

**Q10.** Look at Table 3b. After the three initial enqueues and one dequeue, the `front` index is 1 (not 0). The element at array index 0 is logically gone — but the dequeue operation never moved any data. What did it do instead? What does this tell you about how "removal" works in a circular array queue?

> Your answer:

**Q11.** The circular array queue has a `cnt` field tracking the number of elements. An alternative design uses only `front` and `back` indices and considers the queue empty when `front == back`. What ambiguity arises in that design? Why is the `cnt` field (or an `isFull` flag) needed?

> Your answer:

---

## Model 4: Performance — Stack and Queue Operations at Scale

All four operations — push, pop, enqueue, dequeue — are O(1). But the *constant* hidden inside O(1) differs between a linked implementation and an array implementation. This program measures how long it takes to push/enqueue and pop/dequeue a large number of elements on both implementations.

### Program 4 — `stack_queue_perf.cpp`

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <iomanip>
#include <numeric>
using namespace std;
using Clock = chrono::high_resolution_clock;
using us    = chrono::duration<double, micro>;

struct Node { int data; Node* next; Node(int v):data(v),next(nullptr){} };

// Minimal linked stack/queue (no size tracking for speed)
struct LStack {
    Node* head=nullptr;
    void  push(int v){ Node* n=new Node(v); n->next=head; head=n; }
    int   pop()      { int v=head->data; Node* t=head; head=head->next; delete t; return v; }
};

struct LQueue {
    Node* head=nullptr; Node* tail=nullptr;
    void enqueue(int v){ Node* n=new Node(v); if(!tail){head=tail=n;}
                         else{tail->next=n; tail=n;} }
    int  dequeue()     { int v=head->data; Node* t=head; head=head->next;
                         if(!head)tail=nullptr; delete t; return v; }
};

// Array stack using std::vector (dynamic — no overflow)
struct VStack {
    vector<int> data;
    void push(int v) { data.push_back(v); }
    int  pop()       { int v=data.back(); data.pop_back(); return v; }
};

// Circular array queue (fixed cap — large enough for test)
struct AQueue {
    static const int CAP=5000001;
    vector<int> data;
    int front=0, back=0, cnt=0;
    AQueue(): data(CAP) {}
    void enqueue(int v){ data[back]=v; back=(back+1)%CAP; cnt++; }
    int  dequeue()     { int v=data[front]; front=(front+1)%CAP; cnt--; return v; }
};

template<typename Fn>
double timeUs(Fn fn, int reps=3) {
    auto t0=Clock::now();
    for(int i=0;i<reps;i++) fn();
    return us(Clock::now()-t0).count()/reps;
}

int main() {
    vector<int> sizes = {100000, 500000, 1000000, 5000000};

    cout << "\n--- Push n items then pop all: Linked Stack vs Vector Stack (microseconds) ---\n\n";
    cout << setw(10) << "n"
         << setw(22) << "Linked push+pop"
         << setw(22) << "Vector push+pop"
         << setw(16) << "Ratio"
         << "\n" << string(70,'-') << "\n";

    for (int n : sizes) {
        double lt = timeUs([&]{
            LStack s;
            for(int i=0;i<n;i++) s.push(i);
            for(int i=0;i<n;i++) s.pop();
        });
        double vt = timeUs([&]{
            VStack s;
            s.data.reserve(n);
            for(int i=0;i<n;i++) s.push(i);
            for(int i=0;i<n;i++) s.pop();
        });
        cout << setw(10) << n
             << setw(22) << fixed << setprecision(0) << lt
             << setw(22) << vt
             << setw(15) << setprecision(2) << (lt/vt) << "x\n";
    }

    cout << "\n--- Enqueue n items then dequeue all: Linked Queue vs Array Queue (microseconds) ---\n\n";
    cout << setw(10) << "n"
         << setw(24) << "Linked enq+deq"
         << setw(24) << "Array enq+deq"
         << setw(16) << "Ratio"
         << "\n" << string(74,'-') << "\n";

    for (int n : sizes) {
        double lt = timeUs([&]{
            LQueue q;
            for(int i=0;i<n;i++) q.enqueue(i);
            for(int i=0;i<n;i++) q.dequeue();
        });
        double at = timeUs([&]{
            AQueue q;
            for(int i=0;i<n;i++) q.enqueue(i);
            for(int i=0;i<n;i++) q.dequeue();
        });
        cout << setw(10) << n
             << setw(24) << fixed << setprecision(0) << lt
             << setw(24) << at
             << setw(15) << setprecision(2) << (lt/at) << "x\n";
    }

    cout << "\n";
}
```

### Observation Table 4a — Stack Performance (μs)

| n | Linked push+pop | Vector push+pop | Ratio |
|---|---|---|---|
| 100,000 | | | |
| 500,000 | | | |
| 1,000,000 | | | |
| 5,000,000 | | | |

### Observation Table 4b — Queue Performance (μs)

| n | Linked enq+deq | Array enq+deq | Ratio |
|---|---|---|---|
| 100,000 | | | |
| 500,000 | | | |
| 1,000,000 | | | |
| 5,000,000 | | | |

---

### Critical Thinking Questions — Model 4

**Q12.** Both linked and array implementations are O(1) per operation, so doubling n should double total time for both. When n grows from 100,000 to 5,000,000 (50×), by approximately what factor does each implementation's time grow? Is this consistent with O(n) total work?

> Your answer:

**Q13.** The array/vector implementation is faster than the linked implementation by a consistent ratio. Both do the same logical work. What accounts for the difference? Name at least one concrete reason.

> Your answer:

**Q14.** The vector stack calls `reserve(n)` before pushing. Remove the `reserve` call mentally and predict whether the timing would change significantly. What does `reserve` prevent, and what cost does it avoid?

> Your answer:

**Q15.** Based on your observations across all four models, state a concrete rule for when you would prefer a linked-list implementation of a stack or queue over an array-based one. Your rule should reference at least one specific situation where the linked implementation's properties are genuinely advantageous.

> Your answer:

---


## Extra Credit Questions


**A1.** The call stack in your computer is literally a stack. Every function call pushes a frame; every return pops one. What happens when a recursive function has no base case? Connect what you observed about the array stack's overflow behavior to the term "stack overflow."

**A2.** The linked queue's `dequeue` function has a single special-case line: `if (!head) tail = nullptr`. This line is easy to forget. Given what you traced in Model 3, describe exactly what breaks in a subsequent `enqueue` if this line is missing.

**A3.** `std::stack` and `std::queue` in C++ wrap `std::deque` internally rather than a raw array or linked list. Based on what you observed in Model 4, why might `std::deque` be a better default backing container than either of your two implementations?

---

*Next lab: hash tables — where we abandon sequential structure entirely in favor of O(1) lookup by key.*