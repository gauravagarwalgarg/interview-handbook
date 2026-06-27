# Data Structures & Algorithms  Embedded/Systems Focus

**Focus: Problems relevant to firmware, real-time systems, and resource-constrained environments. Not LeetCode grinding  practical DSA that shows up in production embedded C++.**

---

## How to Use This Section

| Difficulty | Time | When to Ask |
|-----------|------|-------------|
| ⚡ Easy | 5 min | Warm-up, gauge fundamentals |
| 🔬 Medium | 10-15 min | Core assessment |
| 🏗️ Hard | 20 min | Senior/Staff differentiation |

---

## Ring Buffers & Queues

### ⚡ Q1: Implement a Lock-Free SPSC Ring Buffer

**Why it matters:** Every embedded system uses ring buffers  ISR to task, DMA buffers, logging.

**Question:** Implement a single-producer single-consumer ring buffer that:
- Is lock-free (usable from ISR)
- Power-of-2 capacity (for fast modulo)
- Detects full/empty without wasting a slot

```cpp
template <typename T, std::size_t N>
class RingBuffer {
    static_assert((N & (N - 1)) == 0, "N must be power of 2");

public:
    bool push(const T& item) noexcept {
        const auto h = head_.load(std::memory_order_relaxed);
        const auto next = (h + 1) & Mask;
        if (next == tail_.load(std::memory_order_acquire))
            return false;  // Full
        buf_[h] = item;
        head_.store(next, std::memory_order_release);
        return true;
    }

    bool pop(T& item) noexcept {
        const auto t = tail_.load(std::memory_order_relaxed);
        if (t == head_.load(std::memory_order_acquire))
            return false;  // Empty
        item = buf_[t];
        tail_.store((t + 1) & Mask, std::memory_order_release);
        return true;
    }

    std::size_t size() const noexcept {
        return (head_.load(std::memory_order_relaxed) -
                tail_.load(std::memory_order_relaxed)) & Mask;
    }

private:
    static constexpr std::size_t Mask = N - 1;
    std::array<T, N> buf_{};
    alignas(64) std::atomic<std::size_t> head_{0};
    alignas(64) std::atomic<std::size_t> tail_{0};
};
```

**Follow-up:** Why `alignas(64)`? (False sharing prevention  head and tail on separate cache lines.)

---

### 🔬 Q2: Priority Queue for Event Scheduling

**Scenario:** An RTOS scheduler needs a timer queue. Events fire at specific tick counts. Implement an efficient insertion and extraction.

**Expected:** Min-heap with O(log n) insert/extract-min.

```cpp
template <typename T, std::size_t MaxSize>
class MinHeap {
public:
    struct Entry {
        uint32_t priority;  // Lower = sooner
        T data;
        bool operator>(const Entry& o) const { return priority > o.priority; }
    };

    bool insert(uint32_t priority, const T& data) {
        if (size_ >= MaxSize) return false;
        heap_[size_] = {priority, data};
        sift_up(size_);
        ++size_;
        return true;
    }

    bool extract_min(T& out) {
        if (size_ == 0) return false;
        out = heap_[0].data;
        heap_[0] = heap_[--size_];
        sift_down(0);
        return true;
    }

    uint32_t peek_priority() const { return size_ > 0 ? heap_[0].priority : UINT32_MAX; }
    bool empty() const { return size_ == 0; }

private:
    void sift_up(std::size_t i) {
        while (i > 0) {
            std::size_t parent = (i - 1) / 2;
            if (heap_[i] > heap_[parent]) break;
            std::swap(heap_[i], heap_[parent]);
            i = parent;
        }
    }

    void sift_down(std::size_t i) {
        while (2 * i + 1 < size_) {
            std::size_t child = 2 * i + 1;
            if (child + 1 < size_ && heap_[child] > heap_[child + 1]) ++child;
            if (!(heap_[i] > heap_[child])) break;
            std::swap(heap_[i], heap_[child]);
            i = child;
        }
    }

    std::array<Entry, MaxSize> heap_{};
    std::size_t size_ = 0;
};
```

**Follow-up:** What's the time complexity of building a heap from N elements? (O(n) with bottom-up heapify, not O(n log n))

---

## Hash Tables & Lookup

### ⚡ Q3: Fixed-Size Hash Map for Command Dispatch

**Scenario:** You have 32 command IDs (uint8_t). Map each to a handler function pointer with O(1) lookup, no heap.

**Expected:** Direct-indexed array (perfect hash for small key space).

```cpp
using CmdHandler = void(*)(const uint8_t* payload, std::size_t len);

class CommandDispatcher {
public:
    void register_handler(uint8_t cmd_id, CmdHandler handler) {
        table_[cmd_id] = handler;
    }

    bool dispatch(uint8_t cmd_id, const uint8_t* payload, std::size_t len) const {
        if (auto h = table_[cmd_id]) {
            h(payload, len);
            return true;
        }
        return false;  // Unregistered command
    }

private:
    std::array<CmdHandler, 256> table_{};  // Direct index  O(1)
};
```

**Follow-up:** When would you use a hash map instead of direct indexing? (When key space is sparse/large  e.g., 32-bit message IDs with only 50 registered.)

---

### 🔬 Q4: Open-Addressing Hash Map (No Heap)

**Question:** Implement a fixed-capacity hash map with linear probing. Used for firmware symbol tables, configuration stores, or CAN message routing.

```cpp
template <typename K, typename V, std::size_t Capacity>
class FixedHashMap {
public:
    bool insert(const K& key, const V& value) {
        if (size_ >= Capacity * 3 / 4) return false;  // Load factor limit

        std::size_t idx = hash(key) % Capacity;
        while (slots_[idx].occupied) {
            if (slots_[idx].key == key) {
                slots_[idx].value = value;  // Update existing
                return true;
            }
            idx = (idx + 1) % Capacity;  // Linear probe
        }
        slots_[idx] = {key, value, true};
        ++size_;
        return true;
    }

    V* find(const K& key) {
        std::size_t idx = hash(key) % Capacity;
        std::size_t probes = 0;
        while (slots_[idx].occupied && probes < Capacity) {
            if (slots_[idx].key == key) return &slots_[idx].value;
            idx = (idx + 1) % Capacity;
            ++probes;
        }
        return nullptr;
    }

private:
    struct Slot { K key; V value; bool occupied = false; };
    std::array<Slot, Capacity> slots_{};
    std::size_t size_ = 0;

    std::size_t hash(const K& key) const {
        // FNV-1a for bytes, or std::hash for standard types
        return std::hash<K>{}(key);
    }
};
```

**Follow-up:** Why load factor 75%? What happens at higher load? (Probe chains grow exponentially  O(1) degrades toward O(n). At 90%+ it's nearly linear scan.)

---

## Sorting & Searching

### ⚡ Q5: Binary Search on Sorted Sensor Calibration Table

**Scenario:** ADC readings map to physical values via a calibration table (sorted, 64 entries). Implement interpolation lookup.

```cpp
struct CalPoint { uint16_t adc; float physical; };

float interpolate(std::span<const CalPoint> table, uint16_t adc_reading) {
    // Binary search for bounding points
    auto it = std::lower_bound(table.begin(), table.end(), adc_reading,
        [](const CalPoint& p, uint16_t val) { return p.adc < val; });

    if (it == table.end()) return table.back().physical;
    if (it == table.begin()) return table.front().physical;

    const auto& high = *it;
    const auto& low = *(it - 1);

    // Linear interpolation
    float fraction = static_cast<float>(adc_reading - low.adc) /
                     static_cast<float>(high.adc - low.adc);
    return low.physical + fraction * (high.physical - low.physical);
}
```

**Follow-up:** Time complexity? (O(log n) search + O(1) interpolation.)

---

### 🔬 Q6: Insertion Sort for Nearly-Sorted Sensor Data

**Question:** Sensor samples arrive mostly in order but network jitter causes occasional out-of-order delivery (at most K positions out of place). What's the best sort?

**Expected:** Insertion sort is O(nK) for K-sorted data  optimal for small K. Or use a min-heap of size K for streaming (O(n log K)).

```cpp
// Insertion sort  best for nearly-sorted, in-place, stable
template <typename T, typename Compare = std::less<T>>
void insertion_sort(std::span<T> data, Compare cmp = {}) {
    for (std::size_t i = 1; i < data.size(); ++i) {
        T key = std::move(data[i]);
        std::size_t j = i;
        while (j > 0 && cmp(key, data[j - 1])) {
            data[j] = std::move(data[j - 1]);
            --j;
        }
        data[j] = std::move(key);
    }
}
```

**When to use which sort in embedded:**

| Algorithm | Best For | Time | Space | Stable? |
|-----------|----------|------|-------|---------|
| Insertion sort | Nearly sorted, small N (<50) | O(nK) | O(1) | ✅ |
| Heapsort | Guaranteed O(n log n), no extra memory | O(n log n) | O(1) | ❌ |
| Merge sort | Stable sort needed, linked lists | O(n log n) | O(n) | ✅ |
| Counting sort | Small integer keys (e.g., 0-255) | O(n+k) | O(k) | ✅ |
| `std::sort` | General purpose | O(n log n) | O(log n) | ❌ |

---

## Graphs & Trees

### 🔬 Q7: State Reachability Analysis (BFS/DFS)

**Scenario:** Given a state machine definition (states + transitions), verify that:
1. All states are reachable from the initial state
2. No dead-end states exist (every state can reach a terminal or loop)
3. Detect unreachable code paths

```cpp
struct StateMachine {
    std::size_t num_states;
    std::size_t initial_state;
    // adjacency list: transitions[from] = {to1, to2, ...}
    std::vector<std::vector<std::size_t>> transitions;
};

std::vector<std::size_t> find_unreachable(const StateMachine& sm) {
    std::vector<bool> visited(sm.num_states, false);
    std::queue<std::size_t> bfs;

    bfs.push(sm.initial_state);
    visited[sm.initial_state] = true;

    while (!bfs.empty()) {
        auto current = bfs.front(); bfs.pop();
        for (auto next : sm.transitions[current]) {
            if (!visited[next]) {
                visited[next] = true;
                bfs.push(next);
            }
        }
    }

    std::vector<std::size_t> unreachable;
    for (std::size_t i = 0; i < sm.num_states; ++i) {
        if (!visited[i]) unreachable.push_back(i);
    }
    return unreachable;
}
```

**Follow-up:** How do you detect dead-end states? (Reverse the graph, BFS from terminal states. States not reached in reverse are dead-ends.)

---

### 🏗️ Q8: Dependency Graph for Build System (Topological Sort)

**Scenario:** You're implementing a task scheduler (like BitBake). Tasks have dependencies. Determine execution order.

```cpp
enum class TaskState { Pending, InProgress, Complete, Failed };

struct Task {
    std::string name;
    std::vector<std::size_t> depends_on;  // Indices of prerequisite tasks
    TaskState state = TaskState::Pending;
};

// Kahn's algorithm  BFS-based topological sort
std::optional<std::vector<std::size_t>> topological_sort(
    const std::vector<Task>& tasks) {

    const std::size_t n = tasks.size();
    std::vector<std::size_t> in_degree(n, 0);
    std::vector<std::vector<std::size_t>> dependents(n);

    // Build reverse edges + compute in-degrees
    for (std::size_t i = 0; i < n; ++i) {
        for (auto dep : tasks[i].depends_on) {
            dependents[dep].push_back(i);
            ++in_degree[i];
        }
    }

    // Start with tasks that have no dependencies
    std::queue<std::size_t> ready;
    for (std::size_t i = 0; i < n; ++i) {
        if (in_degree[i] == 0) ready.push(i);
    }

    std::vector<std::size_t> order;
    order.reserve(n);

    while (!ready.empty()) {
        auto task_idx = ready.front(); ready.pop();
        order.push_back(task_idx);

        for (auto dependent : dependents[task_idx]) {
            if (--in_degree[dependent] == 0) {
                ready.push(dependent);
            }
        }
    }

    if (order.size() != n) return std::nullopt;  // Cycle detected!
    return order;
}
```

**Follow-up probes:**
- How does BitBake parallelize this? (Tasks at same topological level can run concurrently)
- How do you detect which specific tasks form a cycle? (DFS with coloring: white/gray/black)
- Time complexity? (O(V + E)  linear in tasks + dependencies)

---

## Bit Manipulation

### ⚡ Q9: Embedded Bit Operations

These come up constantly in register programming and protocol parsing.

| Task | Solution | Complexity |
|------|----------|-----------|
| Set bit N | `reg |= (1U << n)` | O(1) |
| Clear bit N | `reg &= ~(1U << n)` | O(1) |
| Toggle bit N | `reg ^= (1U << n)` | O(1) |
| Check bit N | `(reg >> n) & 1U` | O(1) |
| Count set bits | `__builtin_popcount(x)` or Kernighan's algorithm | O(k) where k = set bits |
| Find first set bit | `__builtin_ctz(x)` (count trailing zeros) | O(1) HW instruction |
| Power of 2 check | `x && !(x & (x-1))` | O(1) |
| Round up to power of 2 | `1U << (32 - __builtin_clz(n - 1))` | O(1) |
| Extract bit field | `(reg >> offset) & ((1U << width) - 1)` | O(1) |

**Question:** Extract a 5-bit field starting at bit 12 from a 32-bit register:

```cpp
constexpr uint32_t extract_field(uint32_t reg, uint8_t offset, uint8_t width) {
    return (reg >> offset) & ((1U << width) - 1);
}
// Usage: auto mode = extract_field(status_reg, 12, 5);  // bits [16:12]
```

---

### 🔬 Q10: CRC Calculation (Polynomial Arithmetic)

**Question:** Implement CRC-16-CCITT used in MAVLink/HDLC protocols:

```cpp
constexpr uint16_t crc16_ccitt_byte(uint16_t crc, uint8_t byte) {
    crc ^= static_cast<uint16_t>(byte) << 8;
    for (int i = 0; i < 8; ++i) {
        crc = (crc & 0x8000) ? (crc << 1) ^ 0x1021 : crc << 1;
    }
    return crc;
}

uint16_t crc16_ccitt(std::span<const uint8_t> data, uint16_t init = 0xFFFF) {
    uint16_t crc = init;
    for (auto byte : data) {
        crc = crc16_ccitt_byte(crc, byte);
    }
    return crc;
}
```

**Follow-up:** How do you make this faster? (Lookup table  trade 512 bytes of ROM for ~8x speed. Show `constexpr` table generation.)

---

## Strings & Parsing

### 🔬 Q11: Zero-Copy Token Parser for AT Commands

**Scenario:** Parse AT modem responses like `+CREG: 0,1,"1A2B","3C4D"` without any heap allocation.

```cpp
class TokenParser {
public:
    explicit TokenParser(std::string_view input) : input_(input), pos_(0) {}

    std::string_view next_token(char delimiter = ',') {
        skip_whitespace();
        std::size_t start = pos_;

        if (pos_ < input_.size() && input_[pos_] == '"') {
            // Quoted string
            ++start; ++pos_;
            while (pos_ < input_.size() && input_[pos_] != '"') ++pos_;
            auto token = input_.substr(start, pos_ - start);
            if (pos_ < input_.size()) ++pos_;  // Skip closing quote
            if (pos_ < input_.size() && input_[pos_] == delimiter) ++pos_;
            return token;
        }

        // Unquoted token
        while (pos_ < input_.size() && input_[pos_] != delimiter) ++pos_;
        auto token = input_.substr(start, pos_ - start);
        if (pos_ < input_.size()) ++pos_;  // Skip delimiter
        return token;
    }

    bool has_more() const { return pos_ < input_.size(); }

private:
    void skip_whitespace() {
        while (pos_ < input_.size() && input_[pos_] == ' ') ++pos_;
    }

    std::string_view input_;
    std::size_t pos_;
};

// Usage:
// TokenParser p("+CREG: 0,1,\"1A2B\",\"3C4D\"");
// auto prefix = p.next_token(':');  // "+CREG"
// auto stat = p.next_token();       // " 0"  
// auto reg = p.next_token();        // "1"
// auto lac = p.next_token();        // "1A2B" (quotes stripped)
```

**Key point:** `std::string_view`  zero allocation, points into original buffer.

---

## Memory-Efficient Data Structures

### 🏗️ Q12: Intrusive Linked List (No Heap)

**Scenario:** RTOS task lists, DMA descriptor chains, and timer queues use intrusive lists where the link node is embedded in the object itself.

```cpp
struct IntrusiveListNode {
    IntrusiveListNode* next = nullptr;
    IntrusiveListNode* prev = nullptr;
};

template <typename T, IntrusiveListNode T::*NodeMember>
class IntrusiveList {
public:
    void push_back(T& item) {
        auto* node = &(item.*NodeMember);
        node->prev = tail_;
        node->next = nullptr;
        if (tail_) tail_->next = node;
        else head_ = node;
        tail_ = node;
        ++size_;
    }

    void remove(T& item) {
        auto* node = &(item.*NodeMember);
        if (node->prev) node->prev->next = node->next;
        else head_ = node->next;
        if (node->next) node->next->prev = node->prev;
        else tail_ = node->prev;
        node->next = node->prev = nullptr;
        --size_;
    }

    T* front() {
        return head_ ? container_of(head_) : nullptr;
    }

    std::size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }

private:
    T* container_of(IntrusiveListNode* node) {
        // Calculate offset of NodeMember within T
        constexpr auto offset = reinterpret_cast<std::size_t>(
            &(static_cast<T*>(nullptr)->*NodeMember));
        return reinterpret_cast<T*>(
            reinterpret_cast<std::byte*>(node) - offset);
    }

    IntrusiveListNode* head_ = nullptr;
    IntrusiveListNode* tail_ = nullptr;
    std::size_t size_ = 0;
};

// Usage:
struct TimerEntry {
    uint32_t expiry_tick;
    void (*callback)(void*);
    void* context;
    IntrusiveListNode list_node;  // Embedded link
};

IntrusiveList<TimerEntry, &TimerEntry::list_node> active_timers;
```

**Why intrusive over `std::list`?**
- Zero heap allocation (object already exists)
- Cache-friendly (no separate node allocation scattered in heap)
- Constant-time remove with pointer to element (no search needed)
- Used in Linux kernel (`list_head`), FreeRTOS, DPDK

---

## Complexity Analysis  Quick Reference

### ⚡ Q13: Big-O Quiz

| Operation | Expected Answer |
|-----------|----------------|
| `std::vector::push_back` (amortized) | O(1) |
| `std::map::insert` | O(log n) |
| `std::unordered_map::insert` (average) | O(1) |
| `std::sort` | O(n log n) |
| `std::find` on unsorted vector | O(n) |
| `std::binary_search` | O(log n) |
| Hash table worst-case lookup | O(n)  all keys collide |
| BFS/DFS | O(V + E) |
| Heap insert/extract | O(log n) |
| Counting sort | O(n + k) where k = range |

**Follow-up:** What data structure gives O(1) insert, O(1) delete, O(1) random access? (Hash map + array  or just array with swap-to-back delete)

---

## Algorithm Selection Guide for Interviewers

| Candidate Level | Suitable Questions | Time |
|----------------|-------------------|------|
| Junior (0-3 yr) | Q1, Q3, Q5, Q9, Q13 | 15 min |
| Mid (4-8 yr) | Q2, Q4, Q6, Q10, Q11 | 20 min |
| Senior (9+ yr) | Q7, Q8, Q12, + discuss trade-offs | 15 min |

**Interviewer tip:** For senior candidates, focus less on implementation and more on:
- When to use which data structure
- Space/time trade-off decisions for embedded constraints
- Cache behavior and memory layout awareness
- Real-time determinism (O(1) requirement in ISR context)
