# cuda-swift

This project provides a native Swift interface to CUDA with the following
modules:

- [x] CUDA Driver API `import CUDADriver`
- [x] CUDA Runtime API `import CUDARuntime`
- [x] NVRTC - CUDA Runtime Compiler `import NVRTC`
- [x] cuBLAS - CUDA Basic Linear Algebra Subprograms `import CuBLAS`
- [x] Warp - GPU Acceleration Library `import Warp` ([Thrust](https://github.com/thrust/thrust) counterpart)

Any machine with CUDA 7.0+ and a CUDA-capable GPU is supported. Xcode Playground
is supported as well. Please refer to [Usage](#Usage)
and [Components](#Components).

## Quick look

### Value types

CUDA Driver, Runtime, cuBLAS, and NVRTC (real-time compiler) are wrapped in
native Swift types. Warp provides higher level value types, `DeviceArray` and
`DeviceValue`, with copy-on-write semantics.

```swift
import Warp

/// Initialize two arrays on device
var x: DeviceArray<Float> = [1.0, 2.0, 3.0, 4.0, 5.0]
let y: DeviceArray<Float> = [1.0, 2.0, 3.0, 4.0, 5.0]

/// Scalar map operations
x.incrementElements(by: 2) // x => [2.0, 3.0, 4.0, 5.0, 6.0] on device
x.multiplyElements(by: 2) // x => [2.0, 4.0, 6.0, 8.0, 10.0] on device

/// Addition
x.formElementwise(.addition, with: y) // x => [3.0, 6.0, 9.0, 12.0, 15.0] on device

/// Dot product
x • y // => 165.0

/// Sum
x.sum() // => 15

/// Absolute sum
x.sumOfAbsoluteValues() // => 15

/// Transform by 1-place math functions
x.transform(by: .sin)
x.transform(by: .tanh)
x.transform(by: .ceil)

/// Elementwise operation
x.formElementwise(.addition, with: y)
x.formElementwise(.subtraction, with: y)
x.formElementwise(.multiplication, with: y)
x.formElementwise(.division, with: y)

/// Fill with the same value
var z = y
z.fill(with: 10.0)

/// Composite assignment
x.assign(from: .subtraction, left: y, multipliedBy: 100.0, right: z)
```

### Real-time compilation

#### Compile source string to PTX
```swift
import NVRTC
import CUDADriver
import Warp

let source: String =
  + "extern \"C\" __global__ void saxpy(float a, float *x, float *y, float *out, int n) {"
  + "    size_t tid = blockIdx.x * blockDim.x + threadIdx.x;"
  + "    if (tid < n) out[tid] = a * x[tid] + y[tid];"
  + "}";
let ptx = try Compiler.compile(source)
```

#### JIT-compile and load PTX using Driver API within a device context
```swift
try Device.main.withContext { context in
    let module = try Module(ptx: ptx)
    let function = module.function(named: "saxpy")!
    
    let x: DeviceArray<Float> = [1, 2, 3, 4, 5, 6, 7, 8]
    let y: DeviceArray<Float> = [2, 3, 4, 5, 6, 7, 8, 9]
    var result = DeviceArray<Float>(capacity: 8)

    try function<<<(1, 8)>>>[.float(1.0), .constPointer(to: x), .constPointer(to: y), .pointer(to: &result), .int(8)]
    /// result => [3, 5, 7, 9, 11, 13, 15, 17] on device
}
```

## Package Information

Add a dependency:

```swift
.Package(url: "https://github.com/rxwei/cuda-swift", majorVersion: 1)
```

You may use the `Makefile` in this repository for you own project. No extra path
configuration is needed.

Otherwise, specify the path to your CUDA headers and library at `swift build`.

#### macOS
```
swift build -Xcc -I/usr/local/cuda/include -Xlinker -L/usr/local/cuda/lib
```

#### Linux
```
swift build -Xcc -I/usr/local/cuda/include -Xlinker -L/usr/local/cuda/lib64
```

## Components

### Core

- [x] CUDADriver - CUDA Driver API
    - [x] `Context`
    - [x] `Device`
    - [x] `Function`
    - [x] `PTX`
    - [x] `Module`
    - [x] `Stream`
    - [x] `Unsafe(Mutable)DevicePointer<T>`
    - [x] `DriverError` (all error codes from CUDA C API)
- [x] CUDARuntime - CUDA Runtime API
    - [x] `Unsafe(Mutable)DevicePointer<T>`
    - [x] `Device`
    - [x] `Stream`
    - [x] `RuntimeError` (all error codes from CUDA C API)
- [x] NVRTC - CUDA Runtime Compiler
    - [x] `Compiler`
- [x] CuBLAS - GPU Basic Linear Algebra Subprograms (in-progress)
    - [x] Level 1 BLAS operations
    - [x] Level 2 BLAS operations (GEMV)
    - [x] Level 3 BLAS operations (GEMM)
- [x] Warp - GPU Acceleration Library ([Thrust](https://github.com/thrust/thrust) counterpart)
    - [x] `DeviceArray<T>` (generic array in device memory)
    - [x] `DeviceValue<T>` (generic value in device memory)
    - [x] Acclerated vector operations
    - [x] Type-safe kernel argument helpers

### Optional

- [x] Swift Playground
  - CUDADriver works in the playground. But other modules cause the "couldn't lookup
    symbols" problem for which we don't have a solution until Xcode is fixed.
  - To use the playground, open the Xcode workspace file, and add a library for
    every modulemap under `Frameworks`.

## Dependencies

- [CCUDA (CUDA C System Module)](https://github.com/rxwei/CCUDA)

## License

MIT License

CUDA is a registered trademark of NVIDIA Corporation.
