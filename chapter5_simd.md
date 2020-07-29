# Parallel Programming Chapter 5: SIMD

## 1. SIMD intrinsics

### 1.1. Basics

#### 1.1.1. Possible functionalities

1. load and stores

2. simple arithmetic instructions

3. fused instructions

4. conditional evaluation

5. broadcast

6. shuffles, swizzles, blends

7. inter-lane permutes

#### 1.1.2. Funtion names interpretation

`_mm256_add_pd` can be decommposed into three sections:

* `_mm256` means the vector length, could be: `_mm512`, `_mm256`, `_mm128`
* `add` means the operation, could be: add, mul, sub, load...
* `pd` means packed double, could be:
  * `ps`: packed single
  * `sd`: scalar double
  * `ss`: scalar single

#### 1.1.3. Usual routines of using intrinsics

```c++
#include <mmintrin.h>
void saxpy_simd(float* y, float* x, float a, int n)
{
    int ub = n - (n % 8);
    __m256 vy, vx, va, tmp;
    va = _mm256_set1_ps(a);
    for (int i = 0; i < ub; i+=8)
    {
        vy = _mm256_loadu_ps(&y[i]);
        vx = _mm256_loadu_ps(&x[i]);
        tmp = _mm256_mul_ps(va, vx);
        vy = _mm256_add_ps(tmp, vy);
        _mm256_storeu_ps(&y[i], vy);
    }

    __mmask8 m;
    m = (1 << (minint(ub+8, n) - ub)) - 1;
    vy = _mm256_mask_loadu_ps(vy, m, &y[ub]);
    vx = _mm256_mask_loadu_ps(vx, m, &x[ub]);
    tmp = _mm256_mask_mul_ps(va, m, va, vx);
    vy = _mm256_mask_add_ps(vy, m, tmp, vy);
    _mm256_mask_storeu_ps(&y[ub], m, vy);
}
```

1. include the header for the intrinsics

    ```c++
    #include <mmintrin.h>
    ```

2. create intrinsic vector for intrinsics instructions

    ```c++
    __m256 vy, vx, va, tmp;
    ```

3. if one vector can store k elements(could be float, int, double...), then compute the maximal multiple of k smaller than n.

    ```c++
    // k is 8 here
    int ub = n - (n % 8);
    ```

4. in the iteration, the step size should be k.

    ```c++
    // i+=8 for each step
    for (int i = 0; i < ub; i+=8)
    ```

5. load the array from the main memory into intrinsic vector

    ```c++
    vy = _mm256_loadu_ps(&y[i]);
    vx = _mm256_loadu_ps(&x[i]);
    ```

6. call intrinsics operations on the intrinsics vector

    ```c++
    tmp = _mm256_mul_ps(va, vx);
    vy = _mm256_add_ps(tmp, vy);
    ```

7. store the intrinsics vector result to main memory

    ```c++
    _mm256_mask_storeu_ps(&y[ub], m, vy);
    ```

8. because we only calculate the maximal multiples of k below n number of elements in the array, we still need to calculate the remaining n - ub number of elements. We use mask here. Create a mask with size k and assign the former n - ub masks to true.

    ```c++
    __mmask8 m;
    m = (1 << (minint(ub+8, n) - ub)) - 1;
    ```

9. repeat similar operations from 5 to 7, only that we use mask version of those operations

    ```c++
    vy = _mm256_mask_loadu_ps(vy, m, &y[ub]);
    vx = _mm256_mask_loadu_ps(vx, m, &x[ub]);
    tmp = _mm256_mask_mul_ps(va, m, va, vx);
    vy = _mm256_mask_add_ps(vy, m, tmp, vy);
    _mm256_mask_storeu_ps(&y[ub], m, vy);
    ```

## SIMD for OpenMP

### OpenMP SIMD 

#### Only vectorize loop without parallelization

* no parallelization of the loop body
* cut loop into chunks that fit a SIMD vector register

```c++
#pragma omp simd [clause]
// for-loops
{

}
```

* clause options:

    1. private(var-list)

        uninitialized vectors for variables in var-list

    2. firstprivate(var-list)

        initialized vectors for variables in var-list

    3. reduction(op:var-list)

        create private variables for  var-list and apply reduction operator op at the end of the construct

    4. safelen (length)

        maximum number of iterations that can run concurrently without breaking a dependence. In practice, maximum vector length.

    5. linear (list[:linear-step])

        the variable's value is in relationship with the iteration number
    
    6. aligned (list[:alignment])

        specifies that the list items have a giver alignment
    
    7. collapse (n)

#### Vectorize and parallelize loop

* subdivide chunks as `#pragma omp parallel for` does
* for each chunk, vectorize the chunk of vector

```c++
#pragma omp for simd reduction(+:sum)
```

#### Vectorize customized function

```c++
#pragma omp declare simd [clause]
```

example:

```c++
#pragma omp declare simd
float min(float a, float b)
{
    return a < b ? a : b;
}

// will be convert to
_ZGVZN16vv_min(%zmm0, %zmm1):
    vminps %zmm1, %zmm0, %zmm0
    ret
```

* clauses:

    1. simdlen (length)

        generate function to support a given vector length

    2. uniform(argument-list)

        argument has a constant value between the iterations of given loop

    3. inbranch

        optimize for function always called from inside an if statement

    4. notinbranch

        function never called from insidde an if statement
    
    5. linear (argument-list[:linear-step])
    6. alligned (argument-list[:alignment])

## SIMD considerations

### Data layout

#### Array-of-Structs(AoS)

```text
x | y | z | x | y | z
x | y | z | x | y | z
x | y | z | x | y | z
```

each time read one line

* Pros: good locality of {x, y, z}.
* Cons: potential for gather & scatter operations.

#### Struct-af-Arrays(SoA)

```text
x | x | x | x | x | x |
y | y | y | y | y | y |
z | z | z | z | z | z |
```

each time read one line

* Pros: Contiguous load/store.
* Cons: Poor locality of {x, y, z}.

#### Hybrid(AoSoA)

```text
x | x | y | y | z | z |
x | x | y | y | z | z |
x | x | y | y | z | z |
```

each time read one line

* Pros: contiguous load/store.
* Cons: not a normal layout.

### Data alignment

