# Zarr cloud benchmarks
Benchmarking Zarr readers when reading from cloud object storage.

The plan is to benchmark `zarr-python`, `zarrs-python`, and `zarrs` (from Rust).

The focus will be on reading sharded Zarrs with small chunks, and to look at several different read patterns, such as:
1. Read the entire dataset into RAM
2. Read a few chunks from one shard (with different gaps between the chunks)
3. Read a few chunks from multiple shards

The benchmarks will use real cloud object storage, rather than simulated. And we'll use a virtual machine with a fast network interface card (on the order of 100 gigabits per second).

We'll only look at reading from a single VM because the main use-case we're interested in is reading data whilst training machine learning models, where each GPU is very hungry for data.

## Instrumenting Zarr readers

I'm not just interested in "who is the fastest". I'm also interested in _how_ we can speed up each Zarr reader. So I'll also try to instrument `zarr-python` so we can see a timeline of chunks passing in parallel through the pipeline, which I hope will help us figure out how to speed up `zarr-python`.

I'd also like to try instrumenting these Zarr readers to create precise timelines of when each chunk passes to the next step in the pipeline:

1. Send GET request.
2. Receive first byte from the network.
3. Receive last byte from the network.
4. Start decompression.
5. Copy decompressed chunk into final array.

For example, an optimal Zarr reader might submit GET requests in quick succession, and maintain about 100 network requests in flight at any given moment, and overlap decompression with network IO. The timeline of this optimal Zarr reader might look something like this (assuming we artificially limit parallelism so that no more than two chunks can be processed at once, to make it easier to visualise):

```
       |---------------------> time -------------------->
       
chunk1 | 1----------2-----3-4-5
chunk2 |  1----------2-----3-4-5
chunk3 |                  1----------2-----3-4-5
chunk4 |                   1----------2-----3-4-5
```



## TODO

- [ ] Write script to create sharded Zarr
- [ ] Benchmark `zarr-python` with and without `obstore`, and with and without the ["Coalesce and parallelize partial shard reads" pull request](https://github.com/zarr-developers/zarr-python/pull/3004).
- [ ] Measure raw network throughput (including packet headers) by recording the total number of bytes received by the NIC before and after running the benchmark.
- [ ] Measure throughput for the final array (after decompression).
- [ ] Benchmark `zarrs` (implement the benchmarks in Rust).
- [ ] Assuming `zarrs` is much faster than `zarr-python`, look into instrumenting `zarr-python` to get the exact time that each chunk passes through each step. Maybe also use network capture.
