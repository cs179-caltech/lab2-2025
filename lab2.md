# CS 179
## Assignment 2

Due: Wednesday, April 16, 2025 - 3:00 PM

Put all answers in a file called `README.txt`. After answering all of the
questions, list how long each part took. Feel free to leave any other
feedback.

## Submission Instructions
Put a zip file in your home directory on the lab machine, in the format:

`lab2_2025_submission.zip`

Your submission should be a single archive file (`.zip`) with your README file and all code.

## PART 1

### Question 1.1: Throughput (5 points)
How many 32-bit floating point fused multiply-adds (FMAs) can an RTX A5000 GPU perform per second?

Use these sources:
- [CUDA Docs - Instruction Throughput](https://docs.nvidia.com/cuda/cuda-c-programming-guide/#arithmetic-instructions-throughput-native-arithmetic-instructions)
- [RTX A5000 Specs](https://www.techpowerup.com/gpu-specs/rtx-a5000.c3748)

### Question 1.2: Thread Divergence (6 points)
Let the block shape be (32, 32, 1).

(a)
```cuda
int idx = threadIdx.y + blockSize.y * threadIdx.x;
if (idx % 32 < 16)
    foo();
else
    bar();
```

Does this code diverge? Why or why not?

(b)
```cuda
const float pi = 3.14;
float result = 1.0;
for (int i = 0; i < threadIdx.x; i++)
    result *= pi;
```

Does this code diverge? Why or why not? (This is a bit of a trick question,
either "yes" or "no can be a correct answer with appropriate explanation.)

### Question 1.3: Coalesced Memory Access (9 points)
Let the block shape be (32, 32, 1). Let data be a `(float *)` pointing to global
memory and let data be 128 byte aligned (so `data % 128 == 0`).

Consider each of the following access patterns.

(a)
```cuda
data[threadIdx.x + blockSize.x * threadIdx.y] = 1.0;
```

Is this write coalesced? How many 128 byte cache line requests are made (per warp and per block)?

(b)
```cuda
data[threadIdx.y + blockSize.y * threadIdx.x] = 1.0;
```

Is this write coalesced? How many 128 byte cache line requests are made (per warp and per block)?

(c)
```cuda
data[1 + threadIdx.x + blockSize.x * threadIdx.y] = 1.0;
```

Is this write coalesced? How many 128 byte cache line requests are made (per warp and per block)?

### Question 1.4: Bank Conflicts and Instruction Dependencies (15 points)
Let's consider multiplying a 32 x 128 matrix with a 128 x 32 element matrix.
This outputs a 32 x 32 matrix. We'll use 32 ** 2 = 1024 threads and each thread
will compute 1 output element. Although its not optimal, for the sake of
simplicity let's use a single block, so grid shape = (1, 1, 1),
block shape = (32, 32, 1).

For the sake of this problem, let's assume both the left and right matrices have
already been stored in shared memory are in column major format. This means the
element in the ith row and jth column is accessible at `lhs[i + 32 * j]` for the
left hand side and `rhs[i + 128 * j]` for the right hand side.

This kernel will write to a variable called `output` stored in shared memory.

Consider the following kernel code:

```cuda
int i = threadIdx.x;
int j = threadIdx.y;
for (int k = 0; k < 128; k += 2) {
    output[i + 32 * j] += lhs[i + 32 * k] * rhs[k + 128 * j];
    output[i + 32 * j] += lhs[i + 32 * (k + 1)] * rhs[(k + 1) + 128 * j];
}
```

(a)
Are there bank conflicts in this code? If so, how many ways is the bank conflict
(2-way, 4-way, etc)?

(b)
Expand the inner part of the loop (below)

```cuda
output[i + 32 * j] += lhs[i + 32 * k] * rhs[k + 128 * j];
output[i + 32 * j] += lhs[i + 32 * (k + 1)] * rhs[(k + 1) + 128 * j];
```

into "psuedo-assembly" as was done in the coordinate addition example in lecture 4.

There's no need to expand the indexing math, only to expand the loads, stores,
and math. Notably, the operation `a += b * c` can be computed by a single
instruction called a fused multiply add (FMA), so this can be a single
instruction in your "psuedo-assembly".

Hint: Each line should expand to 5 instructions.

(c)
Identify pairs of dependent instructions in your answer to part b.

(d)
Rewrite the code given at the beginning of this problem to minimize instruction
dependencies. You can add or delete instructions (deleting an instruction is a
valid way to get rid of a dependency!) but each iteration of the loop must still
process 2 values of k.

(e)
Can you think of any other anything else you can do that might make this code
run faster?

## PART 2 - Matrix transpose optimization (50 points)
Optimize the CUDA matrix transpose implementations in `transpose_cuda.cu`. Read
ALL of the TODO comments. Matrix transpose is a common exercise in GPU
optimization, so do not search for existing GPU matrix transpose code on the
internet.

Your transpose code only need to be able to transpose square matrices where the
side length is a multiple of 64.

The initial implementation has each block of 1024 threads handle a 64x64 block
of the matrix, but you can change anything about the kernel if it helps obtain
better performance.

The main method of `transpose.cc` already checks for correctness for all transpose
results, so there should be an assertion failure if your kernel produces incorrect
output.

The purpose of the `shmemTransposeKernel` is to demonstrate proper usage of global
and shared memory. The `optimalTransposeKernel` should be built on top of
`shmemTransposeKernel` and should incorporate any "tricks" such as ILP, loop
unrolling, vectorized IO, etc that have been discussed in class.

The transpose program takes 2 optional arguments: input size and method. Input
size must be one of -1, 512, 1024, 2048, 4096, and method must be one all,
cpu, gpu_memcpy, naive, shmem, optimal. Input size is the first argument and
defaults to -1. Method is the second argument and defaults to all. You can pass
input size without passing method, but you cannot pass method without passing an
input size.

Examples:
```bash
./transpose
./transpose 512
./transpose 4096 naive
./transpose -1 optimal
```

Copy paste the output of `./transpose` into `README.txt` once you are done.
Describe the strategies used for performance in either block comments over the
kernel (as done for `naiveTransposeKernel`) or in `README.txt`.

## PART 3: Profiling (15 points)
Profiling is one of the most important parts of writing CUDA code,
because it allows you to measure what is slowing down your code.
To profile the transpose program:
1. `ncu --export profile.ncu-rep --force-overwrite --set full ./transpose 4096 all`
    - See [NSight Compute Docs](https://docs.nvidia.com/nsight-compute/NsightComputeCli/index.html) for more information
2. [Download NSight Compute GUI](https://developer.nvidia.com/tools-overview/nsight-compute/get-started#latest) on your local computer
3. Download the ncu-rep file from titan to your local computer, for example: `rsync -P titan:lab2-2025-main/build/profile.ncu-rep .`
4. Open NSight Compute GUI, and click File -> Open File -> Select the ncu-rep file
5. Notice the 3 different kernels under the "Summary" tab. Select a kernel and switch to the "Details" tab.

To submit with your assignment:
1. Include screenshots of the Memory Chart under "Memory Workload Analysis" for all 3 kernels. Switch the values to use "Throughput" instead of transfer size.
2. How much global (device) memory bandwidth (GB/s) does each of your kernels use? Add read+write bandwidth together.
    - What percentage of theoretical global memory bandwidth does each kernel achieve? Switch to "Memory Tables" and check "% Peak".
3. Look at the "Source" tab for `shmemTransposeKernel` and view the SASS code. SASS is the low level assembly code that the GPU executes directly.
    - What is one optimization that the compiler performed, which you can notice when you compare your source code to the SASS?

## BONUS (+5 points, maximum set score is 100 even with bonus)
Mathematical scripting environments such as Matlab or Python + Numpy often
encourage expressing algorithms in terms of vector operations because they offer
a convenient and performant interface. For instance, one can add 2 n-component
vectors (a and b) in Numpy with `c = a + b`.

This is often implemented with something like the following code:

```c
void vec_add(float *left, float *right, float *out, int size) {
    for (int i = 0; i < size; i++)
        out[i] = left[i] + right[i];
}
```

Consider the code

```python
a = x + y + z
```

where x, y, z are n-component vectors.

One way this could be computed would be

```c
vec_add(x, y, a, n);
vec_add(a, z, a, n);
```

In what ways is this code (2 calls to vec_add) worse than the following?

```c
for (int i = 0; i < n; i++)
    a[i] = x[i] + y[i] + z[i];
```

List at least 2 ways (you don't need more than a sentence or two for each way).

## Further Readings
There is an advanced technique called "swizzling" to avoid bank conflicts without using padding.
CUDA matrix multiply implementation often use this technique, see [Lei Mao - CuTe Swizzle](https://leimao.github.io/blog/CuTe-Swizzle/).
