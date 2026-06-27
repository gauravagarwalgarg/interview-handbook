# DSA Coding Problems with Solutions

> 25 must-know problems with complete solutions, complexities, and follow-ups.
> Language: Python (for clarity). Solutions are interview-ready.

---

### P1: Two Sum Hash Map O(n)

**Problem:** Given array of integers and target, return indices of two numbers that add to target.

**Approach:** Single-pass hash map. For each number, check if complement exists in map.

```python
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

**Complexity:** O(n) time, O(n) space.
**Follow-up:** What if the array is sorted? → Two pointers O(n) time O(1) space.

---

### P2: Merge Intervals Sort + Merge

**Problem:** Given array of intervals, merge all overlapping intervals.

**Approach:** Sort by start. Iterate and merge if current overlaps with last merged.

```python
def merge(intervals):
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    return merged
```

**Complexity:** O(n log n) time, O(n) space.
**Follow-up:** Insert a new interval into non-overlapping sorted intervals without re-sorting.

---

### P3: LRU Cache Doubly Linked List + Hash Map

**Problem:** Implement LRU cache with O(1) get and put.

**Approach:** OrderedDict or manual DLL + hashmap. Move accessed items to front.

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.cap = capacity

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)
```

**Complexity:** O(1) per operation.
**Follow-up:** Make it thread-safe. → Add read-write lock; or use ConcurrentLinkedHashMap.

---

### P4: Binary Tree Level Order Traversal BFS

**Problem:** Return node values level by level (list of lists).

**Approach:** BFS with queue. Process all nodes at current level before next.

```python
from collections import deque

def level_order(root):
    if not root:
        return []
    result, queue = [], deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(level)
    return result
```

**Complexity:** O(n) time, O(n) space.
**Follow-up:** Zigzag level order → alternate direction each level.

---

### P5: Course Schedule Topological Sort (Cycle Detection)

**Problem:** Given prerequisites, determine if all courses can be finished.

**Approach:** Build directed graph. Detect cycle with DFS coloring or Kahn's algorithm.

```python
from collections import defaultdict, deque

def can_finish(num_courses, prerequisites):
    graph = defaultdict(list)
    in_degree = [0] * num_courses
    for course, prereq in prerequisites:
        graph[prereq].append(course)
        in_degree[course] += 1
    queue = deque(i for i in range(num_courses) if in_degree[i] == 0)
    count = 0
    while queue:
        node = queue.popleft()
        count += 1
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    return count == num_courses
```

**Complexity:** O(V + E) time, O(V + E) space.
**Follow-up:** Return one valid course ordering (topological sort).

---

### P6: Sliding Window Maximum Monotonic Deque

**Problem:** Given array and window size k, return max of each window.

**Approach:** Maintain decreasing deque of indices. Remove smaller elements from back.

```python
from collections import deque

def max_sliding_window(nums, k):
    dq = deque()  # stores indices
    result = []
    for i, num in enumerate(nums):
        while dq and dq[0] <= i - k:
            dq.popleft()
        while dq and nums[dq[-1]] <= num:
            dq.pop()
        dq.append(i)
        if i >= k - 1:
            result.append(nums[dq[0]])
    return result
```

**Complexity:** O(n) time, O(k) space.
**Follow-up:** Sliding window minimum → change comparison direction.

---

### P7: Serialize/Deserialize Binary Tree BFS

**Problem:** Encode a binary tree to string and decode back.

**Approach:** BFS with "null" markers for missing children.

```python
from collections import deque

class Codec:
    def serialize(self, root):
        if not root:
            return ""
        result, queue = [], deque([root])
        while queue:
            node = queue.popleft()
            if node:
                result.append(str(node.val))
                queue.append(node.left)
                queue.append(node.right)
            else:
                result.append("null")
        return ",".join(result)

    def deserialize(self, data):
        if not data:
            return None
        vals = data.split(",")
        root = TreeNode(int(vals[0]))
        queue = deque([root])
        i = 1
        while queue:
            node = queue.popleft()
            if vals[i] != "null":
                node.left = TreeNode(int(vals[i]))
                queue.append(node.left)
            i += 1
            if vals[i] != "null":
                node.right = TreeNode(int(vals[i]))
                queue.append(node.right)
            i += 1
        return root
```

**Complexity:** O(n) time and space.
**Follow-up:** Serialize N-ary tree. → Store child count per node.

---

### P8: Word Ladder BFS Shortest Path

**Problem:** Transform beginWord to endWord changing one letter at a time, using dictionary.

**Approach:** BFS where each state is a word, edges connect words differing by one letter.

```python
from collections import deque

def ladder_length(begin_word, end_word, word_list):
    word_set = set(word_list)
    if end_word not in word_set:
        return 0
    queue = deque([(begin_word, 1)])
    visited = {begin_word}
    while queue:
        word, steps = queue.popleft()
        for i in range(len(word)):
            for c in 'abcdefghijklmnopqrstuvwxyz':
                next_word = word[:i] + c + word[i+1:]
                if next_word == end_word:
                    return steps + 1
                if next_word in word_set and next_word not in visited:
                    visited.add(next_word)
                    queue.append((next_word, steps + 1))
    return 0
```

**Complexity:** O(M² × N) where M=word length, N=dict size.
**Follow-up:** Bidirectional BFS for faster search.

---

### P9: Median of Two Sorted Arrays Binary Search

**Problem:** Find median of two sorted arrays in O(log(m+n)).

**Approach:** Binary search on the shorter array to find correct partition.

```python
def find_median(nums1, nums2):
    if len(nums1) > len(nums2):
        nums1, nums2 = nums2, nums1
    m, n = len(nums1), len(nums2)
    lo, hi = 0, m
    while lo <= hi:
        i = (lo + hi) // 2
        j = (m + n + 1) // 2 - i
        left1 = nums1[i-1] if i > 0 else float('-inf')
        right1 = nums1[i] if i < m else float('inf')
        left2 = nums2[j-1] if j > 0 else float('-inf')
        right2 = nums2[j] if j < n else float('inf')
        if left1 <= right2 and left2 <= right1:
            if (m + n) % 2:
                return max(left1, left2)
            return (max(left1, left2) + min(right1, right2)) / 2
        elif left1 > right2:
            hi = i - 1
        else:
            lo = i + 1
```

**Complexity:** O(log(min(m,n))).
**Follow-up:** Kth element of two sorted arrays → same technique, stop at k.

---

### P10: Trapping Rain Water Two Pointers

**Problem:** Given elevation map, compute trapped water.

**Approach:** Two pointers from both ends. Track left_max and right_max.

```python
def trap(height):
    left, right = 0, len(height) - 1
    left_max = right_max = water = 0
    while left < right:
        if height[left] < height[right]:
            left_max = max(left_max, height[left])
            water += left_max - height[left]
            left += 1
        else:
            right_max = max(right_max, height[right])
            water += right_max - height[right]
            right -= 1
    return water
```

**Complexity:** O(n) time, O(1) space.
**Follow-up:** 3D version (trapping rain water II) → BFS with min-heap on boundaries.

---

### P11: Longest Palindromic Substring Expand Around Center

**Problem:** Find the longest palindromic substring.

**Approach:** For each center (and between-center), expand outward while palindrome.

```python
def longest_palindrome(s):
    def expand(l, r):
        while l >= 0 and r < len(s) and s[l] == s[r]:
            l -= 1
            r += 1
        return s[l+1:r]

    result = ""
    for i in range(len(s)):
        odd = expand(i, i)
        even = expand(i, i+1)
        result = max(result, odd, even, key=len)
    return result
```

**Complexity:** O(n²) time, O(1) space (excluding output).
**Follow-up:** O(n) solution → Manacher's algorithm.

---

### P12: Valid Parentheses Stack

**Problem:** Check if string of brackets is valid.

**Approach:** Push opening brackets. Pop and match on closing brackets.

```python
def is_valid(s):
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}
    for c in s:
        if c in pairs:
            if not stack or stack[-1] != pairs[c]:
                return False
            stack.pop()
        else:
            stack.append(c)
    return len(stack) == 0
```

**Complexity:** O(n) time, O(n) space.
**Follow-up:** Minimum insertions to make valid. → Count unmatched opens and closes.

---

### P13: Kth Largest Element Quickselect

**Problem:** Find kth largest element (not sorted).

**Approach:** Quickselect with random pivot. Average O(n).

```python
import random

def find_kth_largest(nums, k):
    k = len(nums) - k  # convert to kth smallest
    def quickselect(lo, hi):
        pivot_idx = random.randint(lo, hi)
        nums[pivot_idx], nums[hi] = nums[hi], nums[pivot_idx]
        pivot = nums[hi]
        store = lo
        for i in range(lo, hi):
            if nums[i] < pivot:
                nums[i], nums[store] = nums[store], nums[i]
                store += 1
        nums[store], nums[hi] = nums[hi], nums[store]
        if store == k:
            return nums[store]
        elif store < k:
            return quickselect(store + 1, hi)
        else:
            return quickselect(lo, store - 1)
    return quickselect(0, len(nums) - 1)
```

**Complexity:** O(n) average, O(n²) worst.
**Follow-up:** Guaranteed O(n) → median of medians pivot selection.

---

### P14: Clone Graph DFS + HashMap

**Problem:** Deep copy a connected undirected graph.

**Approach:** DFS/BFS with a map from original node → clone.

```python
def clone_graph(node):
    if not node:
        return None
    clones = {}
    def dfs(n):
        if n in clones:
            return clones[n]
        copy = Node(n.val)
        clones[n] = copy
        for neighbor in n.neighbors:
            copy.neighbors.append(dfs(neighbor))
        return copy
    return dfs(node)
```

**Complexity:** O(V + E) time and space.
**Follow-up:** Clone a graph with random pointers (like linked list with random).

---

### P15: Coin Change Bottom-Up DP

**Problem:** Minimum coins to make amount. Return -1 if impossible.

**Approach:** DP where dp[i] = minimum coins for amount i.

```python
def coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

**Complexity:** O(amount × len(coins)) time, O(amount) space.
**Follow-up:** Return the actual coins used → track choices with parent array.

---

### P16: Number of Islands DFS Flood Fill

**Problem:** Count islands in a 2D grid of '1's (land) and '0's (water).

**Approach:** DFS from each unvisited '1', marking connected land as visited.

```python
def num_islands(grid):
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0

    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'  # mark visited
        dfs(r+1, c); dfs(r-1, c); dfs(r, c+1); dfs(r, c-1)

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                dfs(r, c)
                count += 1
    return count
```

**Complexity:** O(M×N) time and space.
**Follow-up:** Count distinct island shapes → normalize DFS paths and use a set.

---

### P17: Rotate Image Transpose + Reverse

**Problem:** Rotate N×N matrix 90° clockwise in-place.

**Approach:** Transpose (swap rows/cols) then reverse each row.

```python
def rotate(matrix):
    n = len(matrix)
    # Transpose
    for i in range(n):
        for j in range(i+1, n):
            matrix[i][j], matrix[j][i] = matrix[j][i], matrix[i][j]
    # Reverse each row
    for row in matrix:
        row.reverse()
```

**Complexity:** O(n²) time, O(1) space.
**Follow-up:** Rotate 90° counter-clockwise → transpose then reverse each column (or reverse rows then transpose).

---

### P18: Design Twitter Feed Heap + Hash

**Problem:** Design simplified Twitter: post, follow, getNewsFeed (10 most recent from followed users).

**Approach:** Each user stores tweets with timestamps. Merge-k-sorted-lists using min-heap.

```python
import heapq
from collections import defaultdict

class Twitter:
    def __init__(self):
        self.time = 0
        self.tweets = defaultdict(list)   # userId -> [(time, tweetId)]
        self.follows = defaultdict(set)   # userId -> set of followees

    def postTweet(self, userId, tweetId):
        self.tweets[userId].append((self.time, tweetId))
        self.time += 1

    def getNewsFeed(self, userId):
        self.follows[userId].add(userId)
        heap = []
        for uid in self.follows[userId]:
            for tweet in self.tweets[uid][-10:]:
                heapq.heappush(heap, tweet)
                if len(heap) > 10:
                    heapq.heappop(heap)
        return [t[1] for t in sorted(heap, reverse=True)]

    def follow(self, followerId, followeeId):
        self.follows[followerId].add(followeeId)

    def unfollow(self, followerId, followeeId):
        self.follows[followerId].discard(followeeId)
```

**Complexity:** getNewsFeed O(N × 10 × log 10) where N = followees.
**Follow-up:** Scale to millions of users → fan-out on write for celebrities.

---

### P19: Find Median from Data Stream Two Heaps

**Problem:** Continuously add numbers and find median.

**Approach:** Max-heap for lower half, min-heap for upper half. Balance sizes.

```python
import heapq

class MedianFinder:
    def __init__(self):
        self.lo = []  # max-heap (negate values)
        self.hi = []  # min-heap

    def addNum(self, num):
        heapq.heappush(self.lo, -num)
        heapq.heappush(self.hi, -heapq.heappop(self.lo))
        if len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))

    def findMedian(self):
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2
```

**Complexity:** O(log n) add, O(1) find.
**Follow-up:** What if 99% of numbers are between 0-100? → Bucket counting for that range + heaps for outliers.

---

### P20: Alien Dictionary Topological Sort

**Problem:** Given sorted alien dictionary, find character order.

**Approach:** Build graph from adjacent word comparisons. Topological sort.

```python
from collections import defaultdict, deque

def alien_order(words):
    graph = defaultdict(set)
    in_degree = {c: 0 for word in words for c in word}
    for i in range(len(words) - 1):
        w1, w2 = words[i], words[i+1]
        if len(w1) > len(w2) and w1[:len(w2)] == w2:
            return ""  # invalid: prefix comes after
        for c1, c2 in zip(w1, w2):
            if c1 != c2:
                if c2 not in graph[c1]:
                    graph[c1].add(c2)
                    in_degree[c2] += 1
                break
    queue = deque(c for c in in_degree if in_degree[c] == 0)
    result = []
    while queue:
        c = queue.popleft()
        result.append(c)
        for neighbor in graph[c]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    return "".join(result) if len(result) == len(in_degree) else ""
```

**Complexity:** O(C) where C = total characters in all words.
**Follow-up:** Multiple valid orderings → return any one (Kahn's gives one valid topo order).

---

### P21: Meeting Rooms II Sweep Line / Min Heap

**Problem:** Minimum conference rooms required for all meetings.

**Approach:** Sort by start time. Use min-heap to track earliest ending meeting.

```python
import heapq

def min_meeting_rooms(intervals):
    if not intervals:
        return 0
    intervals.sort(key=lambda x: x[0])
    heap = [intervals[0][1]]  # end times
    for start, end in intervals[1:]:
        if start >= heap[0]:
            heapq.heappop(heap)  # reuse room
        heapq.heappush(heap, end)
    return len(heap)
```

**Complexity:** O(n log n) time, O(n) space.
**Follow-up:** Return which meetings share which rooms → track room assignments.

---

### P22: Longest Increasing Subsequence DP + Binary Search

**Problem:** Find length of longest strictly increasing subsequence.

**Approach:** Maintain tails array. Binary search for insertion point.

```python
import bisect

def length_of_lis(nums):
    tails = []
    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)
```

**Complexity:** O(n log n) time, O(n) space.
**Follow-up:** Return the actual subsequence → maintain parent pointers.

---

### P23: Word Search II Trie + DFS

**Problem:** Find all words from dictionary that exist in the board.

**Approach:** Build Trie from dictionary. DFS from each cell, walking Trie simultaneously.

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.word = None

def find_words(board, words):
    root = TrieNode()
    for word in words:
        node = root
        for c in word:
            node = node.children.setdefault(c, TrieNode())
        node.word = word

    result = []
    rows, cols = len(board), len(board[0])
    def dfs(r, c, node):
        if node.word:
            result.append(node.word)
            node.word = None  # avoid duplicates
        if r < 0 or r >= rows or c < 0 or c >= cols:
            return
        char = board[r][c]
        if char not in node.children:
            return
        board[r][c] = '#'
        for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
            dfs(r+dr, c+dc, node.children[char])
        board[r][c] = char

    for r in range(rows):
        for c in range(cols):
            dfs(r, c, root)
    return result
```

**Complexity:** O(M×N×4^L) worst case, but Trie prunes heavily.
**Follow-up:** Optimize by removing Trie branches when word is found (pruning).

---

### P24: Implement Trie Node Array

**Problem:** Implement insert, search, startsWith.

**Approach:** Array of 26 children per node + is_end flag.

```python
class Trie:
    def __init__(self):
        self.children = {}
        self.is_end = False

    def insert(self, word):
        node = self
        for c in word:
            if c not in node.children:
                node.children[c] = Trie()
            node = node.children[c]
        node.is_end = True

    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end

    def startsWith(self, prefix):
        return self._find(prefix) is not None

    def _find(self, word):
        node = self
        for c in word:
            if c not in node.children:
                return None
            node = node.children[c]
        return node
```

**Complexity:** O(L) per operation where L = word length.
**Follow-up:** Add wildcard search (`.` matches any char) → DFS branching at wildcards.

---

### P25: Design Hit Counter Circular Buffer

**Problem:** Count hits in last 300 seconds. Record hit(timestamp), getHits(timestamp).

**Approach:** Circular buffer of size 300. Each slot stores timestamp and count.

```python
class HitCounter:
    def __init__(self):
        self.times = [0] * 300
        self.hits = [0] * 300

    def hit(self, timestamp):
        idx = timestamp % 300
        if self.times[idx] != timestamp:
            self.times[idx] = timestamp
            self.hits[idx] = 1
        else:
            self.hits[idx] += 1

    def getHits(self, timestamp):
        total = 0
        for i in range(300):
            if timestamp - self.times[i] < 300:
                total += self.hits[i]
        return total
```

**Complexity:** O(1) hit, O(300)=O(1) getHits.
**Follow-up:** Thread-safe version → atomic operations per bucket or lock-free ring buffer.
