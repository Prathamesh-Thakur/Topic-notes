# Numba with CUDA



### Notebook 1:

##### CPU:

* numba.jit is for **CPU** operations.
* For strict type checking, use jit(nopython = True) or njit.



##### GPU:

* vectorize() decorator can be used to create optimized universal functions.
* For gpu usage, types are necessary. Example: vectorize(\['int64(int64, int64)'], target = 'cuda' or device). The example indicates

that the function takes two int64 parameters and returns an int64 parameter.



Some mathematical functions from **numpy** do not work with GPU numba, and requires the **math** library in Python.



##### CUDA Device functions:

For when ufuncs are not being used, or rather they fall short.

Use numba.cuda.jit

The device = True parameter to jit indicates that the function can only be called from a function running on the GPU.



##### Reduce memory overhead / data transfer

* Transfer the CPU arrays to GPU using .to\_device(). This is used from numba's cuda api and is different from regular python's .to()

method, even though both achieve the same task.



* Create a device array using cuda.device\_array(). Then we can use the out parameter to the function, and use copy\_to\_host method

to transfer that device array back to the CPU.

### 

### Notebook 2:

##### CUDA Thread Hierarchy

###### Terms and definitions:

* Kernels: GPU functions.
* Thread: Smallest unit in the GPU. Parallelization is performed on threads.
* Block: Collection of threads. There can be multiple blocks.
* Grid: Collection of blocks associated with a given kernel launch.



Kernels are launched with execution configuration. It defines number of blocks in the grid as well as

the number of threads in a block. **Every block in the grid has same number of threads.**



###### CUDA provided variables:

* gridDim.x: Number of blocks in the grid
* blockIdx.x: Index of the current block within the grid
* blockDim.x: Number of threads within a single block
* threadIdx.x: Index of thread within the **current** block



###### Ways to parallelize:

Somehow we need to map each unique thread to every variable in the dataset. However, each thread has index

only in it's own block and no index throughout the grid is maintained. However, that can be calculated.



Each thread's unique index wrt grid = threadIdx.x + blockDim.x \* blockIdx.x



**cuda.grid()** is the function which provides the same.

You pass number of dimensions of the grid within the grid function.



###### Functioning of numba.cuda.jit:

When we consider SIMT (Single Instruction Multiple Thread) architecture, each thread receives a copy of the function decorated by

jit. That is why we can use cuda.grid, which returns a scalar value.



Streaming Multiprocessors contain all required resources for the execution of kernel code including many CUDA cores. When a kernel is

launched, each **block** is assigned to a single SM, with potentially many blocks assigned to a single SM. **SMs partition blocks** into further

subdivisions of **32 threads called warps** and it is these warps which are given **parallel instructions to execute**.



It is essential to provide SMs with lot of work to meaningfully utilize the GPU, so that the latency is hidden while switching warps.

This can also be achieved by giving large grid and block dimensions.



###### Grid Stride Loops:

We cannot always have the number of threads equal to the number of data elements. But if we assign fewer threads, work can be left undone.

This is where grid stride loops come in.



* The thread's first element is calculated as usual, using cuda.grid().
* The thread then strides forward by blockDim.x \* gridDim.x steps and continues to do so until it reaches end of data.

Use **cuda.gridsize()** for the above functionality, which returns total number of threads in the grid.

* With all threads working in parallel, all elements are covered.



Plus, with **memory coalescing**, the number of read and writes are reduced as well since the threads will be accessing adjacent elements.



###### Race condition:

Race conditions happen in two scenarios:

* read-after-write hazards: One thread is reading a memory location at the same time another thread might be writing to it.
* write-after-write hazards: Two threads are writing to the same memory location, and only one write will be visible when the kernel is complete.



To resolve, we must either ensure that each thread has responsibility for unique subsets of the dataset, or use **atomic operations.**



###### Atomic operations:

Atomic operations read, modify and update a specific memory location in one single, indivisible step.

Numba has several built in atomic operations.



Example: cuda.atomic.add()





### Notebook 3:

###### Coalesced memory access:

Warp is made of 32 threads, and instructions are executed in parallel at **warp** level.



A line of contiguous memory is basically a segment/block of code that is stored in a contiguous manner.

When contiguous elements are accessed, we say that the memory access is fully coalesced.



Increments to threadIdx.x should map to increments in data in the direction of the **fastest changing index**.



**Fastest changing index:** Computer stores all objects after flattening them. The dimensionality is lost. By default, Python

stores it in row major, meaning if a 2D grid is stored, each row is stored side by side. Thus, x is the fastest changing index

because you don't have to jump the entire row length to get to the next element (which is what would have happened in case of

iterating over column).



###### Memory hierarchy:

Global memory: Can be used by any and all threads across blocks, and persists till the lifetime of the application. (used until now)



Shared memory: User defined **cache** that is shared among **all** threads **in one** block only.



**Shared memory:**

* Can be defined using cuda.array.shared()
* Explicit numba type must be passed
* Shape of the array has to be a constant and cannot be a variable name





###### Shared memory for coalesced access:

View slides of matrix transpose.



###### Bank conflicts in shared memory:

* While logical view of shared memory is continuous, physical memory stores it in columnar "banks".
* Serial access of elements in the same bank leads to bank conflicts, which can slow down the code.
* A way to avoid bank conflicts is to "pad" the shared memory, so that accessing each element naturally becomes accessing different banks.
* Note that padding needs to be added along the fastest changing index.


## Understanding indexing in CUDA, OpenCV and Numpy/Pytorch
* Numpy arrays: (row, column)
Rows are stacked vertically, and columns are packed horizontally. In **Cartesian** terms, since x moves horizontally and y moves vertically, **(row, column)** map to **(y, x)**.

* OpenCV images: (width, height)
If we consider **Cartesian** again, x moves horizontally, y moves vertically. Width is horizontal, and height is vertical, so **(width, height)** maps to **(x, y)**. However, this is when using the **API** or predefined functions.

For memory storage, it acts **same** as numpy arrays, so index according to that for **pixel** level manipulation.

* CUDA Numba GPU: (x, y)
Works in **Cartesian** space; x moves horizontally, y moves vertically.

### Inter mapping
* Numpy to CUDA:
CUDA works in **Cartesian (x, y)**, which maps to **(horizontal, vertical)**. However, numpy works in **(row, column)** which, in Cartesian terms, maps to **(vertical, horizontal)**. So **(row, column) = (y, x)**, if we equate through Cartesian.

* Similar mapping will be for numpy to OpenCV images.

* CUDA and OpenCV images have **same** notation inherently and through Cartesian.