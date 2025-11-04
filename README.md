# rmp-eval

[![CI](https://github.com/roboticsys/rmp-eval/actions/workflows/ci.yml/badge.svg)](https://github.com/roboticsys/rmp-eval/actions/workflows/ci.yml)
[![CodeQL](https://github.com/roboticsys/rmp-eval/actions/workflows/codeql.yml/badge.svg)](https://github.com/roboticsys/rmp-eval/actions/workflows/codeql.yml)
[![Release](https://github.com/roboticsys/rmp-eval/actions/workflows/release.yml/badge.svg)](https://github.com/roboticsys/rmp-eval/actions/workflows/release.yml)

A Linux utility for evaluating PC hardware capability to run the [RMP EtherCAT Motion Controller](https://www.roboticsys.com/rmp-ethercat-motion-controller).

## Purpose

This tool helps determine if your [PC hardware meets the latency requirements](https://support.roboticsys.com/rmp/guide-pc-latency-jitter-bios.html) for running the RMP EtherCAT Motion Controller. RMP requires consistent real-time performance with specific latency thresholds that depend on your sample rate (e.g., 1000Hz requires <125µs latency).

Use this utility during the PC hardware evaluation phase to:

- Test cyclic timing performance of your isolated CPU
- Evaluate network interface card (NIC) performance with hardware timestamps
- Identify timing variability before deploying RMP

## Download

Pre-built packages are available to download for amd64 and arm64 from the [GitHub Releases](https://github.com/roboticsys/rmp-eval/releases) page.

**Debian/Ubuntu Installation:**

```bash
# Install
sudo dpkg -i rmp-eval_*.deb
```

## Usage

**Important:** This tool must be run as root to access raw sockets and set real-time thread priorities. A PREEMPT_RT kernel is required for RMP.

**Basic cyclic test (no NIC):**

```bash
sudo rmp-eval
```

**NIC test with EtherCAT drive (replace `enp2s0` with your NIC name):**

```bash
sudo rmp-eval --nic enp2s0
```

**Note:** The NIC must have an EtherCAT drive connected for this test.

**Check system configuration only (no timing tests):**

```bash
sudo rmp-eval --only-config
```

This runs all configuration checks and exits without starting timing threads.

Press `Ctrl+C` to stop the test and view final results.

## Example Output

```bash
sudo ./rmp-eval --nic enp2s0

CPU: 13th Gen Intel(R) Core(TM) i3-13100TE (4 logical, 4 physical)
Kernel: Linux 6.12.43+deb13-rt-amd64 #1 SMP PREEMPT_RT Debian 6.12.43-1 (2025-08-27) x86_64

System Checks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PREEMPT_RT active                   ✔️    /sys/kernel/realtime=1
Swap disabled                       ✔️    /proc/swaps empty
Timer Migration disabled            ✔️    timer_migration=0
RT throttling disabled              ✔️    sched_rt_runtime_us=-1
Clocksource stable                  ✔️    tsc

Core 3 Checks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RT core isolated                    ✔️    isolated list: 3
nohz_full on RT core                ✔️    nohz_full list: 3
rcu_nocbs includes RT core          ✔️    3
CPU governor = performance          ✔️    governor=performance
CPU current frequency               ❌   cur=800000 kHz, min=800000 kHz, max=4100000 kHz
irqaffinity excludes RT core        ✔️    0,1
No unrelated IRQs on RT core        ✔️    clean
SMT sibling isolated/disabled       ✔️    no sibling
Deep C-states capped                ✔️    intel_idle.max_cstate=0
Turbo/boost disabled                ❌   intel_pstate/no_turbo=0

NIC enp2s0 Checks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
NIC interface present               ✔️    exists
NIC link is UP                      ✔️    operstate=up
NIC is quiet                        ❌   v4=0, v6=1, def4=no, def6=no
NIC IRQs pinned to RT core          ✔️    all pinned to CPU3
RPS disabled on NIC                 ✔️    all zero masks

Target period: 1000 us

|          |       |  Great  |  Good   |  Poor   |   Bad    | Pathetic  | Max Latency  |
| Label    | Count | < 125us | < 250us | < 500us | < 1000us | >= 1000us |   us | index |
|----------+-------+---------+---------+---------+----------+-----------+------+-------+
| Sender   | 62367 |   62367 |       0 |       0 |        0 |         0 |    1 |   161 |
| Receiver | 62367 |   62367 |       0 |       0 |        0 |         0 |   10 | 61567 |
Duration: 00:01:02.445
  Sender max period: 1001µs at index 161 which is Great.
Receiver max period: 1010µs at index 61567 which is Great.
```

Columns:

- Label: Sender or Receiver thread (or Cyclic if no NIC)
- Count: Total observations (cycles)
- < 125us, < 250us, < 500us, < 1000us, >= 1000us: Latency deviation buckets
  - Colored (green → red) based on severity
  - Calculated as deviation from target period
- Max us: Worst-case latency deviation in microseconds (line 208: max - target)
- Max Index: Which iteration had the worst case

## Command-Line Options

```bash
./rmp-eval --help

Options:
--nic, -n                Network interface card name
--iterations, -i         Number of iterations (default: infinite)
--send-sleep, -s         Send sleep duration in microseconds (default: 1000)
--send-priority, -sp     Send thread priority (default: 42)
--receive-priority, -rp  Receive thread priority (default: 45)
--send-cpu, -sc          CPU core to use for the sender thread (default: last core)
--receive-cpu, -rc       CPU core to use for the receiver thread (default: last core)
--verbose, -v            Enable verbose output
--no-config, -nc         Skip system configuration checks
--only-config, -oc       Run system configuration checks only, then exit
--bucket-width, -b       Bucket width in microseconds for counting occurrences (default: auto).
--help, -h               Show this help message
--version                Show version information
```

## Building from Source

**Prerequisites:**

- CMake 3.15 or later
- GCC 12 or later
- C++20 support

**Build steps:**

```bash
# Configure
cmake -B build -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_C_COMPILER=gcc-12 \
      -DCMAKE_CXX_COMPILER=g++-12

# Build
cmake --build build

# Run
sudo ./build/rmp-eval
```

## FAQ

### What are the Sender and Receiver threads?

The **Sender** thread simulates the cyclic thread that sends EtherCAT frames at a fixed rate (default 1000µs/1kHz). The **Receiver** thread waits for and processes EtherCAT responses. These two threads are sufficient to identify worst-case latency in your system.

When no NIC is specified, only the Sender thread runs (labeled "Cyclic") to test basic cyclic timing performance.

### How does this relate to RMP's 8 threads?

While RMP/RMPNetwork uses 8 threads in total for its full operation, this evaluation tool focuses on testing the **worst-case latency** of the critical cyclic timing path. The 2-thread test (sender/receiver) is designed to stress-test the timing characteristics that matter most for real-time performance.

### Should both threads run on the same isolated CPU?

**Yes.** Both the sender and receiver threads should run on the same isolated CPU (default behavior). This matches how RMP operates - all 8 of RMP's threads normally run on a single isolated CPU. You can verify/change this with `--send-cpu` and `--receive-cpu` options if needed, but keeping them on the same core is the recommended configuration.

### Can RMP use multiple isolated CPUs?

RMP is designed to run all its threads on a single isolated CPU. This is the normal and expected configuration for optimal real-time performance.

### Should RMP be running during this test?

No. Do not run rmp-eval while RMP/RMPNetwork is running. This tool is meant to evaluate your hardware **before** deploying RMP, not to test alongside it. Running both simultaneously would interfere with the timing measurements.

### How long should I run the test?

We recommend initially running for at least **5-10 minutes** to capture various system states and potential latency spikes. Longer tests (24+ hours) can reveal issues that only occur under sustained load or periodic system activities. Press `Ctrl+C` to stop and view results.

### Should I test under load?

Yes. To get a realistic assessment, run the test while your system is under typical load conditions expected during RMP operation.

### What do the latency categories/buckets mean?

Based on the RMP [PC hardware latency requirements.](https://support.roboticsys.com/rmp/guide-pc-latency-jitter-bios.html)

- **Great** (< 125µs): Excellent for 1kHz operation, meets RMP requirements
- **Good** (< 250µs): Acceptable but approaching limits
- **Poor** (< 500µs): Marginal, may cause issues under load
- **Bad** (< 1000µs): Inadequate for reliable operation
- **Pathetic** (≥ 1000µs): Unacceptable, missed deadline

For 1kHz (1000µs period), aim for all samples in "Great" category with max latency < 125µs. The buckets will scale proportionally for different target periods. 

### Some config checks failed - can I still use my system?

The timing results table is the ultimate test. If your latencies are consistently in "Great" range (under load) despite some config warnings, your system is still suitable.

### Do I need a PREEMPT_RT kernel to run this tool?

The tool will run on non-RT kernels, but **PREEMPT_RT is required for RMP** itself. This evaluation tool checks for PREEMPT_RT and will report if it's missing. If you're evaluating hardware for RMP deployment, install a PREEMPT_RT kernel first to get accurate results.

### Why do I need an EtherCAT drive connected for NIC testing?

The receiver thread waits for actual EtherCAT responses to measure round-trip timing. Without a connected drive, the receiver will timeout and the test will fail. If you only want to test cyclic timing without NIC hardware, omit the `--nic` parameter to run in cyclic-only mode.

### Can I use this tool for non-RMP real-time applications?

Yes! While designed for RMP evaluation, this tool is useful for testing any Linux real-time system that requires:

- Sub-millisecond cyclic timing
- Low-latency EtherCAT communication
- Isolated CPU performance validation
- PREEMPT_RT kernel verification

Adjust `--send-sleep` to match your application's cycle time.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
