## 操作環境

- Alpine 3.15
- Hadoop 3.3.2

## 事前準備

依照 [Alpine 官方文件](https://wiki.alpinelinux.org/wiki/Docker) 安裝 Docker 

``` sh
$ apk add docker
$ addgroup $USER docker
$ rc-update add docker boot
$ service docker start

$ docker --version
Docker version 20.10.11, build dea9396e184290f638ea873c76db7c80efd5a1d2
```

## 準備編譯環境

使用 Docker 建立 `alpine 3.15` Container

```sh
$ docker run -itd --name hdp alpine:3.15
$ docker exec -it hdp ash
```

安裝 Compile 所需使用到的套件

```sh
$ apk update && apk add \
bash \
fts \
autoconf \
automake \
build-base \
bzip2 \
libressl \
cmake \
curl \
fts \
fuse \
git \
libexecinfo \
libtirpc \
libtool \
maven \
openjdk8 \
snappy \
zlib \
zlib-dev \
findutils \
su-exec \
boost \
boost-dev \
libsasl \
doxygen \
cyrus-sasl \
cyrus-sasl-dev \
musl \
musl-dev \
linux-headers \
protobuf \
protobuf-dev \
zstd \
tar \
openssl \
openssl-dev \
gtest \
gtest-dev \
asio \
asio-dev \
libc6-compat \
gcompat \
syslinux \
syslinux-dev \
libressl-dev \
clang \
libexecinfo-dev \
libtirpc-dev
```

## 編譯 Hadoop

設定環境變數

```sh
$ export JAVA_HOME=/usr/lib/jvm/default-jvm/
$ export PATH=$PATH:/$JAVA_HOME/bin
```

下載 `Hadoop 3.3.2` source code

```sh
$ wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.2/hadoop-3.3.2-src.tar.gz
$ tar zxvf hadoop-3.3.2-src.tar.gz -C /tmp
$ cd /tmp/hadoop-3.3.2-src
```

修正 Compile 時會遇到的錯誤

```sh
$ sed -ri 's/^#if defined\(__sun\).*/#if 1/' /tmp/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c	
$ wget https://raw.githubusercontent.com/chriskohlhoff/asio/443bc17d13eb5e37de780ea6e23157493cf7b3b9/asio/include/asio/impl/error_code.ipp -O /tmp/hadoop-3.3.2-src/hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/libhdfspp/third_party/asio-1.10.2/include/asio/impl/error_code.ipp
$ sed -ri 's/^#warning.*//' /usr/include/sys/poll.h
$ sed -ri '211s/.*/\/\/assert(expr, file, line);/' /tmp/hadoop-3.3.2-src/hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/libhdfspp/third_party/tr2/optional.hpp
$ sed -ri 's/(rt pthread)/execinfo \1/' hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/CMakeLists.txt
```

開始編譯
```sh
$ mvn clean install -DskipTests -DskipShade
$ mvn package -Pdist,native -DskipTests -Dtar
```

成功後在以下目錄可以看到 hadoop-3.3.2.tar.gz

```sh
$ ls /tmp/hadoop-3.3.2/hadoop-dist/target
```	

退出 Container，並使用 `docker cp` 取得編譯好的 .tar.gz 檔

```sh
$ exit
$ docker cp hdp:/tmp/hadoop-3.3.2-src/hadoop-dist/target/hadoop-3.3.2.tar.gz .
```


## Error Log

錯誤訊息:
```
warning redirecting incorrect #include <sys/poll.h> to <poll.h>
```

解決方法:
```
$ sed -ri 's/^#warning.*//' /usr/include/sys/poll.h
```

---

錯誤訊息:
```
# _assert not defind or return strerror_r(value, buf, sizeof(buf));
```

解決方法:
```
$ wget https://raw.githubusercontent.com/chriskohlhoff/asio/443bc17d13eb5e37de780ea6e23157493cf7b3b9/asio/include/asio/impl/error_code.ipp -O /tmp/hadoop-3.3.2-src/hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/libhdfspp/third_party/asio-1.10.2/include/asio/impl/error_code.ipp
$ sed -ri '211s/.*/\/\/assert(expr, file, line);/' /tmp/hadoop-3.3.2-src/hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/libhdfspp/third_party/tr2/optional.hpp
```

---

錯誤訊息:

```
//tar: unrecognized option: B
```

解決方法:

```
$ apk del tar && apk add tar
```

---

錯誤訊息:
```
...backtrace_symbol...
```

解決方法:

```
$ sed -ri 's/(rt pthread)/execinfo \1/' hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/CMakeLists.txt
```

---

錯誤訊息:

```
TIRPC_INCLUDE_DIRS-NOTFOUND
```

解決方法:

```
$ apk add libtirpc-dev
```

---

錯誤訊息:
```
[ERROR] Failed to execute goal on project hadoop-yarn-applications-catalog-webapp: Could not resolve dependencies for project org.apache.hadoop:hadoop-yarn-applications-catalog-webapp:war:3.3.2: Failed to collect dependencies at org.apache.solr:solr-core:jar:7.7.0 -> org.restlet.jee:org.restlet:jar:2.3.0: Failed to read artifact descriptor for org.restlet.jee:org.restlet:jar:2.3.0: Could not transfer artifact org.restlet.jee:org.restlet:pom:2.3.0 from/to maven-default-http-blocker (http://0.0.0.0/): Blocked mirror for repositories: [maven-restlet (http://maven.restlet.org, default, releases+snapshots), apache.snapshots (http://repository.apache.org/snapshots, default, disabled)] -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal on project hadoop-yarn-applications-catalog-webapp: Could not resolve dependencies for project org.apache.hadoop:hadoop-yarn-applications-catalog-webapp:war:3.3.2: Failed to collect dependencies at org.apache.solr:solr-core:jar:7.7.0 -> org.restlet.jee:org.restlet:jar:2.3.0
```

解決方法:

參考自: https://github.com/apache/hadoop/pull/2939#issuecomment-826115290

請依照 **https://github.com/apache/hadoop/pull/2939#issuecomment-826115290** 來修改程式碼

```
$ nano  hadoop-project/pom.xml

找到 <solr.version>7.7.0</solr.version>
修改為 <solr.version>8.8.2</solr.version>
```

```
$ nano hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-catalog/hadoop-yarn-applications-catalog-webapp/pom.xml

找到第 112 行並添加
                <exclusion>
                    <groupId>org.eclipse.jetty</groupId>
                    <artifactId>jetty-http</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.eclipse.jetty</groupId>
                    <artifactId>jetty-client</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.eclipse.jetty</groupId>
                    <artifactId>jetty-io</artifactId>
                </exclusion>

接著找到 148 行並添加
                <exclusion>
                    <groupId>org.eclipse.jetty</groupId>
                    <artifactId>jetty-client</artifactId>
                </exclusion>
```

```
$ nano hadoop-yarn-project/hadoop-yarn/hadoop-yarn-applications/hadoop-yarn-applications-catalog/hadoop-yarn-applications-catalog-webapp/src/test/java/org/apache/hadoop/yarn/appcatalog/application/EmbeddedSolrServerFactory.java

刪除 85,86 行
  //  final SolrResourceLoader loader = new SolrResourceLoader(
  //     solrHomeDir.toPath());
  
接著刪除 88，並替換
        // "embeddedSolrServerNode", loader)
        "embeddedSolrServerNode", solrHomeDir.toPath())

```
---

錯誤訊息:

```
[ERROR] Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.3.1:exec (check-jar-contents) on project hadoop-client-check-invariants: Command execution failed.: Cannot run program "bash" (in directory "/workspace/hadoop-3.3.2-src/hadoop-client-modules/hadoop-client-check-invariants/target/test-classes"): error=2, No such file or directory -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.3.1:exec (check-jar-contents) on project hadoop-client-check-invariants: Command execution failed.
```

解決方法:

```
$ apk add bash
```
---

錯誤訊息:

```
[ERROR] Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: Error executing CMake: Cannot run program "cmake" (in directory "/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native"): error=2, No such file or directory -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: Error executing CMake
```

解決方法:
```
apk add cmake
```
---

錯誤訊息:

```
[WARNING] CMake Warning (dev) in CMakeLists.txt:
[WARNING]   No project() command is present.  The top-level CMakeLists.txt file must
[WARNING]   contain a literal, direct call to the project() command.  Add a line of
[WARNING]   code such as
[WARNING]
[WARNING]     project(ProjectName)
[WARNING]
[WARNING]   near the top of the file, but after cmake_minimum_required().
[WARNING]
[WARNING]   CMake is pretending there is a "project(Project)" command on the first
[WARNING]   line.
[WARNING] This warning is for project developers.  Use -Wno-dev to suppress it.
[WARNING]
[WARNING] CMake Error: CMake was unable to find a build program corresponding to "Unix Makefiles".  CMAKE_MAKE_PROGRAM is not set.  You probably need to select a different build tool.
[WARNING] CMake Error: CMAKE_C_COMPILER not set, after EnableLanguage
[WARNING] CMake Error: CMAKE_CXX_COMPILER not set, after EnableLanguage
[WARNING] -- Configuring incomplete, errors occurred!
[WARNING] See also "/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native/CMakeFiles/CMakeOutput.log".
............
[ERROR] Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: CMake failed with error code 1 -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: CMake failed with error code 1
```

解決方法:

```
apk add build-bash
```

---

錯誤訊息:

```
[WARNING] CMake Warning (dev) in CMakeLists.txt:
[WARNING]   No project() command is present.  The top-level CMakeLists.txt file must
[WARNING]   contain a literal, direct call to the project() command.  Add a line of
[WARNING]   code such as
[WARNING]
[WARNING]     project(ProjectName)
[WARNING]
[WARNING]   near the top of the file, but after cmake_minimum_required().
[WARNING]
[WARNING]   CMake is pretending there is a "project(Project)" command on the first
[WARNING]   line.
[WARNING] This warning is for project developers.  Use -Wno-dev to suppress it.
[WARNING]
[WARNING] -- The C compiler identification is GNU 10.3.1
[WARNING] -- The CXX compiler identification is GNU 10.3.1
[WARNING] -- Detecting C compiler ABI info
[WARNING] -- Detecting C compiler ABI info - done
[WARNING] -- Check for working C compiler: /usr/bin/cc - skipped
[WARNING] -- Detecting C compile features
[WARNING] -- Detecting C compile features - done
[WARNING] -- Detecting CXX compiler ABI info
[WARNING] -- Detecting CXX compiler ABI info - done
[WARNING] -- Check for working CXX compiler: /usr/bin/c++ - skipped
[WARNING] -- Detecting CXX compile features
[WARNING] -- Detecting CXX compile features - done
[WARNING] -- Looking for pthread.h
[WARNING] -- Looking for pthread.h - found
[WARNING] -- Performing Test CMAKE_HAVE_LIBC_PTHREAD
[WARNING] -- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Success
[WARNING] -- Found Threads: TRUE
[WARNING] JAVA_HOME=, JAVA_JVM_LIBRARY=JAVA_JVM_LIBRARY-NOTFOUND
[WARNING] JAVA_INCLUDE_PATH=JAVA_INCLUDE_PATH-NOTFOUND, JAVA_INCLUDE_PATH2=JAVA_INCLUDE_PATH2-NOTFOUND
[WARNING] CMake Error at /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/HadoopJNI.cmake:86 (message):
[WARNING]   Failed to find a viable JVM installation under JAVA_HOME.
[WARNING] Call Stack (most recent call first):
[WARNING]   CMakeLists.txt:42 (include)
[WARNING]
[WARNING]
[WARNING] -- Configuring incomplete, errors occurred!
[WARNING] See also "/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native/CMakeFiles/CMakeOutput.log".
...
[ERROR] Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: CMake failed with error code 1 -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: CMake failed with error code 1
```

解決方法:

```
export JAVA_HOME=/usr/lib/jvm/default-jvm/
export PATH=$PATH:/$JAVA_HOME/bin
```

---

錯誤訊息:

```
[WARNING] CMake Warning (dev) in CMakeLists.txt:
[WARNING]   No project() command is present.  The top-level CMakeLists.txt file must
[WARNING]   contain a literal, direct call to the project() command.  Add a line of
[WARNING]   code such as
[WARNING]
[WARNING]     project(ProjectName)
[WARNING]
[WARNING]   near the top of the file, but after cmake_minimum_required().
[WARNING]
[WARNING]   CMake is pretending there is a "project(Project)" command on the first
[WARNING]   line.
[WARNING] This warning is for project developers.  Use -Wno-dev to suppress it.
[WARNING]
[WARNING] JAVA_HOME=, JAVA_JVM_LIBRARY=/usr/lib/jvm/default-jvm/jre/lib/amd64/server/libjvm.so
[WARNING] JAVA_INCLUDE_PATH=/usr/lib/jvm/default-jvm/include, JAVA_INCLUDE_PATH2=/usr/lib/jvm/default-jvm/include/linux
[WARNING] Located all JNI components successfully.
[WARNING] -- Found JNI: /usr/lib/jvm/default-jvm/jre/lib/amd64/libjawt.so
[WARNING] CMake Error at /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:230 (message):
[WARNING]   Could NOT find ZLIB (missing: ZLIB_INCLUDE_DIR)
[WARNING] Call Stack (most recent call first):
[WARNING]   /usr/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:594 (_FPHSA_FAILURE_MESSAGE)
[WARNING]   /usr/share/cmake/Modules/FindZLIB.cmake:120 (FIND_PACKAGE_HANDLE_STANDARD_ARGS)
[WARNING]   CMakeLists.txt:47 (find_package)
[WARNING]
[WARNING]
[WARNING] -- Configuring incomplete, errors occurred!
[WARNING] See also "/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native/CMakeFiles/CMakeOutput.log".
...
[ERROR] Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: CMake failed with error code 1 -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-common: CMake failed with error code 1
```

解決方法:

```
apk add zlib-dev
```

---

錯誤訊息:

```
[WARNING] [ 55%] Linking C executable test_bulk_crc32
[WARNING] /usr/bin/cmake -E cmake_link_script CMakeFiles/test_bulk_crc32.dir/link.txt --verbose=1
[WARNING] /usr/bin/cc  -g -O2 -Wall -pthread -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE -rdynamic CMakeFiles/test_bulk_crc32.dir/main/native/src/org/apache/hadoop/util/bulk_crc32.c.o CMakeFiles/test_bulk_crc32.dir/main/native/src/org/apache/hadoop/util/bulk_crc32_x86.c.o CMakeFiles/test_bulk_crc32.dir/main/native/src/test/org/apache/hadoop/util/test_bulk_crc32.c.o -o test_bulk_crc32
[WARNING] make[2]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native'
[WARNING] [ 55%] Built target test_bulk_crc32
[WARNING] make[2]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native'
[WARNING] make[2]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native'
[WARNING] make[1]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/target/native'
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c: In function 'terror':
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:114:64: error: missing binary operator before token "("
[WARNING]   114 | #if defined(__sun) || defined(__GLIBC_PREREQ) && __GLIBC_PREREQ(2, 32)
[WARNING]       |                                                                ^
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:118:34: error: 'sys_nerr' undeclared (first use in this function)
[WARNING]   118 |   if ((errnum < 0) || (errnum >= sys_nerr)) {
[WARNING]       |                                  ^~~~~~~~
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:118:34: note: each undeclared identifier is reported only once for each function it appears in
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:121:10: error: 'sys_errlist' undeclared (first use in this function)
[WARNING]   121 |   return sys_errlist[errnum];
[WARNING]       |          ^~~~~~~~~~~
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:123:1: warning: control reaches end of non-void function [-Wreturn-type]
[WARNING]   123 | }
[WARNING]       | ^
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c: In function 'terror':
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:114:64: error: missing binary operator before token "("
[WARNING]   114 | #if defined(__sun) || defined(__GLIBC_PREREQ) && __GLIBC_PREREQ(2, 32)
[WARNING]       |                                                                ^
[WARNING] make[2]: *** [CMakeFiles/hadoop_static.dir/build.make:76: CMakeFiles/hadoop_static.dir/main/native/src/exception.c.o] Error 1
[WARNING] make[2]: *** Waiting for unfinished jobs....
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:118:34: error: 'sys_nerr' undeclared (first use in this function)
[WARNING]   118 |   if ((errnum < 0) || (errnum >= sys_nerr)) {
[WARNING]       |                                  ^~~~~~~~
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:118:34: note: each undeclared identifier is reported only once for each function it appears in
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:121:10: error: 'sys_errlist' undeclared (first use in this function)
[WARNING]   121 |   return sys_errlist[errnum];
[WARNING]       |          ^~~~~~~~~~~
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c:123:1: warning: control reaches end of non-void function [-Wreturn-type]
[WARNING]   123 | }
[WARNING]       | ^
[WARNING] make[2]: *** [CMakeFiles/hadoop.dir/build.make:76: CMakeFiles/hadoop.dir/main/native/src/exception.c.o] Error 1
[WARNING] make[2]: *** Waiting for unfinished jobs....
[WARNING] make[1]: *** [CMakeFiles/Makefile2:87: CMakeFiles/hadoop_static.dir/all] Error 2
[WARNING] make[1]: *** Waiting for unfinished jobs....
[WARNING] make[1]: *** [CMakeFiles/Makefile2:139: CMakeFiles/hadoop.dir/all] Error 2
[WARNING] make: *** [Makefile:91: all] Error 2
```

解決方法:

```
$ sed -ir 's/^#if defined(__sun).*/#if 1/g' hadoop-common-project/hadoop-common/src/main/native/src/exception.c
$ sed -ri '211s/.*/\/\/assert(expr, file, line);/' hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/libhdfspp/third_party/tr2/optional.hpp
```

---

錯誤訊息:

```
[WARNING] [ 21%] Building CXX object CMakeFiles/nativetask.dir/main/native/src/lib/Streams.cc.o
[WARNING] /usr/bin/c++ -Dnativetask_EXPORTS -I/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/target/native/javah -I/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src -I/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/util -I/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib -I/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/test -I/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src -I/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/target/native -I/usr/lib/jvm/default-jvm/include -I/usr/lib/jvm/default-jvm/include/linux -isystem /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/../../../../hadoop-common-project/hadoop-common/src/main/native/gtest/include -g -O2 -Wall -pthread -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE -DNDEBUG -DSIMPLE_MEMCPY -fno-strict-aliasing -fsigned-char -fPIC -MD -MT CMakeFiles/nativetask.dir/main/native/src/lib/Streams.cc.o -MF CMakeFiles/nativetask.dir/main/native/src/lib/Streams.cc.o.d -o CMakeFiles/nativetask.dir/main/native/src/lib/Streams.cc.o -c /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib/Streams.cc
[WARNING] make[2]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/target/native'
[WARNING] make[2]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/target/native'
[WARNING] make[1]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/target/native'
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib/NativeTask.cc:19:10: fatal error: execinfo.h: No such file or directory
[WARNING]    19 | #include <execinfo.h>
[WARNING]       |          ^~~~~~~~~~~~
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib/NativeTask.cc:19:10: fatal error: execinfo.h: No such file or directory
[WARNING]    19 | #include <execinfo.h>
[WARNING]       |          ^~~~~~~~~~~~
[WARNING] compilation terminated.
[WARNING] compilation terminated.
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib/NativeObjectFactory.cc:21:10: fatal error: execinfo.h: No such file or directory
...
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for Apache Hadoop MapReduce NativeTask 3.3.2:
[INFO]
[INFO] Apache Hadoop MapReduce NativeTask ................. FAILURE [ 10.278 s]
[INFO] Apache Hadoop MapReduce Uploader ................... SKIPPED
[INFO] Apache Hadoop MapReduce Examples ................... SKIPPED
[INFO] Apache Hadoop MapReduce ............................ SKIPPED
...
[ERROR] Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-mapreduce-client-nativetask: make failed with error code 2 -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-mapreduce-client-nativetask: make failed with error code 2
    at org.apache.maven.lifecycle.internal.MojoExecutor.execute (MojoExecutor.java:215)
    at org.apache.maven.lifecycle.internal.MojoExecutor.execute (MojoExecutor.java:156)
    at org.apache.maven.lifecycle.internal.MojoExecutor.execute (MojoExecutor.java:148)
    at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject (LifecycleModuleBuilder.java:117)
    at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject (LifecycleModuleBuilder.java:81)
    at org.apache.maven.lifecycle.internal.builder.singlethreaded.SingleThreadedBuilder.build (SingleThreadedBuilder.java:56)
```

解決方法:

```
apk add libexecinfo-dev
```

---

錯誤資訊:

```
[WARNING] /usr/bin/cmake -E cmake_link_script CMakeFiles/nttest.dir/link.txt --verbose=1
[WARNING] /usr/bin/c++  -g -O2 -Wall -pthread -D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE -DNDEBUG -DSIMPLE_MEMCPY -fno-strict-aliasing -fsigned-char -rdynamic CMakeFiles/nttest.dir/main/native/test/lib/TestByteArray.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestByteBuffer.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestComparatorForDualPivotQuickSort.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestComparatorForStdSort.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestFixSizeContainer.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestMemoryPool.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestIterator.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestKVBuffer.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestMemBlockIterator.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestMemoryBlock.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestPartitionBucket.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestReadBuffer.cc.o CMakeFiles/nttest.dir/main/native/test/lib/TestReadWriteBuffer.cc.o CMakeFiles/nttest.dir/main/native/test/util/TestChecksum.cc.o CMakeFiles/nttest.dir/main/native/test/util/TestStringUtil.cc.o CMakeFiles/nttest.dir/main/native/test/util/TestWritableUtils.cc.o CMakeFiles/nttest.dir/main/native/test/TestCommand.cc.o CMakeFiles/nttest.dir/main/native/test/TestConfig.cc.o CMakeFiles/nttest.dir/main/native/test/TestCounter.cc.o CMakeFiles/nttest.dir/main/native/test/TestCompressions.cc.o CMakeFiles/nttest.dir/main/native/test/TestFileSystem.cc.o CMakeFiles/nttest.dir/main/native/test/TestIFile.cc.o CMakeFiles/nttest.dir/main/native/test/TestPrimitives.cc.o CMakeFiles/nttest.dir/main/native/test/TestSort.cc.o CMakeFiles/nttest.dir/main/native/test/TestMain.cc.o CMakeFiles/nttest.dir/main/native/test/test_commons.cc.o -o test/nttest  -Wl,-rpath,/usr/lib/jvm/default-jvm/jre/lib/amd64/server target/usr/local/lib/libnativetask.a libgtest.a -ldl -lrt -lpthread -lz /usr/lib/jvm/default-jvm/jre/lib/amd64/server/libjvm.so
[WARNING] make[2]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/target/native'
[WARNING] make[1]: Leaving directory '/workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/target/native'
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/util/StringUtil.cc:39:21: warning: invalid suffix on literal; C++11 requires a space between literal and string macro [-Wliteral-suffix]
[WARNING]    39 |   snprintf(tmp, 32, "%"PRId64, v);
[WARNING]       |                     ^
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/util/StringUtil.cc:45:21: warning: invalid suffix on literal; C++11 requires a space between literal and string macro [-Wliteral-suffix]
[WARNING]    45 |   snprintf(tmp, 32, "%%%c%"PRId64""PRId64, pad, len);
[WARNING]       |                     ^
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/util/StringUtil.cc:45:34: warning: invalid suffix on literal; C++11 requires a space between literal and string macro [-Wliteral-suffix]
[WARNING]    45 |   snprintf(tmp, 32, "%%%c%"PRId64""PRId64, pad, len);
[WARNING]       |                                  ^
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/util/StringUtil.cc:51:21: warning: invalid suffix on literal; C++11 requires a space between literal and string macro [-Wliteral-suffix]
[WARNING]    51 |   snprintf(tmp, 32, "%"PRIu64, v);
[WARNING]       |                     ^
...
[WARNING] /usr/lib/gcc/x86_64-alpine-linux-musl/10.3.1/../../../../x86_64-alpine-linux-musl/bin/ld: /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib/NativeObjectFactory.cc:49: undefined reference to `backtrace_symbols_fd'
[WARNING] /usr/lib/gcc/x86_64-alpine-linux-musl/10.3.1/../../../../x86_64-alpine-linux-musl/bin/ld: target/usr/local/lib/libnativetask.a(NativeTask.cc.o): in function `NativeTask::HadoopException::HadoopException(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)':
[WARNING] /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib/NativeTask.cc:68: undefined reference to `backtrace'
[WARNING] /usr/lib/gcc/x86_64-alpine-linux-musl/10.3.1/../../../../x86_64-alpine-linux-musl/bin/ld: /workspace/hadoop-3.3.2-src/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/main/native/src/lib/NativeTask.cc:69: undefined reference to `backtrace_symbols'
[WARNING] collect2: error: ld returned 1 exit status
[WARNING] make[2]: *** [CMakeFiles/nttest.dir/build.make:500: test/nttest] Error 1
[WARNING] make[1]: *** [CMakeFiles/Makefile2:90: CMakeFiles/nttest.dir/all] Error 2
[WARNING] make: *** [Makefile:91: all] Error 2
...
[WARNING] collect2: error: ld returned 1 exit status
[WARNING] make[2]: *** [CMakeFiles/nttest.dir/build.make:500: test/nttest] Error 1
[WARNING] make[1]: *** [CMakeFiles/Makefile2:90: CMakeFiles/nttest.dir/all] Error 2
[WARNING] make: *** [Makefile:91: all] Error 2
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for Apache Hadoop MapReduce NativeTask 3.3.2:
[INFO]
[INFO] Apache Hadoop MapReduce NativeTask ................. FAILURE [ 21.484 s]
...
[ERROR] Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-mapreduce-client-nativetask: make failed with error code 2 -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.hadoop:hadoop-maven-plugins:3.3.2:cmake-compile (cmake-compile) on project hadoop-mapreduce-client-nativetask: make failed with error code 2
    at org.apache.maven.lifecycle.internal.MojoExecutor.execute (MojoExecutor.java:215)
    at org.apache.maven.lifecycle.internal.MojoExecutor.execute (MojoExecutor.java:156)
    at org.apache.maven.lifecycle.internal.MojoExecutor.execute (MojoExecutor.java:148)
    at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject (LifecycleModuleBuilder.java:117)
    at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject (LifecycleModuleBuilder.java:81)
    at org.apache.maven.lifecycle.internal.builder.singlethreaded.SingleThreadedBuilder.build (SingleThreadedBuilder.java:56)
```

解決方法:

```
sed -ri 's/(rt pthread)/execinfo \1/' hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-nativetask/src/CMakeLists.txt
```

---

錯誤資訊:

```
[WARNING] -- Found OpenSSL: /usr/lib/libcrypto.so (found version "2.0.0")
[WARNING] -- Checking for module 'libtirpc'
[WARNING] --   Package 'libtirpc', required by 'virtual:world', not found
[WARNING] -- Looking for dlopen in dl
[WARNING] -- Looking for dlopen in dl - found
[WARNING] -- Configuring done
[WARNING] CMake Error: The following variables are used in this project, but they are set to NOTFOUND.
[WARNING] Please set them or make sure they are set and tested correctly in the CMake files:
[WARNING] TIRPC_INCLUDE_DIRS
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]    used as include directory in directory /workspace/hadoop-3.3.2-src/hadoop-tools/hadoop-pipes/src
[WARNING]
[WARNING] CMake Error in CMakeLists.txt:
[WARNING]   Found relative path while evaluating include directories of "hadooppipes":
[WARNING]
[WARNING]     "TIRPC_INCLUDE_DIRS-NOTFOUND"
[WARNING]
[WARNING]
[WARNING]
[WARNING] CMake Error in CMakeLists.txt:
[WARNING]   Found relative path while evaluating include directories of "hadooputils":
[WARNING]
[WARNING]     "TIRPC_INCLUDE_DIRS-NOTFOUND"
```

解決方法:

```
apk add libtirpc-dev
```
