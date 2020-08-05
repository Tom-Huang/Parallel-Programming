# Parallel Programming Chapter 9: MPI advances

<!-- TOC -->

- [1. Advanced MPI](#1-advanced-mpi)
    - [1.1. Send variants](#11-send-variants)
    - [1.2. probe message without receiving](#12-probe-message-without-receiving)
    - [1.3. Bidirectional exchange of data](#13-bidirectional-exchange-of-data)
    - [1.4. Nonblocking operations](#14-nonblocking-operations)
        - [1.4.1. Send operations](#141-send-operations)
        - [1.4.2. Completion operations](#142-completion-operations)
        - [1.4.3. Variants of completion](#143-variants-of-completion)
        - [1.4.4. Bidirectional exchange](#144-bidirectional-exchange)
        - [1.4.5. Persistent communication](#145-persistent-communication)
- [2. Collective operations](#2-collective-operations)
    - [2.1. Synchronization](#21-synchronization)
    - [2.2. Communication](#22-communication)
        - [2.2.1. Broadcast](#221-broadcast)
        - [2.2.2. Gather](#222-gather)
        - [2.2.3. Scatter](#223-scatter)
        - [2.2.4. Variations](#224-variations)
    - [2.3. Reductions](#23-reductions)
    - [2.4. Nonblocking collectives](#24-nonblocking-collectives)
- [3. Communicators](#3-communicators)
    - [3.1. Creating new communicators](#31-creating-new-communicators)

<!-- /TOC -->

## 1. Advanced MPI

### 1.1. Send variants

1. `MPI_Send`

    * standard send operation

2. `MPI_Bsend`

    * buffered send
    * force the use of a send buffer
    * returns immediately, but costs resources

3. `MPI_Ssend`

    * synchronous send
    * only returns once the receive has started
    * adds extra synchronization, but can be costly

4. `MPI_Rsend`

    * ready send
    * user must ensure that receive has been posted
    * enables faster communication, but needs implicit synchronization

### 1.2. probe message without receiving

1. `MPI_Probe`

    * blocks until a matching message was found
    * input: source, tag, communicator

2. `MPI_Iprobe`

    enables to overlap wait time for receive with useful work

    * terminates after checking whether a matching messages is available
    * input: source, tag, communicator
    * output: flag and status object

### 1.3. Bidirectional exchange of data

1. `MPI_Sendrecv(void *sendbuf, int sendcount, MPI_Datatype sendtype, int dest, int sendtag, void *recvbuf, int recvcount, MPI_Datatype recvtype, int source, int recvtag, MPI_Comm comm, MPI_Status *status)`

    * used to send and receive similar data bidirectionally
    * can be matched with regular send/recv calls
    * can be matched with other Sendrecv calls with other destinations

2. `int MPI_Sendrecv_replace(void *buf, int count, MPI_Datatype datatype, int dest, int sendtag, int source, int recvtag, MPI_Comm comm, MPI_Status *status)`

### 1.4. Nonblocking operations

split operations into start and completion

* starting an operation finishes immediately
* completion
  * can be tested for
  * can be waited for
* can complete other work in between
* can start multiple operations

#### 1.4.1. Send operations

1. `MPI_Isend(void* buf, int count, MPI_Datatype dtype, int dest, int tag, MPI_Comm comm, MPI_Request* request)`

    * buf: send buffer
    * count: number of elements
    * dtype: type of data
    * dest: destination rank
    * tag: tag
    * comm: communicator
    * request: Reference to request

2. `MPI_Irecv(void* buf, int count, MPI_Datatype dtype, int source, int tag, MPI_Comm comm, MPI_Request* request)`

    * buf: send buffer
    * count: number of elements
    * dtype: type of data
    * source: source rank
    * tag: tag
    * comm: communicator
    * request: Reference to request

3. Non-blocking send variants

    1. `MPI_Isend`: standard
    2. `MPI_Ibsend`: buffered
    3. `MPI_Issend`: synchronous
    4. `MPI_Irsend`: ready

#### 1.4.2. Completion operations

1. `int MPI_Wait(MPI_Request *request, MPI_Status *status)`

    ```c++
    MPI_Request req;
    MPI_Status status;
    int msg[10];
    //...
    MPI_Irecv(msg, 10, MPI_INT, MPI_ANY_SOURCE, 42, MPI_COMM_WORLD, &req);
    // do some work
    MPI_Wait(&req, &status);
    ```

2. `int MPI_Test(MPI_Request *request, int *flag, MPI_Status *status)`

    * flag: output flag, whether complete or not

    ```c++
    MPI_Request req;
    MPI_Status status;
    int msg[10], flag;
    MPI_Irecv(msg, 10, MPI_INT, MPI_ANY_SOURCE, 42, MPI_COMM_WORLD, &req);
    do
    {
        ...
        MPI_Test(&req, &flag,  &status);
    } while (flag == 0)
    ```

#### 1.4.3. Variants of completion

1. `MPI_Wait`
2. `MPI_Waitall`: returns once all are complete
3. `MPI_Waitany` or `MPI_Waitsome`: returns if any is complete
4. `MPI_Test`
5. `MPI_Testall`: true if all are complete
6. `MPI_Testany` or `MPI_Testsome`: true if any is complete

#### 1.4.4. Bidirectional exchange

```c++
MPI_Request req[2];
MPI_Status status[2];
int msg_in[10], msg_out[10];

MPI_Isend(msg_out, 10, MPI_INT, 1, 42, MPI_COMM_WORLD, &(req[0]));
MPI_Irecv(msg_in, 10, MPI_INT, 1, 42, MPI_COMM_WORLD, &(req[1]));

MPI_Waitall(2, req, status);
```

#### 1.4.5. Persistent communication

```c++
MPI_Request req;
MPI_Status status;
int msg[10];

MPI_Send_start(msg, 10, MPI_INT, 1, 42, MPI_COMM_WORLD, &req):
MPI_Start(&req);
MPI_Wait(&req, status);
MPI_Request_free(&req);
```

## 2. Collective operations

Three classes of collective operations:

* Synchronization
  * Barrier
* Communication
  * Broadcast
  * Gather
  * Scatter
* Reduction
  * global value retuened to one or all processors
  * combination with subsequent scatter
  * parallel prefix operations

### 2.1. Synchronization

1. `MPI_Barrier(MPI_Comm comm)`

### 2.2. Communication

#### 2.2.1. Broadcast

the content of the sent buffer is copied to all other MPI processes

* no tag
* send and receive are the same

1. `int MPI_Bcast(void *buf, int count, MPI_Datatype dtype, int root, MPI_Comm comm)`

    * buf: address of send/receive buffer
    * count: number of elements
    * dtype: data type
    * root: rank of send/broadcasting MPI process
    * comm: communicator

#### 2.2.2. Gather

each process send one message to one process, one process collects all messages sent to it.

1. `int MPI_Gather(void *sendbuf, int sendcount, MPI_Datatype sendtype, void *recvbuf, int recvcount, MPI_Datatype recvtype, int root, MPI_Comm comm)`

    * sendbuf: send buffer
    * sendcount: number of elements sent
    * sendtype: data type
    * recvbuf: receive buffer
    * recvcount: number of elements received
    * recvtype: data type
    * root: rank of receiving MPI process
    * comm: communicator

#### 2.2.3. Scatter

one process distribut a series of data to all proceses. Process k receives sendcount elements starting with `sendbuf + k * sendcount`

1. `int MPI_Scatter(void *sendbuf, int sendcount, MPI_Datatype sendtype, void *recvbuf, int recvcount, MPI_Datatype recvtype, int root, MPI_Comm comm)`

#### 2.2.4. Variations

1. `MPI_Gatherv(void *sendbuf, int sendcount, MPI_Datatype sendtype, void *recvbuf, int *recvcount, int *displs, MPI_Datatype recvtype, int root, MPI_Comm comm)`

    gather operation with different number of data elements per process
2. `MPI_Allgather`: gather operation with broadcast to all processes
3. `MPI_Allgatherv`: gather operation with different number of data elements AND broadcast
4. `MPI_Scatterv`: scatter operation with different number of data elements per process
5. `MPI_Alltoall(void* sendbuf, int sendcount, MPI_Datatype sendtype, void* recvbuf, int recvcount, MPI_Datatype recvtype, MPI_Comm comm)`
    every process sends to every other process (different data)
6. `MPI_Alltoallv`: all to all with different numbers of data elements per process
7. `MIP_Alltoallw`: all to all with different numbers of data elements and types per process

### 2.3. Reductions

1. `MPI_Reduce(void *sbuf, void *rbuf, int count, MPI_Datatype dtype, MPI_Op op, int root, MPI_Comm comm)`

    combines the elements in the send buffer according to operation and delivers the result to root

    1. possible `MPI_Op`

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

2. `MPI_Allreduce`

    reduction followed by broadcast

3. `MPI_Reducescatter_block`

    reduction followed by a scatter operation (equal chunks)

4. `MPI_Reducescatter`

    reduction followed by a scatter operation (varying chunks)

5. `MPI_Scan`

    reduction using a prefix operation

### 2.4. Nonblocking collectives

same concept as non-blocking P2P

## 3. Communicators

* rank is relative to a communicator
* communication across communicators are not possible
* default communicators:
  * `MPI_COMM_WORLD`: all initial MPI processes
  * `MPI_COMM_SELF`: contains only the own MPI process

### 3.1. Creating new communicators

1. `int MPI_Comm_dup(MPI_Comm comm, MPI_Comm *newcomm)`

    copy communicator from comm to newcomm

    * comm: communicator to be copied
    * newcomm: new communicator copy

    * also exists as `MPI_Comm_Idup`

2. `int MPI_Comm_split(MPI_Comm comm, int color, int key, MPI_Comm *newcomm)`

    creates new communicators. All processes that pass the same color will be in same new environment. Key argument determines the rank order in the new communicator.