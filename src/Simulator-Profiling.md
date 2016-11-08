
## Profiling your simulator test

To determine where e.g. time is spend or other resources are being used, you want to profile your application. The recommended way to profile is using the Java Flight Recorder (JFR) which is only available in the Oracle JVM's. The JFR unlike other commercial profilers like JProbe and Yourkit, doesn't make use of sampling or instrumentation and hooks into some internal API's and is quite reliable and causes very little overhead. The problem with most other profilers is that they distort the numbers and frequently point you in the wrong direction; especially when IO or concurrency is involved. Most of the recent performance improvements in Hazelcast are based on using JFR.

To enable the JFR, the JVM settings for the member or client need to be modified depending on what needs to be profiled.

```
JFR_ARGS="-XX:+UnlockCommercialFeatures  \
          -XX:+FlightRecorder \
          -XX:StartFlightRecording=duration=120m,filename=recording.jfr  \
          -XX:+UnlockDiagnosticVMOptions \
          -XX:+DebugNonSafepoints"

coordinator --members 1 \
            --workerVmOptions "$JFR_ARGS" \
            --clients 1 \
            --clientVmOptions "$JFR_ARGS" \
            sometest.properties
```
In the above example, but client and members are configured with JFR. Once the Simulator test has completed, all artifacts including the JFR files, are downloaded. The JFR files can be opened using the Java Mission Control command 'jmc'.

### Reducing fluctuations

Fore more stable performance numbers, set the minimum and maximum heap size to the same value, e.g.

```
coordinator --members 1 \
            --workerVmOptions "-Xmx4g -Xms4g" \
            --clients 1 \
            --clientVmOptions "-Xmx1g -Xms1g" \
            sometest.properties
```

Also set the minumum cluster size to the expected number of members using the following property:
```
-Dhazelcast.initial.min.cluster.size=4
```
This prevents Hazelcast cluster to start before the miniumum number of nodes has been reached. Otherwise the benchmark numbers of the tests can be distorted due to partition migrations during the test. Especially with a large number of partitions and short tests, this can lead to a very big impact on the benchmark numbers.

### Enabling diagnostics

Hazelcast 3.7+ has a diagnostics systems which provides detailed insights what is happening inside the client or server HazelcastInstance. It is designed to run in production and has very little performance overhead. It has so little overhead that we always enable it when doing benchmarks.

```
coordinator --members 1 \
            --workerVmOptions "-Dhazelcast.diagnostics.enabled=true \
                               -Dhazelcast.diagnostics.metric.level=info \
                               -Dhazelcast.diagnostics.invocation.sample.period.seconds=30 \
                               -Dhazelcast.diagnostics.pending.invocations.period.seconds=30 \
                               -Dhazelcast.diagnostics.slowoperations.period.seconds=30" \
            --clients 1 \
            --clientVmOptions "-Dhazelcast.diagnostics.enabled=true \
                               -Dhazelcast.diagnostics.metric.level=info" \
            sometest.properties
```

Using the above example, both client and server have diagnostics enabled. And both will write a diagnostics file. Once the simulator run completes and the artifacts are downloaded, the diagnostics files can be analyzed.

### Enabling different profilers or other startup customizations

If you want to use a different profiler than JFR and you require more than simple JVM args, or you want to play with features like numactrl, OpenOnload etc, you need to override the worker startup script. This is done by copying the startup script to the working directory. For example to modify a member worker:

```
cp $SIMULATOR_HOME/conf/worker-hazelcast-member.sh .
```

This bash script control the startup of the member. This particular file also contains full examples for:
- Yourkit
- Intel VTune
- Linux Perf
- HProf
- numactrl
- dstat
- OpenOnload

It also allows for experimenting with different profilers like honest-profiler, https://github.com/RichardWarburton/honest-profiler.

To upload artifacts required, create an 'upload' directory in the working directory. This upload directory will automatically be copied to all worker machines can be be found in the parent directory of the worker, so ../upload/someartifact. 
            