# Parallel Programming Chapter 6: Performance analysis

## 1. Sequential performance

### 1.1. Performance Analysis tool types

1. source code instrumentation: exact, but invasive
2. automatic addition of tool functionality to source code
3. compiler instrumentation: requires source, but transparent
4. binary instrumentation: can be transparent, but still costly
5. link-level: transparent, less costly, but limited to APIs
6. wrapping the entire execution with a timing command

### 1.2. Performance analysis focuses

1. identifying computational intensive parts
   1. where am I spending my time?
      1. modules/libraries
      2. loops
      3. statement
      4. functions
   2. is the time spent in computational kernel?
      1. does this match my intuition?
   3. can the code be vectorize?
   4. is the memory hierarchy impacting?
      1. do I have excessive cache misses?
      2. how is my data locality?
      3. impact of TLB misses?
   5. is my I/O efficient?
   6. time spent in system libraries?

#### 1.2.1. Inclusive vs. exclusive timing

* `Exclusive` timing: time spent inside a function only
* `Inclusive` timing: time spent inside a function and its children
* if similar, children executions are insignificant
* if inclusive time is significantly greater than exclusive time: focus attention to the execution times of the children

#### 1.2.2. Hotpath analysis

* which paths takes the most time?

#### 1.2.3. Butterfly analysis (best known from `gprof`)

* shows split of time in callees and callers

## Synchronization overheads

good locking behavior:

* only use locks when needed
* reduce time spent in locks
* carefully order lock usage to avoid deadlocks
* if critical sections have to be substantial, try overlapping

alternative: lock-free data structures

* avoid use of mutex/critical sections
* carefully manipulate memory to avoid bad "intermediate" state
* check [http://www.rossbencina.com/code/lockfree?q=~rossb/code/lockfree/](http://www.rossbencina.com/code/lockfree?q=~rossb/code/lockfree/) for more reference to lock-free data structures

## Cache behavior and locality

improving cache performance is critical for sequential code.

### False sharing

one thread only wants to modify one element a[0] in array a. Another thread only wants to modify one element a[1] in array a. But everytime a thread loads the element in main memeory or higher level of cache, it loads a cache line(for example: 4 contiguous elements a[0]-a[3]). So these two threads will load the same cache line initially and modify only one element in the cache line. When they try to store the result back to the higher cache, the a[1] in the first thread is not updated and the a[0] in the second thread is not updated. So this behavior will cause errors and is named false sharing.

### OpenMP thread locations

we can use ICV(internal control variables) to control thread location.

1. `OMP_PROC_BIND`
   1. true: threads are bound to cores/hw-threads
   2. false: threads are not bound and can be migrated
   3. master: new threads are always co-located with master
   4. close: new threads are located "close" to their master threads
   5. spread: new threads are spread out as much as possible
2. `OMP_PLACES`
   example:
   OMP_PLACES={start: length: stride}