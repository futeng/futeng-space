# Pulsar 中使用 JWT 进行身份认证

Pulsar 使用标准的 `io.jsonwebtoken` 库支持使用 JWT 做身份认证，同时也提供了 JWT 客户端，可以方便的制作密钥和生成 token。

# JWT 简介

JSON Web Token（RFC-7519）是在 Web 应用中常用的一种认证方案。形式上这个 token 只是一个 JSON 字符串，并做了 base64url 编码和签名。

一个 JWT 的 token 字符串包含三个部分，例如：xxxxx.yyyyy.zzzzz（通过点来分隔）：

- Header（下图中红色部分）：内容是一个 JSON 串，主要指定签名算法（Pulsar 中默认使用 HS256），使用 base64url 编码。
- Payload（下图中紫色部分）：内容也是一个 JSON 串，主要指定用户名（sub）和过期时间，也是用 base64url 编码。
- Signature（下图蓝色部分）：内容由使用特定算法，计算编码后的 Header 和 Payload 加一个 secret 值而来。签名保证了 token 不被中途篡改。

```shell
# 一个使用 HMAC SHA256 算法的签名示例
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

## token 内容的查看和验证

![image-20220626104218918](https://tva1.sinaimg.cn/large/e6c9d24ely1h3lg6o8m2kj21nt0u0jvf.jpg)

我们可以去 https://jwt.io/ 查看 Token 的组成内容，或者使用任意 base64 解密工具查看 header 和 payload 部分。

Pulsar 客户端也提供了验证功能：

- bin/pulsar tokens show: 查看 token 的 header 和 payload：

	```shell
	bin/pulsar tokens show -i eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0LXVzZXIiLCJleHAiOjE2NTY3NzYwOTh9.awbp6DreQwUyV8UCkYyOGXCFbfo4ZoV-dofXYTnFXO8
	
	
	{"alg":"HS256"}
	---
	{"sub":"test-user","exp":1656776098}
	```

- bin/pulsar tokens validate：使用 secret key 或者 public key 验证 token：

  ```shell
  bin/pulsar tokens validate -pk  /Users/futeng/workspace/github/futeng/pulsar-pseudo-cluster/pulsar-1/my-public.key -i "eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.ijp-Qw4JDn1aOQbYy4g4YGBbXYIgLA9lCVrnP-heEtPCdDq11_c-9pQdQwc6RdphvlSfoj50qwL5OtmFPysDuF2caSYzSV1kWRWN-tFzrt-04_LRN-vlgb6D06aWubVFJQBC4DyS-INrYqbXETuxpO4PI9lB6lLXo6px-SD5YJzQmcYwi2hmQedEWszlGPDYi_hDG9SeDYmnMpXTtPU3BcjaDcg9fO6PlHdbnLwq2MfByeIj-VS6EVhKUdaG4kU2EJf5uq2591JJAL5HHiuTZRSFD6YbRXuYqQriw4RtnYWSvSeVMMbcL-JzcSJblNbMmIOdiez43MPYFPTB7TMr8g"
  
  {sub=admin}
  ```


## Pulsar 中使用 JWT 的几个注意点

基于 JWT 的特性，在 Pulsar 中使用需要注意的几个点：

1. 使用 JWT 可以做认证和授权，但数据仍然处于暴露状态，在安全要求较高的环境，建议开启 TLS 对数据传输加密（会牺牲一小部分性能），以进一步巩固安全。
2. JWT 最重要的特性是不需要借助第三方服务（例如 Kerberos 之于 KDC），凭借 JWT 内容本身，就能验证 token 是否有效。这也带来一个问题，即 token 一旦签发，在有效期间将会一直有效，无法撤回。因此对于执行某些重要操作的 token，有效期要尽量设置的短。
3. Pulsar 支持 2 种 JWT 加密方式，即使用对称秘钥（secret key）和使用非对称密钥（private/public key），选择其中一个即可。
4. Token 的配置容易出错用户（例如需要操作 pulsar-admin 但仅给了test-user 的 token），建议配置后可使用 validate 命令做验证。
5. Pulsar Broker 会缓存客户端的认证信息，并会在一个固定时间（默认 60 秒）来检查每个连接的认证是否过期，这个刷新时间可以配置 broker.conf 中的 authenticationRefreshCheckSeconds 参数来自定义。



# 配置 JWT

注意 Pulsar 提供了使用对称密钥（secret key）和使用非对称密钥（private/public key）两种方式，可根据安全强度要求选择其中一个即可。

提示：

1. Pulsar 中既可以对 broker 的访问做 JWT 认证，也可以对 bookie 的访问做 JWT 认证。两者配置方式类似，本文重点在于区分两种密钥配置的不同，因此将仅针对 broker 对配置做说明。

2. token 参数的配置，既可使用字符串形式，也可以使用文件形式，可择优选择。

   ```shell
   # 字符串形式
   brokerClientAuthenticationParameters={"token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0LXVzZXIifQ.9OHgE9ZUDeBTZs7nSMEFIuGNEX18FLR3qvy8mqxSxXw"}
   # 从文件读取
   brokerClientAuthenticationParameters={"file":"///path/to/proxy-token.txt"}
   ```

3. standalone 的配置内容相同。

## 使用对称密钥（secret key）

Pulsar 基于 [JSON Web Tokens](https://jwt.io/introduction/) ([RFC-7519](https://tools.ietf.org/html/rfc7519)) 提供了一个标准的 JWT 客户端，可以生成密钥、制作 token 和检测密钥有效性等功能。

### Step 1. 生成 secret key

```shell
bin/pulsar tokens create-secret-key --output my-secret.key
```

- `bin/pulsar tokens create-secret-key` ：生成一个对称密钥，保存到文件 `my-secret.key` 中。后续将使用这个密钥文件来创建 token。

### Step 2. 生成用于超级管理员的 token

我们将超级管理员命名为 admin（对应 pulsar 认证概念里对 role）。这里不指定超时时间（ `--expiry-time`），则默认将不过期。

```shell
bin/pulsar tokens create --secret-key my-secret.key --subject admin
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.Ra9pwWHTWjB67v5GkVuuDMqXWwfeTJuwflyvmhxYk_c
```

### Step 3. 生成给测试用户的 token

```shell
bin/pulsar tokens create --secret-key my-secret.key --subject test-user --expiry-time 7d
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0LXVzZXIiLCJleHAiOjE2NTY3NzYwOTh9.awbp6DreQwUyV8UCkYyOGXCFbfo4ZoV-dofXYTnFXO8
```

### Step 4. 配置 broker

```shell
# 开启认证
authenticationEnabled=true
# 认证提供者
authenticationProviders=org.apache.pulsar.broker.authentication.AuthenticationProviderToken
# 开启授权
authorizationEnabled=true
# 超级管理员
superUserRoles=admin
# broker Client 使用等认证插件
brokerClientAuthenticationPlugin=org.apache.pulsar.client.impl.auth.AuthenticationToken
# broker Client 通讯使用的 token（需要 admin role）
brokerClientAuthenticationParameters={"token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.Ra9pwWHTWjB67v5GkVuuDMqXWwfeTJuwflyvmhxYk_c"}
# 使用 secretKey 的密钥文件位置（file://开头）
tokenSecretKey=file:///Users/futeng/workspace/github/futeng/pulsar-pseudo-cluster/pulsar-1/my-secret.key
```

### Step 5. 重启 broker

```shell
bin/pulsar-daemon stop broker
bin/pulsar-daemon start broker
```

### Step 6. 测试

#### Step 6.1. 验证 broker token

```shell
bin/pulsar tokens validate -sk  /Users/futeng/workspace/github/futeng/pulsar-pseudo-cluster/pulsar-1/my-secret.key -i "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.Ra9pwWHTWjB67v5GkVuuDMqXWwfeTJuwflyvmhxYk_c"

# 打印：{sub=admin}
```

- 注意 broker 的 token 需要使用超级管理员。

#### Step 6.2. 测试超级管理员用户访问

```shell
# produce as admin role
bin/pulsar-client \
--url "pulsar://127.0.0.1:6650" \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \
--auth-params {"token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.Ra9pwWHTWjB67v5GkVuuDMqXWwfeTJuwflyvmhxYk_c"} \
produce public/default/test -m "hello pulsar" -n 10
```

#### Step 6.3. 测试普通用户访问

```shell
# produce as test-user role
bin/pulsar-client \
--url "pulsar://127.0.0.1:6650" \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \
--auth-params {"token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0LXVzZXIiLCJleHAiOjE2NTY3NzYwOTh9.awbp6DreQwUyV8UCkYyOGXCFbfo4ZoV-dofXYTnFXO8"} \
produce public/default/test -m "hello pulsar" -n 10
```

- 尚未给普通用户赋权，因此命令执行需要报缺少权限的错误。

#### Step 6.4. 测试给普通用户赋权

```shell
bin/pulsar-admin \
--admin-url "http://127.0.0.1:8080/" \
--auth-params {"token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.Ra9pwWHTWjB67v5GkVuuDMqXWwfeTJuwflyvmhxYk_c"} \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \
namespaces grant-permission public/default --role test-user --actions produce,consume
```

- 赋权后，可再次测试普通用户访问，需要可以正常发送数据。
- 注意 `bin/pulsar-admin` 命令默认使用的是管理流 8080 端口。
- 注意 `auth-params` 需要使用超级管理员 token 来执行赋权。

#### Step 6.5. 测试给普通用户回收权限

```shell
bin/pulsar-admin \
--admin-url "http://127.0.0.1:8080/" \
--auth-params {"token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.Ra9pwWHTWjB67v5GkVuuDMqXWwfeTJuwflyvmhxYk_c"} \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \ 
namespaces revoke-permission public/default --role test-user
```



## 使用非对称密钥（private/public key）

### Step 1. 生成秘钥对

```shell
bin/pulsar tokens create-key-pair --output-private-key my-private.key --output-public-key my-public.key
```

### Step 2. 生成用于超级管理员的 token

我们将超级管理员命名为 admin（对应 pulsar 认证概念里对 role）。这里不指定超时时间（ `--expiry-time`），则默认将不过期。

```shell
bin/pulsar tokens create --private-key my-private.key --subject admin

eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.ijp-Qw4JDn1aOQbYy4g4YGBbXYIgLA9lCVrnP-heEtPCdDq11_c-9pQdQwc6RdphvlSfoj50qwL5OtmFPysDuF2caSYzSV1kWRWN-tFzrt-04_LRN-vlgb6D06aWubVFJQBC4DyS-INrYqbXETuxpO4PI9lB6lLXo6px-SD5YJzQmcYwi2hmQedEWszlGPDYi_hDG9SeDYmnMpXTtPU3BcjaDcg9fO6PlHdbnLwq2MfByeIj-VS6EVhKUdaG4kU2EJf5uq2591JJAL5HHiuTZRSFD6YbRXuYqQriw4RtnYWSvSeVMMbcL-JzcSJblNbMmIOdiez43MPYFPTB7TMr8g
```

### Step 3. 生成给测试用户的 token

```shell
bin/pulsar tokens create --private-key my-private.key --subject test-user --expiry-time 7d

eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0LXVzZXIiLCJleHAiOjE2NTY4MDMzODh9.0dAXdyl1dVsLZbhnvJDKPXFmyNlqwDYMMwzOoJ1L2Rl9gfcgVB4DzEfBFesU1F07P5oiM_X5hmxdI5YDSDxU4VGb_Sy3MakOAlROq3a4qzT45eY15-N3IxyfaI66BellDsZWyXVwsWnPYmwMBOlqZXgZAEhPL8HqC3c1IMBeMo78lDNobP7k0SVWsy9jhhmVOcas2ZQ4B-vOC8f0pHAWD29Rf_AV34A5w6Wu5XbQoHpMp5n0KRv2K_oFed_Zmg79uvtLv3Ujd8DaXN9a2vjXRatFYY2iZN8OhB1SV4WjpXB5hyG5Sv9uAHC559W39g8-AznG8NA5J79d-tIftIr8Dg
```

### Step 4. 配置 broker

```shell
# 开启认证
authenticationEnabled=true
# 认证提供者
authenticationProviders=org.apache.pulsar.broker.authentication.AuthenticationProviderToken
# 开启授权
authorizationEnabled=true
# 超级管理员
superUserRoles=admin
# broker Client 使用等认证插件
brokerClientAuthenticationPlugin=org.apache.pulsar.client.impl.auth.AuthenticationToken
# broker Client 通讯使用的 token（需要 admin role）
brokerClientAuthenticationParameters={"token":"eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.ijp-Qw4JDn1aOQbYy4g4YGBbXYIgLA9lCVrnP-heEtPCdDq11_c-9pQdQwc6RdphvlSfoj50qwL5OtmFPysDuF2caSYzSV1kWRWN-tFzrt-04_LRN-vlgb6D06aWubVFJQBC4DyS-INrYqbXETuxpO4PI9lB6lLXo6px-SD5YJzQmcYwi2hmQedEWszlGPDYi_hDG9SeDYmnMpXTtPU3BcjaDcg9fO6PlHdbnLwq2MfByeIj-VS6EVhKUdaG4kU2EJf5uq2591JJAL5HHiuTZRSFD6YbRXuYqQriw4RtnYWSvSeVMMbcL-JzcSJblNbMmIOdiez43MPYFPTB7TMr8g"}
# 使用 tokenPublicKey 的公钥文件位置（file://开头）
tokenPublicKey=file:///Users/futeng/workspace/github/futeng/pulsar-pseudo-cluster/pulsar-1/my-public.key
```

### Step 5. 重启 broker

```shell
bin/pulsar-daemon stop broker
bin/pulsar-daemon start broker
```

### Step 6. 测试

#### Step 6.1. 验证 broker token

```shell
bin/pulsar tokens validate -pk  /Users/futeng/workspace/github/futeng/pulsar-pseudo-cluster/pulsar-1/my-public.key -i "eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.ijp-Qw4JDn1aOQbYy4g4YGBbXYIgLA9lCVrnP-heEtPCdDq11_c-9pQdQwc6RdphvlSfoj50qwL5OtmFPysDuF2caSYzSV1kWRWN-tFzrt-04_LRN-vlgb6D06aWubVFJQBC4DyS-INrYqbXETuxpO4PI9lB6lLXo6px-SD5YJzQmcYwi2hmQedEWszlGPDYi_hDG9SeDYmnMpXTtPU3BcjaDcg9fO6PlHdbnLwq2MfByeIj-VS6EVhKUdaG4kU2EJf5uq2591JJAL5HHiuTZRSFD6YbRXuYqQriw4RtnYWSvSeVMMbcL-JzcSJblNbMmIOdiez43MPYFPTB7TMr8g"

# 打印：{sub=admin}
```

- 注意 broker 的 token 需要使用超级管理员。
- 注意是使用公钥来验证 token（public.key）。

#### Step 6.2. 测试超级管理员用户访问

```shell
# produce as admin role
bin/pulsar-client \
--url "pulsar://127.0.0.1:6650" \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \
--auth-params {"token":"eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.ijp-Qw4JDn1aOQbYy4g4YGBbXYIgLA9lCVrnP-heEtPCdDq11_c-9pQdQwc6RdphvlSfoj50qwL5OtmFPysDuF2caSYzSV1kWRWN-tFzrt-04_LRN-vlgb6D06aWubVFJQBC4DyS-INrYqbXETuxpO4PI9lB6lLXo6px-SD5YJzQmcYwi2hmQedEWszlGPDYi_hDG9SeDYmnMpXTtPU3BcjaDcg9fO6PlHdbnLwq2MfByeIj-VS6EVhKUdaG4kU2EJf5uq2591JJAL5HHiuTZRSFD6YbRXuYqQriw4RtnYWSvSeVMMbcL-JzcSJblNbMmIOdiez43MPYFPTB7TMr8g"} \
produce public/default/test -m "hello pulsar" -n 10
```

#### Step 6.3. 测试普通用户访问

```shell
# produce as test-user role
bin/pulsar-client \
--url "pulsar://127.0.0.1:6650" \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \
--auth-params {"token":"eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ0ZXN0LXVzZXIiLCJleHAiOjE2NTY4MDMzODh9.0dAXdyl1dVsLZbhnvJDKPXFmyNlqwDYMMwzOoJ1L2Rl9gfcgVB4DzEfBFesU1F07P5oiM_X5hmxdI5YDSDxU4VGb_Sy3MakOAlROq3a4qzT45eY15-N3IxyfaI66BellDsZWyXVwsWnPYmwMBOlqZXgZAEhPL8HqC3c1IMBeMo78lDNobP7k0SVWsy9jhhmVOcas2ZQ4B-vOC8f0pHAWD29Rf_AV34A5w6Wu5XbQoHpMp5n0KRv2K_oFed_Zmg79uvtLv3Ujd8DaXN9a2vjXRatFYY2iZN8OhB1SV4WjpXB5hyG5Sv9uAHC559W39g8-AznG8NA5J79d-tIftIr8Dg"} \
produce public/default/test -m "hello pulsar" -n 10
```

- 尚未给普通用户赋权，因此命令执行需要报缺少权限的错误。

#### Step 6.4. 测试给普通用户赋权

```shell
bin/pulsar-admin \
--admin-url "http://127.0.0.1:8080/" \
--auth-params {"token":"eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.ijp-Qw4JDn1aOQbYy4g4YGBbXYIgLA9lCVrnP-heEtPCdDq11_c-9pQdQwc6RdphvlSfoj50qwL5OtmFPysDuF2caSYzSV1kWRWN-tFzrt-04_LRN-vlgb6D06aWubVFJQBC4DyS-INrYqbXETuxpO4PI9lB6lLXo6px-SD5YJzQmcYwi2hmQedEWszlGPDYi_hDG9SeDYmnMpXTtPU3BcjaDcg9fO6PlHdbnLwq2MfByeIj-VS6EVhKUdaG4kU2EJf5uq2591JJAL5HHiuTZRSFD6YbRXuYqQriw4RtnYWSvSeVMMbcL-JzcSJblNbMmIOdiez43MPYFPTB7TMr8g"} \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \
namespaces grant-permission public/default --role test-user --actions produce,consume
```

- 赋权后，可再次测试普通用户访问，需要可以正常发送数据。
- 注意 `bin/pulsar-admin` 命令默认使用的是管理流 8080 端口。
- 注意 `auth-params` 需要使用超级管理员 token 来执行赋权。

#### Step 6.5. 测试给普通用户回收权限

```shell
bin/pulsar-admin \
--admin-url "http://127.0.0.1:8080/" \
--auth-params {"token":"eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.ijp-Qw4JDn1aOQbYy4g4YGBbXYIgLA9lCVrnP-heEtPCdDq11_c-9pQdQwc6RdphvlSfoj50qwL5OtmFPysDuF2caSYzSV1kWRWN-tFzrt-04_LRN-vlgb6D06aWubVFJQBC4DyS-INrYqbXETuxpO4PI9lB6lLXo6px-SD5YJzQmcYwi2hmQedEWszlGPDYi_hDG9SeDYmnMpXTtPU3BcjaDcg9fO6PlHdbnLwq2MfByeIj-VS6EVhKUdaG4kU2EJf5uq2591JJAL5HHiuTZRSFD6YbRXuYqQriw4RtnYWSvSeVMMbcL-JzcSJblNbMmIOdiez43MPYFPTB7TMr8g"} \
--auth-plugin "org.apache.pulsar.client.impl.auth.AuthenticationToken" \
namespaces revoke-permission public/default --role test-user 
```

# 配合使用 TLS 传输加密

可参考：https://pulsar.apache.org/docs/security-tls-transport 

# 问题

## broker clinet 的 token 需要超级管理员权限

```shell
2022-06-25T23:45:22,824+0800 [pulsar-io-4-5] ERROR org.apache.pulsar.broker.service.SystemTopicBasedTopicPoliciesService - [public/default] Failed to create reader on __change_events topic
java.util.concurrent.CompletionException: org.apache.pulsar.client.api.PulsarClientException$AuthorizationException: {"errorMsg":"Proxy Client is not authorized to Get Partition Metadata","reqId":4248733629549842796, "remote":"/127.0.0.1:16650", "local":"/127.0.0.1:51687"}
```

## 如何使用 Supplier 来提供 token

```java
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar://broker.example.com:6650/")
    .authentication(
        AuthenticationFactory.token(() -> {
            // Read token from custom source
            return readToken();
        }))
    .build();
```

通常不建议对 token 硬编码。Pulsar 在 client 中也对应提供了 Supplier 方式的编程（函数式编程）。用户可以根据自己的环境，实现 Supplier 的方法，这样可以方便的从外部去更新 token。
示例：可以任意定义获取 token 的方法，注意名称和 Supplier 函数名保持一致（感谢 @Jun Fu 提供示例）。

```java
private String getToken() {
    String token = "${get token from db or from token file}";
    return token;
}
```

## 关于多次创建 Token 问题

多次创建相同 role 的 token ，内容会变动吗？

我们可以参考 `org.apache.pulsar.utils.auth.tokens.TokensCliUtils` 代码可以知道，如果没有指定过期时间，每次创建的 token 将会是相同的。

```java
# org.apache.pulsar.utils.auth.tokens.TokensCliUtils
String token = AuthTokenUtils.createToken(signingKey, subject, optExpiryTime);
```

## 关于官方的 token 工具

可以直接查看 `bin/pulsar` 命令，调用的是 `org.apache.pulsar.utils.auth.tokens.TokensCliUtils` 代码。

在调用时会启动一个 JVM 虚拟机执行对应代码。

```
vim bin/pulsar

elif [ $COMMAND == "tokens" ]; then
    exec $JAVA $OPTS org.apache.pulsar.utils.auth.tokens.TokensCliUtils $@
```

其实不用官方工具去制作 token 等都相关。

## 参考

[1]: https://pulsar.apache.org/docs/security-jwt "Client authentication using tokens based on JSON Web Tokens"

[2]: https://blog.angular-university.io/angular-jwt/ "JWT: The Complete Guide to JSON Web Tokens"
