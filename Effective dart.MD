Last week I have created a [Flutter app](https://github.com/maxim-saplin/flutter_python_streaming) to run the Julia set generator on a gRPC Python server. This was a sample project discussing [Flutter and Python integration](https://dev.to/maximsaplin/integrating-flutter-all-6-platforms-and-python-part-2-live-talk-542e).

I had a prejudice towards Python that it is super slow. And I was very much surprised to discover that using Numba and a few simple alterations (`@njit(parallel=True)` and `prange()`) to the original Python code I could get ~350x boost in performance! Measured in frames-per-second in the UI! All of that happening over gRPC with serialisation/deserialisation overhead between Flutter and Python processes.

![Python, Numba, Flutter performance diffs, Julia set generation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/taf6zl0e3le2947ji0wl.png)

What surprised me even more... When out of curiosity I implemented Dart analog to Python code, I received 3 times slower FPS - running it natively, in process, with UI stutters. I also had a prejudice that Flutter is super fast out of the box, with Ahead-of-Time optimising compiler, type/null-safe Dart and expected it would be way ahead of any Python implementation.

Can we get Dart up-to-speed and recover? Let's see...

## About the Task

Julia set is a fractal closely related to the Mandelbrot set. Calculation of Julia/Mandlebrot can stress CPU and is used in many benchmarks (here's one of the recent ones from [Mojo](https://www.modular.com/blog/how-mojo-gets-a-35-000x-speedup-over-python-part-1)). I've picked Julia instead of Mandelbrot since it allows to easily create animated fractals by changing one of the formula params.

I used escape algorithm to calculate the set for a given region of points. I.e. for each (x,y) coordinate, which represents a point on a complex plain, we iterate in a loop making up to `threshold` iterations (by default it is 100), calculating if the point belongs to the set or not (diverges). For each point we can store the number of iterations it took to verify it and then use it to color and render pixel. Essentialy, we create a method that can create a `width`x`height` pixel map with each pixel described by number of loop iterations in took to validate (x,y) number.

## App Overview

You can find the [Git repo here](https://github.com/maxim-saplin/flutter_python_streaming). Flutter project is located in `app/` folder, rest of the folders can be ignored (part of Python/gRPC example):
```
app/
|-- grpc_generated/
|-- julia_native/
|-- julia_service.dart
|-- main.dart
```

1) `main.dart` contains the UI and rendering logic, it uses `RawImage` widget to display Julia set received from a Stream provided by...
2) `julia_service.dart` provides an abstract interface to different generators
3) `julia_native/` contains 6 variants of Dart implementations reviewed below.

## V1 - A Direct Python-to-Dart Translation

V1 is a straightforward conversion of the Python code running in the UI thread. This version runs single-threaded and serves as our baseline for performance measurements. The average FPS achieved by this variant is approximately 37.3.

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
In V1, the Julia set generation is calculated using a custom _Complex class for complex number operations and the abs() function to check for the escape condition.

## V2 - Parallelization using [IsolatePool](https://pub.dev/packages/isolate_pool_2)

V2 takes the V1 implementation and parallelizes it using the IsolatePool library. By utilizing 10 isolates, V2 achieves an average FPS of 97.1, which is a significant improvement compared to the baseline performance.

Code snippet:

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
In V2, the Julia set generation is parallelized using the IsolatePool library, and the same logic from V1 is wrapped inside a GetBlockJob class. This class is later scheduled using the IsolatePool mechanism to handle parallelism.

## V3 - Using Uint8List instead of List<int>

In V3, the less efficient List<int> data structure from V2 is replaced with a more performant Uint8List. This change results in an average FPS of 112.2 when using 10 isolates, a further improvement compared to V2.

Code snippet:

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
In V3, the only change made is replacing List<int> with Uint8List when creating the list to store the Julia set generation results.

## V4 - Improved Single-Threaded Calculations

V4 is an enhanced version of V1, using more efficient calculations without the need for a complex number data structure and sqrt function. This single-threaded implementation achieves an average FPS of 47.2, showing an improvement over the baseline V1, but still lacking the performance of the parallelized V2 and V3.

Code snippet:

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

In V4, a more efficient implementation of the _checkInJuliaSet function is used, removing the _Complex class and replacing the abs() function with a direct calculation of the sum of squares for the escape condition.

## V5 - Combining Optimizations from V3 and V4

V5 incorporates the improved calculations from V4 into the parallelized V3 implementation. Depending on the number of isolates used (8, 10, or 12), the average FPS varies between 107.2 to 119.6, demonstrating the effectiveness of combining the optimizations.

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

In V5, we use the improved _checkInJuliaSet function from V4 while maintaining the parallel structure of V3 with Uint8List.

## V6 - Optimized Work Distribution among Isolates

V6 introduces a more advanced load distribution strategy for the isolates. By using a multiplier (x2, x4, or x6) for interleaving rows among isolates, V6 ensures a more even workload for all isolates. This optimization results in the highest average FPS of 149.4 when using 9 isolates and a 4x load distribution, which is about four times the performance of the baseline V1.

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

In conclusion, the test results show that significant performance improvements can be achieved by optimizing the Dart code for generating the Julia set. By applying these optimization techniques, you can improve the performance of your applications and make them more responsive to user interactions. The V6 implementation demonstrates the highest performance gain, indicating that distributing the workload more evenly among isolates can lead to significant improvements in frame rates.

## Measurement and Assumptions

Frames-per-second (FPS) is used as a measure. FPS counter is displayed at the top left corner of the window. To start measurement the Play button is double clicked and animation is cycled 3 times with average FPS registered.

FPS shows how many Julia sets (pixel maps flattened as `List<int>`) are received in a given second in `Stream.listen` event. We ignore rasterisation time (consider it 0). Essentially, the FPS counter includes the calculations of Julia set, widget builds times, async/stream/list creation overhead and whatever is happening in the UI thread. It doesn't measure the pure Julia loops execution time but loads the entire Flutter pipeline (with Julia set generation being the place where most of the CPU time is spent).

Rendering/rasterisation is ignored as a factor since it is 
- Assumed super fast
- Taking place in a separate rasterisation thread 

To support the assumptions, if you check Dart Dev Tools Performance Page while animation is running, there're almost no frames with more than 1ms raster time - most of the CPU work happening in UI thread (computations, creating widgets, allocating lists etc.):

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iixdenvpnslgovzfrikz.png)