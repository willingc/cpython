.. _thread-safety-howto:

***********************
Thread safety in Python
***********************

Why is thread safety important?
===============================

Data integrity

Predictability of actions and results

What is a thread?
=================

We probably shouldn't assume that people know this.

 A thread is the smallest unit of execution that can be scheduled by an operating system. It's a lightweight subprocess that runs within a process and shares
 the process's memory space and resources.

What is thread safety?
======================

Thread safety refers to the property of code that ensures correct execution when accessed by multiple threads
simultaneously. A piece of code is considered thread-safe if it can be called concurrently by multiple threads without
causing data races, corruption, or inconsistent state.

data race:

data corruption:

inconsistent state:
In programming, an inconsistent state occurs when data or system components violate expected invariants or constraints, leading to contradictory or invalid
  conditions.

  Common examples:
  - Race conditions: Multiple threads modifying shared data without proper synchronization, causing partial updates
  - Partial transactions: Database operations that complete some but not all steps, violating ACID properties
  - Broken invariants: Object fields that should maintain relationships (e.g., a list's size field not matching actual element count)
  - Stale cache: Cached data that doesn't reflect the current state of the source data

  This typically happens during concurrent operations, system failures, or bugs in state management logic.



In Python, thread safety is particularly important in
multi-threaded applications where shared resources (such as data structures, files, or network connections) may be
accessed by multiple threads at the same time. Thread-safe code typically uses synchronization mechanisms like locks,
mutexes, or atomic operations to coordinate access to shared resources, preventing race conditions and ensuring that
operations complete atomically without interference from other threads.


When to consider thread safety?
================================

Shared mutable data

Non-atomic operations (context changes)

External library code might not be designed for thread safety

In Python, thread safety is guaranteed in specific situations due to the Global Interpreter Lock (GIL) in the standard
CPython implementation. The GIL ensures that only one thread executes Python bytecode at a time, which makes many basic
operations thread-safe by default. Simple operations like reading or replacing a single variable, appending to a list,
or accessing dictionary items are atomic and therefore thread-safe.

However, thread safety is not guaranteed for
compound operations (like checking if a key exists then adding it), operations that release the GIL (such as I/O
operations or calls to C extensions), or when using Python implementations without a GIL (like Jython, IronPython, or
the experimental free-threaded builds of CPython 3.13+). Additionally, even with the GIL, you must still use proper
synchronization when performing non-atomic operations, modifying shared mutable objects across threads, or when the
logical correctness of your program depends on the ordering of operations between threads.

Is there ever a possibility that a read could be unsafe with multiple threads reading the same data?
===================================================================================================

In most cases, multiple threads reading the same immutable data simultaneously is safe in Python. Pure read operations
on immutable objects (like integers, strings, and tuples) are inherently thread-safe because the data cannot change.
For built-in mutable objects in CPython, simple read operations are generally safe due to the GIL ensuring atomicity.
However, reads can become unsafe in several scenarios::

* (1) when reading from objects with custom ``__getitem__`` or ``__getattr__`` methods that modify state or have side effects,
* (2) when iterating over a collection that another thread is modifying (leading to RuntimeError: "dictionary changed size during iteration"),
* (3) when the read operation itself is non-atomic and involves multiple steps (like reading multiple related values that must be consistent),
* (4) in Python implementations without a GIL or in free-threaded builds where even basic operations may not be atomic, and
* (5) when reading from I/O streams or file objects where the read operation changes the file position, making concurrent reads produce unpredictable results.

Therefore, while simple reads are typically safe, complex read operations or reads with side effects may require
synchronization even when no thread is writing.

Are object creations thread-safe?
==================================

Object creation in Python is generally thread-safe for built-in types due to the GIL in CPython. Creating new instances
of basic types (int, str, list, dict, etc.) from multiple threads simultaneously is safe because each thread gets its
own new object. The allocation of memory and initialization of built-in objects are atomic operations under the GIL.
However, thread safety issues can arise in several circumstances::

(1) when creating instances of custom classes with ``__new__`` or ``__init__`` methods that access or modify shared state,

(2) when using class-level variables or metaclasses that maintain shared state during instantiation,

(3) when object creation involves singleton patterns or object pools that aren't properly synchronized,

(4) when using factory functions or builders that rely on global state, and

(5) in free-threaded Python builds or alternative implementations without a GIL.

Additionally, while the act of creating an object is thread-safe, any subsequent initialization that involves shared
resources still requires proper synchronization. For example, creating a new thread-safe counter object is safe, but if
its ``__init__`` method increments a global counter without locking, that operation would not be thread-safe.

When working with a single thread are all actions thread-safe?
================================================================

Yes, when your Python program uses only a single thread, all actions are inherently thread-safe because there is no
concurrent access to shared resources. Thread safety concerns only arise when multiple threads execute simultaneously
and potentially access the same data. In a single-threaded program, operations execute sequentially, eliminating race
conditions, data races, and synchronization issues. However, there are important caveats to consider::

- (1) some Python libraries may internally create additional threads (like certain networking or GUI libraries),
  making your seemingly single-threaded program multi-threaded,
- (2) signal handlers in Python can interrupt your single-threaded code at almost any point, creating concurrency issues
  similar to those in multi-threaded programs,
- (3) if your single-threaded Python code interacts with external multi-threaded systems (databases, web services,
  shared memory), you still need to consider thread safety at those boundaries, and
- (4) asynchronous programming with asyncio, while single-threaded, can still have concurrency issues between coroutines
  if shared state is modified without proper care.

Therefore, while a truly single-threaded program doesn't have thread safety issues by definition, modern Python
applications often involve hidden concurrency that requires careful consideration.

What are common ways thread-safety is broken?
==============================================

Thread safety is commonly broken in Python programs through several patterns that developers should avoid:

**1. Race Conditions in Check-Then-Act Patterns**: The most common mistake is checking a condition then acting on it
without atomic execution. For example, ``if key not in dict: dict[key] = value`` is not thread-safe because another
thread might add the key between the check and the assignment. Use ``dict.setdefault()`` or locks instead.

**2. Assuming Compound Operations are Atomic**: Operations like ``counter += 1`` appear simple but actually involve
multiple steps (read, modify, write) that can be interrupted. Even ``list.append()`` followed by ``list.pop()`` isn't
safe when multiple threads are involved. Always use locks for compound operations or use thread-safe alternatives like
``queue.Queue``.

**3. Improper Lock Management**: Using multiple locks incorrectly can cause deadlocks (threads waiting for each other
forever) or failing to use locks consistently (some code paths protected, others not). Always acquire multiple locks in
the same order across all threads and use context managers (``with lock:``) to ensure locks are properly released.

**4. Sharing Mutable Default Arguments**: Functions with mutable default arguments like
``def func(items=[]): items.append(x)`` share the same list object across all calls and threads. Use ``None`` as the
default and create new objects inside the function.

**5. Relying on the GIL for Complex Operations**: While the GIL makes individual bytecode operations atomic, it doesn't
protect sequences of operations. The GIL can switch between threads at any Python bytecode instruction, not just at
statement boundaries.

**6. Incorrect Use of Thread-Local Storage**: Accidentally sharing objects that should be thread-local, or assuming
thread-local storage persists across thread pool task executions. Each thread pool task might run on a different thread.

**7. File and I/O Race Conditions**: Multiple threads writing to the same file or reading/writing without proper
synchronization corrupts data. File operations that seem atomic (like writing a line) may not be, especially with
buffering.

**8. Forgetting About Iterator Invalidation**: Modifying a collection while another thread is iterating over it causes
"RuntimeError: dictionary changed size during iteration" or silent data corruption. Use locks or create copies for
iteration.

What are typical patterns to ensure thread-safety in your code?
================================================================

Thread safety can be achieved through several well-established patterns and best practices:

**1. Use Threading Primitives Properly**::

    import threading

    # Use Lock for mutual exclusion
    lock = threading.Lock()
    with lock:
        # Critical section - only one thread can execute this at a time
        shared_resource.modify()

    # Use RLock for re-entrant locking (same thread can acquire multiple times)
    rlock = threading.RLock()

    # Use Semaphore to limit concurrent access
    semaphore = threading.Semaphore(3)  # Max 3 threads

**2. Prefer Thread-Safe Data Structures**: Use ``queue.Queue``, ``queue.LifoQueue``, or ``queue.PriorityQueue``
instead of regular lists for inter-thread communication. These provide thread-safe ``put()`` and ``get()`` operations
with optional blocking and timeouts.

**3. Immutable Objects Pattern**: Design your data structures to be immutable. Instead of modifying objects, create new
ones. This eliminates synchronization needs since immutable objects can be safely shared::

    # Instead of modifying
    config.setting = new_value  # Not thread-safe

    # Create new immutable object
    config = config._replace(setting=new_value)  # Thread-safe with namedtuple

**4. Thread-Local Storage**: Use ``threading.local()`` for data that should be isolated per thread::

    thread_local_data = threading.local()
    thread_local_data.connection = create_connection()  # Each thread gets its own

**5. Atomic Operations with Threading Events**: Use ``threading.Event`` for simple signaling between threads without
sharing complex state::

    stop_event = threading.Event()
    # In worker thread
    while not stop_event.is_set():
        do_work()
    # In main thread
    stop_event.set()  # Signal all workers to stop

**6. Context Managers and Decorators**: Create reusable synchronization patterns::

    def synchronized(lock):
        def decorator(func):
            def wrapper(*args, **kwargs):
                with lock:
                    return func(*args, **kwargs)
            return wrapper
        return decorator

    shared_lock = threading.Lock()

    @synchronized(shared_lock)
    def critical_operation():
        # Automatically thread-safe
        pass

**7. Message Passing Instead of Shared State**: Use queues to pass messages between threads rather than sharing mutable
objects. This follows the "Don't communicate by sharing memory; share memory by communicating" principle.

**8. Copy-on-Write Pattern**: When reading shared data, work with copies to avoid interference::

    with lock:
        data_copy = shared_data.copy()
    # Work with data_copy outside the lock
    process(data_copy)

**9. Double-Checked Locking for Lazy Initialization** (use with caution)::

    _instance = None
    _lock = threading.Lock()

    def get_singleton():
        global _instance
        if _instance is None:  # First check (no lock)
            with _lock:
                if _instance is None:  # Second check (with lock)
                    _instance = create_expensive_object()
        return _instance

**10. Use Higher-Level Abstractions**: Prefer ``concurrent.futures.ThreadPoolExecutor`` or ``multiprocessing.Pool``
which handle synchronization internally, over manually managing threads and locks.



Does free-threading have different considerations than Python with a GIL with regards to thread safety?
========================================================================================================

Yes, free-threading (Python without the GIL) has significantly different and more stringent thread safety considerations
compared to traditional CPython with the GIL. The differences are fundamental and require a complete shift in how
developers approach concurrent Python code:

**Key Differences in Free-Threading:**

**1. No Automatic Atomicity**: In CPython with the GIL, individual bytecode operations are atomic. In free-threading,
even simple operations like ``x = y`` or ``list.append()`` may not be atomic. Multiple threads can truly execute Python
code simultaneously, creating genuine parallel execution and real race conditions.

**2. All Built-in Operations Need Review**: Operations that were safe due to the GIL now require explicit synchronization::

    # With GIL - safe
    my_list.append(item)
    my_dict[key] = value

    # Free-threading - needs locking
    with lock:
        my_list.append(item)
        my_dict[key] = value

**3. Reference Counting Challenges**: The GIL protected Python's reference counting mechanism. Free-threading
implementations must use atomic operations or other techniques for reference counting, which can impact performance and
correctness. Object deletion and garbage collection become more complex.

**4. C Extension Compatibility**: Many C extensions assume GIL protection and are not thread-safe. In free-threading
builds, these extensions may cause crashes or data corruption unless specifically updated for free-threading compatibility.

**5. Performance Characteristics Change**:
- True parallelism is possible, potentially offering significant speedups for CPU-bound multi-threaded code
- However, the overhead of fine-grained locking can make some operations slower
- Memory usage may increase due to additional synchronization structures

**6. Standard Library Adaptations**: The free-threaded builds require modifications to standard library modules. As
seen in the release notes, many modules like ``heapq``, ``sqlite3``, ``ssl``, and others have been specifically updated
for thread safety in free-threaded builds.

**7. Debugging Becomes Harder**: Race conditions that were impossible with the GIL can now occur, making bugs more
subtle and harder to reproduce. Tools like Thread Sanitizer (TSAN) become essential for detecting race conditions.

**Migration Considerations:**

- **Audit all shared state**: Every shared variable needs protection, not just complex operations
- **Use synchronization primitives liberally**: What worked with the GIL may fail catastrophically without it
- **Test extensively**: Code that has worked for years may have latent race conditions exposed by true parallelism
- **Monitor third-party dependencies**: Many packages assume GIL protection and may not be free-threading ready
- **Consider alternative approaches**: Sometimes switching to multiprocessing or async patterns may be safer than
  adapting code for free-threading

**Best Practices for Free-Threading:**

1. **When in doubt, assume something may not be thread-safe** unless explicitly documented
2. **Use immutable data structures** wherever possible
3. **Prefer message passing** over shared memory
4. **Use thread-safe collections** from the ``queue`` module
5. **Implement fine-grained locking** but watch for deadlocks
6. **Run with Thread Sanitizer** during development
7. **Document thread safety guarantees** in your own code

The free-threaded builds (available experimentally in Python 3.13+) represent a fundamental change in Python's
concurrency model. While they offer the promise of true parallelism for CPU-bound tasks, they require much more careful
attention to thread safety than traditional Python development.

How do you use the thread sanitizer?
=====================================

Thread Sanitizer (TSan) is a powerful runtime tool that detects data races and other threading bugs in C/C++ programs,
including CPython and its extensions. TSan works by instrumenting memory accesses and synchronization operations,
tracking the happens-before relationships between threads to identify potential race conditions that might not manifest
during normal testing.

**Building Python with Thread Sanitizer:**

To use TSan with CPython, you need to build Python from source with TSan enabled::

    # Configure Python build with TSan
    ./configure --with-thread-sanitizer --with-pydebug

    # For free-threaded builds, also add:
    ./configure --with-thread-sanitizer --with-pydebug --disable-gil

    # Build Python
    make -j$(nproc)

The ``--with-thread-sanitizer`` flag enables TSan instrumentation, while ``--with-pydebug`` provides additional
debugging symbols and assertions that help TSan provide more detailed reports.

**Running Python with Thread Sanitizer:**

Once built with TSan, simply run your Python scripts normally. TSan will automatically monitor all thread operations::

    # Run your script
    ./python my_threaded_script.py

    # Run the test suite with TSan
    ./python -m test -j$(nproc)

    # Run specific tests
    ./python -m test test_threading test_concurrent_futures

**Understanding TSan Output:**

When TSan detects a race condition, it produces a detailed report showing:

1. **The type of race** (data race, use-after-free, etc.)
2. **Stack traces** for the conflicting accesses
3. **Thread information** showing which threads were involved
4. **Memory addresses** where the race occurred

Example TSan output::

    WARNING: ThreadSanitizer: data race (pid=12345)
      Write of size 8 at 0x7b0400000000 by thread T2:
        #0 increment_counter counter.c:10 (python+0x123456)
        #1 worker_thread threading.c:50 (python+0x234567)

      Previous read of size 8 at 0x7b0400000000 by thread T1:
        #0 read_counter counter.c:15 (python+0x345678)
        #1 main_thread threading.c:30 (python+0x456789)

**Suppressing False Positives:**

CPython maintains suppression files for known benign races or intentional lock-free algorithms. These are located in
``Tools/tsan/`` directory:

- ``suppressions_free_threading.txt``: Suppressions for free-threaded builds
- ``suppressions.txt``: General suppressions for builds with GIL

To use custom suppressions, set the TSAN_OPTIONS environment variable::

    export TSAN_OPTIONS="suppressions=my_suppressions.txt"
    ./python my_script.py

**Common TSan Findings in Python Code:**

1. **Missing Locks**: Accessing shared data without proper synchronization
2. **Lock Order Violations**: Potential deadlocks from inconsistent lock ordering
3. **Race on Reference Counts**: Particularly important in free-threaded builds
4. **Unsynchronized Callbacks**: C extensions calling back into Python without proper locking

**Best Practices for TSan Testing:**

1. **Test with Realistic Workloads**: Use multi-threaded scenarios that stress concurrent access patterns
2. **Enable TSan in CI**: Include TSan builds in continuous integration to catch regressions
3. **Review All Reports**: Even "benign" races can indicate design issues
4. **Test C Extensions**: Many race conditions occur at the Python/C boundary
5. **Use with Free-Threading**: TSan is especially valuable for testing free-threaded Python builds where
   races are more likely to manifest

**Performance Considerations:**

Programs running under TSan typically experience:
- 5-15x CPU slowdown
- 5-10x memory overhead
- Increased thread synchronization overhead

Therefore, TSan should be used during development and testing, not in production environments.

**Integration with Development Workflow:**

For developers working on CPython or C extensions:

1. Build a TSan-enabled Python for testing alongside your regular development build
2. Run your test suite under TSan before submitting patches
3. Pay special attention to any new warnings introduced by your changes
4. Document any intentional lock-free algorithms that might trigger false positives

Standard Library Modules
========================

The following table lists all documentation in the CPython repository that mentions thread safety:

.. list-table:: Thread Safety References in CPython Documentation
   :header-rows: 1
   :widths: 30 25 45

   * - Page/Section Title
     - File Name
     - Snippet
   * - **How-to Guides**
     -
     -
   * - Thread safety in Python
     - ``Doc/howto/thread-safety.rst``
     - Document about thread safety (this document)
   * - **FAQ**
     -
     -
   * - Programming FAQ
     - ``Doc/faq/programming.rst``
     - "By using global variables. This isn't thread-safe, and is not recommended."
   * - Library FAQ
     - ``Doc/faq/library.rst``
     - "What kinds of global value mutation are thread-safe?"
   * - **Glossary**
     -
     -
   * - Python Glossary
     - ``Doc/glossary.rst``
     - "free-threading CPython does not guarantee the thread-safety of iterator operations"
   * - Python Glossary
     - ``Doc/glossary.rst``
     - "adding thread-safety, tracking object creation"
   * - **C API Documentation**
     -
     -
   * - Introduction
     - ``Doc/c-api/intro.rst``
     - "the preferred, thread-safe way to access the exception state"
   * - Dictionary Objects
     - ``Doc/c-api/dict.rst``
     - "The function is not thread-safe in the free-threaded build"
   * - Memory Management
     - ``Doc/c-api/memory.rst``
     - "These functions are thread-safe, so a thread state does not need to be attached"
   * - Memory Management
     - ``Doc/c-api/memory.rst``
     - "For the PYMEM_DOMAIN_RAW domain, the allocator must be thread-safe"
   * - Memory Management
     - ``Doc/c-api/memory.rst``
     - "All allocators must be thread-safe"
   * - Perf Maps
     - ``Doc/c-api/perfmaps.rst``
     - "create a lock to ensure thread-safe writes to the file"
   * - Perf Maps
     - ``Doc/c-api/perfmaps.rst``
     - "This function is thread safe"
   * - Bytes Objects
     - ``Doc/c-api/bytes.rst``
     - "The API is not thread safe: a writer should only be used by a single thread"
   * - **What's New Documentation**
     -
     -
   * - What's New in 3.10
     - ``Doc/whatsnew/3.10.rst``
     - "consider using asyncio.run_coroutine_threadsafe instead"
   * - What's New in 3.11
     - ``Doc/whatsnew/3.11.rst``
     - "sqlite3 now sets sqlite3.threadsafety based on the default threading mode"
   * - **Internal Documentation**
     -
     -
   * - Asyncio Internals
     - ``InternalDocs/asyncio.md``
     - "Thread safety: Before Python 3.14, concurrent iterations over WeakSet was not thread safe"
   * - Asyncio Internals
     - ``InternalDocs/asyncio.md``
     - "improve the performance and thread safety of tasks management"
   * - Garbage Collector
     - ``InternalDocs/garbage_collector.md``
     - "global interpreter lock for thread safety"
   * - Garbage Collector
     - ``InternalDocs/garbage_collector.md``
     - "performing a collection for thread safety"
   * - **Test Suppressions**
     -
     -
   * - TSAN Suppressions
     - ``Tools/tsan/suppressions_free_threading.txt``
     - "Range iteration is not thread-safe yet"
   * - TSAN Suppressions
     - ``Tools/tsan/suppressions_free_threading.txt``
     - "will probably need to rewritten as explicit loops to be thread-safe"
   * - **Release Notes (Selected)**
     -
     -
   * - Python 3.5.0a1
     - ``Misc/NEWS.d/3.5.0a1.rst``
     - "Writing to ZipFile and reading multiple ZipExtFiles is threadsafe now"
   * - Python 3.5.0a1
     - ``Misc/NEWS.d/3.5.0a1.rst``
     - "subprocess's Popen.wait() is now thread safe"
   * - Python 3.5.0a1
     - ``Misc/NEWS.d/3.5.0a1.rst``
     - "Improved thread-safety in logging cleanup during interpreter shutdown"
   * - Python 3.7.0a1
     - ``Misc/NEWS.d/3.7.0a1.rst``
     - "webbrowser.register() is now thread-safe"
   * - Python 3.8.0a1
     - ``Misc/NEWS.d/3.8.0a1.rst``
     - "Fixed thread-safety of error handling in _ssl"
   * - Python 3.9.0a1
     - ``Misc/NEWS.d/3.9.0a1.rst``
     - "In posix, use ttyname_r instead of ttyname for thread safety"
   * - Python 3.10.0a1
     - ``Misc/NEWS.d/3.10.0a1.rst``
     - "Fix a race condition in the call_soon_threadsafe() method"
   * - Python 3.11.0a1
     - ``Misc/NEWS.d/3.11.0a1.rst``
     - "Clarify that shutil.make_archive is not thread-safe"
   * - Python 3.11.0a1
     - ``Misc/NEWS.d/3.11.0a1.rst``
     - "Remove the unqualified claim that tkinter is threadsafe"
   * - Python 3.11.0a7
     - ``Misc/NEWS.d/3.11.0a7.rst``
     - "Fix thread safety of zipfile._SharedFile.tell"
   * - Python 3.12.0a1
     - ``Misc/NEWS.d/3.12.0a1.rst``
     - "On macOS, the libc syslog() function is not thread-safe"
   * - Python 3.12.0b1
     - ``Misc/NEWS.d/3.12.0b1.rst``
     - "thread-safety. If the module has external dependencies... that isn't thread-safe"
   * - Python 3.12.0b1
     - ``Misc/NEWS.d/3.12.0b1.rst``
     - "write to perf-map files in a thread safe manner"
   * - Python 3.13.0a2
     - ``Misc/NEWS.d/3.13.0a2.rst``
     - "Make hashlib related modules thread-safe without the GIL"
   * - Python 3.13.0a4
     - ``Misc/NEWS.d/3.13.0a4.rst``
     - "The free-threaded build now has its own thread-safe GC implementation"
   * - Python 3.13.0a4
     - ``Misc/NEWS.d/3.13.0a4.rst``
     - "Make queue.SimpleQueue thread safe when the GIL is disabled"
   * - Python 3.13.0a4
     - ``Misc/NEWS.d/3.13.0a4.rst``
     - "Make methods on collections.deque thread-safe when the GIL is disabled"
   * - Python 3.14.0a2
     - ``Misc/NEWS.d/3.14.0a2.rst``
     - "Fixed thread safety in ssl in the free-threaded build"
   * - Python 3.14.0a3
     - ``Misc/NEWS.d/3.14.0a3.rst``
     - "Make operator.methodcaller thread-safe and re-entrant safe"
   * - Python 3.14.0a3
     - ``Misc/NEWS.d/3.14.0a3.rst``
     - "grp: Make grp.getgrall thread-safe by adding a mutex"
   * - Python 3.14.0a3
     - ``Misc/NEWS.d/3.14.0a3.rst``
     - "Make linecache.checkcache thread safe and GC re-entrancy safe"
   * - Python 3.14.0a3
     - ``Misc/NEWS.d/3.14.0a3.rst``
     - "Fix non-thread-safe object resurrection when calling finalizers"
   * - Python 3.14.0a5
     - ``Misc/NEWS.d/3.14.0a5.rst``
     - "test suite is not yet reviewed for thread-safety"
   * - Python 3.14.0a5
     - ``Misc/NEWS.d/3.14.0a5.rst``
     - "Fix thread safety of PyList_Insert in free-threading builds"
   * - Python 3.14.0a5
     - ``Misc/NEWS.d/3.14.0a5.rst``
     - "Fix thread safety of PyList_SetItem in free-threading builds"
   * - Python 3.14.0a6
     - ``Misc/NEWS.d/3.14.0a6.rst``
     - "reversed is still not thread-safe in the sense that concurrent iterations"
   * - Python 3.14.0a7
     - ``Misc/NEWS.d/3.14.0a7.rst``
     - "Fix several thread-safety issues in ctypes on the free threaded build"
   * - Python 3.14.0b1
     - ``Misc/NEWS.d/3.14.0b1.rst``
     - "This makes using the context manager thread-safe in multi-threaded programs"
   * - **Upcoming Changes**
     -
     -
   * - Core and Builtins
     - ``Misc/NEWS.d/next/Core_and_Builtins/``
     - "Make cProfile thread-safe on the free threaded build"
   * - Core and Builtins
     - ``Misc/NEWS.d/next/Core_and_Builtins/``
     - "Make functions in grp thread-safe on the free threaded build"
   * - Core and Builtins
     - ``Misc/NEWS.d/next/Core_and_Builtins/``
     - "Make methods in heapq thread-safe on the free threaded build"
   * - Core and Builtins
     - ``Misc/NEWS.d/next/Core_and_Builtins/``
     - "Make functions in syslog thread-safe on the free threaded build"
   * - Core and Builtins
     - ``Misc/NEWS.d/next/Core_and_Builtins/``
     - "Make functions in pwd thread-safe on the free threaded build"
   * - Library
     - ``Misc/NEWS.d/next/Library/``
     - "Fix libc thread safety issues with pwd by locking access"
   * - Library
     - ``Misc/NEWS.d/next/Library/``
     - "Fix thread-safety issues in linecache"
   * - Library
     - ``Misc/NEWS.d/next/Library/``
     - "Fix libc thread safety issues with os by replacing getlogin"
   * - Library
     - ``Misc/NEWS.d/next/Library/``
     - "Update Python implementation of io.BytesIO to be thread safe"
   * - Library
     - ``Misc/NEWS.d/next/Library/``
     - "Fix libc thread safety issues with dbm by performing stateful operations"


Thread Safety References in Python Source Code
===============================================

The following table lists Python source files that mention thread safety in their code or comments:

.. list-table:: Thread Safety References in Python Source Files
   :header-rows: 1
   :widths: 25 35 40

   * - Module/Class
     - File Name
     - Context/Snippet
   * - **Core Library**
     -
     -
   * - asyncio.Future
     - ``Lib/asyncio/futures.py``
     - "This class is not thread-safe" (class docstring)
   * - asyncio.events._ThreadSafeHandle
     - ``Lib/asyncio/events.py``
     - "_ThreadSafeHandle is used for callbacks scheduled with call_soon_threadsafe and is thread safe unlike Handle"
   * - asyncio.base_events
     - ``Lib/asyncio/base_events.py``
     - "call_soon_threadsafe(): Like call_soon(), but thread-safe"
   * - asyncio.base_events
     - ``Lib/asyncio/base_events.py``
     - "Non-thread-safe methods of this class make this assumption"
   * - asyncio.base_events
     - ``Lib/asyncio/base_events.py``
     - "Non-thread-safe operation invoked on an event loop other than the current one"
   * - asyncio.tasks
     - ``Lib/asyncio/tasks.py``
     - "run_coroutine_threadsafe: Submit a coroutine object to a given event loop"
   * - asyncio.tasks
     - ``Lib/asyncio/tasks.py``
     - "uses itertools.count() instead of += 1 because the latter is not thread safe"
   * - functools
     - ``Lib/functools.py``
     - "The internals of the lru_cache are encapsulated for thread safety"
   * - functools
     - ``Lib/functools.py``
     - "lock = RLock() # because linkedlist updates aren't threadsafe"
   * - logging
     - ``Lib/logging/__init__.py``
     - "Add thread safety in case someone mistakenly calls basicConfig() from multiple threads"
   * - multiprocessing.heap
     - ``Lib/multiprocessing/heap.py``
     - "appending and retrieving from a list is not strictly thread-safe but under CPython it's atomic thanks to the GIL"
   * - multiprocessing.managers
     - ``Lib/multiprocessing/managers.py``
     - "self.id_to_obj[ident] = (None, (), None) # thread-safe"
   * - multiprocessing.context
     - ``Lib/multiprocessing/context.py``
     - "gh-84559: We changed everyones default to a thread safeish one in 3.14"
   * - optparse.OptionParser
     - ``Lib/optparse.py``
     - "OptionParser is not thread-safe" (class docstring)
   * - random
     - ``Lib/random.py``
     - "The random() method is implemented in C... and is, therefore, threadsafe" (module docstring)
   * - random.Random.gauss
     - ``Lib/random.py``
     - "Not thread-safe without a lock around calls" (method docstring)
   * - sched
     - ``Lib/sched.py``
     - "localize variable access to minimize overhead and to improve thread safety"
   * - urllib.request.CacheFTPHandler
     - ``Lib/urllib/request.py``
     - "XXX this stuff is definitely not thread safe"
   * - importlib._bootstrap
     - ``Lib/importlib/_bootstrap.py``
     - "block on taking the lock, which is what we want for thread safety"
   * - **Test Suite**
     -
     -
   * - test_atexit
     - ``Lib/test/test_atexit.py``
     - "test_atexit_thread_safety: GH-126907: atexit was not thread safe"
   * - test_multiprocessing
     - ``Lib/test/_test_multiprocessing.py``
     - "test_thread_safety: bpo-24484: _run_finalizers() should be thread-safe"
   * - test_free_threading.test_io
     - ``Lib/test/test_free_threading/test_io.py``
     - "ThreadSafetyMixin: Test pretty much everything that can break under free-threading"
   * - test_io
     - ``Lib/test/test_io/__init__.py``
     - "test_free_threading/test_io - tests thread safety of io objects"
   * - test_ssl
     - ``Lib/test/test_ssl.py``
     - "test_load_cert_chain_thread_safety: gh-134698: _ssl detaches the thread state"
   * - test_sqlite3
     - ``Lib/test/test_sqlite3/test_dbapi.py``
     - "test_thread_safety: self.assertIn(sqlite.threadsafety, {0, 1, 3})"
   * - test_ctypes
     - ``Lib/test/test_ctypes/test_cfuncs.py``
     - "test_thread_safety: only meaningful on free-threading"
   * - test_ctypes
     - ``Lib/test/test_ctypes/test_arrays.py``
     - "test_thread_safety: only meaningful if the GIL is disabled"
   * - test_concurrent_futures
     - ``Lib/test/test_concurrent_futures/test_future.py``
     - "Test concurrent access (basic thread safety check)"
   * - **Build/Tools**
     -
     -
   * - Mac Build Script
     - ``Mac/BuildScript/build-installer.py``
     - "--enable-threadsafe" (SQLite configuration option)
   * - C Analyzer
     - ``Tools/c-analyzer/cpython/_analyzer.py``
     - "Local static mutexes are used to wrap libc functions that aren't thread safe"
   * - Distutils (C Analyzer)
     - ``Tools/c-analyzer/distutils/cygwinccompiler.py``
     - "-mthreads: Support thread-safe exception handling on Mingw32"

