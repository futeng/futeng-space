# 概念

Pulsar 使用 Java 的 JAAS 机制来支持通过 Kerberos 做身份认证。JAAS 中一个用户信息对应一个 `section` 。对于 Kerberos 认证而言，一个用户信息最重要的是 principle 和 keytab，现在可以方便的封装成一个 section 里。最后将这些用户信息拼装到 `jaas.conf ` 文件中。

 `jaas.conf` 样例：

```shell
 SectionName {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   storeKey=true
   useTicketCache=false
   keyTab="/etc/security/keytabs/pulsarbroker.keytab"
   principal="broker/localhost@EXAMPLE.COM";
};
 AnotherSectionName {
  ...
};
```

- `SectionName`：指定了用户名标识，内部封装了一个 Kerberos 用户信息。

对于 Pulsar 如何使用 Kerberos 认证，从配置上而言，就是告知进程该以哪个 section 的身份来启动程序。以 broker 为例：

1. 在 `broker.conf` 中指定服务使用的 section 。
2. 而 section 信息是在 `pulsar_env.sh` 中通过指定 `jaas.conf ` 文件来加载获得的。

# 注意事项

## Notice 1：broker service principal 的 {hostname} 和advertisedAddress 保持一致

结论：例如 broker service 使用的 Principal 是 `"broker/172.17.0.7@SNIO"`，则 advertisedAddress 也必须配置为相同的 IP： `172.17.0.7`.

原因：客户端从 TGT Server 获取到的 broker 服务端 Principal 里面的 hostname 部分是advertisedAddress定义的 IP。

可从 kdc 日志（/var/log/krb5kdc.log）查看到问题。broker 服务会涉及两个 Principal，一个是 broker 服务注册的 Principal，另一个是 TGT Server 返回的 Principal。这会导致如果两个 Principal 内部的 IP 不一致，用户会永远无法访问到服务。毕竟用户一次只能携带一个服务端Principal 信息。

```shell
# ① broker 服务以 broker/172.17.0.7@SNIO 的 Principal 向 KDC 数据库注册自己是认证服务的身份
Jun 16 14:29:19 mysql-server-for-ranger krb5kdc[2003](info): AS_REQ (3 etypes {3 1 16}) 172.17.0.7: ISSUE: authtime 1655389759, etypes {rep=3 tkt=16 ses=1}, broker/172.17.0.7@SNIO for krbtgt/SNIO@SNIO
# ② 稍后有客户端 client@SNIO 申请获取 broker/172.17.0.7@SNIO TGT 票据
Jun 16 14:29:38 mysql-server-for-ranger krb5kdc[2003](info): AS_REQ (3 etypes {3 1 16}) 172.17.0.7: ISSUE: authtime 1655389778, etypes {rep=3 tkt=16 ses=1}, client@SNIO for krbtgt/SNIO@SNIO
# ③ TGT 返回的 Principal 是 broker/172.17.0.7@SNIO，其中IP 地址部分是由 advertisedAddress 确定的
Jun 16 14:29:38 mysql-server-for-ranger krb5kdc[2003](info): TGS_REQ (3 etypes {3 1 16}) 172.17.0.7: ISSUE: authtime 1655389778, etypes {rep=1 tkt=16 ses=1}, client@SNIO for broker/172.17.0.7@SNIO
```

一个错误访问的日志：

```shell
# ① broker 服务以 broker/172.17.0.7@SNIO 的 Principal 向 KDC 数据库注册自己是认证服务的身份
Jun 16 14:29:19 mysql-server-for-ranger krb5kdc[2003](info): AS_REQ (3 etypes {3 1 16}) 172.17.0.7: ISSUE: authtime 1655389759, etypes {rep=3 tkt=16 ses=1}, broker/172.17.0.7@SNIO for krbtgt/SNIO@SNIO
# ② 客户端 client@SNIO 申请获取 broker/20.232.197.169@SNIO TGT 票据
Jun 16 14:29:59 mysql-server-for-ranger krb5kdc[2003](info): AS_REQ (3 etypes {3 1 16}) 172.17.0.7: ISSUE: authtime 1655389799, etypes {rep=3 tkt=16 ses=1}, client@SNIO for krbtgt/SNIO@SNIO
# ③ 并未找到 broker/20.232.197.169@SNIO 服务（broker 报错： Failed to evaluate client token:，客户端报错：Unable to authenticate）
Jun 16 14:30:00 mysql-server-for-ranger krb5kdc[2003](info): TGS_REQ (3 etypes {3 1 16}) 172.17.0.7: ISSUE: authtime 1655389799, etypes {rep=1 tkt=16 ses=1}, client@SNIO for broker/20.232.197.169@SNIO
```

测试环境有特殊性，使用的是 Azure 公有云虚拟机，由 public IP 和 private IP 构成的 Principal 会被看成是两个不同的服务：

- public IP: 20.232.197.169
- private IP: 172.17.0.7
- Hostname: mysql-server-for-ranger
- pulsar: 2.9.0-snapshot
- openjdk version "11.0.15"

 试验方法：通过 `bin/pulsar-client --url` 指定 URL 的方式来生产消息。URL 的选择为：

- pulsar://172.17.0.7:6650
- pulsar://20.232.197.169:6650
- pulsar://mysql-server-for-ranger:6650
- 不加 URL

注意：将 broker service principal 机器部分用 hostname 代替（非 IP），KDC 再交给客户端的 TGT 仍然是带 IP 的，这可能是个 BUG 或者跟环境有关系。

## Notice 2：关于 KDC 和  krb5.conf 文件

- 请确保 KDC 服务正常，网络可达、可正常创建用户等。
- 获取 krb5.conf 文件：服务进程需要 krb5.conf 来找到 KDC 机器位置等，需要在每个服务节点存放。另 krb5.conf 文件格式极易修改错误，出错后很难发现问题，请极力避免。

## Notice 3：关于服务端 Principal 命名规则

关于服务端 Principal 命名规则：

1. 必须包含完整的三部分：service/hostname@REALM。
2. service 命名部分，pulsar 里建议使用关键字如 `broker`、`proxy` （其他的报 warn）。
3. 主机地址部分为了防止多块网卡的 IP 配置问题，均建议使用 hostname，但是如果没有 DNS 解析环境，请参考 Notice1，可使用 IP，保证服务 IP 和advertisedAddressIP 一致。
4. REALM 域名建议使用大写，注意不要复制错。

Broker 的 Principal 命名格式示例：

```shell
# 正确示范
broker/host1@MY.REALM

# 错误示例
broker@MY.REALM
pulsarbroker/host1@MY.REALM
```

## Notice 4：当使用 Oracle JDK 且开启 KDC ace-256 加密时

当使用 Oracle JDK 且开启 KDC ace-256 加密时，需要额外引入配置文件（Oracle JDK 的 Ulimit JCE 策略文件），并放置在 `$JAVA_HOME/jre/lib/security` ；

注意部分 JDK 版本不支持 krb5.conf 的`includedir`开头，可能需要去除。

## Notice 5：配置 FQDN 解析

Kerberos 所有主机都需要支持 FQDN 解析（少量机器可在 `/etc/hosts` 配置所有主机DNS信息）。

# 部署

## Step 1. 创建 broker service 用户

我们需要给所有参与 Pulsar 集群服务的进程，如 Broker、Proxy 和 Client 等，都生成对应的 Kerberos 用户和票据。

在 KDC 机器使用 `kadmin.local` 工具创建 broker 用户（注意更换 REALM 换成实际 KDC 的域名；本次使用 外网 IP 作为 Principal 中间项，原因参考 notice1）：

```shell
export REALM="SNIO"
export MYHOST="20.232.197.169"
sudo mkdir -p /etc/security/keytabs

# public IP 版本
sudo kadmin.local -q "addprinc -randkey broker/$MYHOST@$REALM"
sudo kadmin.local -q "ktadd -k /etc/security/keytabs/broker-$MYHOST.keytab  broker/$MYHOST@$REALM"
```

可以通过 `kinit -kt keytab_path principal` 来验证票据是否可正常缓存 TGT。

同时建议创建 broker 服务其他 Principal 版本的认证信息：

```shell
# hostname 版本
sudo kadmin.local -q "addprinc -randkey broker/$(hostname)@$REALM"
sudo kadmin.local -q "ktadd -k /etc/security/keytabs/broker-$(hostname).keytab  broker/$(hostname)@$REALM"

# private IP 版本
export PRIVATE_IP="172.17.0.7"
sudo kadmin.local -q "addprinc -randkey broker/$PRIVATE_IP@$REALM"
sudo kadmin.local -q "ktadd -k /etc/security/keytabs/broker-$PRIVATE_IP.keytab  broker/$PRIVATE_IP@$REALM"
```

## Step 2. 创建 broker client 用户

```shell
sudo kadmin.local -q "addprinc -randkey client@$REALM"
sudo kadmin.local -q "ktadd -k /etc/security/keytabs/client.keytab client@$REALM"
```

## Step 3. 赋予 keytab 可读权限

我们创建了用户的 keytab 如下，我们可以使用 `sudo chown $USER /etc/security/keytabs/*.keytab`：

```shell
$ ll /etc/security/keytabs
-rw------- 1 pulsar root 254 Jun 16 13:30 broker-172.17.0.7.keytab
-rw------- 1 pulsar root 270 Jun 16 12:54 broker-20.232.197.169.keytab
-rw------- 1 pulsar root 306 Jun 16 12:32 broker-mysql-server-for-ranger.keytab
-rw------- 1 pulsar root 206 Jun 16 13:16 client.keytab
```

## Step 4. 配置 broker 使用 Kerberos 认证

### 4.1 封装 JAAS 文件

这里将创建的 broker service 和 broker client 用户封装到一个 JAAS 文件中，放置于：`conf/pulsar_jaas.conf`，完整目录是 `/home/pulsar/demo/pulsar-pseudo-cluster/pulsar-1/conf/pulsar_jaas.conf`。

```shell
   PulsarBroker {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   storeKey=true
   useTicketCache=false
   keyTab="/etc/security/keytabs/broker-mysql-server-for-ranger.keytab"
   principal="broker/mysql-server-for-ranger@SNIO";
};

 PulsarClient {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   storeKey=true
   useTicketCache=false
   keyTab="/etc/security/keytabs/pulsarclient.keytab"
   principal="pulsarclient@SNIO";
};
```

### 4.2 配置 broker JVM 参数引入 JAAS 文件

修改 Pulsar的启动脚本 `pulsar_env.sh` ，将 JAAS 文件和 kerb5.conf 的位置配置到 `conf/pulsar_env.sh` 文件 `PULSAR_EXTRA_OPTS` 项的末尾。

```shell
-Djava.security.auth.login.config=/home/pulsar/demo/pulsar-pseudo-cluster/pulsar-1/conf/pulsar_jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf 
```

完整的被修改行：

```shell
PULSAR_EXTRA_OPTS=${PULSAR_EXTRA_OPTS:-" -Dpulsar.allocator.exit_on_oom=true -Dio.netty.recycler.maxCapacity.default=1000 -Dio.netty.recycler.linkCapacity=1024 -Djava.security.auth.login.config=/home/pulsar/demo/pulsar-pseudo-cluster/pulsar-1/conf/pulsar_jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf"}
```

### 4.3 配置 broker.conf 指定 section

修改 broker 的 `conf/broker.conf` 配置中，新增下面两段配置。

```shell
# 保持一致
advertisedAddress=20.232.197.169

authenticationEnabled=true
authenticationProviders=org.apache.pulsar.broker.authentication.AuthenticationProviderSasl
saslJaasClientAllowedIds=.*client.*
saslJaasBrokerSectionName=PulsarBroker

## Authentication settings of the broker itself. Used when the broker connects to other brokers
brokerClientAuthenticationPlugin=org.apache.pulsar.client.impl.auth.AuthenticationSasl
brokerClientAuthenticationParameters={"saslJaasClientSectionName":"PulsarClient", "serverType":"broker"}
```

- `saslJaasClientAllowedIds` ：正则表达式，配置允许哪些客户端（Kerberos Principal）连接 Broker。
- `saslJaasBrokerSectionName` ：配置 Broker 使用哪个 Section 连接集群。
- `brokerClientAuthenticationParameters` ：JSON字符串，配置 Broker 内部 admin 客户端使用哪个 section 连接其他 Broker。服务类型为 broker ，后期也可能配置为proxy。

## Step 5. 重启 broker 服务

在重启前，确保以下文件存在，且当前用户有可读权限。

- /etc/security/keytabs/pulsarbroker.keytab
- /etc/security/keytabs/pulsarclient.keytab
- conf/pulsar_jaas.conf
- /etc/krb5.conf 

```shell
bin/pulsar-daemon stop broker
bin/pulsar-daemon start broker
```

正常日志会打印出 broker login 信息：successfully logged in.

![image-20220616203807569](https://tva1.sinaimg.cn/large/e6c9d24egy1h3ad7gz75nj22l40aidmm.jpg)

## Step 6. 配置 cli

常用 pulsar 客户端工具包括 `bin/pulsar-client`, `bin/pulsar-perf` 和 `bin/pulsar-admin` ，都需要配置 Kerberos 认证。

### 6.1 配置 Client JVM 引入 JAAS 文件

配置  conf/pulsar_tools_env.sh，在 `PULSAR_EXTRA_OPTS` 末尾追加JVM参数：

```shell
   -Djava.security.auth.login.config=/home/pulsar/demo/pulsar-pseudo-cluster/pulsar-1/conf/pulsar_jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf  
```

完整的被修改行：

```shell
PULSAR_EXTRA_OPTS="${PULSAR_EXTRA_OPTS} ${PULSAR_MEM} ${PULSAR_GC} ${PULSAR_GC_LOG} -Dio.netty.leakDetectionLevel=disabled -Djava.security.auth.login.config=/home/pulsar/demo/pulsar-pseudo-cluster/pulsar-1/conf/pulsar_jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf"
```

### 6.2 配置 conf/client.conf 指定 section

```shell
authPlugin=org.apache.pulsar.client.impl.auth.AuthenticationSasl
authParams={"saslJaasClientSectionName":"PulsarClient", "serverType":"broker"}
```

将沿用 PulsarClient 这个 section 连接 broker 节点。

## Step 7. Admin 和生产消费测试

admin 用户需要在 conf/broker 里定义，例如本例定义如下：

```shell
superUserRoles=client@SNIO
```

客户端保持同样的 Principal ，访问时携带服务 URL：

```shell
bin/pulsar-admin --admin-url http://20.232.197.169:8080/ clusters list
```

![image-20220617000950043](https://tva1.sinaimg.cn/large/e6c9d24egy1h3ajbofzumj22nk0j6dqv.jpg)

使用 produce和 consumer 访问测试，需要指定 url 参数：

```shell
bin/pulsar-client --url pulsar://20.232.197.169:6650 produce persistent://public/default/test -n 10  -m "hello pulsar"
bin/pulsar-client  --url pulsar://20.232.197.169:6650 consume persistent://public/default/test -n 10 -s "consumer-test"  -t "Exclusive" 
```