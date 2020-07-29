# Parallel Programming Chapter 4: OpenMP Advances

## 1. OpenMP Runtime Routines and ICV

### 1.1. Runtime routines

```c++
// set the number of threads
omp_set_num_threads(count);

// get the maximum number of threads for threads creation
int omp_get_max_threads();

// get the number of threads in current team
int omp_get_num_threads();

// get the current thread id
int omp_get_thread_num();

// get the number of processors
int omp_get_num_procs();
```

### 1.2. Environment variables (ICVs)

1. `OMP_NUM_THREAD=4`
    * set the number of threads in a team of parallel region

2. `OMP_SCHEDULE="dynamic" OMP_SCHEDULE="GUIDED,4"`
    * selects scheduling strategy to be applied at runtime
    * schedule clause in the code takes precedence

3. `OMP_DYNAMIC=TRUE`
    * allow runtime system to determine the number of threads

4. `OMP_NESTED=TRUE`
    * allow nesting of parallel regions
    * if supported by the runtime

## 2. Data races

Three kinds of data races:

* Read after Write(RAW)
* Write after Read(WAR)
* Write after Write(WAW)

For determinning the source: the source should always be the first operation: write in RAW, read in WAR and first write in WAW.

### 2.1. Data dependencies

#### 2.1.1. Three types of dependences

1. True/Flow dependence (read after write, write is source)

    ```c++
    s1: x = ...
    s2: ... = x
    ```

2. Anti dependence (write after read, read is source)

    ```c++
    s1: ... = x
    s2: x = ...
    ```

3. Output dependence (write after write, first wirte is source)

    ```c++
    s1: x = ...
    s2: x = ...
    ```

#### 2.1.2. Loop terminology

* Loop's nesting level = number of surrounding loops + 1
* Iteration number = value of the iteration index normalized to (0 to n-1)
* Iteration vector = describes one iteration at the innermost loop

    ```c++
    // 1st level
    for(int i = 0; i < 100; i++)
    {
        // 2nd level
        for(int j = 0; j < 100; j++)
        {
            // 3rd level
            for(int k = 0; k < 100; k++)
            {

            }
        }
    }
    ```

    example: a tuple I = (3,2,7) describes the 3th iteration at the first level, the 2nd iteration at the second level and the 7th iteration at the 3rd level.

* dependence distance for S(I)(destination) S(J)(source): J-I (for a d-level loop then J-I is also d-dimensional)(can use the index of the array in that expression to represent)

    for example:

    ```c++
    for(int i = 1; i < 100; i++)
    {
        for(int j = 1; j < 100; j++)
        {
            a[i+1] = a[i] + 1;
        }
    }
    ```

    the iteration vector for `a[i+1]` can be represented by `i+1`, `a[i]` by `i`

* dependence direction: tuple with `<`(if 0 < distance), `=`, `>`(if 0 > distance) for each loop showing sign of the distance. destination iteration (`>``=``<`) source iteration.
* on the dependence graph: always source -> destination

#### 2.1.3. Loop transformation

1. loop interchange

    if the leftmost non-`=`  loop dependency is always `>`(only depends on the previous iteration)

    ```c++
    for (i = 0; i < 100; i++)
        for (j = 0; j < 100; j++)
            for (k = 0; k < 100; k++)
                // do something
    // is the same as
    for (i = 0; i < 100; i++)
        for (k = 0; k < 100; k++)
            for (j = 0; j < 100; j++)
                // do something that does not depends on
    ```

2. loop distribution

    * loop distribution

    can seperate parallelizable part and non-parallelizable part of code

    ```c++
    for(...)
    {
        s1: a[i] = b[i] + 2;
        s2: c[i] = a[i-1] * 2;
    }
    // can be distributed to two independent loops
    for(...)
    {
        s1: a[i] = b[i] + 2;
    }
    for(...)
    {
        s2: c[i] = a[i-1] * 2;
    }
    ```
3. loop fusion

    increases granularity

    ```c++
    for(...)
    {
        a[i] = b[i] + 2;
    }
    for(...)
    {
        c[i] = d[i+1] * a[i];
    }
    // can be fused into 
    for(...)
    {
        a[i] = b[i] + 2;
        c[i] = d[i+1] * a[i];
    }
    ```

4. loop alignment

    ```c++
    for(...) // i = 2:n
    {
        s1: a[i] = b[i] + 2;
        s2: c[i] = a[i-1] * 2;
    }
    // can be aligned in this way
    for(...) // i = 1:n
    {
        if (i > 1) a[i] = b[i] + 2;
        if (i < n) c[i+1] = a[i] * 2;
    }
    // or
    c[2] = a[1] * 2;
    for(...) // i = 2:n-1
    {
        s1: a[i] = b[i] + 2;
        s2: c[i+1] = a[i] * 2;
    }
    a[n] = b[n] + 2;
    ```

## 3. OpenMP advances

### 3.1. Task

#### 3.1.1. Task basics

Tasks are independent piece of work

* Tied tasks vs untied tasks

  * tied(default): once a task starts it will remain on the same thread

  * untied: tasks can move to a different thread

* task syntax `#pragma omp task [clause list]{...}`

* clauses
  * `if(scalar-expression)`: if false, execution starts immediately by the creating thread
  * `untied`: task is not tied to the thread starting its execution
  * `default (shared | none)`, private, firstprivate, shared: default is firstprivate
  * `priority(value)`: hint to influence order of execution, must not be used to rely on task ordering

    ```c++
    #pragma omp parallel
    {
        #pragma omp single
        {
            for(...)
            {
                #pragma omp task
                    process();

            }
        }
    }
    ```

#### 3.1.2. Task wait and task yield

1. waits for completion of immediate child tasks

    ```c++
    #pragma omp taskwait
    ```

2. specifies that the current task can be suspended

    ```c++
    #pragma omp taskyield
    ```

#### 3.1.3. Task dependencies

defines in/out dependencies between tasks

* Out: variables produced by this task
* In: variables consumed by this task
* Inout: variables that are both in and out

```c++
#pragma omp task depend(dependency-type: list)
```

example:

```c++
#pragma omp task shared(x, ...) depend(out: x)
```

### 3.2. Flush

because complier optimization will change the order of the code. Flush can avoid those problems

```c++
#pragma omp flush [(list)]
```

* synchronizes data of the executing thread with main memory
* it does not update implicit copies at other threads
* loads/stores executed before the flush have to be finished
* load/stores following the flush are not allowed to be executed early





