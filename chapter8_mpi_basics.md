# Parallel Programming Chapter 8: MPI basics

<!-- TOC -->

- [1. MPI basics](#1-mpi-basics)
    - [1.1. how to use](#11-how-to-use)
        - [1.1.1. Steps](#111-steps)
        - [1.1.2. All the code](#112-all-the-code)
    - [1.2. Basic functions](#12-basic-functions)

<!-- /TOC -->

## 1. MPI basics

### 1.1. how to use

#### 1.1.1. Steps

1. include <mpi.h>

    ```c++
    #include <mpi.h>
    ```

2. initialize and finalize MPI

    ```c++
    MPI_Init(&argc, &argv);
    // some code here
    MPI_Finalize();
    ```

3. get the size and rank

    ```c++
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    ```

4. send and receive

    ```c++
    if (rank > 0)
        MPI_Recv(&value, 1, MPI_DOUBLE, rank-1, 0, MPI_COMM_WORLD, &s);
    if (rank < size - 1)
        MPI_Send(&value, 1, MPI_DOUBLE, rank+1, 0, MPI_COMM_WORLD);
    ```

#### 1.1.2. All the code

```c++
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
int main (int argc, char** argv)
{
    double value;
    int size, rank;
    MPI_Status s;
    MPI_Init (&argc, &argv);
    MPI_Comm_size (MPI_COMM_WORLD, &size);
    MPI_Comm_rank (MPI_COMM_WORLD, &rank);
    value=MPI_Wtime();
    printf("MPI Process %d of %d (value=%f)\n", rank, size, value);
    if (rank>0)
        MPI_Recv(&value, 1, MPI_DOUBLE, rank-1, 0, MPI_COMM_WORLD, &s);
    if (rank<size-1)
        MPI_Send(&value, 1, MPI_DOUBLE, rank+1, 0, MPI_COMM_WORLD);
    if (rank==size-1)
        printf("Value from MPI Process 0: %f\n",value);
    MPI_Finalize ();
}
```

### 1.2. Basic functions

1. `int MPI_Comm_size(MPI_Comm comm, int *size)`

    returns the number of processes in the process group of the given communicator

2. `int MPI_Comm_rank(MPI_Comm comm, int *rank)`

    returns the rank of the executing process relative to the communicator

3. `int MPI_Send(void *buf, int count, MPI_Datatype dtype, int dest, int tag, MPI_Comm comm)`

    * buf: address of the send buffer
    * count: number of data elements of type dtype to be sent
    * dtype: data type to be sent
    * dest: receiver rank
    * tag: message tag
    * comm: communicator

    1. Common MPI data types

        * MPI_SHORT
        * MPI_INT
        * MPI_LONG
        * MPI_LONG_LONG
        * MPI_UNSIGNED_SHORT
        * MPI_UNSIGNED
        * MPI_UNSIGNED_LONG
        * MPI_UNSIGNED_LONG_LONG
        * MPI_FLOAT
        * MPI_DOUBLE
        * MPI_C_COMPLEX
        * MPI_C_DOUBLE_COMPLEX
        * MPI_BYTE
        * MPI_PACKED

4. `int MPI_Recv(void *buf, int count, MPI_Datatype dtype, int source, int tag, MPI_Comm comm, MPI_Status *status)`

    * buf: address of the receive buffer
    * count: number of data elements of type dtype to be received
    * dtype: data type
    * source: sender(must match the sender rank, can be set to `MPI_ANY_SOURCE`)
    * tag message tag(must match the sender tag, can be set to `MPI_ANY_TAG`)
    * comm: communicator
    * status: status information

    1. `MPI_Status`: structure containing information on incoming message, including:

        * MPI_SOURCE: sender of the message
        * MPI_TAG: message tag
        * MPI_ERROR: error code

5. `int MPI_Get_count(const MPI_Status *status, MPI_Datatype dtype, int *count)`

    * status: MPI_Status object returned from the receive call
    * dtype: data type used in receive
    * count: number of data elements actually received

6. `double MPI_Wtime(void)`

    returns wall clock time(seconds) since a reference time stamp in the past

7. `double MPI_Wtick(void)`

    returns resolution of `MPI_Wtime`. Number of seconds between two successive clock ticks

