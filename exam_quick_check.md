# Exam quick check: template of code

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
pthread_cond_t condition;
pthread_mutex_t condition_mutex;

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