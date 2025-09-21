---
title: "Go HTTP: Impact of NOT Reading Response Bodies to EOF"
date: 2025-09-21T12:58:36+02:00
draft: false
toc: false
images:
tags: ["Go", "Benchmarking", "Performance", "Microoptimization"]
---

Recently, a colleague and I were discussing whether draining the response body (my shorthand for “reading to EOF before closing”) was actually necessary when the response contains an error code. In that situation, the client already reads the full body on successful requests, so my colleague argued that skipping the drain on error responses would have negligible performance impact.

That reasoning made sense at first glance, but it left me curious. I knew the theory, but I wanted to quantify the actual performance impact of not draining response bodies.

This post documents that investigation.

> Source code of this experiment can be found in [this GitHub repository](https://github.com/pessolato/httpmicrobench).

## Why This Matters

HTTP keep-alive and connection reuse are critical for performance in modern applications. If clients fail to properly consume responses, they can inadvertently disable connection pooling, forcing expensive new TCP (and potentially TLS) handshakes on each request.

This problem is subtle: your code may work fine in tests, but at scale, the performance degradation can be dramatic.

## How Go’s HTTP Transport Works

When you issue a request with `http.Client.Do(req)`:

1. The request is dispatched through the configured `Transport` (by default, `http.Transport`).
2. The transport either opens or reuses a TCP connection to the server.
3. The server replies with headers followed by a body (either length-delimited or chunked).
4. Go exposes the body as `resp.Body`, which is more than a simple `io.ReadCloser`. It manages the underlying connection and determines whether it can be reused.

### Case 1: Closing Without Reading to EOF

If you call `resp.Body.Close()` before consuming all bytes:

* **Unread bytes remain in the buffer.**
  Suppose the server sends `Content-Length: 1000`, but you only read 100 bytes. The remaining 900 bytes still sit in the TCP buffer.

* **Connection reuse becomes unsafe.**
  The `Transport` cannot guarantee the connection is aligned with the next request boundary.

* **The connection is discarded.**
  Instead of returning the connection to the idle pool, Go closes it.

* **Practical effect:** The next request must open a fresh TCP (and TLS) connection, which increases latency and CPU load.

### Case 2: Reading to EOF and Closing

If you fully consume the response body:

* The `Transport` can confirm the stream is aligned.
* Closing returns the connection to the idle pool.
* Subsequent requests reuse the same connection, avoiding extra handshakes.

## Designing the Experiment

To measure the performance impact, I built a controlled test environment using only Go’s standard library and Docker for orchestration. The setup consisted of a simple HTTP client and server, plus infrastructure to collect and analyze metrics.

### Server

The server responds to any request with a payload of configurable size. A path parameter controls the number of bytes to send back, which the server generates as random data. This allowed me to simulate different response sizes without external dependencies.

### Client

The client was more involved. Key requirements:

* **Protocol selection:** Ability to enable only HTTP/1.1 or HTTP/2.
* **Tracing:** Integration with `httptrace.ClientTrace` to record connection and request lifecycle events.
* **Configurable draining:** Optionally read responses to EOF before closing.

Each client issues a “template” request `n` times in sequence, logging response times and tracing events. Configuration is done via environment variables.

### Orchestration with Docker

The most complex part was orchestration. I used the Docker SDK to build images, manage networks, and control containers. While arguably over-engineered, the setup gave me fine-grained control over test runs.

The workflow:

1. **Image and network setup**

   * Compile client and server into separate binaries.
   * Build Docker images if missing.
   * Create a dedicated network for isolation.

2. **Container creation**

   * Six containers in total:

     * Four clients: HTTP/1.1 with/without draining, HTTP/2 with/without draining.
     * Two servers: one for draining clients, one for non-draining.

3. **Data collection**

   * Logs and metrics streamed via the Docker API.
   * Data stored in JSONL format for easy parsing.

4. **Cleanup**

   * Containers are removed after each run.

## Running the Benchmark

Each client issued 10,000 sequential requests with the following command:

```sh
NUMBER_OF_REQUESTS=10000 go run ./cmd/bench/
```

To summarize results, I ran:

```sh
BENCH_RESULTS_DIRECTORY="benchresults/20250920155945" go run ./cmd/stats/
```

The stats tool computed min, max, mean, and median for response times and CPU usage. While I focused on latency and CPU, the dataset also allows analyzing connection reuse, memory usage, and error rates.

## Results

The following is the output of the stats tool:

```text
Summarizing result logs from file: benchresults/20250920155945/client-http-1-drain-0-logs.jsonl
Request Time:
- Min: 462.968µs
- Max: 4.002694458s
- Mean: 1.279138ms
- Median: 836.046µs

Summarizing result stats from file: benchresults/20250920155945/client-http-1-drain-0-stats.jsonl
CPU Usage:
- Min: 0.00%
- Max: 114.49%
- Mean: 72.64%
- Median: 108.39%

Summarizing result logs from file: benchresults/20250920155945/client-http-1-drain-1-logs.jsonl
Request Time:
- Min: 104.666µs
- Max: 4.01174869s
- Mean: 705.606µs
- Median: 285.943µs

Summarizing result stats from file: benchresults/20250920155945/client-http-1-drain-1-stats.jsonl
CPU Usage:
- Min: 0.00%
- Max: 89.40%
- Mean: 35.87%
- Median: 0.00%

Summarizing result logs from file: benchresults/20250920155945/client-http-2-drain-0-logs.jsonl
Request Time:
- Min: 452.761µs
- Max: 4.002149171s
- Mean: 1.273813ms
- Median: 829.245µs

Summarizing result stats from file: benchresults/20250920155945/client-http-2-drain-0-stats.jsonl
CPU Usage:
- Min: 0.00%
- Max: 113.48%
- Mean: 71.06%
- Median: 107.84%

Summarizing result logs from file: benchresults/20250920155945/client-http-2-drain-1-logs.jsonl
Request Time:
- Min: 105.359µs
- Max: 4.001713071s
- Mean: 707.408µs
- Median: 289.669µs

Summarizing result stats from file: benchresults/20250920155945/client-http-2-drain-1-stats.jsonl
CPU Usage:
- Min: 0.00%
- Max: 89.51%
- Mean: 34.10%
- Median: 0.00%

Summarizing result stats from file: benchresults/20250920155945/server-drain-0-stats.jsonl
CPU Usage:
- Min: 0.00%
- Max: 82.52%
- Mean: 51.61%
- Median: 79.07%

Summarizing result stats from file: benchresults/20250920155945/server-drain-1-stats.jsonl
CPU Usage:
- Min: 0.00%
- Max: 120.90%
- Mean: 27.23%
- Median: 0.00%
```

Highlights from the results:

* **Draining improved mean request times by \~44.5%** (both HTTP/1.1 and HTTP/2).
* **Median latency improved by \~65%** — the effect on typical requests is even stronger than on the average.
* **Client CPU usage dropped by 50–52%** when draining.
* **Server CPU usage dropped by \~47%** when clients drained responses.
* Without draining, both client and server CPUs often saturated near 100%; with draining, they frequently idled near 0%.

In other words, the cost of not draining is not just a small inefficiency—it can double CPU usage and dramatically increase latency.

## Conclusion

Failing to drain HTTP response bodies in Go **significantly harms performance** for both clients and servers.

* **Request latency:** \~44.5% faster on average, \~65% faster at median.
* **Client CPU:** \~50% lower.
* **Server CPU:** \~47% lower.
* **Connection reuse:** Preserved with draining, disabled without it.

### Practical Takeaway

Always drain the response body before closing, even when handling error responses you don’t care about. If you don’t need the content, discard it by reading to EOF and ignoring the bytes.

This small step ensures connection reuse, reduces CPU load, and dramatically improves performance—both for your client and the servers it communicates with.
