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
    - [2.2. typical concepts](#22-typical-concepts)
        - [2.2.1. nested](#221-nested)
        - [2.2.2. sections and task](#222-sections-and-task)
        - [2.2.3. single and critical and master](#223-single-and-critical-and-master)
        - [2.2.4. ordered](#224-ordered)
        - [2.2.5. reduction](#225-reduction)
        - [2.2.6. runtime routines](#226-runtime-routines)

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

### 2.2. typical concepts

#### 2.2.1. nested

* two levels nested parallel for loop
* set number of threads in pragma

```c++
omp_set_nested(2);
#pragma omp parallel for num_threads(6)
```

#### 2.2.2. sections and task

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

#### 2.2.3. single and critical and master

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

#### 2.2.4. ordered

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

#### 2.2.5. reduction

```c++
#pragma omp parallel for reduction(+:var)
for(int i = 0; i < 100; i++)
{

}
```

#### 2.2.6. runtime routines

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