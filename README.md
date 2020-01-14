# Binlog

A high performance C++ log library to produce structured binary logs.

    BINLOG_INFO("Log anything! {}, {} or even {}", 1.2f, std::vector{3,4,5}, AdaptedStruct{1, "Foo"});

## Motivation

Consider the following log excerpt, taken from an application, that
computes a float value for each request using multiple threads:

    INFO [2019/12/10 12:54:47.062805012] thread1 Process request #1, result: 6.765831 (bin/app/processor.cpp:137)
    INFO [2019/12/10 12:54:47.062805881] thread1 Process request #2, result: 7.835833 (bin/app/processor.cpp:137)
    INFO [2019/12/10 12:54:47.062806406] thread2 Process request #3, result: 2.800832 (bin/app/processor.cpp:137)
    INFO [2019/12/10 12:54:47.062806903] thread3 Process request #5, result: 5.765831 (bin/app/processor.cpp:137)
    INFO [2019/12/10 12:54:47.062807397] thread2 Process request #4, result: 1.784832 (bin/app/processor.cpp:137)
    INFO [2019/12/10 12:54:47.062807877] thread2 Process request #7, result: 2.777844 (bin/app/processor.cpp:137)
    INFO [2019/12/10 12:54:47.062808437] thread3 Process request #6, result: 3.783869 (bin/app/processor.cpp:137)

There's a lot of **redundancy** (the severity, the format string, the file path are printed several times),
**wasted space** (the timestamp is represented using 29 bytes, where 8 bytes of information would be plenty),
and **loss of precision** (the originally computed floating point _result_ is lost, only its text representation
remains). Furthermore, the log was **expensive to produce** (consider the string conversion of each timestamp
and float value) and **not trivial to parse** using automated tools.

Binlog solves these issues by using _structured binary logs_.
The static parts (severity, format string, file and line, etc) are saved only once to the logfile.
While logging, instead of converting the log arguments to text, they are simply copied as-is,
and the static parts are referenced by a single identifier.
This way the logfile becomes **less redundant** (static parts are only written once),
smaller (e.g: timestamps are stored on fewer bytes without loss of precision),
and possibly **more precise** (e.g: the exact float is saved, without having the representation
interfering with the value).
Because of to the smaller representation, _binary logfiles_ are much **faster to produce**,
saving precious cycles on the hot path of the application.
Thanks to the _structured_ nature, binary logfiles can be consumed by other programs,
without reverting to fragile text processing methods.

_Binary logfiles_ are not human readable, but they can be converted to text using the `bread` program.
The format of the text log is configurable, independent of the logfile.
For further information, please refer to the [Documentation][].

## Features

 - Log (almost) anything:
   - Fundamentals (char, bool, integers, floats)
   - Containers (vector, deque, list, map, set and more)
   - Strings (const char pointer, string, string_view)
   - Pointers (raw, unique_ptr, shared_ptr and optionals)
   - Pairs and Tuples
   - Enums
   - Custom structures (when adapted)
 - High performance: log in a matter of tens of nanoseconds
 - Asynchronous logging via lockfree queues
 - Format strings with `{}` placeholders
 - Vertical separation of logs via custom categories
 - Horizontal separation of logs via runtime configurable severities

## Performance

The performance of creating log events is benchmarked using the [Google Benchmark][] library.
Benchmark code is located at `test/perf`.
The benchmark measures the time it takes to create a timestamped log event with various
log arguments (e.g: one integer, one string, three floats) in a single-producer,
single-consumer queue (the logging is asynchronous), running in a tight loop.
There's a benchmark that does not timestamp the log event, to indirectly measure the cost
of `std::chrono::system_clock::now()` or `clock_gettime`.
There's a bechmark trying to guess the cache effect, that invalidates
the data cache on each iteration by writing a large byte buffer.

The results reported below are a sample of several runs,
only to show approximate performance, and not for direct comparison.
Machine configuration: Intel Xeon E5-2698 2.20 GHz, RHEL 2.6.32.

| Benchmark                   | Std. Dev. | Median     |
|:----------------------------|----------:|-----------:|
| One integer                 |      0 ns |      44 ns |
| One integer (no clock)      |      0 ns |      11 ns |
| One integer (poison cache)  |      4 ns |    1015 ns |
| One string                  |      1 ns |      50 ns |
| Three floats                |      0 ns |      45 ns |

## Install

See `INSTALL.md`.
In general, Binlog requires a C++14 conforming platform.
Binlog is built and tested on Linux, macOS and Windows,
with various recent versions of GCC, Clang and MSVC.

[Documentation]: http://binlog.org/UserGuide.html
[Google Benchmark]: https://github.com/google/benchmark