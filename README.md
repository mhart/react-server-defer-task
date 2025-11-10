# An exploration of changing deferTask in RSC streams in Next.js

Based on [the Next.js benchmark from t3dotgg](https://github.com/t3dotgg/cf-vs-vercel-bench/tree/main/next-bench) (with the "math" CPU stuff removed, leaving just rendering work).

This illustrates the impact of changing [the `MAX_ROW_SIZE` threshold used to determine whether to `deferTask` or not](https://github.com/facebook/react/blob/0fa32506dab4293dfffae662e181d2f970aa95ba/packages/react-server/src/ReactFlightServer.js#L3460-L3462) during flight stream serialization for rendering on the server.

Changing this check essentially has a huge knock-on effect that removes a lot of the work involved in rendering large pages because these chunks are no longer lazy, so they don't need to be resolved later, thrown in deep pending stacks, reserialized, etc.

This logic and value (`const MAX_ROW_SIZE = 3200`) were originally introduced in [#33030](https://github.com/facebook/react/pull/33030) to prevent large rows from blocking painting.
However, it has a big effect when SSR'ing in Next.js (multiplied somewhat because it tees the flight stream and applies a chain of transforms).

The magnitude of performance impact suggests that it could perhaps be configurable in some way on the server, and ideally able to be determined ahead of time whether it's necessary or not (potentially during the compilation/bundling phase)

The patch here makes this configurable, to illustrate the change in SSR performance. It's essentially just:

```diff
  const element: ReactElement = (value: any);

- if (serializedSize > MAX_ROW_SIZE) {
+ if (serializedSize > process.env.MAX_ROW_SIZE) {
    return deferTask(request, task);
  }

  if (__DEV__) {
```

## Testing the patch

```bash
pnpm install
pnpm build # Includes patching Next.js

MAX_ROW_SIZE=3200 NODE_ENV=production node .next/standalone/server.js
```

Using `3200` will yield the same results as pre-patch, increasing this shows the performance benefits.

## Testing w/ other versions of Next.js

```bash
pnpm install next@canary
pnpm build # Works at least until Next.js 16.0.2-canary.12
```

## Result Summary

(MB Pro M1, Next.js 15.5.6 and Node.js 22.21.1, other versions below)

Default wrk settings (2 threads, 10 connections)

```bash
wrk -d30 http://localhost:3000
```

- Avg w/ `MAX_ROW_SIZE=3200`: 2.29s (default setting)
- Avg w/ `MAX_ROW_SIZE=12800`: 1.54s (1.49x faster)
- Avg w/ `MAX_ROW_SIZE=18500`: 1.27s (1.8x faster)
- Avg w/ `MAX_ROW_SIZE=65100`: 1.19s (1.92x faster)

With `node --single-threaded` to simulate a single core (like on Lambda) and then running a single request at a time (1 thread, 1 connection):

```bash
wrk -c1 -t1 -d30 http://localhost:3000
```

- Avg w/ `MAX_ROW_SIZE=3200`: 253ms (default setting)
- Avg w/ `MAX_ROW_SIZE=12800`: 170ms (1.49x faster)
- Avg w/ `MAX_ROW_SIZE=18500`: 140ms (1.81x faster)
- Avg w/ `MAX_ROW_SIZE=65100`: 125ms (2.02x faster)

Other runtimes see large impact too:

- Bun on Next.js 15: 1.5x – 2.04x
- Deno on Next.js 15: 1.47x – 2.3x

**NB:** On Next.js 16, the results are less pronounced though still non-trivial, in the 1.26x – 1.52x range on Node.js for latest canary (16.0.2-canary.12).

- Bun on Next.js 16: 1.41x – 1.77x
- Deno on Next.js 16: 1.2x – 1.37x

## Full Results

(note that Bun always runs multi-threaded GC, I'm not sure what the `--single-threaded` equivalent is)

### Next.js 15.5.6

- Node.js 22.21.1, single-threaded, 3200 = 253ms -> 12800 = 170ms (1.49x), 18500 = 140ms (1.81x), 65100 = 125ms (2.02x)
- Node.js 22.21.1, 10 concurrents, 3200 = 2.29s -> 12800 = 1.54s (1.49x), 18500 = 1.27s (1.8x), 65100 = 1.19s (1.92x)

- Node.js 24.11.0, single-threaded, 3200 = 213ms -> 12800 = 147ms (1.45x), 18500 = 134ms (1.59x), 65100 = 122ms (1.75x)
- Node.js 24.11.0, 10 concurrents, 3200 = 1.94s -> 12800 = 1.42s (1.37x), 18500 = 1.23s (1.58x), 65100 = 1.2s (1.62x)

- Node.js 25.1.0, single-threaded, 3200 = 208ms -> 12800 = 149ms (1.4x), 18500 = 130ms (1.6x), 65100 = 120ms (1.73x)
- Node.js 25.1.0, 10 concurrents, 3200 = 1.86s -> 12800 = 1.39s (1.34x), 18500 = 1.2s (1.55x), 65100 = 1.16s (1.6x)

- Bun 1.3.2, "single-threaded", 3200 = 143ms -> 12800 = 93ms (1.54x), 18500 = 77ms (1.86x), 65100 = 70ms (2.04x)
- Bun 1.3.2, 10 concurrents, 3200 = 1.42s -> 12800 = 946ms (1.5x), 18500 = 792ms (1.79x), 65100 = 722ms (1.97x)

- Deno 2.5.6, single-threaded, 3200 = 264ms -> 12800 = 161ms (1.64x), 18500 = 136ms (1.94x), 65100 = 115ms (2.3x)
- Deno 2.5.6, 10 concurrents, 3200 = 2.15s -> 12800 = 1.46s (1.47x), 18500 = 1.17s (1.84x), 65100 = 1.12s (1.92x)

### Next.js 16.0.2-canary.12

- Node.js 22.21.1, single-threaded, 210ms (3200) -> 12800 = 167ms (1.26x), 18500 = 142ms (1.48x), 65100 = 143ms (1.47x)
- Node.js 22.21.1, multi-threaded, 1.82s (3200) -> 12800 = 1.45s (1.26x), 18500 = 1.29s (1.41x), 65100 = 1.2s (1.52x)

- Node.js 24.11.0, single-threaded, 180ms (3200) -> 12800 = 144ms (1.25x), 18500 = 128ms (1.41x), 65100 = 125ms (1.44x)
- Node.js 24.11.0, multi-threaded, 1.58s (3200) -> 12800 = 1.31s (1.21x), 18500 = 1.2s (1.32x), 65100 = 1.14s (1.39x)

- Node.js 25.1.0, single-threaded, 177ms (3200) -> 12800 = 144ms (1.22x), 18500 = 130ms (1.36x), 65100 = 126ms (1.4x)
- Node.js 25.1.0, multi-threaded, 1.57s (3200) -> 12800 = 1.33s (1.18x), 18500 = 1.19s (1.32x), 65100 = 1.18s (1.33x)

- Bun 1.3.2, "single-threaded", 122ms (3200) -> 12800 = 84ms (1.45x), 18500 = 73ms (1.67x), 65100 = 69ms (1.77x)
- Bun 1.3.2, "multi-threaded", 1.18s (3200) -> 12800 = 837ms (1.41x), 18500 = 747ms (1.58x), 65100 = 701ms (1.68x)

- Deno 2.5.6, single-threaded, 158ms (3200) -> 12800 = 132ms (1.2x), 18500 = 117ms (1.35x), 65100 = 116ms (1.36x)
- Deno 2.5.6, multi-threaded, 1.49s (3200) -> 12800 = 1.22s (1.22x), 18500 = 1.12s (1.33x), 65100 = 1.09s (1.37x)
