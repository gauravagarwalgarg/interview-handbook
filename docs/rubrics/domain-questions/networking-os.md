# Networking & Operating Systems Domain Questions

## TCP/UDP & Networking

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 1 | Explain the TCP 3-way handshake. What happens if the final ACK is lost? | SDE 2 | SYN → SYN-ACK → ACK. If final ACK lost: server stays in SYN_RECEIVED, retransmits SYN-ACK. Client is in ESTABLISHED (can send data which acts as ACK). Server eventually establishes on receiving data or times out. | What is SYN flood attack? How does SYN cookies mitigate it? What's TCP Fast Open? |
| 2 | When would you choose UDP over TCP? Design a protocol on top of UDP that guarantees ordering. | SDE 2-3 | UDP for: low latency (gaming, VoIP), broadcast/multicast, small messages (DNS). Ordering on UDP: sequence numbers per packet + reorder buffer + timeout for gaps. Basically reinventing parts of TCP selectively. | How does QUIC improve on TCP? What's head-of-line blocking in HTTP/2 over TCP? |
| 3 | Explain TCP congestion control (slow start, congestion avoidance, fast retransmit). | SDE 3 | Slow start: cwnd doubles each RTT until ssthresh. Congestion avoidance: cwnd += 1/cwnd per ACK (linear). Fast retransmit: 3 duplicate ACKs → retransmit without timeout; halve cwnd. BBR: model-based (bandwidth × RTT). | How does BBR differ from CUBIC? What's the impact of buffer bloat? |
| 4 | How does DNS resolution work end-to-end? What are the caching layers? | SDE 2 | App → OS resolver cache → /etc/hosts → recursive resolver (ISP/8.8.8.8) → root NS → TLD NS → authoritative NS. Caching at each layer with TTL. Response: A/AAAA/CNAME records. | What's DNS-over-HTTPS? How do CDNs use DNS for load balancing? What's the amplification attack vector? |

## Socket Programming

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 5 | Explain the difference between blocking, non-blocking, and async I/O. | SDE 2-3 | Blocking: thread waits until data ready. Non-blocking: returns EAGAIN immediately; app must poll. Async (io_uring): kernel completes I/O and notifies; no polling needed. Multiplexing (epoll): single thread monitors many FDs. | How does Node.js achieve concurrency with a single thread? What's the C10K problem? |
| 6 | Compare `select`, `poll`, `epoll`, and `io_uring`. When is each appropriate? | SDE 3 | select: O(n) scan, 1024 FD limit. poll: no FD limit but still O(n). epoll: O(1) for events, edge/level triggered, scales to millions. io_uring: zero-copy async I/O, batched syscalls, kernel-side polling. | What's the thundering herd problem with epoll? How does SO_REUSEPORT help? |
| 7 | Design a TCP server that handles 10K concurrent connections efficiently. | SDE 3 | Event loop + epoll (level-triggered for safety); connection pool; non-blocking sockets; worker thread pool for CPU-bound tasks. Alternatively: thread-per-connection with green threads (Go goroutines). | How do you handle slow clients (backpressure)? What about connection draining during shutdown? |

## Linux Process Model

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 8 | Explain fork(), exec(), and wait(). What is copy-on-write? | SDE 2 | fork(): duplicates process (new PID, same code). exec(): replaces process image with new program. wait(): parent blocks until child exits. COW: pages shared until written; avoids copying entire address space on fork. | What's vfork()? When does fork() fail? How does clone() relate to threads? |
| 9 | What is a zombie process vs an orphan process? How do you prevent zombies? | SDE 2 | Zombie: child exited but parent hasn't called wait(); keeps entry in process table. Orphan: parent exited; init/systemd adopts and reaps. Prevention: signal(SIGCHLD, SIG_IGN), double-fork, or async waitpid. | How do you find zombies in production? What's PID 1's special role in containers? |
| 10 | Explain Linux signals. What's the difference between SIGTERM, SIGKILL, SIGSTOP? | SDE 2 | SIGTERM: graceful shutdown request (catchable). SIGKILL: immediate termination (uncatchable). SIGSTOP: pause process (uncatchable). Signal handlers run asynchronously; only async-signal-safe functions allowed in handlers. | What functions are async-signal-safe? How do you handle signals in multi-threaded programs? |

## Virtual Memory & Memory Management

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 11 | Explain virtual memory, page tables, and TLB. What causes a page fault? | SDE 2-3 | Virtual address → page table → physical frame. TLB caches translations. Page fault: page not in RAM → OS loads from disk (swap) or allocates (demand paging). Major fault: disk I/O. Minor fault: no disk (e.g., COW, zero page). | What are huge pages? When would you use mlock()? How does ASLR work? |
| 12 | What is memory-mapped I/O (mmap)? When is it faster than read()/write()? | SDE 2-3 | mmap maps file directly into process address space; avoids kernel→user buffer copy. Faster for: random access patterns, shared memory between processes, large files. Worse for: sequential streaming (read-ahead better), small files (page granularity overhead). | How does mmap interact with the page cache? What are the dangers of MAP_SHARED? |

## Scheduling & File Systems

| # | Question | Level | Expected Answer | Follow-ups |
|---|----------|-------|-----------------|------------|
| 13 | Explain the CFS (Completely Fair Scheduler). How does it handle priority? | SDE 3 | Red-black tree of tasks keyed by virtual runtime (vruntime). Lower vruntime → picked next. Nice values adjust time weight (lower nice = more CPU). CFS doesn't use time slices directly; uses target latency divided by runnable tasks. | How does CFS handle CPU-bound vs I/O-bound? What are cgroups' role in scheduling? Real-time scheduling classes? |
| 14 | Compare ext4, XFS, and Btrfs. When would you choose each? | SDE 3 | ext4: general purpose, stable, good for small files, journaling. XFS: large files, high throughput, parallel allocation. Btrfs: COW, snapshots, checksums, compression, but less mature for production. | What is journaling and why does it matter? How does fsync guarantee durability? What's the write amplification in COW filesystems? |
| 15 | Explain the Linux I/O stack from application to disk. Where are the bottlenecks? | SDE 3 | App → syscall (write) → VFS → filesystem → page cache → block layer (I/O scheduler) → device driver → hardware. Bottlenecks: page cache thrashing, I/O scheduler queuing, disk IOPS limits. Direct I/O (O_DIRECT) bypasses page cache. | What's the difference between CFQ, deadline, and none I/O schedulers? When would you use O_DIRECT? How does NVMe change the I/O stack? |
