# 如何在 Apple M1 上使用 Apache Pulsar

## 从 RocksDB 说起

Apache Pulsar 将在 2.11 版本升级 RocksDB[^1]（ 6.10.2 -> 6.29.4.1），届时将原生支持在 Apple M1 上运行 Pulsar。

[^1]: RocksDB 用来存储 ledger 的索引。

在这之前的 Pulsar 版本，因为依赖的 RocksDB 尚不支持 osx-arm64 架构，这会让 Bookie 在启动阶段报一个 `UnsatisfiedLinkError`错误，导致无法正确启动。

解压 `org.rocksdb-rocksdbjni-6.10.2.jar` ，可以看到仅有一个支持 osx-x86 架构的  librocksdbjni-osx.jnilib 包。

```shell
rocksdbjni-6.10.2
├── HISTORY-JAVA.md
├── META-INF
├── librocksdbjni-linux-aarch64-musl.so
├── librocksdbjni-linux-aarch64.so
├── librocksdbjni-linux-ppc64le-musl.so
├── librocksdbjni-linux-ppc64le.so
├── librocksdbjni-linux32-musl.so
├── librocksdbjni-linux32.so
├── librocksdbjni-linux64-musl.so
├── librocksdbjni-linux64.so
├── librocksdbjni-osx.jnilib
├── librocksdbjni-win64.dll
└── org
```

终于：

> - [Java: Support Apple Silicon/M1 machines #7720](https://github.com/facebook/rocksdb/issues/7720#issuecomment-1079648907) 
>
> "RocksDB 6.29.4.1 has been released to Maven Central which should now work on M1 macs."

Apache Pulsar 也很快做了跟进：

> - [Upgrade Rocksdb to 6.29.4.1 #14886](https://github.com/apache/pulsar/pull/14886)
>
> "Upgrade to newest version of RocksDb which has support for Mac with M1 architecture."

在最新的 2.11.0-SNAPSHOT里，rocksdb 版本已经确定在 ：`<rocksdb.version>6.29.4.1</rocksdb.version>` 。

解压 `rocksdbjni-6.29.4.1.jar`，可以看到增加了对 `osx-arm64` 对支持。

```shell
cksdbjni-6.29.4.1
├── HISTORY-JAVA.md
├── META-INF
├── librocksdbjni-linux-aarch64-musl.so
├── librocksdbjni-linux-aarch64.so
├── librocksdbjni-linux-ppc64le-musl.so
├── librocksdbjni-linux-ppc64le.so
├── librocksdbjni-linux-s390x-musl.so
├── librocksdbjni-linux-s390x.so
├── librocksdbjni-linux32-musl.so
├── librocksdbjni-linux32.so
├── librocksdbjni-linux64-musl.so
├── librocksdbjni-linux64.so
├── librocksdbjni-osx-arm64.jnilib
├── librocksdbjni-osx-x86_64.jnilib
├── librocksdbjni-win64.dll
└── org
```



## 推荐使用最新的 2.11 master 分支

所以在目前的时间点，在 M1 环境运行 Apache Pulsar 最佳的方案是使用最新的 Master 分支代码（2.11.0-SNAPSHOT）。不过目前还有点[兼容性问题][1]。


## 老版本如何使用


除此之外，我们可以分环境来说明，如何在 M1 下使用老版本 Apache Pulsar：

1. **裸机环境（使用 arm64 JDK）**：可将 `lib/rocksdbjni-6.10.2.jar` 替换 `rocksdbjni-6.29.4.1.jar`，可兼容大部分消息场景（已简单测试 2.10、2.9.3 和 2.8.2 版本）。处理 Ledger 索引、Pulsar IO 和一些客户端可能会有问题。
2. **裸机环境（使用 x86 JDK）**：可直接使用默认的 apache pulsar（推荐方案）。推荐使用 [sdkman][3]工具，通过配置 `sdkman_rosetta2_compatible=true` ，让本地 JDK 通过 rosetta2 来运行 Pulsar。配置步骤参考链接 [2]。
3. **Docker 环境（arm64）**：可自行制作 arm64 的 pulsar 镜像，注意也可以升级下 rocksdb。也可以催促下官方的 arm64 镜像[7]。
4. **Docker 环境（模拟 x86）**：推荐使用 [colima][4] 虚拟机工具来运行 x86 架构的 docker 运行时（例如 `colima start --arch x86_64 --cpu 4 --memory 8` 可开启一个本地的 x86 虚拟机），可支持 x86 架构的 pulsar 镜像运行。


### arm 镜像制作示例

对于 arm64 的 pulsar 镜像，一个简单制作和使用示例如下，参考 [5]。

1. 制作 Dockerfile：

   ```yaml
   FROM --platform=arm64v8 adoptopenjdk:11-hotspot AS build
   # Prepare environment
   ENV PULSAR_HOME=/pulsar
   ENV PATH=$PULSAR_HOME/bin:$PATH
   RUN groupadd --system --gid=9999 pulsar && useradd --system --home-dir $PULSAR_HOME --uid=9999 --gid=pulsar pulsar
   WORKDIR $PULSAR_HOME
   
   ARG PULSAR_VERSION
   ENV PULSAR_VERSION 2.10.0
   # Install Pulsar
   RUN set -ex; \
     apt-get update && apt-get install -y wget; \
     PULSAR_VERSION=$PULSAR_VERSION; \
     wget -O pulsar.tgz "http://mirrors.ustc.edu.cn/apache/pulsar/pulsar-${PULSAR_VERSION}/apache-pulsar-${PULSAR_VERSION}-bin.tar.gz"; \
     tar -xf pulsar.tgz --strip-components=1; \
     mv lib/org.rocksdb-rocksdbjni-6.10.2.jar lib/org.rocksdb-rocksdbjni-6.10.2.jar.bak; \
     wget -O lib/rocksdbjni-6.29.4.1.jar https://maven.aliyun.com/repository/public/org/rocksdb/rocksdbjni/6.29.4.1/rocksdbjni-6.29.4.1.jar; \
     rm pulsar.tgz; \
     \
     chown -R pulsar:pulsar .;
   
   EXPOSE 6650 8080
   CMD [ "bin/pulsar","standalone" ]
   ```

2. 编译命令（这里是我编译好的镜像 [6]）：

   ```shell
   docker buildx build --platform linux/arm64 -t futeng/pulsar:2.10.0-rocksdbjni-6.29.4.1 .
   ```

3. 启动命令：

   ```shell
   docker run -it -p 6650:6650  -p 8080:8080 \
   --mount source=pulsardata,target=/pulsar/data \
   --mount source=pulsarconf,target=/pulsar/conf \
   futeng/pulsar:2.10.0-rocksdbjni-6.29.4.1 \
   bin/pulsar standalone
   ```

4. 查看架构

   ```shell
   root@29fba9e349d1:/pulsar# uname -a
   Linux 29fba9e349d1 5.10.104-linuxkit #1 SMP PREEMPT Thu Mar 17 17:05:54 UTC 2022 aarch64 aarch64 aarch64 GNU/Linux
   ```

对于在 M1 上运行 arm 容器，速度上还是有很大优势的。

## 鸣谢

感谢 @Yisheng Cai 安利 colima，感谢 @Yunze Xu 推荐 sdkman，感谢 @Kai Wang 介绍 Pulsar ARM 相关使用。

## 结束语

最完美的方案还是等待官方 2.11.0 版本的 release 啊！

> 更多资料和链接可点击阅读原文。



[1]: https://github.com/apache/pulsar/issues/14961 "Pulsar IO integration tests"
[2]: https://pulsar.apache.org/docs/getting-started-standalone/#install-jdk-on-m1 "install x86 jdk on m1"
[3]: https://sdkman.io "Home - SDKMAN! the Software Development Kit Manager"
[4]: https://github.com/abiosoft/colima "Colima - container runtimes on macOS (and Linux) with minimal setup."
[5]: https://www.zhaojianyun.com/archives/run-pulsar-on-m1-mac "借助DOCKER让PULSAR可以在M1上原生运行"
[6]: https://hub.docker.com/r/futeng/pulsar/tags "docker pull futeng/pulsar:2.10.0-rocksdbjni-6.29.4.1"
[7]: https://github.com/apache/pulsar/issues/12944 "#12944 ARM based docker image"



----

# 彩蛋 



## 资料栈

- 问题

  - [Ticket 2260：ARM Compatible Release of Pulsar Standalone](https://support.streamnative.io/a/tickets/2260)
    - 2.11 已经原生支持 M1
    - Install JDK on M1
      - 提到 Pulsar 不支持的原因，是因为 BookKeeper 依赖的 RocksDB 只支持 X86 的 JDK。所以给出的方案是 `sdkman_rosetta2_compatible=true` ，让本地 JDK 通过 rosetta2 来支持运行 Pulsar。
    - 切换 x86 JDK 步骤：https://pulsar.apache.org/docs/getting-started-standalone/#install-jdk-on-m1
    

  - Pulsar 还不支持的 M1
    - https://github.com/apache/pulsar/issues/13153
    


- Pulsar 对跟进

  - https://github.com/apache/pulsar/pull/14886 ：14886 ，merlimat 表示 2.11： "Upgrade to newest version of RocksDb which has support for Mac with M1 architecture."
    - https://github.com/apache/pulsar/issues/14961：还有问题：Pulsar IO integration tests 集成测试失败了
      - https://github.com/apache/pulsar/pull/14962#issuecomment-1084192562：解释了原因，即，RocksDB <6.17.3 以下是不兼容最新的 6.29.4.1 版本的

- RocksDB 6.29.4.1 has been released to Maven Central which should now work on M1 macs.

  https://github.com/facebook/rocksdb/issues/7720#issuecomment-1079648907


## 我的测试

### 裸机测试

使用脚本：https://github.com/futeng/pulsar-pseudo-cluster

![image-20220623122602373](https://tva1.sinaimg.cn/large/e6c9d24ely1h3i2bn5vjyj21qo0qwwld.jpg)



结论：

1. 可以更换 jar 包，但是有不兼容的风险（虽然简单测试是通过了），Pulsar IO 有风险

   1. 裸机 SDK rosetta2=false，测试失败（bookie无法启动）
   2. 裸机 SDK rosetta2=false，测试更换 6.29.4.1 之后的，测试通过（bookie可成功启动，简单消费，管理可以），有一定意义。
   3. 裸机 SDK rosetta2=true，测试正常
   4. 裸机 SDK rosetta2=ture，测试正常

   

2. docker 环境
   1. colima with x86，测试正常，不过经常卡主。
   1. 自己编译更换高版本后的 ARM包，测试正常。

可能结论：最好的使用 2.11 并等待 release。



### docker 测试

```shell
➜  docker colima start --arch x86_64 --cpu 4 --memory 8
INFO[0000] starting colima
INFO[0000] runtime: docker
INFO[0000] preparing network ...                         context=vm
INFO[0000] starting ...                                  context=vm
INFO[0066] provisioning ...                              context=docker
INFO[0066] starting ...                                  context=docker
INFO[0073] done

➜  docker colima status
INFO[0000] colima is running
INFO[0000] arch: x86_64
INFO[0000] runtime: docker
INFO[0000] mountType: sshfs
INFO[0000] socket: unix:///Users/futeng/.colima/default/docker.sock
```

### arm 镜像 Dockerfile

```shell
FROM --platform=arm64v8 adoptopenjdk:11-hotspot AS build
# Prepare environment
ENV PULSAR_HOME=/pulsar
ENV PATH=$PULSAR_HOME/bin:$PATH
RUN groupadd --system --gid=9999 pulsar && useradd --system --home-dir $PULSAR_HOME --uid=9999 --gid=pulsar pulsar
WORKDIR $PULSAR_HOME

ARG PULSAR_VERSION
ENV PULSAR_VERSION 2.10.0
# Install Pulsar
RUN set -ex; \
  apt-get update && apt-get install -y wget; \
  PULSAR_VERSION=$PULSAR_VERSION; \
  wget -O pulsar.tgz "http://mirrors.ustc.edu.cn/apache/pulsar/pulsar-${PULSAR_VERSION}/apache-pulsar-${PULSAR_VERSION}-bin.tar.gz"; \
  tar -xf pulsar.tgz --strip-components=1; \
  mv lib/org.rocksdb-rocksdbjni-6.10.2.jar lib/org.rocksdb-rocksdbjni-6.10.2.jar.bak; \
  wget -O lib/rocksdbjni-6.29.4.1.jar https://maven.aliyun.com/repository/public/org/rocksdb/rocksdbjni/6.29.4.1/rocksdbjni-6.29.4.1.jar; \
  rm pulsar.tgz; \
  \
  chown -R pulsar:pulsar .;

EXPOSE 6650 8080
CMD [ "bin/pulsar","standalone" ]
```

编译命令

```shell

docker buildx build --platform linux/arm64 -t futeng/pulsar:2.10.0-rocksdbjni-6.29.4.1 .
```

启动

```shell
docker run -it -p 16650:6650  -p 18080:8080 \
--mount source=pulsardata,target=/Users/futeng/workspace/docker/pulsar/data \
--mount source=pulsarconf,target=/Users/futeng/workspace/docker/pulsar/conf \
futeng/pulsar:2.10.0-rocksdbjni-6.29.4.1 \
bin/pulsar standalone
```

会报异常：

```shell
java.lang.NoSuchMethodError: 'org.rocksdb.ReadOptions org.rocksdb.ReadOptions.setIterateUpperBound(org.rocksdb.Slice)'
	at org.apache.bookkeeper.bookie.storage.ldb.KeyValueStorageRocksDB.getFloor(KeyValueStorageRocksDB.java:257) ~[org.apache.bookkeeper-bookkeeper-server-4.14.4.jar:4.14.4]
	at org.apache.bookkeeper.bookie.storage.ldb.EntryLocationIndex.getLastEntryInLedgerInternal(EntryLocationIndex.java:106) ~[org.apache.bookkeeper-bookkeeper-server-4.14.4.jar:4.14.4]
	at org.apache.bookkeeper.bookie.storage.ldb.EntryLocationIndex.removeOffsetFromDeletedLedgers(EntryLocationIndex.java:220) ~[org.apache.bookkeeper-bookkeeper-server-4.14.4.jar:4.14.4]
	at org.apache.bookkeeper.bookie.storage.ldb.SingleDirectoryDbLedgerStorage.lambda$checkpoint$7(SingleDirectoryDbLedgerStorage.java:638) ~[org.apache.bookkeeper-bookkeeper-server-4.14.4.jar:4.14.4]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515) [?:?]
	at java.util.concurrent.FutureTask.run(FutureTask.java:264) [?:?]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304) [?:?]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128) [?:?]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628) [?:?]
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30) [io.netty-netty-common-4.1.74.Final.jar:4.1.74.Final]
	at java.lang.Thread.run(Thread.java:829) [?:?]
```

