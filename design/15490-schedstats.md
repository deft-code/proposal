# Proposal: Runtime SchedStats

Author(s): Nick Terry

Last updated: 2016-05-10

Discussion at https://golang.org/issue/15490.

## Abstract

The SchedStats api is used to cheaply monitor scheduler performance within a
program. The intent is to enable large scale monitoring of a fleet of servers via
expvars or a similar interface.

## Background

The `runtime` package provides monitoring access to the allocator and garbage
collector via `runtime.MemStats`. Equivalent access to scheduler metrics are severly
limited: `GOMAXPROCS(0)` and `NumGoroutine()`.

A simplified set of goroutine states:

* **Blocked**
A blocked goroutine is waiting on some other goroutine or input. E.g. receiving from a `time.Ticker.C`.
* **Scheduled**
A scheduled goroutine is ready to run but no Proc has run it yet. E.g. after `time.sendTime` sends on the `time.Ticker.C`.
* **Executing**  
An executing goroutine is actively being run by a Proc. E.g. after the receive on `time.Ticker.C` completes.

## Proposal

SchedStats and it associated functions are in the `runtime` to allow
inspecting the scheduler. The implementation favors fast over exact metrics
since any returned metrics will be immediately stale. Goroutines are periodically sampled to
measure the delay between being scheduled and executing. The sampling frequency
is time based rather than cardinality based.

New scheduler metrics:
* Cumulative total of goroutines created.
* Cumulative times any goroutine executed.
* Number of executing goroutines. Almost always equal to the number of threads,
  but possible differing if some threads are spinning.
* Cumulative times any goroutine was scheduled. E.g. `runtime.goready()`
* Number of scheduled goroutines. The sum of each proc's runq and the global runq.
* Cumulative times a thread started. E.g. `runtime.startm()`
* Number of threads. All non-idle `m`.
* Samples of the delay between a goroutine being scheduled and executing.

The existing scheduler metrics in `runtime` are include in `SchedStats` for completeness.


## Rationale

`GODEBUG` provides access to similar scheduler details via `schedtrace` and
`scheddetail`. However, programmatic access to the metrics is cumbersome and ill
suited to large scale monitoring.

## Compatibility

Some metrics should not be included in the go1 guarantee, e.g. thread use.  The
concern is that a metric may cease to be meaningful as the scheduler
continues to evolve.

The unexported metrics are accessible via the `SchedStats` method `GetMetric(string) interface{}`. The
names of the metrics are listed in the comment with a warning about
their instability.

## Implementation

```
type SchedStats struct {
  TotalGoroutineExec uint64
  NumGoroutineExec   int

  TotalGoroutineSched uint64
  NumGoroutineSched   int

  TotalGoroutine uint64
  NumGoroutine   int

  TotalCGoCalls uint64
  NumCGoCalls   int

  TotalSysCalls uint64
  NumSysCalls   int

  MaxProcs int
  NumProcs int

  totalThreadStarts uint64
  numThreads        int

  GoroutineSchedNs [256]uint64 // circular buffer of recent goroutine execution delays, most recent at [(NumSamples+255)%256]
  NumSamples       int
  SampleHz         int
}

// GetMetric provides access to all scheduler metrics.
// All exported members can be retrieved.
//
// Some unexported metrics are also available:
// * totalThreadStarts int64
// * numThreads int
// Unexported metrics are not covered by the go1compat guarantee.
// 
// Returns nil if name is invalid.
//
// Example usage:
// var s runtime.SchedStats
// runtime.ReadScheStats(&s)
// x := s.GetMetric("TotalGoroutineExec").(uint64)
// y, ok := s.GetMetric("numThreads").(int)
func (s *SchedStats) GetMetric(name string) interface{}

func SetSchedulerProfileRate(hz int)

func ReadSchedStats(s *SchedStats)
```

Most of the metrics are maintained like `runtime.m.ncgocalls`. `ReadSchedStats`
loops over `runtime.allm` and accumulates the values. Currently
`runtime.m.ncgocalls` is racey on 32 bit architectures, but the race is only
evident once 2^31 cgo calls have been made. `SchedStats`'s other 64bit metrics are
similarly racey, except it is reasonable for `TotalGoroutineExec` to exceed 2^31
within a year, making the race more plausible. However, the race can only cause
the 64bit values to return a smaller value. This is an acceptable risk. Should
this change in the furture spinlock synchronization can be added.

The sampling is managed by the sysmon goroutine, though any regularly
executed goroutine is sufficient. Sysmon atomically sets a
global flag whenever a sample should be taken. When any goroutine is enqueued that
flag is atomically consumed and if true the current time is recorded in
`g.enqueuedTime`. Before a proc executes a goroutine it checks if `g.enqueueTime` is non-zero. If so
the delta is computed and synchronously added to the global samples array.
`g.enqueueTime` is cleared so the sample is only performed once.
`ReadSchedStats` synchronously copies the global sample array. The array synchronization will be
nearly contention free so a spinlock can be use instead of mutex.

##Bike Shedding

`SchedStats` tries to mirror `MemStats` as much as possible. This creates some
oddities.

* `NumCGoCall()` returns `int64` while `MemStats` uses `uint64`.
* `MemStats.TotalCGoCall` is equivalent to `runtime.NumCGoCall()` but the names differ.
* It feels more idiomatic to have `ReadSchedStats` return pointer rather than use an output parameter.
* Some of `SchedStats` members obviously duplicate existing `runtime` functions.
