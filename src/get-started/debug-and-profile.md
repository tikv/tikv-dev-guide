# Debug and Profile

In previous chapter, we introduce how to build TiKV from source, and in this chapter, we will focus on how to debug and profile TiKV from the view of a developer.

## Prerequisites

* rust-gdb or rust-lldb  
rust-gdb and rust-lldb are installed in rust. But lldb is not installed by default. So if you want to use rust-lldb, please install lldb firstly.  
rust-gdb is an enhanced tools based on gdb for rust programming, and the usage is almost the same with gdb, if you want to learn more about gdb, please see [GDB](https://www.sourceware.org/gdb/).  

* perf  
Perf is common Linux profiler. It's powerful: it can instrument CPU performance counters, tracepoints, kprobes, and uprobes (dynamic tracing). It can be installed as following:
```bash
Ubuntu: sudo apt-get install linux-tools
CentOS: sudo yum install perf
```

*For simplicity, we will introduce the debugging with rust-gdb, audience can also use rust-lldb.*

## Running TiKV with GDB

### Debug a unit test binary in TiKV

1. Build the unit test binary, for example we want to debug the test case: [test_raw_get_key_ttl](https://github.com/tikv/tikv/blob/a7d1595f5486616be34e0cf2bbf372edb5f0e85a/src/storage/mod.rs#L5352-L5356)

Firstly, we can get the binary file with cargo command, like: 
```bash
cargo test tikv test_raw_get_key_ttl
```
A binary file located in `target/debug/deps/tikv-some-hash` will be produced.

2. Debug the binary with rust-gdb:

```bash
rust-gdb --args target/debug/deps/tikv-4a32c89a00a366cb test_raw_get_key_ttl
```
3. Now the standard gdb interface is shown. We can debug the unit test with gdb command. Here are some simple usage examples.

* r(run) to start the program.
* b(break) file_name:line_number to set a breakpoint.
* p(print) args to print args.
* ls to show the surrounding codes of breakpoint.
* s(step) to step in the function.
* n(next) to step to next line.
* c(continue) to continue the program.
* watch to set a data watch breakpoint.

### Debug TiKV cluster with specified tikv-server binary

1. Build tikv-server binary with the guide in [previous chapter](./build-tikv-from-source.md).
2. The binary files are in \${TIKV_SOURCE_CODE}/target/debug/, we can also get one release mode binary with make release, and the binary is in \${TIKV_SOURCE_CODE}/target/release/.
3. TiUP is recommanded to deploy a TiKV cluster. It's easy to deploy a local TiKV cluster with tiup playground. Please refer [Get start in 5 minutes](https://tikv.org/docs/5.1/concepts/tikv-in-5-minutes/#set-up-a-local-tikv-cluster-with-the-default-options). With TiUP, we can also specify the tikv-server binary file during deploy. The following is an emxample:

```bash
TIKV_BIN=~/tikv/target/release/tikv-server

tiup playground v5.0.4 --host 0.0.0.0 --db 0 --tag clst01 \
  --kv 3 --kv.binpath ${TIKV_BIN} --kv.config ~/cluster_config/tikv.toml \
  --pd 1 --pd.config ~/cluster_config/pd.toml
```

4. Now we get one TiKV cluster with three TiKV virtual nodes and one pd node. we can use rust-gdb to attach the tikv-server process.

```bash
rust-gdb attach ${PID}
```

pid of tikv-server can be abtained with following command:

```bash
ps -aux|grep tikv-server
```
Now the standard GDB interface is shown, we can use rust-gdb as same an gdb. 

## Profiling TiKV
When we want to find the cpu bottleneck of one program, we can use [perf Linux profiler](https://www.brendangregg.com/perf.html) to find the events. It can also be used for profiling TiKV.  

Here is one example:

```bash
perf record -g -p `pidof tikv-server`

perf report
```
## Monitor TiKV cluster

Prometheus and Grafana are used to monitor and alert the TiKV cluster. Please ref [TiKV website](https://tikv.org/docs/5.1/deploy/monitor/deploy/#configure-grafana).


