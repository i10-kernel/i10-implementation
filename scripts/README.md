# Test scripts:

## (1) FIO
1. Install FIO: https://github.com/axboe/fio

2. Run the FIO scripts (e.g., fio/randread.fio). You can modify the scripts to use different remote devices (/dev/nvme1n1) or different parameters (e.g, block size, io-depth, cpus, number of jobs, etc.):

   ```
   fio fio/randread.fio
   ```

## (2) RocksDB
1. Install RocksDB: https://github.com/facebook/rocksdb

2. Clean page cache every 1s as background (with root) to reduce the effect of page cache:

   ```
   while true; do eche 1 > /proc/sys/vm/drop_caches; free -h; sleep 1; done;
   ```

3. Run the RocksDB benchmarking tool (db_bench) using 'benchmark.sh'. The below example is to run 'readrandom' workload with a single core (core0):

   ```
   cd rocksdb
   taskset -c 0 tools/benchmark.sh readrandom
   ```

4. If you see “Too many open files” error when writing, increase the number of open files ([ref](https://community.pivotal.io/s/article/Session-failures-with-Too-many-open-files)):

   ```
   ulimit -n 524288
   ```

## (3) Filebench
1. Install Filebench: https://github.com/filebench/filebench

2. Run the Filebench scripts (e.g., filebench/randomread.f). In the scripts, you should specify the target directory where the remote device is mounted (e.g., /mnt):

   ```
   taskset -c 0 filebench -f filebench/randomread.f
   ```
