Title: Efficient Dart: Optimizing CPU-bound Load in Flutter without Missing a Frame

Last week, I created a [Flutter app](https://github.com/maxim-saplin/flutter_python_streaming) to run the Julia set generator on a gRPC Python server. This sample project discusses [Flutter and Python integration](https://dev.to/maximsaplin/integrating-flutter-all-6-platforms-and-python-part-2-live-talk-542e). I had a prejudice towards Python being super slow. Surprisingly, by using Numba and a few simple alterations (`@njit(parallel=True)` and `prange()`) to the original Python code, I achieved a ~350x performance boost measured in frames-per-second (FPS) in the UI, all happening over gRPC with serialization/deserialization overhead between Flutter and Python processes.

![Python, Numba, Flutter performance diffs, Julia set generation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/taf6zl0e3le2947ji0wl.png)

What surprised me even more was implementing a Dart analogue to the Python code. I expected the state-of-art Dart and Flutter to be fast, but I found it to be three times slower in terms of FPS than Python, running natively in-process, with noticeable UI stutters. Can we optimize Dart's performance for this CPU-bound task? Let's find out.

## About the Task

The Julia set is a fractal closely related to the Mandelbrot set. Calculating Julia/Mandelbrot sets can stress a CPU and are used in many benchmarks, such as [Mojo](https://www.modular.com/blog/how-mojo-gets-a-35-000x-speedup-over-python-part-1). I chose the Julia set to create animated fractals by changing one of the formula parameters.

I used the escape algorithm to calculate the set for a given region of points. For each (x, y) coordinate representing a point on a complex plane, we iterate in a loop making up to a `threshold` number of iterations (default is 100), calculating if the point belongs to the set or not (diverges). For each point, we can store the number of iterations it took to verify it, and then use it to color and render the pixel. Essentially, we create a method that can generate a `width`x`height` pixel map with each pixel described by the number of loop iterations it took to validate the (x, y) number.

## App Overview

You can find the [Git repo here](https://github.com/maxim-saplin/flutter_python_streaming). The Flutter project is located in the `app/` folder. The rest of the folders can be ignored (part of Python/gRPC example):

```
app/
|-- grpc_generated/
|-- julia_native/
|-- julia_service.dart
|-- main.dart
```

1) `main.dart` contains the UI and rendering logic, using the `RawImage` widget to display the Julia set received from a stream provided by...
2) `julia_service.dart` offers an abstract interface to different generators.
3) `julia_native/` contains six Dart implementation variants of Julia set generator reviewed below.

## Measurement and Assumptions

We use frames-per-second (FPS) as a measure. The FPS counter is displayed at the top left corner of the window. To start the measurement, double-click the Play button, and the animation will cycle three times, with the average FPS registered.

FPS shows how many Julia sets (pixel maps flattened as `List<int>`) are received in a given second in `Stream.listen` event. The FPS counter includes the calculations of Julia set, widget build times, and async/stream/list creation overhead, among other factors. It does not measure the pure Julia loop execution time but loads the entire Flutter pipeline assuming that most of the CPU time is spent on Julia set generation.

Rendering/rasterization is ignored as a factor since it is assumed to be fast and takes place in a separate rasterization thread. As seen in the Dart Dev Tools Performance page, almost no frames exceed 1ms raster time, indicating that most CPU work occurs in the UI thread (computations, creating widgets, allocating lists, etc.).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iixdenvpnslgovzfrikz.png)

Now let's examine the six different Dart implementation variants of the Julia set generator and evaluate their performance improvements based on provided test results.

**V1 - A Direct Python-to-Dart Translation**

V1 is a straightforward conversion of the Python code running in the UI thread. This version runs single-threaded and serves as our baseline for performance measurements. The average FPS achieved by this variant is approximately 37.3.

*Code snippet:*

```dart
// julia_native/v1.dart

class _Complex {
  double real;
  double imag;

  _Complex(this.real, this.imag);

  _Complex operator +(_Complex other) =>
      _Complex(real + other.real, imag + other.imag);
  _Complex operator *(_Complex other) => _Complex(
      real * other.real - imag * other.imag,
      real * other.imag + imag * other.real);

  double abs() => sqrt(real * real + imag * imag);
}

int _checkInJuliaSet(
    double zx, double zy, double constX, double constY, int threshold) {
  _Complex z = _Complex(zx, zy);
  _Complex c = _Complex(constX, constY);

  for (int i = 0; i < threshold; i++) {
    z = z * z + c;
    if (z.abs() > 4.0) {
      return i;
    }
  }

  return threshold - 1;
}
```

In V1, the Julia set generation is calculated using a custom `_Complex` class for complex number operations and the `abs()` function to check for the escape condition.

**V2 - Parallelization using IsolatePool**

V2 takes the V1 implementation and parallelizes it using the IsolatePool library. By utilizing 10 isolates, V2 achieves an average FPS of 97.1, which is a significant improvement compared to the baseline performance.

*Code snippet:*

```dart
// julia_native/v2.dart

// Skipping imports and _canceled, cancelSetGeneration(), and _position for brevity

late IsolatePool _pool;

Future<List<int>> _getSetAsHeightMapParallel(
    int widthPoints, int heightPoints, IsolatePool pool, int threshold) async {
  // ...
}

class GetBlockJob extends PooledJob<List<int>> {
  // ...

  @override
  Future<List<int>> job() async {
    List<int> result = List<int>.filled(widthPoints * im.length, 0);
    // ...
    for (int i = 0; i < im.length; i++) {
      for (int j = 0; j < widthPoints; j++) {
        result[i * widthPoints + j] =
            _checkInJuliaSet(re[j], im[i], cx, cy, threshold);
      }
    }

    return result;
  }
}
```

In V2, the Julia set generation is parallelized using the `IsolatePool` library, and the same logic from V1 is wrapped inside a `GetBlockJob` class. This class is later scheduled using the `IsolatePool` mechanism to handle parallelism.

**V3 - Using Uint8List instead of List<int>**

In V3, the less efficient List<int> data structure from V2 is replaced with a more performant Uint8List. This change results in an average FPS of 112.2 when using 10 isolates, a further improvement compared to V2.

*Code snippet:*

```dart
// julia_native/v3.dart

// Skipping imports for brevity

Future<List<int>> _getSetAsHeightMapParallel(
    int widthPoints, int heightPoints, IsolatePool pool, int threshold) async {
  var list = Uint8List(widthPoints * heightPoints); // changed from List<int>
  // ...
}

class GetBlockJob extends PooledJob<List<int>> {
  // ...

  @override
  Future<List<int>> job() async {
    List<int> result = List<int>.filled(widthPoints * im.length, 0);
    // ...
    }
    return result;
  }
}
```

In V3, the only change made is to replace `List<int>` with `Uint8List` when creating the list to store the Julia set generation results.

**V4 - Improved Single-Threaded Calculations**

V4 is an enhanced version of V1, using more efficient calculations without the need for a complex number data structure and sqrt function. This single-threaded implementation achieves an average FPS of 47.2, showing an improvement over the baseline V1 but still lacking the performance of the parallelized V2 and V3.

*Code snippet:*

```dart
// julia_native/v4.dart

// Skipping imports for brevity

int _checkInJuliaSet(
    double zx, double zy, double constX, double constY, int threshold) {
  var zx0 = zx;
  var zy0 = zy;

  for (int i = 0; i < threshold; i++) {
    final zx0zy0 = zx0 * zy0;
    zx0 = zx0 * zx0 - zy0 * zy0 + constX;
    zy0 = zx0zy0 + zx0zy0 + constY;
    if ((zx0 * zx0 + zy0 * zy0) > 16) {
      return i;
    }
  }

  return threshold - 1;
}

```

In V4, a more efficient implementation of the `_checkInJuliaSet` function is used, removing the `_Complex` class and replacing the `abs()` function with a direct calculation of the sum of squares for the escape condition.

**V5 - Combining Optimizations from V3 and V4**

V5 incorporates the improved calculations from V4 into the parallelized V3 implementation. Depending on the number of isolates used (8, 10, or 12), the average FPS varies between 107.2 to 119.6, demonstrating the effectiveness of combining the optimizations.

*Code snippet:*

```dart
// julia_native/v5.dart
// V5 source code is identical to V3 source code, but with _checkInJuliaSet from V4 implementation
// ...

int _checkInJuliaSet(
    double zx, double zy, double constX, double constY, int threshold) {
  var zx0 = zx;
  var zy0 = zy;

  for (int i = 0; i < threshold; i++) {
    final zx0zy0 = zx0 * zy0;
    zx0 = zx0 * zx0 - zy0 * zy0 + constX;
    zy0 = zx0zy0 + zx0zy0 + constY;
    if ((zx0 * zx0 + zy0 * zy0) > 16) {
      return i;
    }
  }

  return threshold - 1;
}
```

In V5, we use the improved `_checkInJuliaSet` function from V4 while maintaining the parallel structure of V3 with `Uint8List`.

**V6 - Optimized Work Distribution among Isolates**

V6 introduces a more advanced load distribution strategy for the isolates. By using a multiplier (x2, x4, or x6) for interleaving rows among isolates, V6 ensures a more even workload for all isolates. This optimization results in the highest average FPS of 149.4 when using 9 isolates and a 4x load distribution, which is about four times the performance of the baseline V1.

*Code snippet:*

```dart
// julia_native/v6.dart
// Skipping imports for brevity

// The source code for V6 Dart implementation is same as V5 Dart implementation
// with additional modifications related to work distribution for the isolates in
// _getSetAsHeightMapParallel()

Future<List<int>> _getSetAsHeightMapParallel(
    int widthPoints, int heightPoints, IsolatePool pool, int threshold) async {
  // ...
  int blockSize = (heightPoints / (pool.numberOfIsolates * 4)).ceil();
  // ...
}

// Rest of the code remains the same as V5
```

In V6, the primary change from V5 is the new blockSize calculation that incorporates a 4x factor for distributing workload more evenly among isolates.

**Summary and Conclusion**

In this article, we have evaluated the performance improvements for generating the Julia set in Flutter by optimizing the Dart code. Using the provided test results, we have analyzed six different Dart implementation variants:

| Variant       | Average FPS  | Description                                     |
|---------------|--------------|-------------------------------------------------|
| V1            | 37.3         | Baseline, direct Python-to-Dart translation     |
| V2            | 97.1         | Parallelization using IsolatePool               |
| V3            | 112.2        | Using Uint8List instead of List<int>            |
| V4            | 47.2         | Improved single-threaded calculations           |
| V5            | 107.2 - 119.6| Combining optimizations from V3 and V4          |
| V6            | 149.4        | Optimized work distribution among isolates      |

The test results show that significant performance improvements can be achieved by optimizing the Dart code for generating the Julia set. The V6 implementation demonstrates the highest performance gain, indicating that distributing the workload more evenly among isolates can lead to significant improvements in frame rates. This article has provided code snippets, performance comparisons, and explanations for each improvement made in the Julia set generation algorithm, showcasing efficient Dart optimizations and inspiring developers to utilize these techniques in their own projects.