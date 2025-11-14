# Improving batching in RSC streams in Next.js

This illustrates the impact of changing [the `MAX_ROW_SIZE` threshold](https://github.com/facebook/react/blob/0fa32506dab4293dfffae662e181d2f970aa95ba/packages/react-server/src/ReactFlightServer.js#L3460-L3462) during flight stream serialization for rendering on the server. This shows a speedup of **1.75x** on Bun, for example.

**EDIT**: Opened [a PR against React](https://github.com/facebook/react/pull/35089) to expose this as a configurable option.

**EDIT 2**: Opened [an issue](https://github.com/facebook/react/issues/35125) to discuss batching behaviour more broadly.

This logic was originally introduced in [#33030](https://github.com/facebook/react/pull/33030) to prevent large rows from blocking painting.

The problem is that after the limit is reached, all subsequent children will be rendered as lazy tasks individually until the next sibling is processed. This can lead to an explosion of lazy chunks, each with only one element (not necessary even close to the default 3200 serialized size limit), and because each chunk has noticeable overhead, it slows the rendering down much more than if every row was processed in roughly the same batch size of elements/chunks.

## Demo reproduction

This reproduction is a Next.js 16 app that renders a page with 20 sections, each having 103 paragraphs. The first three paragraphs have real text in them, the following 100 are just "Paragraph X". The entire page is ~120kb of HTML.

If you look at the flight (RSC) stream for this page, the first row contains the first few paragraphs, and all the rest are rendered as lazy chunks. This means there are roughly ~2000 rows in the flight stream.

Changing the `MAX_ROW_SIZE` from the default `3200` to `18700` shows what happens if the rows are batched together in roughly equal batches – that ends up with ~20 rows – and processes _much_ faster.

This illustrates that each row has non-trivial overhead, and that the batching mechanism could be improved to alleviate this.

The build script patches `MAX_ROW_SIZE` to make this configurable. It's essentially just:

```diff
  const element: ReactElement = (value: any);

- if (serializedSize > MAX_ROW_SIZE) {
+ if (serializedSize > process.env.MAX_ROW_SIZE) {
    return deferTask(request, task);
  }

  if (__DEV__) {
```

## Testing

```bash
pnpm install
pnpm build # Includes patching Next.js

MAX_ROW_SIZE=3200 NODE_ENV=production bun .next/standalone/server.js
# OR:
MAX_ROW_SIZE=3200 NODE_ENV=production node .next/standalone/server.js
```

Using `3200` will yield the same results as pre-patch, increasing this shows the performance benefits.

## Result Summary

(MB Pro M1, Next.js 16.0.3 and Bun 1.3.2, other versions below)

Default wrk settings (2 threads, 10 connections)

```bash
wrk -d30 --timeout=30 --latency http://localhost:3000
```

- Avg w/ `MAX_ROW_SIZE=3200`: 158.21ms (default setting)
- Avg w/ `MAX_ROW_SIZE=18700`: 90.65ms (**1.75x** faster)

Other runtimes:

- Node.js 25.2.0: **1.39x** faster
- Node.js 24.11.1: **1.4x** faster
- Node.js 22.21.1: **1.4x** faster

## Full results

#### `MAX_ROW_SIZE=3200 NODE_ENV=production bun .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency   158.21ms   16.08ms 279.35ms   91.86%
  Req/Sec    32.58     15.52    50.00     45.54%
Latency Distribution
    50%  156.18ms
    75%  161.37ms
    90%  169.75ms
    99%  252.32ms
1891 requests in 30.05s, 620.66MB read
```

#### `MAX_ROW_SIZE=18700 NODE_ENV=production bun .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency    90.65ms    4.74ms 111.15ms   75.92%
  Req/Sec    55.12     12.81   101.00     85.07%
Latency Distribution
    50%   89.30ms
    75%   93.68ms
    90%   96.82ms
    99%  104.46ms
3306 requests in 30.02s, 0.96GB read
```

#### `MAX_ROW_SIZE=3200 NODE_ENV=production volta run --node=22.21.1 node .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency   206.19ms   22.67ms 544.78ms   93.33%
  Req/Sec    24.03      8.95    40.00     73.06%
Latency Distribution
    50%  201.36ms
    75%  209.50ms
    90%  219.67ms
    99%  316.81ms
1453 requests in 30.04s, 476.84MB read
```

#### `MAX_ROW_SIZE=18700 NODE_ENV=production volta run --node=22.21.1 node .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency   147.60ms   15.36ms 487.48ms   94.94%
  Req/Sec    33.84      6.86    50.00     90.57%
Latency Distribution
    50%  144.40ms
    75%  149.82ms
    90%  159.16ms
    99%  170.66ms
2032 requests in 30.03s, 606.94MB read
```

#### `MAX_ROW_SIZE=3200 NODE_ENV=production volta run --node=24.11.1 node .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency   187.19ms   19.51ms 557.32ms   94.21%
  Req/Sec    26.51      9.21    40.00     68.74%
Latency Distribution
    50%  184.73ms
    75%  190.97ms
    90%  198.30ms
    99%  240.83ms
1602 requests in 30.05s, 525.83MB read
```

#### `MAX_ROW_SIZE=18700 NODE_ENV=production volta run --node=24.11.1 node .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency   134.06ms   10.03ms 343.82ms   92.49%
  Req/Sec    37.20     11.47    50.00     44.44%
Latency Distribution
    50%  134.48ms
    75%  137.67ms
    90%  141.10ms
    99%  151.21ms
2235 requests in 30.04s, 667.66MB read
```

#### `MAX_ROW_SIZE=3200 NODE_ENV=production volta run --node=25.2.0 node .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency   184.46ms   18.89ms 570.90ms   97.12%
  Req/Sec    26.96     10.69    40.00     53.11%
Latency Distribution
    50%  182.47ms
    75%  187.11ms
    90%  195.08ms
    99%  214.24ms
1625 requests in 30.03s, 533.49MB read
```

#### `MAX_ROW_SIZE=18700 NODE_ENV=production volta run --node=25.2.0 node .next/standalone/server.js`

```text
Thread Stats   Avg      Stdev     Max   +/- Stdev
  Latency   132.55ms    9.27ms 339.69ms   92.84%
  Req/Sec    37.64      9.79    50.00     61.85%
Latency Distribution
    50%  131.92ms
    75%  134.61ms
    90%  140.24ms
    99%  151.23ms
2262 requests in 30.04s, 675.60MB read
```
