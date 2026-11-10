# memtier_benchmark with Homa and SMT

This tree is [memtier_benchmark](https://github.com/redis/memtier_benchmark), Redis Ltd.'s load
generator for Redis and Memcached, with two more transports and options for a fleet of independent,
open-loop clients:

- **Homa** ([PlatformLab/HomaModule](https://github.com/PlatformLab/HomaModule)): a message-based
  datacenter transport.
- **SMT** (Secure Message Transport, [uoenoplab/smt](https://github.com/uoenoplab/smt)): Homa with
  TLS records inside the transport, from our S&P'26 paper "Designing Transport-Level Encryption for
  Datacenter Networks".

The Redis server with these transports is [uoenoplab/smt-redis](https://github.com/uoenoplab/smt-redis).
memtier_benchmark's own README is [README-memtier.md](README-memtier.md).

> [!NOTE]
> We rebase this branch onto upstream `master` from time to time and force-push it.

## Build

The dependencies are in [DEVELOPMENT.md](DEVELOPMENT.md). Then:

```
autoreconf -ivf && ./configure && make -j
```

## Options we added

| Option | What it does |
|---|---|
| `--homa` | Each client is one Homa socket, and each write is one Homa RPC. `--server` must be an IPv4 address, and `--port` is the server's Homa port. |
| `--smt` | `--homa` with SMT encryption. Each socket gets SMT's built-in test key for every peer (TLS 1.2 records, AES-GCM-128). The server must use the same key, as `redis-server --smt-port` does. A test key protects nothing: use it for measurements only. |
| `--rate-poisson=RATE` | Each client's requests arrive as a Poisson process of RATE per second. A request's latency counts from its arrival, so it includes the time that the request waits for its client's previous request. |
| `--sample-mix` | Each request's type (by `--ratio`) and value size (by `--data-size-list` weights) are drawn at random. Upstream memtier rotates through them in a fixed order. |
| `--warmup=SECS` | Each client runs SECS seconds before `--test-time` starts. The results do not count the requests sent in that time. |
| `--no-per-second-percentiles` | Each client skips the per-second latency percentiles. With thousands of clients, that pass stalls the worker threads and inflates the tail. Totals and percentiles of the run do not change. |

Over Homa, a client has one RPC in flight at a time: Homa can complete a later, smaller response
first, and memtier's parser expects the responses in order. With `--pipeline` above 1, the requests
that memtier writes together go in one RPC.

## Example

16 threads x 64 clients over Homa against a Redis server at 10.0.1.1, Homa port 2000. All 1,024
clients together send 60,000 requests per second, about 59 per client:

```
memtier_benchmark -s 10.0.1.1 -p 2000 --homa --protocol=redis -t 16 -c 64 \
  --rate-poisson=58.59 --sample-mix --randomize --distinct-client-seed \
  --ratio=7:93 --data-size-list=27:5751,607:1172,434:732 --key-maximum=100000 \
  --warmup=5 --test-time=20 --no-per-second-percentiles --hide-histogram
```

## Results

[uoenoplab/smt-redis `artifact/fleet`](https://github.com/uoenoplab/smt-redis/tree/artifact/fleet)
used these options for 4 to 24,576 clients against one Redis shard, with request mixes from Twitter's
cache trace. It ran the version before the rebase, on branch `archive/fleet`.

## Branches

| Branch | What |
|---|---|
| `main` | this tree |
| `archive/fleet` | the version that `artifact/fleet` measured (upstream `7a6394e` and our first five commits) |
