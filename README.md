# JavaScript Self-Profiling API Proposal

[Slides from TPAC 2018](https://github.com/vdjeric/js-self-profiling/blob/master/doc/tpac-2018-slides.pdf)

[Draft spec](https://vdjeric.github.io/js-self-profiling)

Discussion in GitHub issues + [WICG discourse thread](https://discourse.wicg.io/t/proposal-an-api-to-allow-webpage-javascript-to-profile-its-own-performance/2818)

## Motivation

Currently it is difficult for web developers to understand how their applications perform in the wide variety of conditions encountered on real user devices. A programmable JS profiling API is needed to collect JS profiles from real end-user environments.

A native self-profiling API for JS code would also allow web developers to efficiently find hotspots in their JS code during page loads and user interactions, to assign CPU budgets to individual JS-implemented features on the page, to find unnecessary work being done on the client, and to find low-priority JS code executing in the background and wasting device power.

Currently JS self-profiling can be accomplished by instrumenting individual JS functions with timing code but this is cumbersome, bloats JS size, changes the code (potentially altering performance characteristics), adds overhead from timing calls, and risks missing out on hotspots in unexpected corners.

## Facebook's Profiler Polyfill

In an attempt to polyfill the missing self-profiling functionality, Facebook built and deployed its own in-page JS profiler implemented with JavaScript and SharedArrayBuffers. This JS profiler was implemented using a worker thread that signaled to the main thread when it needed to record its current stack. The worker thread would toggle a “capture stack now” flag in a SharedArrayBuffer every few milliseconds, and the value of this flag was read by instrumentation code that was inserted at transpilation time into the beginning of interesting JS functions running on the main thread. If the instrumentation code saw that the flag was set, it would capture the current JS stack (using the Error object) and add it to the running profile.

This JS profiler was enabled for only a small percentage of Facebook users, and only instrumented functions of 10 statements or more in order to limit performance impact and to limit the quantity of profiling data collected. Nevertheless, we found it extremely valuable for understanding Facebook.com's performance in the field and for finding optimization opportunities. 

This polyfill implementation had some downsides:

* The size overhead of the added instrumentation required limiting the fraction of functions instrumented (resulting in incomplete coverage)
* The sampling “interrupt” was not instant so stacks were collected from the **_next_** instrumented function which added a lot of noise to the dataset and made it difficult to reason about
* The performance overhead limited the sampling frequency and fraction of pageloads profiled
* Our design depended on SharedArrayBuffers, which have been disabled on the Web for the time-being, so the profiler is disabled for now

A browser-implemented profiling API would avoid these downsides.

## Wider Industry Interest

It is cumbersome to manage instrumentation for performance measurement, regardless of whether it is inserted by build-time tooling (like Facebook's polyfill above) or inserted by hand in functions of interest. This API eliminates the need to maintain performance instrumentation and therefore allows smaller sites to deploy in-page JS profiling.

We also expect that third-party analytics providers will offer libraries or infrastructure to record, ingest, aggregate and automatically analyze the collected profiles for optimization opportunities, thus further lowering the barrier to entry.

Finally, several other Web properties with large codebases have expressed interest to us for using this API to better monitor their webapps performance in the field.

## API Overview

Developers will be able to spin up a new `profiler` via `performance.profile(options)`, where `options` contains the following required fields:

- Target sample rate
- Maximum sample capacity
- Call frame categories that should be traced (currently only "js")

The returned `profiler` (wrapped in a promise) will begin sampling. The UA may choose a different sample rate than the one that the user requested (which must be the next lowest valid sampling interval), or it may refuse to spin up a profiler for any reason (e.g. if there are too many profilers running concurrently) by rejecting the promise. The true sample rate of the profiler may be accessible via `profiler.options.sampleInterval`.

Each time the sampling interval has elapsed, the following algorithm is run:

1. Let `env` be the script execution environment of the script that started the profiler.
2. Let `stack` be the stack of execution contexts associated with `env`.
3. For each execution context `ec` on `stack`:
    1. Let `ecs` be the script associated with `ec`.
    2. If the source of the classic script tag associated with `ecs` shares an origin with the browsing context or has the `crossorigin` bit set (and it sends a valid CORS header), record the stack frame.
    3. If the source of the module script tag associated with `ecs` sends a valid CORS header, record the stack frame.
    4. Otherwise, omit the stack frame.
4. Report a new sample with all stack frames recorded in the algorithm, associated with the current timestamp relative to the browsing context's time origin.

    _In a nutshell, this collects data from all stack frames that come from a script that is same-origin or participates in CORS, masking those that do not._

The UA is free to elide any stack frames that may be optimized out (e.g. as a result of inlining).

Calling `profiler.stop()` returns a promise containing a trace object that can be sent to a server for aggregation. This trace is encoded in a trie format similar to the GeckoProfiler and Chrome tracing formats.

If the sample buffer capacity is reached, the `onsamplebufferfull` event is sent to the `profiler` object. This stops profiling immediately. The trace may still be collected via `profiler.stop()` when this occurs.

## (old) Possible JSON Profile Format

> 2018-10-31: We're currently exploring a more compact binary trace format for traces used for aggregation.

Space-efficient encodings are possible given the tree-like structure of profile stacks and the repeated strings in function names and filenames. A trie seems like a natural choice. A trie is also desirable because the raw uncompressed memory representation causes huge memory pressure and is more expensive to analyze (e.g. to count stack frequencies).

There are examples of trie-based approaches in browsers today:

* Chrome's Trace Event format, specifically the stackFrames field: [https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview#heading=h.yr703knxre9f](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/)
* Firefox's Gecko Profiler format, specifically the stackTable field: https://github.com/devtools-html/perf.html/blob/master/docs-developer/gecko-profile-format.md#source-data-format

An idea for a possible format:

```javascript
{
    version: 1,
    options: {
        <configuration actually used for profiling>
    },
    profile: {
        strings: [
            "start @ main.js:12:34"
            "render @ newsfeed.js:56:78",
            "getLikes @ stories.js:90:12",
            "getComments @ stories.js:11:22"
        ]
        stackFrames: {
            ...
            101: { name: 0 },
            102: { name: 1, parent: 101 },
            103: { name: 2, parent: 102 },
            104: { name: 3, parent: 102 }
        },
        main-thread: [
            {
                "type": "stack",
                "timestamp": 12345.67890000,
                "stack": 103,
            },
            {
                "type": "stack",
                "timestamp": 12346.00000000,
                "stack": 104,
            },
            ...
        ]
    }
}    
```

Open questions:

* How to represent source locations for <script> tags, inline event handlers, etc.
* How to represent a call to native DOM like innerHTML or "style recalc" or JS-builtins like Math.random()
* How should async events between threads be annotated?

## Visualization

Mozilla's perf.html visualization tool for Firefox profiles or Chrome's trace-viewer (chrome://tracing) UI could be trivially adapted to visualize the data produced by this profiling API.

## perf.html

As an illustration, a screenshot below from Mozilla's perf.html project shows the JS stack aggregation and timeline. It is able to show gaps where JavaScript was not executing, areas where there were long running events (red), and an aggregate view of the samples in the selected time range such as the 15 contiguous samples in function 'user.ts' highlighted in the screenshot below.

![Mozilla's perf.html UI for visualizing Gecko profiles](perf.html.png)

