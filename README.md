# Scala GraalVM Example Using Static Native Image

### Building the Project
You can accept he default responses to the configuration questions.

```
$ sbt -sbt-version 1.3.12 new http4s/http4s.g8 -b 0.21
[info] welcome to sbt 1.3.12 (Oracle Corporation Java 11.0.10)
[info] set current project to test (in build file:/home/sseighma/code/graalvm/scala-demos/quickstart/test/)
[info] set current project to test (in build file:/home/sseighma/code/graalvm/scala-demos/quickstart/test/)
name [quickstart]:
organization [com.example]:
package [com.example.quickstart]:
scala_version [2.13.4]:
sbt_version [1.4.7]:
http4s_version [0.21.16]:
circe_version [0.13.0]:
logback_version [1.2.3]:
munit_version [0.7.20]:
munit_cats_effect_version [0.13.0]:
graal_native_image [TRUE/false]:
is_linux_build [true/FALSE]:
scala_assembly_target [scala-2.13]:

Template applied in /home/sseighma/code/graalvm/scala-demos/quickstart/test/./quickstart
```
Next, change to the `quickstart` directory:
```
$ cd quickstart/
```
Run the project:
```
$ sbt run
[info] welcome to sbt 1.4.7 (Oracle Corporation Java 11.0.10)
[info] loading settings for project scala-graalvm-example-build from plugins.sbt ...
[info] loading project definition from /home/sseighma/code/graalvm/scala-demos/Scala-GraalVM-Example/project
[info] loading settings for project root from build.sbt ...
[info] set current project to quickstart (in build file:/home/sseighma/code/graalvm/scala-demos/Scala-GraalVM-Example/)
[info] running com.example.quickstart.Main
[ioapp-compute-0] INFO  o.h.b.c.n.NIO1SocketServerGroup - Service bound to address /0:0:0:0:0:0:0:0:8080
[ioapp-compute-0] INFO  o.h.s.b.BlazeServerBuilder -
  _   _   _        _ _
 | |_| |_| |_ _ __| | | ___
 | ' \  _|  _| '_ \_  _(_-<
 |_||_\__|\__| .__/ |_|/__/
             |_|
[ioapp-compute-0] INFO  o.h.s.b.BlazeServerBuilder - http4s v0.21.16 on blaze v0.14.14 started at http://[::]:8080/

```

Test the `hello/$USER` endpoint:
```
$ http http://localhost:8080/hello/$USER
HTTP/1.1 200 OK
Content-Length: 29
Content-Type: application/json
Date: Mon, 10 May 2021 18:15:54 GMT

{
    "message": "Hello, sseighma"
}
```

Assemble the project:
```
$ sbt assembly
```

Run the `jar` file to start the server:
```
$ java -jar target/scala-2.13/quickstart-assembly-0.0.1-SNAPSHOT.jar
[ioapp-compute-0] INFO  o.h.b.c.n.NIO1SocketServerGroup - Service bound to address /0:0:0:0:0:0:0:0:8080
[ioapp-compute-0] INFO  o.h.s.b.BlazeServerBuilder -
  _   _   _        _ _
 | |_| |_| |_ _ __| | | ___
 | ' \  _|  _| '_ \_  _(_-<
 |_||_\__|\__| .__/ |_|/__/
             |_|
[ioapp-compute-0] INFO  o.h.s.b.BlazeServerBuilder - http4s v0.21.16 on blaze v0.14.14 started at http://[::]:8080/

```

Test the `joke` endpoint:

```
$ http http://localhost:8080/joke
HTTP/1.1 200 OK
Content-Length: 74
Content-Type: application/json
Date: Mon, 10 May 2021 18:37:37 GMT

{
    "joke": "People who don't eat gluten are really going against the grain."
}
```
### Gather Benchmark Info

I'll be using [`wrk2`](https://github.com/giltene/wrk2) for the actual benchmarks and an extension called [`wrk2img`](https://github.com/PPACI/wrk2img) to create some nice graphs of our benchmark results.

Assuming you have both tools installed (and the server running), let's get started!
```
$ wrk --latency --rate 100 --connections 5 --threads 2 --duration 60s http://localhost:8080/joke > results/jar_resuts.txt
```

### Building a Static Native Image

See this [link](https://docs.oracle.com/en/graalvm/enterprise/21/docs/reference-manual/native-image/StaticImages/) for information on downloading and building `musl` and `zlibc`.

If you have `musl-gcc` on the path, you can build a native image statically linked against `muslc` with the following options: `--static --libc=musl`. To verify that `musl-gcc` is on the path, run `musl-gcc -v`:
```
$ musl-gcc -v
Using built-in specs.
Reading specs from /usr/local/musl/lib/musl-gcc.specs
rename spec cpp_options to old_cpp_options
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-redhat-linux/10/lto-wrapper
OFFLOAD_TARGET_NAMES=nvptx-none
OFFLOAD_TARGET_DEFAULT=1
Target: x86_64-redhat-linux
Configured with: ../configure --enable-bootstrap --enable-languages=c,c++,fortran,objc,obj-c++,ada,go,d,lto --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --with-bugurl=http://bugzilla.redhat.com/bugzilla --enable-shared --enable-threads=posix --enable-checking=release --enable-multilib --with-system-zlib --enable-__cxa_atexit --disable-libunwind-exceptions --enable-gnu-unique-object --enable-linker-build-id --with-gcc-major-version-only --with-linker-hash-style=gnu --enable-plugin --enable-initfini-array --with-isl --enable-offload-targets=nvptx-none --without-cuda-driver --enable-gnu-indirect-function --enable-cet --with-tune=generic --with-arch_32=i686 --build=x86_64-redhat-linux
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 10.3.1 20210422 (Red Hat 10.3.1-1) (GCC)

```

Build the statically linked native image:
```
$ native-image --static --libc=musl \
-H:+ReportExceptionStackTraces \
--allow-incomplete-classpath \
--no-fallback \
--initialize-at-build-time \
--enable-http \
--enable-https \
--enable-all-security-services \
--verbose -jar target/scala-2.13/quickstart-assembly-0.0.1-SNAPSHOT.jar \
http4sNative
```
You can confirm you built a statically linked native image executable:
```
$ file ./http4sNative
./http4sNative: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped
```

Run the native image version:
```
$ ./http4sNative
[ioapp-compute-0] INFO  o.h.b.c.n.NIO1SocketServerGroup - Service bound to address /0:0:0:0:0:0:0:0:8080
[ioapp-compute-0] INFO  o.h.s.b.BlazeServerBuilder -
  _   _   _        _ _
 | |_| |_| |_ _ __| | | ___
 | ' \  _|  _| '_ \_  _(_-<
 |_||_\__|\__| .__/ |_|/__/
             |_|
[ioapp-compute-0] INFO  o.h.s.b.BlazeServerBuilder - http4s v0.21.16 on blaze v0.14.14 started at http://[::]:8080/
```

Test the endpoint:
```
$ http http://localhost:8080/hello/$USER
HTTP/1.1 200 OK
Content-Length: 29
Content-Type: application/json
Date: Mon, 10 May 2021 18:15:54 GMT

{
    "message": "Hello, sseighma"
}

```

Once again, gather some benchmark data using the native image version:

```
$ wrk --latency --rate 100 --connections 5 --threads 2 --duration 60s http://localhost:8080/joke > results/native_resuts.txt
```

We'll view the results via a latency graph later in this example.


### Create the Benchmark Graph
Combine the two benchmarks and graph the results:
```
$ cat results/native_results.txt results/jar_results.txt | wrk2img --log -n "native", "jar" results/graph-final.png  >> /dev/null
```

![](//wsl$/Fedora/home/sseighma/code/graalvm/scala-demos/Scala-GraalVM-Example/results/graph.png)