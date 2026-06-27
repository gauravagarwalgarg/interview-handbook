# Low-Level Design Problems

> 25 real LLD interview problems with class structures, design decisions, and level expectations.
> Focus: SOLID principles, design patterns, thread-safety, extensibility.

---

### P1: Design Parking Lot

**Problem:** Multi-floor parking lot with different vehicle sizes. Track spots, entry/exit, billing.

**Key Classes:**
```python
class VehicleSize(Enum):
    MOTORCYCLE, COMPACT, LARGE = 1, 2, 3

class ParkingSpot:
    def __init__(self, spot_id: str, size: VehicleSize, floor: int):
        self.spot_id = spot_id
        self.size = size
        self.floor = floor
        self.vehicle = None  # None if free
    def can_fit(self, vehicle) -> bool:
        return self.vehicle is None and vehicle.size <= self.size

class Vehicle:
    def __init__(self, license_plate: str, size: VehicleSize): ...

class ParkingLot:
    def __init__(self, floors: int, spots_per_floor: int): ...
    def park(self, vehicle: Vehicle) -> Optional[ParkingSpot]: ...
    def unpark(self, spot_id: str) -> float:  # returns fee
    def available_spots(self, size: VehicleSize) -> int: ...

class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, duration: timedelta, size: VehicleSize) -> float: ...
```

**Design Decisions:** Strategy pattern for pricing. Spot allocation via size-based free lists. Floor-level locking for concurrency.

**SDE 2 vs 3:** SDE 2 gets basic CRUD working. SDE 3 adds: multi-entry gates with load balancing, dynamic pricing, EV charging spots, capacity alerts.

---

### P2: Design LRU Cache (Thread-Safe)

**Problem:** Fixed-capacity cache with O(1) get/put, evicts least recently used on overflow.

**Key Classes:**
```python
class Node:
    def __init__(self, key, value):
        self.key, self.value = key, value
        self.prev = self.next = None

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache: Dict[str, Node] = {}
        self.head = Node(0, 0)  # dummy
        self.tail = Node(0, 0)  # dummy
        self.head.next = self.tail
        self.tail.prev = self.head
        self.lock = threading.RLock()

    def get(self, key: str) -> Optional[Any]:
        with self.lock:
            if key in self.cache:
                self._move_to_front(self.cache[key])
                return self.cache[key].value
            return None

    def put(self, key: str, value: Any) -> None:
        with self.lock:
            if key in self.cache:
                self.cache[key].value = value
                self._move_to_front(self.cache[key])
            else:
                if len(self.cache) >= self.capacity:
                    self._evict()
                node = Node(key, value)
                self.cache[key] = node
                self._add_to_front(node)
```

**Design Decisions:** DLL for O(1) move/remove. RLock for thread-safety. Could use RWLock for read-heavy workloads.

**SDE 2 vs 3:** SDE 3 adds: TTL per entry, eviction callbacks, size-based eviction (not just count), sharded locks for higher concurrency.

---

### P3: Design Rate Limiter (Token Bucket)

**Problem:** Limit API calls per user. Support multiple rate limit tiers.

**Key Classes:**
```python
class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()
        self.lock = threading.Lock()

    def allow(self) -> bool:
        with self.lock:
            self._refill()
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

class RateLimiter:
    def __init__(self, rules: List[RateLimitRule]):
        self.buckets: Dict[str, TokenBucket] = {}
        self.rules = rules

    def is_allowed(self, client_id: str, endpoint: str) -> bool:
        rule = self._match_rule(client_id, endpoint)
        bucket = self._get_or_create_bucket(client_id, rule)
        return bucket.allow()
```

**Design Decisions:** Token bucket vs sliding window vs fixed window. Per-client vs per-endpoint buckets. Distributed: Redis + Lua scripts for atomic operations.

**SDE 2 vs 3:** SDE 3 adds: sliding window log for burst handling, distributed rate limiting with Redis, graceful degradation, rate limit headers in response.

---

### P4: Design Snake Game

**Problem:** Snake game on grid. Track snake body, direction, food spawning.

**Key Classes:**
```python
class Direction(Enum):
    UP, DOWN, LEFT, RIGHT = (-1,0), (1,0), (0,-1), (0,1)

class SnakeGame:
    def __init__(self, width: int, height: int, food: List[Tuple[int,int]]):
        self.width, self.height = width, height
        self.snake = deque([(0, 0)])  # head at front
        self.snake_set = {(0, 0)}
        self.food = deque(food)
        self.score = 0

    def move(self, direction: str) -> int:
        head_r, head_c = self.snake[0]
        dr, dc = Direction[direction].value
        new_head = (head_r + dr, head_c + dc)

        # Check wall collision
        if not (0 <= new_head[0] < self.height and 0 <= new_head[1] < self.width):
            return -1  # game over

        # Check food
        if self.food and self.food[0] == new_head:
            self.food.popleft()
            self.score += 1
        else:
            tail = self.snake.pop()
            self.snake_set.remove(tail)

        # Check self collision
        if new_head in self.snake_set:
            return -1

        self.snake.appendleft(new_head)
        self.snake_set.add(new_head)
        return self.score
```

**Design Decisions:** Deque for O(1) head/tail operations. Set for O(1) collision check. Tail removed before self-collision check (can move into own tail's previous position).

---

### P5: Design Tic-Tac-Toe

**Problem:** N×N board. O(1) move validation and winner detection.

**Key Classes:**
```python
class TicTacToe:
    def __init__(self, n: int):
        self.n = n
        self.rows = [0] * n      # sum per row
        self.cols = [0] * n      # sum per col
        self.diag = 0            # main diagonal sum
        self.anti_diag = 0       # anti-diagonal sum

    def move(self, row: int, col: int, player: int) -> int:
        val = 1 if player == 1 else -1
        self.rows[row] += val
        self.cols[col] += val
        if row == col:
            self.diag += val
        if row + col == self.n - 1:
            self.anti_diag += val
        if abs(self.rows[row]) == self.n or abs(self.cols[col]) == self.n \
           or abs(self.diag) == self.n or abs(self.anti_diag) == self.n:
            return player
        return 0
```

**Design Decisions:** O(1) per move using running sums instead of scanning. Player 1 adds +1, player 2 adds -1. Win when absolute sum equals N.

---

### P6: Design In-Memory File System

**Problem:** Support mkdir, ls, addContent, readContent operations.

**Key Classes:**
```python
class FSNode:
    def __init__(self, name: str, is_file: bool = False):
        self.name = name
        self.is_file = is_file
        self.content = ""
        self.children: Dict[str, FSNode] = {}

class FileSystem:
    def __init__(self):
        self.root = FSNode("/")

    def mkdir(self, path: str) -> None:
        node = self._traverse(path, create=True)

    def ls(self, path: str) -> List[str]:
        node = self._traverse(path)
        if node.is_file:
            return [node.name]
        return sorted(node.children.keys())

    def add_content(self, path: str, content: str) -> None:
        node = self._traverse(path, create=True, is_file=True)
        node.content += content

    def read_content(self, path: str) -> str:
        return self._traverse(path).content

    def _traverse(self, path, create=False, is_file=False):
        parts = [p for p in path.split('/') if p]
        node = self.root
        for part in parts:
            if part not in node.children:
                if not create: raise FileNotFoundError
                node.children[part] = FSNode(part)
            node = node.children[part]
        if is_file: node.is_file = True
        return node
```

**Design Decisions:** Trie structure for path resolution. Separate file/directory types. Content appended (log-structured).

**SDE 2 vs 3:** SDE 3 adds: permissions, file locking, hard/soft links, inode-style metadata, journaling.

---

### P7: Design Task Scheduler

**Problem:** Schedule tasks with priorities, dependencies, and concurrency limits.

**Key Classes:**
```python
class Task:
    def __init__(self, task_id: str, priority: int, fn: Callable,
                 dependencies: List[str] = None):
        self.task_id = task_id
        self.priority = priority
        self.fn = fn
        self.dependencies = dependencies or []
        self.status = TaskStatus.PENDING

class TaskScheduler:
    def __init__(self, max_workers: int):
        self.tasks: Dict[str, Task] = {}
        self.ready_queue = PriorityQueue()
        self.executor = ThreadPoolExecutor(max_workers)
        self.lock = threading.Lock()

    def submit(self, task: Task) -> str: ...
    def _check_ready(self, task: Task) -> bool: ...
    def _on_complete(self, task_id: str): ...
    def cancel(self, task_id: str) -> bool: ...
    def get_status(self, task_id: str) -> TaskStatus: ...
```

**Design Decisions:** Priority queue for ready tasks. DAG for dependencies. Callback on completion to unblock dependents.

---

### P8: Design Pub/Sub System

**Problem:** In-process publish/subscribe with topic filtering.

**Key Classes:**
```python
class Message:
    def __init__(self, topic: str, payload: Any, timestamp: float = None):
        self.topic, self.payload = topic, payload
        self.timestamp = timestamp or time.time()

class Subscriber(ABC):
    @abstractmethod
    def on_message(self, message: Message) -> None: ...

class PubSub:
    def __init__(self):
        self.subscribers: Dict[str, List[Subscriber]] = defaultdict(list)
        self.lock = threading.RLock()

    def subscribe(self, topic: str, subscriber: Subscriber) -> None: ...
    def unsubscribe(self, topic: str, subscriber: Subscriber) -> None: ...
    def publish(self, message: Message) -> None:
        with self.lock:
            subs = list(self.subscribers.get(message.topic, []))
        for sub in subs:  # notify outside lock
            sub.on_message(message)
```

**Design Decisions:** Topic-based routing. Copy subscribers list before notification (prevent deadlock). Async delivery option with queue per subscriber.

---

### P9: Design Logger (Thread-Safe, Rotating)

**Problem:** Logging framework with levels, formatting, file rotation.

**Key Classes:**
```python
class LogLevel(Enum):
    DEBUG, INFO, WARN, ERROR, FATAL = 0, 1, 2, 3, 4

class LogSink(ABC):
    @abstractmethod
    def write(self, entry: LogEntry) -> None: ...

class RotatingFileSink(LogSink):
    def __init__(self, path: str, max_size_mb: int, max_files: int):
        self.path = path
        self.max_size = max_size_mb * 1024 * 1024
        self.max_files = max_files
        self.lock = threading.Lock()

class Logger:
    _instance = None

    def __init__(self, min_level: LogLevel, sinks: List[LogSink]):
        self.min_level = min_level
        self.sinks = sinks

    def log(self, level: LogLevel, msg: str, **context) -> None:
        if level.value < self.min_level.value:
            return
        entry = LogEntry(level, msg, context, time.time(), threading.current_thread().name)
        for sink in self.sinks:
            sink.write(entry)
```

**Design Decisions:** Sink abstraction (console, file, network). Rotation by size/time. Async buffer for high-throughput (ring buffer + flush thread).

---

### P10: Design Connection Pool

**Problem:** Reusable database connections with health checks and timeouts.

**Key Classes:**
```python
class Connection:
    def __init__(self, conn_id: str): ...
    def execute(self, query: str) -> Any: ...
    def is_alive(self) -> bool: ...
    def close(self) -> None: ...

class ConnectionPool:
    def __init__(self, factory: Callable, min_size: int, max_size: int, timeout: float):
        self.pool: Queue = Queue(maxsize=max_size)
        self.factory = factory
        self.size = 0
        self.max_size = max_size
        self.timeout = timeout
        self.lock = threading.Lock()

    def acquire(self) -> Connection:
        try:
            conn = self.pool.get(timeout=self.timeout)
            if not conn.is_alive():
                conn.close()
                return self._create()
            return conn
        except Empty:
            if self.size < self.max_size:
                return self._create()
            raise PoolExhaustedError()

    def release(self, conn: Connection) -> None:
        if conn.is_alive():
            self.pool.put(conn)
        else:
            conn.close()
            with self.lock:
                self.size -= 1
```

**Design Decisions:** Queue-based pool. Health check on acquire. Background evictor thread for idle connections. Max lifetime to prevent stale connections.

---

### P11: Design Elevator System

**Problem:** Multiple elevators serving N floors. Optimize wait time.

**Key Classes:**
```python
class ElevatorState(Enum):
    IDLE, MOVING_UP, MOVING_DOWN = 0, 1, 2

class Elevator:
    def __init__(self, elevator_id: int, capacity: int):
        self.id = elevator_id
        self.current_floor = 0
        self.state = ElevatorState.IDLE
        self.destinations: SortedSet = SortedSet()
        self.passengers = 0

class ElevatorController:
    def __init__(self, num_elevators: int, num_floors: int):
        self.elevators = [Elevator(i, 10) for i in range(num_elevators)]

    def request(self, floor: int, direction: Direction) -> Elevator:
        return self._find_best_elevator(floor, direction)

    def _find_best_elevator(self, floor, direction):
        # SCAN/LOOK algorithm: prefer elevator already moving in same direction
        ...
```

**Design Decisions:** SCAN algorithm (elevator algorithm). Separate hall requests from cab requests. Weight: distance + load + direction match.

---

### P12: Design Vending Machine (State Machine)

**Problem:** State transitions: Idle → HasMoney → Dispensing → Idle.

**Key Classes:**
```python
class State(ABC):
    @abstractmethod
    def insert_money(self, machine, amount): ...
    @abstractmethod
    def select_product(self, machine, product_id): ...
    @abstractmethod
    def dispense(self, machine): ...

class IdleState(State):
    def insert_money(self, machine, amount):
        machine.balance += amount
        machine.set_state(HasMoneyState())
    def select_product(self, machine, product_id):
        raise InvalidOperationError("Insert money first")

class VendingMachine:
    def __init__(self):
        self.state = IdleState()
        self.balance = 0
        self.inventory: Dict[str, Product] = {}

    def set_state(self, state: State):
        self.state = state
```

**Design Decisions:** State pattern for clean transitions. Each state class encapsulates valid operations and transition logic.

---

### P13: Design Chess Game

**Problem:** Board, pieces with movement rules, turn management, check/checkmate detection.

**Key Classes:**
```python
class Piece(ABC):
    def __init__(self, color: Color, position: Position):
        self.color, self.position = color, position
    @abstractmethod
    def valid_moves(self, board: 'Board') -> List[Position]: ...

class King(Piece): ...
class Queen(Piece): ...
class Rook(Piece): ...

class Board:
    def __init__(self):
        self.grid: List[List[Optional[Piece]]] = [[None]*8 for _ in range(8)]
    def move(self, from_pos, to_pos) -> MoveResult: ...
    def is_check(self, color: Color) -> bool: ...
    def is_checkmate(self, color: Color) -> bool: ...

class Game:
    def __init__(self):
        self.board = Board()
        self.current_turn = Color.WHITE
        self.move_history: List[Move] = []
    def make_move(self, from_pos, to_pos) -> MoveResult: ...
    def undo(self) -> None: ...
```

**Design Decisions:** Piece hierarchy with polymorphic `valid_moves`. Board validates moves (no self-check). Command pattern for undo.

---

### P14: Design Hotel Booking System

**Problem:** Room types, availability search, booking with concurrency control.

**Key Classes:**
```python
class Room:
    def __init__(self, room_id: str, room_type: RoomType, floor: int, price: float): ...

class Reservation:
    def __init__(self, guest: Guest, room: Room, check_in: date, check_out: date): ...

class HotelBookingSystem:
    def search_available(self, room_type: RoomType, check_in: date,
                        check_out: date) -> List[Room]: ...
    def book(self, guest: Guest, room_id: str, check_in: date,
            check_out: date) -> Reservation: ...
    def cancel(self, reservation_id: str) -> float:  # returns refund
```

**Design Decisions:** Optimistic locking on booking (check + reserve atomically). Date-range interval tree for fast availability queries. Overbooking strategy for revenue optimization.

---

### P15: Design ATM Machine

**Problem:** Card validation, PIN check, balance inquiry, withdrawal, receipt.

**Key Classes:**
```python
class ATM:
    def __init__(self, atm_id: str, cash_dispenser: CashDispenser, bank_service: BankService):
        self.state = IdleState()
    def insert_card(self, card: Card) -> None: ...
    def enter_pin(self, pin: str) -> bool: ...
    def withdraw(self, amount: float) -> WithdrawalResult: ...

class CashDispenser:
    def __init__(self, denominations: Dict[int, int]):  # denomination -> count
        ...
    def dispense(self, amount: int) -> Dict[int, int]:
        # Greedy: largest denominations first
        ...
    def can_dispense(self, amount: int) -> bool: ...
```

**Design Decisions:** State pattern (Idle → CardInserted → Authenticated → Transaction). Cash dispenser uses greedy algorithm. Bank service is external dependency (interface).

---

### P16: Design Thread Pool

**Problem:** Fixed pool of worker threads executing submitted tasks.

**Key Classes:**
```python
class ThreadPool:
    def __init__(self, num_threads: int):
        self.task_queue: Queue = Queue()
        self.workers: List[Thread] = []
        self.shutdown_flag = threading.Event()
        for _ in range(num_threads):
            t = Thread(target=self._worker_loop, daemon=True)
            t.start()
            self.workers.append(t)

    def submit(self, fn: Callable, *args) -> Future:
        future = Future()
        self.task_queue.put((fn, args, future))
        return future

    def _worker_loop(self):
        while not self.shutdown_flag.is_set():
            try:
                fn, args, future = self.task_queue.get(timeout=1)
                try:
                    result = fn(*args)
                    future.set_result(result)
                except Exception as e:
                    future.set_exception(e)
            except Empty:
                continue

    def shutdown(self, wait: bool = True):
        self.shutdown_flag.set()
        if wait:
            for w in self.workers:
                w.join()
```

**Design Decisions:** Blocking queue for task distribution. Future for result retrieval. Graceful shutdown with drain option.

---

### P17: Design Object Pool / Memory Pool

**Problem:** Pre-allocate objects to avoid repeated allocation/deallocation overhead.

**Key Classes:**
```cpp
// C++ typical for HFT/game engines
template<typename T, size_t PoolSize>
class ObjectPool {
    union Slot {
        T object;
        Slot* next;
    };
    Slot pool_[PoolSize];
    Slot* free_list_;

public:
    ObjectPool() {
        free_list_ = &pool_[0];
        for (size_t i = 0; i < PoolSize - 1; i++)
            pool_[i].next = &pool_[i + 1];
        pool_[PoolSize - 1].next = nullptr;
    }

    T* allocate() {
        if (!free_list_) return nullptr;
        Slot* slot = free_list_;
        free_list_ = slot->next;
        return new (&slot->object) T();  // placement new
    }

    void deallocate(T* obj) {
        obj->~T();
        Slot* slot = reinterpret_cast<Slot*>(obj);
        slot->next = free_list_;
        free_list_ = slot;
    }
};
```

**Design Decisions:** Free list for O(1) alloc/dealloc. Union trick reuses memory. No heap allocation after construction.

---

### P18: Design Circuit Breaker

**Problem:** Prevent cascading failures by short-circuiting calls to failing services.

**Key Classes:**
```python
class CircuitState(Enum):
    CLOSED, OPEN, HALF_OPEN = 0, 1, 2

class CircuitBreaker:
    def __init__(self, failure_threshold: int, recovery_timeout: float,
                 half_open_max: int = 1):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = 0

    def call(self, fn: Callable, *args):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenError()
        try:
            result = fn(*args)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
```

**Design Decisions:** Three states: Closed (normal), Open (fail-fast), Half-Open (probe). Sliding window for failure counting. Per-service breaker instances.

---

### P19: Design Command Pattern (Undo/Redo)

**Problem:** Text editor with undo/redo support.

**Key Classes:**
```python
class Command(ABC):
    @abstractmethod
    def execute(self) -> None: ...
    @abstractmethod
    def undo(self) -> None: ...

class InsertCommand(Command):
    def __init__(self, document, position, text):
        self.doc, self.pos, self.text = document, position, text
    def execute(self): self.doc.insert(self.pos, self.text)
    def undo(self): self.doc.delete(self.pos, len(self.text))

class CommandHistory:
    def __init__(self):
        self.undo_stack: List[Command] = []
        self.redo_stack: List[Command] = []

    def execute(self, cmd: Command):
        cmd.execute()
        self.undo_stack.append(cmd)
        self.redo_stack.clear()

    def undo(self):
        if self.undo_stack:
            cmd = self.undo_stack.pop()
            cmd.undo()
            self.redo_stack.append(cmd)

    def redo(self):
        if self.redo_stack:
            cmd = self.redo_stack.pop()
            cmd.execute()
            self.undo_stack.append(cmd)
```

---

### P20: Design Plugin System

**Problem:** Extensible application where plugins register at runtime.

**Key Classes:**
```python
class Plugin(ABC):
    @abstractmethod
    def name(self) -> str: ...
    @abstractmethod
    def initialize(self, context: 'AppContext') -> None: ...
    @abstractmethod
    def execute(self, data: Any) -> Any: ...

class PluginManager:
    def __init__(self):
        self.plugins: Dict[str, Plugin] = {}
        self.hooks: Dict[str, List[Callable]] = defaultdict(list)

    def register(self, plugin: Plugin) -> None:
        plugin.initialize(self.context)
        self.plugins[plugin.name()] = plugin

    def load_from_directory(self, path: str) -> None:
        # Dynamic loading via importlib
        ...

    def emit(self, hook_name: str, *args) -> List[Any]:
        return [fn(*args) for fn in self.hooks[hook_name]]
```

**Design Decisions:** Abstract base class defines contract. Dynamic loading via importlib/reflection. Hook system for extension points.

---

### P21: Design Observable / Event Bus

**Problem:** Decoupled event-driven communication between components.

**Key Classes:**
```python
class EventBus:
    def __init__(self):
        self.listeners: Dict[Type, List[Callable]] = defaultdict(list)
        self.async_executor = ThreadPoolExecutor()

    def subscribe(self, event_type: Type, handler: Callable) -> Subscription:
        self.listeners[event_type].append(handler)
        return Subscription(self, event_type, handler)

    def publish(self, event: Any) -> None:
        for handler in self.listeners.get(type(event), []):
            handler(event)

    def publish_async(self, event: Any) -> None:
        for handler in self.listeners.get(type(event), []):
            self.async_executor.submit(handler, event)
```

---

### P22: Design Producer-Consumer Queue

**Problem:** Thread-safe bounded queue with blocking semantics.

**Key Classes:**
```python
class BoundedQueue:
    def __init__(self, capacity: int):
        self.buffer = [None] * capacity
        self.capacity = capacity
        self.head = self.tail = self.count = 0
        self.lock = threading.Lock()
        self.not_full = threading.Condition(self.lock)
        self.not_empty = threading.Condition(self.lock)

    def put(self, item, timeout=None) -> bool:
        with self.not_full:
            while self.count == self.capacity:
                if not self.not_full.wait(timeout):
                    return False
            self.buffer[self.tail] = item
            self.tail = (self.tail + 1) % self.capacity
            self.count += 1
            self.not_empty.notify()
            return True

    def take(self, timeout=None) -> Optional[Any]:
        with self.not_empty:
            while self.count == 0:
                if not self.not_empty.wait(timeout):
                    return None
            item = self.buffer[self.head]
            self.head = (self.head + 1) % self.capacity
            self.count -= 1
            self.not_full.notify()
            return item
```

---

### P23: Design Leader Election

**Problem:** Select one node as leader in a distributed cluster.

**Key Classes:**
```python
class LeaderElection:
    def __init__(self, node_id: str, peers: List[str], heartbeat_interval: float):
        self.node_id = node_id
        self.leader = None
        self.term = 0
        self.state = NodeState.FOLLOWER
        self.election_timeout = random.uniform(150, 300) / 1000

    def start_election(self):
        self.term += 1
        self.state = NodeState.CANDIDATE
        votes = 1  # vote for self
        for peer in self.peers:
            if self.request_vote(peer, self.term):
                votes += 1
        if votes > len(self.peers) // 2:
            self.become_leader()

    def become_leader(self):
        self.state = NodeState.LEADER
        self.leader = self.node_id
        self._start_heartbeat()
```

**Design Decisions:** Raft-style: random election timeout prevents split votes. Majority quorum. Leader sends heartbeats to maintain authority.

---

### P24: Design Retry with Exponential Backoff

**Problem:** Resilient RPC calls with configurable retry policy.

**Key Classes:**
```python
class RetryPolicy:
    def __init__(self, max_retries: int = 3, base_delay: float = 1.0,
                 max_delay: float = 60.0, jitter: bool = True,
                 retryable_exceptions: Tuple = (Exception,)):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.jitter = jitter
        self.retryable = retryable_exceptions

    def execute(self, fn: Callable, *args) -> Any:
        for attempt in range(self.max_retries + 1):
            try:
                return fn(*args)
            except self.retryable as e:
                if attempt == self.max_retries:
                    raise
                delay = min(self.base_delay * (2 ** attempt), self.max_delay)
                if self.jitter:
                    delay *= random.uniform(0.5, 1.5)
                time.sleep(delay)
```

**Design Decisions:** Exponential backoff prevents thundering herd. Jitter decorrelates retries across clients. Circuit breaker wraps retry for total failure cutoff.

---

### P25: Design Config Manager (Hot Reload)

**Problem:** Centralized config that supports live updates without restart.

**Key Classes:**
```python
class ConfigManager:
    def __init__(self, source: ConfigSource, poll_interval: float = 5.0):
        self.config: Dict[str, Any] = {}
        self.source = source
        self.watchers: Dict[str, List[Callable]] = defaultdict(list)
        self.lock = threading.RLock()
        self._start_polling(poll_interval)

    def get(self, key: str, default=None) -> Any:
        with self.lock:
            return self.config.get(key, default)

    def watch(self, key: str, callback: Callable) -> None:
        self.watchers[key].append(callback)

    def _reload(self):
        new_config = self.source.load()
        with self.lock:
            for key, value in new_config.items():
                if self.config.get(key) != value:
                    self.config[key] = value
                    for cb in self.watchers.get(key, []):
                        cb(key, value)

class ConfigSource(ABC):
    @abstractmethod
    def load(self) -> Dict[str, Any]: ...

class FileConfigSource(ConfigSource): ...
class RemoteConfigSource(ConfigSource): ...  # Consul, etcd, etc.
```

**Design Decisions:** Polling vs push (webhook/watch). Watcher pattern for reactive updates. Layered sources (file < env < remote). Atomic swap for consistency.
