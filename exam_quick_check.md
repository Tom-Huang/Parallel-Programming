# Exam quick check: template of code
<!-- TOC -->

- [1. pthread](#1-pthread)
    - [1.1. compile](#11-compile)
    - [1.2. basics operations](#12-basics-operations)
    - [1.3. Implement a barrier with conditional variables](#13-implement-a-barrier-with-conditional-variables)
    - [1.4. Implement dynamic tasks allocation as `#pragma omp parallel schedule(dynamic,1)`](#14-implement-dynamic-tasks-allocation-as-pragma-omp-parallel-scheduledynamic1)
    - [1.5. Communication between threads with conditional variables](#15-communication-between-threads-with-conditional-variables)
- [2. OpenMP](#2-openmp)
    - [2.1. compile](#21-compile)
    - [2.2. some attention points](#22-some-attention-points)
        - [2.2.1. schedule](#221-schedule)
    - [2.3. typical concepts](#23-typical-concepts)
        - [2.3.1. nested](#231-nested)
        - [2.3.2. sections and task](#232-sections-and-task)
        - [2.3.3. single and critical and master](#233-single-and-critical-and-master)
        - [2.3.4. ordered](#234-ordered)
        - [2.3.5. reduction](#235-reduction)
        - [2.3.6. runtime routines](#236-runtime-routines)
        - [2.3.7. Environment variables (ICVs)](#237-environment-variables-icvs)
        - [2.3.8. lock](#238-lock)
- [3. SIMD](#3-simd)
    - [3.1. compile](#31-compile)
    - [3.2. basics](#32-basics)
    - [3.3. operations](#33-operations)
- [4. MPI](#4-mpi)
    - [4.1. compile](#41-compile)
    - [4.2. basics](#42-basics)
        - [4.2.1. MPI_Comm_split](#421-mpi_comm_split)
        - [4.2.2. MPI_Sendrecv](#422-mpi_sendrecv)
        - [4.2.3. reuse processes for parallelization](#423-reuse-processes-for-parallelization)
    - [4.3. all sorts of collectives](#43-all-sorts-of-collectives)
        - [4.3.1. MPI_Gather](#431-mpi_gather)
        - [4.3.2. MPI_Allgather](#432-mpi_allgather)
        - [4.3.3. MPI_Reduce](#433-mpi_reduce)
        - [4.3.4. MPI_Allreduce](#434-mpi_allreduce)
        - [4.3.5. MPI_Iallreduce](#435-mpi_iallreduce)
        - [4.3.6. MPI_Bcast](#436-mpi_bcast)
        - [4.3.7. MPI_Scatter](#437-mpi_scatter)
    - [4.4. window allocation](#44-window-allocation)
        - [4.4.1. allocate and free window](#441-allocate-and-free-window)
        - [4.4.2. create window](#442-create-window)
        - [4.4.3. dynamically create](#443-dynamically-create)

<!-- /TOC -->
## 1. pthread

### 1.1. compile

name the sequential implementation as sequential_implementation.cpp, name the submission as student_submission.cpp

```Makefile
all: student_submission sequential

sequential: sequential_implementation.cpp
g++ -std=c++17 -Wall -Wextra -o sequential_implementation -O3 -g sequential_implementation.cpp -lpthread -lm

student_submission: student_submission.cpp
g++ -std=c++17 -Wall -Wextra -o student_submission -O3 -g student_submission.cpp -lpthread -lm
```

### 1.2. basics operations

```c++
// include library
#include <pthread.h>

// define number of threads
#define NUM_THREAD 4

// create threads
pthread_t threads[NUM_THREAD];
for(int i = 0; i < NUM_THREAD; i++){
    pthread_create(&threads[i], nullptr, kernel, (void *)args);
}

// joining threads
for(int i = 0; i < NUM_THREAD; i++){
    pthread_join(threads[i], nullptr);
}

```

design of the kernel

```c++
// should be this type
void *kernel(void *args){

    // read the arguments
    int input = *(*int *) args;

    // if we want return values, use input args pointer to return
    int result;
    *(int *) args = result;

    return nullptr;
}
```

### 1.3. Implement a barrier with conditional variables

kernel:

```c++
// make the kernel waits until the condition is satisfied

int num_threads = 4;
int counter = 0;
pthread_cond_t condition = PTHREAD_COND_INITIALIZER;
pthread_mutex_t condition_mutex = PTHREAD_MUTEX_INITIALIZER;

void* kernel(void* arg){
    int id = *(int*) arg;
    pthread_mutex_lock(&condition_mutex);
    counter++;

    if (counter == num_threads) {
        counter = 0;
        // if the condition is satisfied, send signal to all other threads
        pthread_cond_broadcast(&condition);
    }
    else {
        // this operation will make the thread wait and unlock the mutex lock
        pthread_cond_wait(&condition, &condition_mutex);
        // once the signal is received, the lock will be locked again
    }
    pthread_mutex_unlock(&condition_mutex);
    return nullptr;
}
```

main:

```c++
#include <pthread.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char** argv){
    pthread_t threads[num_threads];
    int ids[num_threads];

    for (int i = 0; i < num_threads; i++){
        ids[i] = i;
        pthread_create(threads+i, NULL, kernel, ids+i);
    }

    for (int i = 0; i < num_threads; i++){
        pthread_join(threas[i], NULL);
    }
    return 0;
}
```

### 1.4. Implement dynamic tasks allocation as `#pragma omp parallel schedule(dynamic,1)`

kernel:

```c++
#include <pthread.h>
#define NUM_THREADS 4
#define NUM_TASKS 1000

// use of shared data structure to distribute
unsigned int nextTask = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int results[NUM_TASKS];

// define a kernel to process task, which take task id as input
int process(unsigned int task_id){
    ...
    return result;
}

void* kernel(void* args){
    while(true){
        pthread_mutex_lock(&lock);
        if (nextTask >= NUM_TASKS){
            pthread_mutex_unlock(&lock);
            break;
        }
        unsigned task = nextTask;
        nextTask++;
        pthread_mutex_unlock(&lock);
        results[task] = process(task);
    }
    return nullptr;
}
```

### 1.5. Communication between threads with conditional variables

```c++
//thread 1:
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
bool condition = false;

void* thread1(){
pthread_mutex_lock(&mutex);
while (!condition)
    pthread_cond_wait(&cond, &mutex);
/* do something that requires holding the mutex and condition is true */
pthread_mutex_unlock(&mutex);
}

//thread2:
void* thread2(){
pthread_mutex_lock(&mutex);
/* do something that might make condition true */
pthread_cond_signal(&cond);
pthread_mutex_unlock(&mutex);
}

```

## 2. OpenMP

### 2.1. compile

```Makefile
CXX = g++
CXX_FLAGS = --std=c++17 -Wall -Wextra -fopenmp -O3

all: sequential_implementation student_submission test

sequential_implementation: sequential_implementation.cpp
$(CXX) $(CXX_FLAGS) -o sequential_implementation sequential_implementation.cpp 

student_submission: student_submission.cpp 
$(CXX) $(CXX_FLAGS) -o student_submission student_submission.cpp 
```

### 2.2. some attention points

#### 2.2.1. schedule

* if for loop index is defined outside of the loop, check if you need to make it firstprivate. If the index is still used after the loop, check if you need to make it lastprivate.
* when creating `task`, remember to specify `shared(var)` when you need variable to be shared from global, because by default `task` is `firstprivate`.
* when creating `task`, remember to let only one thread to create
    ```c++
    #pragma omp parallel
    {
        #pragma omp single
        {
            #pragma omp task
            {

            }
        }
    }
    ```
* when you try to write a variable defined outside of the parallel block, check if you need to privatize it

### 2.3. typical concepts

#### 2.3.1. nested

* two levels nested parallel for loop
* set number of threads in pragma

```c++
omp_set_nested(2);
#pragma omp parallel for num_threads(6)
```

#### 2.3.2. sections and task

[sections](./chapter3_openmp_basics.md)

[task](./chapter4_openmp_advances.md)

* sections have implicit barrier after the block of sections. Task only creates a queue of tasks for spare threads to execute. But task can also specify `taskwait` to create a barrier for the completion of the immediate child tasks.
* one section block can only be executed by one thread, default tied task also can be run only on one thread. But untied task can move across thread when task is being executed.
* task's default schedule is firstprivate

```c++
#pragma omp sections
{
    #pragma omp section
    {

    }

    #pragma omp section
    {

    }
}
```

```c++
#pragma omp parallel
{
    #pragma omp task
    {

    }

    #pragma omp task
    {

    }
}
```

#### 2.3.3. single and critical and master

[single and critical and master](./chapter3_openmp_basics.md)

* single only one thread can execute this block. critical only one thread at a time can execute this block(can be executed many times by different threads, but at different time). master only master thread can execute this block.
* single has implicit barrier at the end.
* critical can have name but can not have name conflicts.
* master is used for printing to screen, file I/O and so on.

```c++
#pragma omp master
#pragma omp single [parameters]
#pragma omp critical [name]

```

#### 2.3.4. ordered

* make sure that the block is executed in sequential for loop ordered

```c++
#pragma omp parallel for ordered
for(int i = 0; i < 100; i++)
{
    process();
    #pragma omp ordered
    {

    }
}
```

#### 2.3.5. reduction

```c++
#pragma omp parallel for reduction(+:var)
for(int i = 0; i < 100; i++)
{

}
```

#### 2.3.6. runtime routines

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

#pragma omp proc_bind(KEYWORD)
// KEYWORD could be:
// true: threads are bound to cores
// false: threads are not bound and can be migrated
// master: new threads are located "close" to their master threads
// close: new threads are located "close" to their master threads
// spread: new threads are spread out as much as possible
```

#### 2.3.7. Environment variables (ICVs)

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

#### 2.3.8. lock

```c++
#include <omp.h>
int id;
omp_lock_t lock;

omp_init_lock(lock);

#pragma omp parallel shared(lock) private(id) {
    id = omp_get_thread_num();
    omp_set_lock(&lock); //Only a single thread writes
    printf("My Thread num is: %d", id);
    omp_unset_lock(&lock);

    while (!omp_test_lock(&lock)) other_work(id); //Lock not obtained
    real_work(id); //Lock obtained
    omp_unset_lock(&lock); //Lock freed
}
omp_destroy_lock(&lock);
```

## 3. SIMD

### 3.1. compile

```c++
CXX = g++
CXX_FLAGS = --std=c++17 -Wall -Wextra -mavx -O3 -g

all: sequential_implementation student_submission

sequential_implementation: sequential_implementation.cpp
$(CXX) $(CXX_FLAGS) -o sequential_implementation sequential_implementation.cpp

student_submission: student_submission.cpp
$(CXX) $(CXX_FLAGS) -o student_submission student_submission.cpp

clean:
rm -f sequential_implementation student_submission
```

### 3.2. basics

```c++
#include <immintrin.h>
#define SIZE 1511
// define aligned iteration number
const int aligned = SIZE - SIZE % 8;

float a[SIZE];
float b[SIZE];
float c = 0;

__m256 partial_sum = _mm256_set1_ps(0);

// iteration step set to 8
for(int k = 0; k < aligned; k+=8)
{
    // define intrinsics variable
    __m256 a_i = _mm256_loadu_ps(a + k);
    __m256 b_i = _mm256_loadu_ps(b + k);
    partial_sum = _mm256_add_ps(partial_sum, _mm256_mul_ps(a_i, b_i));
}

float result[8];

// copy the vector(8 floats) to main memory
_mm256_storeu_ps(resutl, partial_sum);

for (int i = 0; i < 8; i++)
{
    c += result[i];
}

// calculate the remainder
for(int k = aligned; k < SIZE; k++)
{
    c += a[k] * b[k];
}
```

### 3.3. operations

```

```

## 4. MPI

### 4.1. compile

```Makefile
CXX = mpicxx
CXX_FLAGS = --std=c++17 -Wall -Wextra -march=native -O3 -g -DOMPI_SKIP_MPICXX
# this compiler definition is needed to silence warnings caused by the openmpi CXX
# bindings that are deprecated. This is needed on gcc 8 forward.
# see: https://github.com/open-mpi/ompi/issues/5157

all: sequential_implementation student_submission

sequential_implementation: sequential_implementation.cpp
	$(CXX) $(CXX_FLAGS) -o sequential_implementation sequential_implementation.cpp

student_submission: student_submission.cpp
	$(CXX) $(CXX_FLAGS) -o student_submission student_submission.cpp

clean:
	rm -f sequential_implementation student_submission
```

### 4.2. basics

* first iteration doesn't need to receive
* last iteration doesn't need to send
* make use of non-blocking time between start and complete

```c++
#include <mpi.h>

int main(int argc, char** argv)
{
    // init
    MPI_Init(&argc, &argv);

    // get rank and size
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    // send and receive
    // don't receive for the first rank
    if (rank > 0)
        MPI_Recv(&value, 1, MPI_DOUBLE, rank-1, 0, MPI_COMM_WORLD, &s);
    // don't send for the last rank
    if (rank < size - 1)
        MPI_Send(&value, 1, MPI_DOUBLE, rank+1, 0, MPI_COMM_WORLD);
    // output in the last rank
    if (rank == size -1 )
        printf("Value from MPI Process 0: %f\n",value);

    // finalize
    MPI_Finalize();
}
```

#### 4.2.1. MPI_Comm_split

```c++
int main ( int argc , char ** argv )
{
    int value , temp ;
    int size , rank , row , col[4] ;
    MPI_Comm c[4];

    MPI_Init (& argc , & argv );
    MPI_Comm_size ( MPI_COMM_WORLD , & size );
    MPI_Comm_rank ( MPI_COMM_WORLD , & rank );
    
    col[0] = rank & (1 << 3);
    col[1] = rank & (1 << 2);
    col[2] = rank & (1 << 1);
    col[3] = rank & (1 << 0);

    MPI_Comm_split ( MPI_COMM_WORLD , col[0] , rank , c );
    MPI_Comm_split ( c[0]           , col[1] , rank , c + 1 );
    MPI_Comm_split ( c[1]           , col[2] , rank , c + 2 );
    MPI_Comm_split ( c[2]           , col[3] , rank , c + 3 );


    MPI_Allreduce (& rank , & value , 1 , MPI_INT , MPI_MIN , c[3] );
    if (rank == 10) printf("Value 4 %i\n", value);
    temp = value ;

    MPI_Allreduce (& temp , & value , 1 , MPI_INT , MPI_SUM , c[1] );
    if (rank == 11) printf("Value 5 %i\n", value);
    temp = value ;

    MPI_Allreduce (& temp , & value , 1 , MPI_INT , MPI_MAX , c[2] );
    if (rank == 12) printf("Value 6 %i\n", value);

    MPI_Finalize ();
}
```

#### 4.2.2. MPI_Sendrecv

```c++
int main1(int argc, char* argv[]) {
    int rank, size, buf;
    MPI_Init(&argc, &argv); /* starts MPI */
    MPI_Comm_rank(MPI_COMM_WORLD, &rank); /* process id */
    MPI_Comm_size(MPI_COMM_WORLD, &size); /* number processes */
    buf = rank;

    if (rank == 0){
        MPI_Recv(&buf, 1, MPI_INT, (rank + size - 1) % size /*source*/, 0 /*tag*/, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        MPI_Send(&buf, 1, MPI_INT, (rank + 1) % size /*dest*/,0 /*tag*/,MPI_COMM_WORLD );
    } else {
        MPI_Send(&buf, 1, MPI_INT, (rank + 1) % size /*dest*/, 0 /*tag*/, MPI_COMM_WORLD);
        MPI_Recv(&buf, 1, MPI_INT, (rank + size - 1) % size /*source*/, 0 /*tag*/, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
    }
    MPI_Finalize();
    return 0;
}
// can be replace by
int main2(int argc, char* argv[]) {
    int rank, size, buf;
    MPI_Init(&argc, &argv); /* starts MPI */
    MPI_Comm_rank(MPI_COMM_WORLD, &rank); /* process id */
    MPI_Comm_size(MPI_COMM_WORLD, &size); /* number processes */
    buf = rank;

    MPI_Sendrecv(&buf, 1, MPI_INT,(rank + 1) % size /*send dest*/, 0 /*tag*/,
                 &buf, 1, MPI_INT,(rank + size - 1) % size /*recv source*/, 0 /*tag*/,
                 MPI_COMM_WORLD, MPI_STATUS_IGNORE);

    MPI_Finalize();
    return 0;
}
```

#### 4.2.3. reuse processes for parallelization

* basic idea of parallelization with MPI: using non-blocking send and receive to overlap processing processes

```c++
#include <mpi.h>
#define SIZE 1024

int main(int argc, char** argv)
{
    // init
    MPI_Init(&argc, &argv);

    // get rank and size
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    MPI_Request send_request = MPI_REQUEST_NULL, receive_request = MPI_REQUEST_NULL;

    // pay attention to the iteration step
    // one round should be an ordered execution of all processes
    // after executing current i, the process should get ready for index i+size
    for(int i = rank; i < SIZE; i += size)
    {
        // first iteration should not have receive
        // receive from last rank
        if(i != 0){
            MPI_Irecv(void* buf, int count, MPI_Datatype type, int source, int tag, MPI_Comm comm, MPI_Request *request);
        }

        // do some works that are irrelevant to the message but heavy

        // complete the receive
        MPI_Wait(&receive_request, MPI_STATUS_IGNORE);

        // do some works that are relevant to the message

        // last iteration should not have send
        // send to the next rank
        if (i != SIZE - 1)
        {
            MPI_Send(void *buf, int count, MPI_Datatype type, int dest, int tag, MPI_Comm comm);
        }
        else
        {
            // output the last block's result
        }
    }

    // finalize
    MPI_Finalize();
}
```

### 4.3. all sorts of collectives

#### 4.3.1. MPI_Gather

```c++
int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    int data[4];
    data[0] = rank; data[1] = 0; data[2] = 0; data[3] = 0;
    
    MPI_Gather(data, 1, MPI_INT, // send
               data, 1, MPI_INT, // recv
               0 /*root*/, MPI_COMM_WORLD);
    MPI_Finalize ();
}
```

#### 4.3.2. MPI_Allgather

```c++
int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    int data[4];
    data[0] = rank; data[1] = 0; data[2] = 0; data[3] = 0;
    MPI_Allgather(data, 1, MPI_INT, data, 1, MPI_INT, MPI_COMM_WORLD);
    MPI_Finalize();
}
```

#### 4.3.3. MPI_Reduce

```c++
int main(int argc, char** argv){
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    int local_data = 1, global_data = 0;
    MPI_Reduce(&local_data /*send buffer*/, 
               &global_data /*recv buffer*/, 1, MPI_INT, 
               MPI_SUM/*op*/, 0/*root*/, MPI_COMM_WORLD);
    MPI_Finalize();
}
```

#### 4.3.4. MPI_Allreduce

```c++
int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    int local_data = 1, global_data = 0;
    MPI_Allreduce(&local_data, &global_data, 1, MPI_INT, MPI_SUM, MPI_COMM_WORLD);
    MPI_Finalize();
}
```

operations:

* MPI_MAX
* MPI_MIN
* MPI_SUM
* MPI_PROD: product
* MPI_LAND: logical and
* MPI_BAND: bit-wise and
* MPI_LOR: logical or
* MPI_BOR: bit-wise or
* MPI_LXOR: logical xor
* MPI_BXOR: bit-wise xor

#### 4.3.5. MPI_Iallreduce

add one more arguments `MPI_Request* req` at the end of `MPI_Allreduce`

```c++
int main2(int argc , char* argv []) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank); MPI_Comm_size(MPI_COMM_WORLD, &size);
    double a[SIZE], b[SIZE],c[SIZE];
    double sum_a = 0, sum_b = 0, sum_c = 0, avg_a = 0, avg_b = 0, avg_c = 0;
    double min_a = MAX_VAL, min_b = MAX_VAL, min_c = MAX_VAL, max_a = -1 , max_b = -1 , max_c = -1;
    initValues(a, b, c); // Fill our input arrays with data

    for (int i = 0; i < SIZE ; ++i) {
        sum_a += a[i]; // partial sums over array "a"
    }
    avg_a = sum_a / SIZE;
    MPI_Allreduce(&avg_a, &avg_a, 1, MPI_DOUBLE, MPI_SUM, MPI_COMM_WORLD);
    avg_a /= size; // aggregate the average over all processes
    for (int i = 0; i < SIZE; ++i) {
        b[i] *= avg_a; // Do some work
    }
    for (int i = 0; i < SIZE; ++i) {
        min_b = MIN(min_b, b[i]); max_b = MAX(max_b, b[i]); // Calculate min , max
    }
    MPI_Request req_min, req_max;
    MPI_Iallreduce(MPI_IN_PLACE, &min_b, 1, MPI_DOUBLE, MPI_MIN, MPI_COMM_WORLD, &req_min);
    MPI_Iallreduce(MPI_IN_PLACE, &max_b, 1, MPI_DOUBLE, MPI_MAX, MPI_COMM_WORLD, &req_max);

    for (int i = 0; i < SIZE; ++i) c[i] += avg_a; // Do work
    MPI_Wait(&req_max, MPI_STATUS_IGNORE); // Wait for needed value
    for (int i = 0; i < SIZE; ++i) c[i] += max_b / 2.0; // Do more work
    MPI_Wait(&req_min, MPI_STATUS_IGNORE); // Wait again
    for (int i = 0; i < SIZE; ++i) c[i] += min_b / 2.0; // Rest of the work
    for (int i = 0; i < SIZE; ++i) sum_c += c[i];

    avg_c = sum_c / SIZE;
    MPI_Allreduce(&avg_c, &avg_c, 1, MPI_DOUBLE, MPI_SUM, MPI_COMM_WORLD);
    avg_c /= size;
    MPI_Finalize(); return 0;
}
```

#### 4.3.6. MPI_Bcast

```c++
#include <mpi.h>

int main(int argc, char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    int data[4];
    if (rank == 0) {  data [0] = 0; data [1] = 1; data [2] = 2; data [3] = 3; }
    else {            data [0] = 0; data [1] = 0; data [2] = 0; data [3] = 0; }
    MPI_Bcast(data, 4, MPI_INT, 0 /*root*/, MPI_COMM_WORLD);
    MPI_Finalize();
}
```

#### 4.3.7. MPI_Scatter

```c++
int main( int argc , char** argv) {
    int rank, size;
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    int data[4];
    if (rank == 0) { data [0] = 0; data [1] = 1; data [2] = 2; data [3] = 3; }
    else { data [0] = 0; data [1] = 0; data [2] = 0; data [3] = 0; }
    MPI_Scatter(data, 1, MPI_INT, // send
                data, 1, MPI_INT, // recv
                0 /*root*/, MPI_COMM_WORLD);
    MPI_Finalize();
}
```

### 4.4. window allocation

#### 4.4.1. allocate and free window

you don't need to allocate memory yourself

```c++
int main(int argc, char** argv) {
    int *a;
    MPI_Win win;
    MPI_Init(&argc, &argv);
    /* collectively create remotely accessible memory in the window */
    MPI_Win_allocate(1000*sizeof(int), sizeof(int), MPI_INFO_NULL, MPI_COMM_WORLD, &a, &win);
    /* Array ‘a’ is now accessible from all processes in MPI_COMM_WORLD */
    MPI_Win_free(&win);
    MPI_Finalize();
    return 0;
}
```

#### 4.4.2. create window

you need to allocate memory yourself

```c++
int main(int argc, char** argv) {
    int *a;
    MPI_Win win;
    MPI_Init(&argc, &argv);
    /* create private memory in every MPI process */
    a = (void *) malloc(1000 * sizeof(int));
    /* use private memory like you normally would */
    a[0] = 1; a[1] = 2;
    /* collectively declare memory as remotely accessible */
    MPI_Win_create(a, 1000*sizeof(int), sizeof(int), MPI_INFO_NULL, MPI_COMM_WORLD, &win);
    /* Array ‘a’ is now accessibly by all processes in MPI_COMM_WORLD */
    MPI_Win_free(&win);
    MPI_Finalize();
    return 0;
}
```

#### 4.4.3. dynamically create

```c++
int main(int argc, char** argv) {
    int *a;
    MPI_Win win;
    MPI_Init(&argc, &argv);
    MPI_Win_create_dynamic(MPI_INFO_NULL, MPI_COMM_WORLD, &win);
    /* create private memory in every MPI process */
    a = (void *) malloc(1000 * sizeof(int));
    
    /* use private memory like you normally would */
    a[0] = 1; a[1] = 2;

    /* locally declare memory as remotely accessible */
    MPI_Win_attach(win, a, 1000*sizeof(int));

    /* Array ‘a’ is now accessible from all processes in MPI_COMM_WORLD*/

    /* undeclare public memory */
    MPI_Win_detach(win, a);
    MPI_Win_free(&win);
    MPI_Finalize();
    return 0;
}
```