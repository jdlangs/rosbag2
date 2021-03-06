# Rosbag2 writer benchmarking

The primary package to test transport-less performance of the rosbag2 writer and storage.
Enables parametrized batch execution of benchmarks and tries to replicate the flow to capture message loss in queues.

## How it works

Use `scripts/benchmark.sh` to run an entire set of benchmarks.
These are currently aimed at several 100Mb/s scenarios.
Parameters are easy to change inside the script.

By default, results will be written to `/tmp/rosbag2_test/[current_date]`.
The summary of benchmarks goes into `results.csv` file, which includes rows of execution parameters and results.
Benchmarks also produce execution logs in a series of sub-directories in `size[size]_inst[inst]_cache[cache]/` format.

Database (bag) files are removed after recording to avoid filling up the disk.
To modify this behavior, modify the benchmark.sh script.

## Building

To build the package in the rosbag2 build process, make sure to turn `BUILD_ROSBAG2_BENCHMARKS` flag on (e.g. `colcon build --cmake-args -DBUILD_ROSBAG2_BENCHMARKS=1`)

If you already built rosbag2, you can use `packages-select` option to build benchmarks.
Example: `colcon build --packages-select rosbag2_performance_writer_benchmarking --cmake-args -DBUILD_ROSBAG2_BENCHMARKS=1`.

## General knowledge: I/O benchmarking

#### Background: benchmarking disk writes on your system

It might be useful to first understand what limitation your disk poses to the throughput of data recording.
Performance of bag write can't be higher over extended period of time (you can only use as much memory).

**Using dd command**

`dd if=/dev/zero of=/tmp/output conv=fdatasync bs=384k count=1k; rm -f /tmp/output`

This method is not great for benchmarking the disk but an easy way to start since it requires no dependencies.
This will write zeros to the /tmp/output file with block size 384k, 1000 blocks, ends when write finishes.
Make sure to benchmark the disk which your bags will be written to (check your mount points and change “/tmp/output” to another path if needed).
Note: this depends on parameters used and whatever else is running on your system but can give you a ballpark figure when ran several times.

**Using fio**

For more sophisticated & accurate benchmarks, see the `fio` command. An example for big data blocks is: `fio --name TEST --eta-newline=5s --filename=fio-tempfile.dat --rw=write --size=500m --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=60 --group_reporting`.

#### Profiling bags I/O with tools

Tools that can help in I/O profiling: `sudo apt-get install iotop ioping sysstat`
* `iotop` works similar as `top` command, but shows disk reads, writes, swaps and I/O %. Can be used at higher frequency in batch mode with specified process to deliver data that can be plotted.
  *  Example use: `sudo iotop -h -d 0.1 -t -b -o -p <PID>` after running the bag.  
* `ioping` can be used to check latency of requests to device
* `strace` can help determine syscalls associated with the bottleneck.
  *  Example use: `strace -c ros2 bag record /image --max-cache-size 10 -o ./tmp`. You will see a report after finishing recording with Ctrl-C.
