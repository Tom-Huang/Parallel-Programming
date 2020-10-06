# Parallel programming Chapter 2: pthread

<!-- TOC -->

- [1. Threads](#1-threads)
    - [1.1. What is a thread](#11-what-is-a-thread)
- [2. pthread](#2-pthread)
    - [2.1. how to use pthread](#21-how-to-use-pthread)
        - [2.1.1. create/fork/join thread](#211-createforkjoin-thread)
        - [2.1.2. how to use pthread](#212-how-to-use-pthread)
        - [pthread routines](#pthread-routines)
        - [2.1.3. how to create lock](#213-how-to-create-lock)
    - [2.2. lock granularity](#22-lock-granularity)
        - [2.2.1. Coarse grained locking: one single lock for all data](#221-coarse-grained-locking-one-single-lock-for-all-data)
        - [2.2.2. Fine grained locking: one lock for each data element](#222-fine-grained-locking-one-lock-for-each-data-element)
    - [2.3. how to use condition variables](#23-how-to-use-condition-variables)
        - [2.3.1. some basics](#231-some-basics)
        - [2.3.2. how to use](#232-how-to-use)

<!-- /TOC -->

## 1. Threads

### 1.1. What is a thread

hardware thread:

software thread:

user-level thread:

## 2. pthread

### 2.1. how to use pthread

#### 2.1.1. create/fork/join thread

```c++
// create thread
pthread_t a_thread;
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);

// join thread
int pthread_join(pthread_t thread, void **retval);
```

common thread function

```c++
// get own thread ID:
pthread_self()

// compare two thread ID:
pthread_equal(t1, t2)

// run a particular function once in a process
pthread_once(ctrl, fct)
```

#### 2.1.2. how to use pthread

1. include header file

    ```c++
    #include <pthread.h>
    ```

2. create a kernel function

    ```c++
    void* perform_work(void* argument){
        int real_argument;
        real_argument = *((int*) argument);
        return NULL;
    }
    ```

3. call the kernel function from the main function

    ```c++
    // define thread
    p_thread thread;
    // define thread argument
    int thread_arg;
    // define result code
    int result_code;
    // create thread
    result_code = pthread_create(&thread, NULL, perform_work, &thread_arg);
    ```

#### pthread routines

```c++
// get own thread ID
pthread_self();

// compare two threads IDs
pthread_equal(t1, t2);

// run a particular function once in a process
pthread_once(ctrl, fct);
```

#### 2.1.3. how to create lock

1. define a mutex lock globally and initialize it with `PTHREAD_MUTEX_INITIALIZER` or dynamically

    ```c++
    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    // or init like this
    pthread_mutex_init(&mutex);
    // and destroy like this
    pthread_mutex_destroy(&mutex);
    ```

2. call `pthread_mutext_unlock` and `pthread_mutex_trylock`

    ```c++
    // it will lock all the global variables until lock is freed by other threads
    pthread_mutex_lock(&mutex);
    ```

    ```c++
    // it will try to lock the variables and return immediately
    // usually it is used with while loop
    pthread_mutex_trylock(&mutex);
    ```

3. unlock a mutex

    ```c++
    pthread_mutex_unlock(&mutex);
    ```

### 2.2. lock granularity

#### 2.2.1. Coarse grained locking: one single lock for all data

Pros: eases implementations

Cons: limits concurrency

#### 2.2.2. Fine grained locking: one lock for each data element

Pros: maximizes concurrency

Cons: increases overhead for locking and unlocking

### 2.3. how to use condition variables

#### 2.3.1. some basics

waiting based on the value of a shared variable. One thread needs to wait for a condition to be true. Second thread checks condition and signals other thread when condition is met.

#### 2.3.2. how to use

1. create a global variable of `pthread_cond_t` and initialize it with `PTHREAD_COND_INITIALIZER`

    ```c++
    pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
    // or dynamically
    pthread_cond_init(&cond);
    // and destroy like this
    pthread_cond_destroy(&cond);
    ```

2. create a global mutex lock

    ```c++
    pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    ```

3. one thread waits until the condition to be set in another thread

    mutex must be locked before calling

    ```c++
    pthread_mutex_lock(&mutex);
    while (some condition){
        pthread_cond_wait(&condition, &mutex);
    }
    pthread_mutex_unlock(&mutex);
    ```

    `pthread_cond_wait(&cond, &mutex)` releases the mutex lock before it sleeps. But when it receives the signal, it will try to lock the mutex and then end this expression.

4. another thread signals a condition to a waiting thread

    ```c++
    if (some other condition){
        // this will unlock only a thread that is waiting for cond
        pthread_cond_signal(&cond);
        // or send to all thread waiting for cond to make them wake up
        pthread_cond_broadcast(&cond);
    }
    ```

5. an example

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

    timeline:

    1. thread 1 locks the mutex.
    2. thread 1 check if the condition is satisfied and enter the while loop.
    3. thread 1 runs `pthread_cond_wait()` and finds that the condition signal is not received. So it unlock the mutex and starts sleeping.
    4. thread 2 tries to lock the mutex all the time and doesn't lock until step 3 is finished. 
    5. thread 2 do something that can make the condition to be true.
    6. thread 2 makes the condition become true and call `pthread_cond_signal()`.
    7. thread 1 receives the signal and tries to lock the mutex. But because at this point, thread 1 still locks the mutex, thread 2 waits for the lock.
    8. thread 2 unlocks the mutex.
    9. thread 1 successfully locks the mutex and end the expression of `pthread_cond_wait()`. Thread 1 runs the while loop check and fails, so it escapes from the loop.
    10. thread 1 runs the code that needs the condition to be true.
    11. thread 1 unlocks the mutex.

