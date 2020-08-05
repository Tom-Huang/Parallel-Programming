# Parallel Programming Chapter 11: Scaling

<!-- TOC -->

- [1. Scaling](#1-scaling)
    - [1.1. Weak scaling](#11-weak-scaling)
    - [1.2. Strong scaling](#12-strong-scaling)
    - [1.3. Impace on programmability](#13-impace-on-programmability)
        - [1.3.1. Debugging at scale](#131-debugging-at-scale)
        - [1.3.2. Performance analysis](#132-performance-analysis)
- [2. Mapping codes to machine](#2-mapping-codes-to-machine)
    - [2.1. MPI interfaces](#21-mpi-interfaces)
        - [2.1.1. Grid topologies](#211-grid-topologies)
        - [2.1.2. Graph topologies](#212-graph-topologies)

<!-- /TOC -->

## 1. Scaling

### 1.1. Weak scaling

keeping the problem size per HW thread/core/node constant. Larger machine -> larger problem. Most common type of scaling

* advantages:
    * execution properties ofter stay fixed
    * easier to scale, as overheads stay roughly constant as well

* challenges:
    * keeping load constant can be tricky for complex applications
    * multiple ways to repartition a workload

### 1.2. Strong scaling

keeping the total problem constant. Larger machine -> faster execution

* need to adjust problem distribution

* challenges:
    * harder to scale, as overheads grow with smaller per HW thread workloads
    * changing executing characteristics

### 1.3. Impace on programmability

#### 1.3.1. Debugging at scale

* tradictional interactive debugging does not work (too much information to present to user)
* first need to reduce the search space to manageable subset
* use stack traces tool: STAT

#### 1.3.2. Performance analysis

* tools: mpiP
* performance analysis is becoming statistics, needs multiple runs

## 2. Mapping codes to machine

### 2.1. MPI interfaces

#### 2.1.1. Grid topologies

1. creates a new communicator with cartesian topology attached

```c++
int MPI_Cart_create(MPI_Comm comm_old, int ndims,
                    int dims[], int periods[], int reorder,
                    MPI_Comm *comm_cart)
```

* IN comm_old : Starting communicator
* IN ndims: number of dimensions
* IN dims: array with lengths in each dimension
* IN periods: logical array specifying whether the grid is periodic
* IN reorder: Reorder ranks allowed?
* OUT comm_cart: resulting 

2. get a rank from a tuple of coordinates

```c++
int MPI_Cart_rank(MPI_Comm comm, int coords[], int *rank)
```

3. get a tuple of coordinates from a rank

```c++
int MPI_Cart_coords(MPI_Comm comm, int rank, int maxdims, int coords[])
```

4. get a send/receive pair for use with MPI_Sendrecv

```c++
int MPI_Cart_shift(MPI_Comm comm, int direction, int disp, int *rank_source, int *rank_dest)
```

#### 2.1.2. Graph topologies