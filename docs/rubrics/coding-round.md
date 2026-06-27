# Coding & Problem-Solving Rubric

## 4-Tier Scorecard

| Dimension | 1 Strong No-Hire | 2 No-Hire | 3 Lean Hire | 4 Strong Hire |
|-----------|---------------------|-------------|---------------|-----------------|
| **Algorithmic Thinking** | Cannot identify problem type; no approach | Identifies pattern but fails to implement | Correct approach with minor bugs | Optimal approach, clean implementation |
| **Complexity Analysis** | Cannot state Big-O | States Big-O but incorrect | Correct Big-O, cannot optimize | Analyzes multiple approaches, picks optimal |
| **Edge Cases** | Ignores edge cases entirely | Handles 1-2 when prompted | Identifies most edge cases independently | Exhaustive coverage; discusses adversarial inputs |
| **Code Quality** | Unreadable; no structure | Functional but messy; long functions | Clean, modular; reasonable naming | Production-ready; self-documenting; defensive |
| **Communication** | Cannot explain thinking | Explains only when asked | Thinks aloud; structured approach | Drives discussion; explores alternatives |
| **Testing** | Cannot verify own code | Traces one example when prompted | Tests happy path + edge cases | Writes formal tests; discusses test strategy |

## Language-Specific Expectations

| Language | SDE 2 Bar | SDE 3 Bar |
|----------|-----------|-----------|
| Python | Idiomatic (list comp, generators); knows stdlib | Async/await, decorators, memory profiling |
| C++ | RAII, STL containers, iterators | Move semantics, constexpr, template metaprogramming |
| Java | Collections framework, generics | Concurrency (ExecutorService), Stream API internals |
| Go | Goroutines, channels basics | Context propagation, race detector, interface design |

---

## Example Problem 1: Merge K Sorted Lists

**Level:** SDE 2-3 bar | **Time:** 25 min | **Category:** Heap / Divide & Conquer

### Problem Statement
Given `k` sorted linked lists, merge them into one sorted linked list.

### Expected Approach
- Use a min-heap of size `k` to track the smallest element across all lists
- Pop min, push its next node
- Alternative: divide-and-conquer merge (like merge sort)

### Solution (Python Heap)
```python
import heapq
from typing import List, Optional

class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
    def __lt__(self, other):
        return self.val < other.val

def merge_k_lists(lists: List[Optional[ListNode]]) -> Optional[ListNode]:
    heap = []
    for node in lists:
        if node:
            heapq.heappush(heap, node)
    
    dummy = ListNode(0)
    curr = dummy
    while heap:
        smallest = heapq.heappop(heap)
        curr.next = smallest
        curr = curr.next
        if smallest.next:
            heapq.heappush(heap, smallest.next)
    return dummy.next
```

### Complexity
- **Time:** O(N log k) where N = total nodes
- **Space:** O(k) for heap

### Follow-ups
1. What if lists are stored on different machines? (distributed merge)
2. How would you handle lists arriving as streams?
3. Can you do this with O(1) extra space? (in-place merge)

### Scoring Guide
| Score | Signal |
|-------|--------|
| 1 | Cannot merge two sorted lists |
| 2 | Brute-force merge all + sort (O(N log N)) |
| 3 | Heap approach with minor bugs (e.g., forgets None check) |
| 4 | Clean heap solution, discusses alternatives, handles follow-ups |

---

## Example Problem 2: Thread-Safe LRU Cache

**Level:** SDE 3+ bar | **Time:** 30 min | **Category:** Concurrency + Data Structures

### Problem Statement
Design and implement a thread-safe LRU cache with O(1) get/put operations.

### Expected Approach
- Doubly-linked list (order) + HashMap (O(1) lookup)
- Thread safety: lock granularity discussion (global lock vs striped locks)
- Bonus: read-write lock for concurrent reads

### Solution (Python Threading)
```python
import threading
from collections import OrderedDict

class ThreadSafeLRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.lock = threading.RLock()
    
    def get(self, key: int) -> int:
        with self.lock:
            if key not in self.cache:
                return -1
            self.cache.move_to_end(key)
            return self.cache[key]
    
    def put(self, key: int, value: int) -> None:
        with self.lock:
            if key in self.cache:
                self.cache.move_to_end(key)
            self.cache[key] = value
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)
```

### Complexity
- **Time:** O(1) for get/put (amortized)
- **Space:** O(capacity)

### Follow-ups
1. How would you reduce lock contention? (sharded cache)
2. What about distributed LRU across multiple nodes?
3. How do you handle cache stampede?
4. Compare RLock vs Lock when does it matter?

### Scoring Guide
| Score | Signal |
|-------|--------|
| 1 | Cannot implement basic LRU |
| 2 | LRU works but no thread safety awareness |
| 3 | Correct LRU + global lock; acknowledges contention |
| 4 | Discusses lock granularity, RWLock, sharding; production concerns |

---

## Example Problem 3: Find Median in a Stream

**Level:** SDE 2 bar | **Time:** 20 min | **Category:** Heap / Design

### Problem Statement
Design a class that supports `addNum(int)` and `findMedian() -> float` for a stream of integers.

### Expected Approach
- Two heaps: max-heap (left half), min-heap (right half)
- Balance heaps so sizes differ by at most 1
- Median = top of larger heap, or average of both tops

### Solution (Python)
```python
import heapq

class MedianFinder:
    def __init__(self):
        self.lo = []  # max-heap (inverted)
        self.hi = []  # min-heap
    
    def addNum(self, num: int) -> None:
        heapq.heappush(self.lo, -num)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))
    
    def findMedian(self) -> float:
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2.0
```

### Complexity
- **Time:** O(log n) add, O(1) median
- **Space:** O(n)

### Follow-ups
1. What if you need the median of a sliding window of size k?
2. How would you handle this in a distributed setting?
3. Can you support `removeNum`?

### Scoring Guide
| Score | Signal |
|-------|--------|
| 1 | No idea how to maintain running median |
| 2 | Sorts on every query (O(n log n) per median) |
| 3 | Two-heap approach with minor rebalancing bugs |
| 4 | Clean two-heap, discusses extensions, thread safety |
