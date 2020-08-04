# Parallel Programming Chapter 3: OpenMP Basics

## 1. OpenMP Basics

### 1.1. Compilation

```bash
gcc -O3 -fopenmp openmp.c
```

## 2. OpenMP syntax

Control degree of parallelism by Internal Control Variable `OMP_NUM_THREADS`

### 2.1. Basic parallelism

the code will run the parallel block OMP_NUM_THREADS times

```c++
#include <omp.h>
#pragma omp parallel [parameter]
{

}
```

### 2.2. Sections

* must be in a parallel region
* each section within a sections block is executed once by one thread
* Threads that finished their section wait at an implicit barrier at the end of the sections block

```c++
#pragma omp parallel sections [parameters]
{
    #pragma omp section
    {

    }

    #pragma omp section
    {

    }
}
```

### 2.3. For loop

* no synchronization at the beginning
* synchronization at an implicit barrier unless parameter `nowait` is specified
* iterations must be independent

```c++
#pragma omp parallel for [parameters]
for(int i = 0; i < 100; ++i)
{

}
```

#### 2.3.1. Schedule

```c++
// fixed sized chunks, distributed in one time. default size is (total num/thread num)
#pragma omp for schedule(static)
// fix sized chunks, distributed one by one as chunks finished. default size is 1
#pragma omp for schedule(dynamic)
// start with large chunks, then exponentially decreasing size, distributed one by one
#pragma omp for schedule(guided)
// controlled at the runtime using control variable
#pragma omp for schedule(runtime)
// compiler/runtime can choose
#pragma omp for schedule(auto)
```

### 2.4. Shared memory

#### 2.4.1. Usage

```c++
void main(){
int t;
#pragma omp parallel
{
    #pragma omp for private(t)
    for(int i = 0; i < 100; ++i)
    {

    }
}
```

global variable: t but privatized in each thread

local variable: i

#### 2.4.2. Sharing attributes

1. private(var-list)
    * variables in var-list are private for each thread
    * they are independent

2. shared(var-list)
    * variables in var-list are shared

3. default(private | shared | none)
    * set the default attribute for all variables in this region

4. firstprivate(var-list)
    * initialize the values with the values before the region
    * variables are private but initialized with the value of the shared copy before the region

5. lastprivate(var-list)
    * copy the last to out of the region
    * variables are private but the value of the thread executing the last iteration of a parallel loop in sequential order is copied to the variable outside of the region(irrelevant to the threads, only relevant to the last loop iterations)
   
* the default sharing attribute for global and static variable is `shared`
* the default sharing attribute for `#pragma omp task` region is `firstprivate`
* orphaned task region (only one task) has default attribute `firstprivate`
* non-orphaned task inherit the `shared` attribute

### 2.5. Synchronization

#### 2.5.1. Barrier

* each parallel region has an implicit barrier at the end
* the implicit barrier can be switch off by adding `nowait`
* explicit barrier is as follow

```c++
#pragma omp barrier
```

* can cause load imbalance
* use when really needed

#### 2.5.2. Master region

* a master region enforces that only the master executes the code block
* other thread skip the region

```c++
#pragma omp master
```

#### 2.5.3. Single region

* a single region enforces that only a single thread executes the code block
* other thread skip the region
* implicit barrier synchronization at the end of region
* possible uses: initialization of data structures

```c++
#pragma omp single
```

#### 2.5.4. Critical section

* a critical section is a block of code
* can only be executed by only one thread at a time
* critical section name(global, should not have conflicts) identifies the specific critical section
* all unnamed critical directives map to the same name
* avoid long critical sections for performance reasons

```c++
#pragma omp critical [name]
```

#### 2.5.5. Atomic statements

* the ATOMIC directive ensures that a specific memory location is updated atomically
* useful for simple/fast updates to shared data structures
* avoid locking

```c++
#pragma ATOMIC
    expression-statement
```

### 2.6. Locks

```c++
// necessary initialization and finalization
omp_init_lock(&lockvar);
// set and unset lock
omp_set_lock(&lockvar);
omp_unset_lock(&lockvar);
// test lock combined with while loop

while(!omp_test_lock(&lockvar))
{
    // lock not obtained
}
// lock obtained

// return the lock
omp_unset_lock(&lockvar);
// destroy lock
omp_destroy_lock(&lockvar);
```

### 2.7. Ordered construct

* construct must be within the `dynamic` extent of an `omp for` construct with an ordered clause
* ordered construct are executed strictly in the order in which they would be executed in a sequential execution of the loop

```c++
#pragma omp for ordered
for(...)
{
    #pragma omp ordered
    {
        ...
    }
}
```

### 2.8. Reductions

this clause performs a reduction:

* syntax `reduction(operator: list)`
* on the variables that appear in list
* with the operator *operator*
* across the values "at thread end" or "last iteration" of each thread

```c++
int a = 0;
#pragma omp parallel for reduction(+: a)
for (int i = 0; i < 4; i++)
{
    a = i;
}
printf("%f",a);
```

if OMP_NUM_THREADS=4

Output: `0 + 1 + 2 + 3 = 6`

```txt
4
```

if OMP_NUM_THREADS=2

Output: `1 + 3 = 4`

```txt
4
```
