## 操作環境

- Alpine 3.15 ( 建議 3.12 以上 )
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

使用 Docker 建立 `alpine 3.13` Container

```sh
$ docker run -itd --name hdp alpine:3.13
$ docker exec -it hdp ash
```


更換 Alpine package repositories source

```sh
$ cat << EOF | tee /etc/apk/repositories > /dev/null 
http://dl-cdn.alpinelinux.org/alpine/v3.13/main
http://dl-cdn.alpinelinux.org/alpine/v3.13/community
http://dl-cdn.alpinelinux.org/alpine/v3.12/main
EOF
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
if defined(__sun) || defined(__GLIBC_PREREQ) && __GLIBC_PREREQ(2, 32)
MT-Safe under Solaris which doesn't support sys_errlist/sys_nerr
```

解決方法:

```
$ sed -ir 's/^#if defined\(__sun\).*/#if 1/' /tmp/hadoop-3.3.2-src/hadoop-common-project/hadoop-common/src/main/native/src/exception.c
$ sed -ri '211s/.*/\/\/assert(expr, file, line);/' /tmp/hadoop-3.3.2-src/hadoop-hdfs-project/hadoop-hdfs-native-client/src/main/native/libhdfspp/third_party/tr2/optional.hpp
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

