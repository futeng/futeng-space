# Kerberos 服务端安装

服务端重点三个配置文件：

- /etc/krb5.conf
- /var/kerberos/krb5kdc/kdc.conf
- /var/kerberos/krb5kdc/kadm5.acl

注意点：

- 防止配置文件格式错误，如编辑过程中导致内容缺失。
- 查询软件对加密算法的支持程度，如降低版本 hadoop 需要去除掉 aes 和 camellia 相关加密算法支持。
- 配置文件不要加注释。
- 域名大小写敏感，建议统一使用大写（default_realm）。
- Java 使用 `aes256-cts` 验证方式需要安装额外的  jar 包，简单起见科删除  aes256-cts 加密方式。

## 前置准备

- 确认域名：本例为 `MYREALM`。
- 确保所有进入 Kerberos 认证的机器时间同步。

## Step 1. 安装依赖

```shell
sudo yum install -y krb5-server krb5-libs krb5-auth-dialog krb5-workstation
```

## Step 2. 配置 krb5.conf

默认位置：`/etc/krb5.conf` ，用于客户端配置。

```shell
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
 default_realm = MYREALM
 default_ccache_name = KEYRING:persistent:%{uid}
 default_tkt_enctypes = des-cbc-md5 des-cbc-crc des3-cbc-sha1 
 default_tgs_enctypes = des-cbc-md5 des-cbc-crc des3-cbc-sha1 
 permitted_enctypes = des-cbc-md5 des-cbc-crc des3-cbc-sha1
 allow_weak_crypto = true

[realms]
 MYREALM = {
  kdc = host128137
  admin_server = host128137
 }

[domain_realm]
host128137=MYREALM
```

说明：

- `[logging]`：服务端 志打印位置。
- `[libdefaults]`：连接的默认配置 。
  - `default_realm`：设置默认领域。多个领域是配置在 [realms] 章节。
  - `udp_preference_limit=1`：禁止使用udp（可以防止一个Hadoop中的错误）
  - `ticket_lifetime`： 凭证生效的时限，一般为24小时。
  - `renew_lifetime`： 凭证最长可以被延期的时限，一般为7 天。
  - 去掉 `des256` 加密支持，因为该加密算法需要 JCE 的额外支持，增加了对环境的依赖。
- `[realms]`：配置所有需要访问的 Kerberos 域。 
  - `kdc`：kdc服务器地址（机器:端口），默认端口 88。
  - admin_server： admin服务地址（机器:端口），默认端口749。
  - default_domain： 指定默认的域名。

## Step 3. 配置 kdc.conf

默认位置：`/var/kerberos/krb5kdc/kdc.conf`，这是 KDC 服务端配置文件。

```shell
[kdcdefaults]
 kdc_ports = 88  # 需要开通的端口
 kdc_tcp_ports = 88

[realms]
 MYREALM = {
  acl_file = /var/kerberos/krb5kdc/kadm5.acl 
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal 
  max_life = 24h
  max_renewable_life = 10d
 }
```

说明：

- `[kdcdefaults]` ：KDC 默认行为配置，全局生效。
- `[realms]`：配置每个域的信息。
  - `MYREALM`： 域名，建议统一使用大写。
  - `acl_file`：  admin 的用户权限，需要用户自己创建。
  - `supported_enctypes`：支持的校验方式。
  - `admin_keytab`：KDC 进行校验的管理员 keytab。

## Step 4. 配置 kadm5.acl

文件位置： `/var/kerberos/krb5kdc/kadm5.acl` ，用于 ACL（访问控制列表），此处提前配置管理员。

```shell
[futeng@host128137 ~]$ sudo cat /var/kerberos/krb5kdc/kadm5.acl
*/admin@MYREALM	*
```

## Step 5. 安装 KDC 数据库

安装命令：`kdb5_util create -r MYREALM -s`

- `-r`：指定域名，注意和配置文件域名保持一致。
- `-s`：指定将数据库的主节点密钥存储在文件中，从而可以在每次启动KDC时自动重新生成主节点密钥。
- `-d`： 指定数据库名，默认名为 principal。

```shell
$ sudo kdb5_util create -r MYREALM -s
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'MYREALM',
master key name 'K/M@MYREALM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key:  # 输入 KDC 数据库密码，重要，futeng123
Re-enter KDC database master key to verify: #futeng123
```

说明：

- 该命令会在` /var/kerberos/krb5kdc/` 目录下创建 `principal` 数据库文件。在重装时，可能需要需要删除数据库文件。
- 对于 KDC 数据库密码，非常重要，请妥善保存。

## Step 6. 启动服务

```shell
sudo chkconfig --level 35 krb5kdc on
sudo chkconfig --level 35 kadmin on
sudo service krb5kdc start
sudo service kadmin start
```

## Step 7. 创建 Kerberos 管理员

可使用 `kadmin.local` （只允许在本机执行）或 `kadmin` 工具，管理 KDC 服务。

创建超级管理员的 Principle：`root/admin`。

```shell
sudo kadmin.local -q "addprinc root/admin"
Authenticating as principal root/admin@MYREALM with password.
WARNING: no policy specified for root/admin@MYREALM; defaulting to no policy
Enter password for principal "root/admin@MYREALM": # root
Re-enter password for principal "root/admin@MYREALM": # root
Principal "root/admin@MYREALM" created.
```

- 一条命令方式创建：`sudo echo -e "root\nroot" | kadmin.local -q "addprinc root/admin"`

## Step 8. 创建普通用户

1、创建测试用户 test，此时用户只能通过密码方式访问（在生成 keytab 文件后，将只能通过keytab访问）。

```shell
$  echo -e "test\ntest" | sudo kadmin.local -q "addprinc test"
# 注意上面创建了用户 test，密码也是 test
Authenticating as principal root/admin@MYREALM with password.
WARNING: no policy specified for test@MYREALM; defaulting to no policy
Enter password for principal "test@MYREALM":
Re-enter password for principal "test@MYREALM":
Principal "test@MYREALM" created.
```

2、用户基本操作示例

```shell
# 1. 登录
$ kinit test
Password for test@MYREALM: # test

# 2. 查看登录缓存
$ klist -e
Ticket cache: KEYRING:persistent:1000:1000
Default principal: test@MYREALM

Valid starting       Expires              Service principal
08/19/2020 14:27:28  08/20/2020 14:33:00  krbtgt/MYREALM@MYREALM
	renew until 08/26/2020 14:32:41, Etype (skey, tkt): des-cbc-crc, des3-cbc-sha1
# 3. 更新票据缓存有效期
$ kinit -R
$ klist -e
Ticket cache: KEYRING:persistent:1000:1000
Default principal: test@MYREALM

Valid starting       Expires              Service principal
08/19/2020 14:35:17  08/20/2020 14:35:17  krbtgt/MYREALM@MYREALM
	renew until 08/26/2020 14:32:41, Etype (skey, tkt): des-cbc-crc, des3-cbc-sha1
	### 可以看到 Expires 时间已经更新，注意 renew until 是相对不变的
	
# 4. 销毁登录缓存
$ kdestroy
$ klist
klist: Credentials cache keyring 'persistent:1000:1000' not found
```

3、查看安装版本

```shell
$ klist -V
```

# Kerberos 客户端安装

## Step 1. 安装依赖

```shell
$ sudo yum install krb5-devel krb5-workstation  -y
```

## Step 2. 配置 krb5.conf

可从 KDC 机器获取 krb5.conf 文件

# 	Kerberos 使用手册

> [Wikipedia](https://en.wikipedia.org/wiki/Kerberos_(protocol))
>
> **Kerberos** ([/ˈkɜːrbərɒs/](https://en.wikipedia.org/wiki/Help:IPA/English)) is a [computer-network](https://en.wikipedia.org/wiki/Computer_network) [authentication](https://en.wikipedia.org/wiki/Authentication) [protocol](https://en.wikipedia.org/wiki/Cryptographic_protocol) that works on the basis of *tickets* to allow [nodes](https://en.wikipedia.org/wiki/Node_(networking)) communicating over a non-secure network to prove their identity to one another in a secure manner. 

> [百科](https://baike.baidu.com/item/Kerberos/5561682?fr=aladdin)
>
> Kerberos是一种计算机网络授权协议，用来在非安全网络中，对个人通信以安全的手段进行身份认证。这个词又指麻省理工学院为这个协议开发的一套计算机软件。

## 区分认证和授权

认证是证明你就是你本人自己（是系统当初录入的那个合法用户）的过程，通常通过验证一个凭据真伪来实现。认证通过后，就可以登录系统，在权限范围内可执行一定的操作。例如，SSH 是一种认证协议，用户在通过 ssh 认证后，可以登录远程服务器。Kerberos 也是一种认证协议，很多常见开源软件使用 Kerberos 验证登录用户身份的真实性，仅当用户通过 Kerberos 认证，才会被认为是一个合法的用户，可以进行相应操作。SSH 的认证凭据是用户名、密码，Kerberos 的凭据是 Principal 和 keytab（密码文件）。当别人拿到你的票据就可以伪装成你，直到票据过期。

授权是给用户授予操作范围和程度的过程，通常通过对用户申请的权限是否批准来实现。授权通过后，用户就可以执行该权限限定的操作。

认证和授权是两个完全不同却极易混淆的概念。

### Kerberos认证对比 ssh认证

| 对比事项 | SSH(Linux)                                                   | Kerberos(hadoop/spark...)                                    |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 用户概念 | user（可通过 `whoami` 查询当前登录用户）                     | principal（可通过 `klist` 查询当前登录用户）                 |
| 用户创建 | 由 Linux 管理员通过 `user add USER` 创建，可通过 `cat /etc/passwd` 查询 | 由 Kerberos 管理员使用 `kadmin.local add_principal` 创建，可通过 `kadmin.local list_principals` 查看所有用 |
| 用户登录 | 可通过 ssh 协议，远程登录，登录命令：`ssh username@host`     | 可通过 Kerberos 协议，远程登录，登录命令：`kinit username` 或者使用密钥文件形式登录 `kinit -kt keytab principal` |
| 退出登录 | `exit` 可退出 ssh 登录                                       | `kdestory` 可退出 Kerberos 登录                              |

## 注意事项

### 关于 sudo

注意，在命令行下使用 `sudo` 执行 Kerberos 命令，实际生效的是 root 环境，而非当前登录的用户。

### 关于两种方式登录

Kerberos 支持两种方式登录：

- 使用 Principal 和输入密码方式登录（默认）。
- 使用 Principal 和 keytab 密码文件方式登录（通常大数据组件采用的登录方式）。

在生成 keytab 密码文件后，默认密码方式就不可用。

### 关于 Kerberos 用户

我们通常用 Principal  来代指 Kerberos 用户，注意格式，像 admin/host1 和 admin 因为权限不同，可以认为是不同的用户。

## Kerberos 服务管理

1、启动服务

```shell
sudo chkconfig --level 35 krb5kdc on
sudo chkconfig --level 35 kadmin on
sudo service krb5kdc start
sudo service kadmin start
```

2、服务状态

```shell
sudo service krb5kdc status
sudo service kadmin status
```

3、关闭服务

```shell
sudo service krb5kdc stop
sudo service kadmin stop
```

4、重启服务

```shell
sudo service krb5kdc restart
sudo service kadmin restart
```

5、查看日志

````shell
sudo tail -f /var/log/krb5kdc.log
sudo tail -f /var/log/kadmind.log
````

## Kerberos 用户管理

KDC 支持两种方式管理：

- 在KDC本机使用  `kadmin.local` 管理，需要登录 KDC，无需输入 Kerberos 管理员密码
- 远程或在 KDC本机使用 `kadmin`管理，需要 Kerberos 管理员密码

两种操作相同，为行文简单，本文均讨论使用 `kadmin.local` 方式。

- 方式 1 使用登录：`$ sudo kadmin.local`

```shell
sudo kadmin.local
Authenticating as principal hive/admin@MYREALM with password.
kadmin.local: # 可执行命令
```

- 方式 2：使用 kadmin 管理用户

```shell
sudo kadmin -p root/admin -q "addprinc -randkey futeng/host128138@MYREALM"
```

### 查询所有用户（`list_principals`）

```shell
kadmin.local:  list_principals
```

### 查询用户详情（get_principal）

```shell
kadmin.local:  get_principal futeng
```

### 新增用户（add_principal USER）

1、使用默认配置

```shell
kadmin.local:  add_principal futeng # 新增 futeng 用户
WARNING: no policy specified for futeng@MYREALM; defaulting to no policy
Enter password for principal "futeng@MYREALM": # 输入密码
Re-enter password for principal "futeng@MYREALM":# 确认密码
Principal "futeng@MYREALM" created.
kadmin.local:  quit # 退出

# 测试
$ kinit futeng
Password for futeng@MYREALM: # 输入密码
$ klist 
Ticket cache: FILE:/tmp/krb5cc_1000 # 登录成功
Default principal: futeng@MYREALM

Valid starting     Expires            Service principal
09/07/20 17:14:45  09/08/20 17:14:45  krbtgt/MYREALM@MYREALM
	renew until 09/14/20 17:14:45
```

2、指定有效期

```shell
kadmin.local:  addprinc
usage: add_principal [options] principal
	options are:
		[-randkey|-nokey] [-x db_princ_args]* [-expire expdate] [-pwexpire pwexpdate] [-maxlife maxtixlife]
		[-kvno kvno] [-policy policy] [-clearpolicy]
		[-pw password] [-maxrenewlife maxrenewlife]
		[-e keysaltlist]
		[{+|-}attribute]

# 用户有效期 -expire
addprinc -expire "2020-9-9 20:00" test5
# 用户有效期 + 用户密码有效期 -pwexpire
addprinc -expire "2020-9-9 20:00" -pwexpire "2020-9-10 20:00" test6
# 用户有效期 + 用户密码有效期 + 票据生命有效期
modify_principal -expire "6/6/2009 12:01am EST" 
    -pwexpire "6/7/2009 12:01am EST" -maxlife "12:00" user1
```

### 修改用户（modify_principal）

```shell
usage: modify_principal [options] principal
	options are:
		[-x db_princ_args]* [-expire expdate] [-pwexpire pwexpdate] [-maxlife maxtixlife]
		[-kvno kvno] [-policy policy] [-clearpolicy]
		[-maxrenewlife maxrenewlife] [-unlock] [{+|-}attribute]
	attributes are:
		allow_postdated allow_forwardable allow_tgs_req allow_renewable
		allow_proxiable allow_dup_skey allow_tix requires_preauth
		requires_hwauth needchange allow_svr password_changing_service
		ok_as_delegate ok_to_auth_as_delegate no_auth_data_required
		lockdown_keys

where,
	[-x db_princ_args]* - any number of database specific arguments.
			Look at each database documentation for supported arguments
```

### 生成 keytab（ktadd）

```shell
$ sudo kadmin.local
kadmin.local:  ktadd -k /tmp/futeng.keytab futeng
Entry for principal futeng with kvno 4, encryption type des3-cbc-sha1 added to keytab WRFILE:/tmp/futeng.keytab.
Entry for principal futeng with kvno 4, encryption type arcfour-hmac added to keytab WRFILE:/tmp/futeng.keytab.
Entry for principal futeng with kvno 4, encryption type des-hmac-sha1 added to keytab WRFILE:/tmp/futeng.keytab.
Entry for principal futeng with kvno 4, encryption type des-cbc-md5 added to keytab WRFILE:/tmp/futeng.keytab.
```

### 删除用户（delete_principal）

```shell
kadmin.local:  delete_principal futeng
Are you sure you want to delete the principal "futeng@MYREALM"? (yes/no): yes
Principal "futeng@MYREALM" deleted.
Make sure that you have removed this principal from all ACLs before reusing.
```

## 有效期管理

Kerberos 可以支持两个层面的有效期管理，包括登录（票据）有效期（ticket lifetime）和用户有效期（user lifetime）。

### 票据（登录）有效期

登录有效期用于控制用户在客户端登录前后的生命周期管理，包括登录、销毁和刷新等操作（可类比 web 系统登录，可以登入、登出和刷新 session）。登录有效期受客户端配置、KDC 端配置和 Principal 属性控制。

使用 `klist` 可以看到当前登录有效期信息：

```shell
$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: test3h@MYREALM

①Valid starting     ②Expires            Service principal
09/08/20 09:53:04  09/08/20 10:53:04  krbtgt/MYREALM@MYREALM
	③renew until 09/15/20 09:53:04
```

重点参数：

- ①`Valid starting`：Ticket 认证开始时间
- ②`Expires`：Ticket 认证过期时间
- ③`renew until`：Ticket 认证的==刷新截止时间==

当然Ticket 认证过期时间不是随机的，而是有客户端和 kdc 端重点参数 **Ticket 生命有效期（ticket_lifetime）**规定的。同样的，最大刷新时间也不是无限制可刷新，而是有参数 **Ticket 最大可刷新有效期（renew_lifetime）** 控制的：

- ④`ticket lifetime`：Ticket 生命有效期 
- ⑤`renew_lifetime`：Ticket 最大可刷新有效期

例如，客户端配置：

```shell
$ sudo cat /etc/krb5.conf

[libdefaults]
  ⑤renew_lifetime = 7d
  forwardable = true
  default_realm = MYREALM
  ④ticket_lifetime = 1h
  ...
```

### 用户有效期

用户有效期用于控制 Kerberos 用户可登录系统的生命周期。用户有效期只有使用 `kadmin` 管理员工具配置。可配置对Kerberos 用户的用户有效期和用户密码有效期：

例如：

```shell
# 72小时后
kadmin.local: modify_principal 
							-expire "72 hours" ①
							-pwexpire "6/7/2020 12:01am EST" principal_name ②
	
#以上为行为方便，实际需要保持在同一行执行
```

参数：

- `expire`：用户有效期
- `pwexpire`：用户密码有效期

### 参数一览


| 分类     | 控制参数         | klist          | krb5.conf       | kdc.conf           | kadmin principal |
| -------- | ---------------- | -------------- | --------------- | ------------------ | ---------------- |
| 登录控制 | 认证开始时间     | Valid starting |                 |                    |                  |
| 登录控制 | 认证过期时间     | Expires        |                 |                    |                  |
| 登录控制 | 刷新截止时间     | renew until    |                 |                    |                  |
| 登录控制 | 票据有效周期     |                | ticket_lifetime | max_life           |                  |
| 登录控制 | 最大刷新周期     |                | renew_lifetime  | max_renewable_life |                  |
| 用户级   | 用户过期时间     |                |                 |                    | expire           |
| 用户级   | 用户密码过期时间 |                |                 |                    | pwexpire         |

### 支持的时间格式

参考：[Supported date and time formats](https://web.mit.edu/kerberos/krb5-1.12/doc/basic/date_format.html#getdate)

|                | Format       | Example  |
| -------------- | ------------ | -------- |
| Date           | mm/dd/yy     | 07/27/12 |
| month dd, yyyy | Jul 27, 2012 |          |
| yyyy-mm-dd     | 2012-07-27   |          |
| Absolute time  | HH:mm[:ss]pp | 08:30 PM |
| hh:mm[:ss]     | 20:30        |          |
| Relative time  | N tt         | 30 sec   |
| Time zone      | Z            | EST      |
| z              | -0400        |          |

缩写含义：

- *month* : locale’s month name or its abbreviation;
- *dd* : day of month (01-31);
- *HH* : hours (00-12);
- *hh* : hours (00-23);
- *mm* : in time - minutes (00-59); in date - month (01-12);
- *N* : number;
- *pp* : AM or PM;
- *ss* : seconds (00-60);
- *tt* : time units (hours, minutes, min, seconds, sec);
- *yyyy* : year;
- *yy* : last two digits of the year;
- *Z* : alphabetic time zone abbreviation;
- *z* : numeric time zone;

示例：

```shell
# 2020-9-9 20:00 过期
modify_principal -expire "2020-9-9 20:00" test3h
# 2020年9月6日 东部时间 12:01 上午过期
modify_principal -expire "9/6/2020 12:01am EST" principal_name
# 2020年1月13日 晚上10:05
modify_principal -expire "January 23, 2020 10:05pm" principal_name
# 今天22点
modify_principal -expire "22:00" principal_name
# 30分钟后过期
modify_principal -expire "30 minutes" principal_name
# 72小时后
modify_principal -expire "72 hours" principal_name
```

### 关于 ticket renew

可类比 web 系统，假设系统规定10分钟用户无操作将默认登出，但是另设有『刷新』按钮，允许用户点击刷新，从而将登录的 session 有效期后延。

## Kerberos 常用操作

本文讨论的 Kerberos 票据（tickets），指使用 kinit 登录后的登录凭据，表现为由 Kerberos 客户端维护的一份登录缓存，可使用 klist 查看。 

常规客户端票据管理步骤：

1. 使用 `kinit -kt keytab principal` 登录，连接服务执行业务操作
2. 本次登录有效期（session）控制：
   - 本次 session 还未彻底过期：使用 `kinit -R` 刷新登录有效期，重新计时
   - 本次 session 已彻底过期：使用 `kinit -kt keytab principal` 再次登录

3. 使用 `kdestory` 登出

未控制访问安全，建议每次登出前执行登出操作，也可以将 `kdestory` 命令写到 `.logout` 配置中以执行自动登出。

###  查看认证缓存（klist）

klist 常用选项：

- `-k`：列出 keytab 文件中包含的所有 keys（类似 Principal）
- `-e`：列出加密类型
- `-l`：查询缓存状态

```shell
$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: futeng@MYREALM

Valid starting     Expires            Service principal
09/02/20 15:50:29  09/03/20 15:50:29  krbtgt/MYREALM@MYREALM
	renew until 09/09/20 15:50:29
	
$ sudo klist -e
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: hdfs-cluster137@MYREALM

Valid starting       Expires              Service principal
08/20/2020 14:35:41  08/21/2020 14:35:41  krbtgt/MYREALM@MYREALM
	renew until 08/27/2020 14:35:41, Etype (skey, tkt): des3-cbc-sha1, des3-cbc-sha1
```

klist 命令呈现2部分信息：

- 基本信息
  - `Ticket cache`：缓存文件位置
  - `Default principal`：默认 Principal（会以当前 TTY 登录用户+krb5.conf
- 有效期信息
  -  `Valid starting`：开始认证的时间
  -  `Expires`：认证过期时间
  -  `Service principal`：TGT 票据会以 `krbtgt` 打头，后面包含域名（realm name）
  -  `renew until`：票据的最大有效期

### 查看登录状态（`klist -l`）

```shell
$ klist -l
Principal name                 Cache name
--------------                 ----------
futeng@MYREALM                    FILE:/tmp/krb5cc_1000 (Expired)
```

### 查看 keytab 对应的 Principal（`klist -ke keytab`）

可通过 `klist -ke keytab` 查询 keytab 文件中包含了哪些可用的 Principal

    $ sudo klist -ke /etc/security/keytabs/hdfs.headless.keytab
    Keytab name: FILE:/etc/security/keytabs/hdfs.headless.keytab
    KVNO Principal
    ---- --------------------------------------------------------------------------
       2 hdfs-cluster137@MYREALM (des-cbc-md5)
       2 hdfs-cluster137@MYREALM (aes256-cts-hmac-sha1-96)
       2 hdfs-cluster137@MYREALM (arcfour-hmac)
       2 hdfs-cluster137@MYREALM (des3-cbc-sha1)
       2 hdfs-cluster137@MYREALM (aes128-cts-hmac-sha1-96)

### 使用 kinit 获取缓存票据

通常用户登录都会伴随 session 控制，例如登录 web 系统，再一段时间没有操作后会强制 session 失效，用户需要再次登录方能有效。

Kerberos 体系中，我们可以通过 `kinit` 工具来控制登录，例如在登录过期后，再次登录。

- 方式 1：密码方式登录（kinit）

```shell
$ kinit futeng
Password for futeng@MYREALM: # 输入密码即可
```

- 方式 2：keytab 方式登录（`kinit -kt keytab principal`）

```shell
$ sudo kinit -kt /etc/security/keytabs/hdfs.headless.keytab hdfs-cluster137@MYREALM
```

### 延长登录（`kinit -R`）

延长登录常见与强 session 控制的 web 系统。例如有警务查询系统，一次查询时间控制在15分钟，当你查询到第10分钟时，可以点击【延长登录】按钮，此时，查询剩余时间将重新从15分钟开始计时。不过，一旦一次查询超过控制，则无法延长，需要重新登录。

在 Kerberos 体系中，也提供了延长登录的机制（`kinit -R`）。

```shell
$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: futeng@MYREALM

Valid starting     Expires            Service principal
09/07/20 16:21:57  09/08/20 16:21:57  krbtgt/MYREALM@MYREALM 
	renew until 09/14/20 16:21:57
# futeng 用户
# 本次登录开始时间为：09/07/20 16:21:57
# 本次登录的过期时间是：09/08/20 16:21:57，可以发现有效期为 24h

# 延长登录时间，认证开始时间将重置为当前时刻
$ kinit -R  
$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: futeng@MYREALM

Valid starting     Expires            Service principal
09/07/20 16:22:05  09/08/20 16:22:05  krbtgt/MYREALM@MYREALM
	renew until 09/14/20 16:21:57
	
# 当本次登录已经失效，则无法刷新（ticket expired while renewing credentials），需要重新登录
$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: futeng@MYREALM
$ klist -l
Principal name                 Cache name
--------------                 ----------
futeng@MYREALM                    FILE:/tmp/krb5cc_1000 (Expired)
Valid starting     Expires            Service principal
09/02/20 15:50:29  09/03/20 15:50:29  krbtgt/MYREALM@MYREALM
	renew until 09/09/20 15:50:29
[futeng@host128137 keytabs]$ kinit -R
kinit: Ticket expired while renewing credentials
```

### 指定登录有效期（`kinit -l`）

可通过 `kinit -l` 指定档次登录的有效期，支持配置：s 秒，m 分钟，h小时，d 天。

```shell
$ kinit -l 1h -kt futeng.keytab futeng@MYREALM
$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: futeng@MYREALM

Valid starting     Expires            Service principal
09/07/20 16:33:58  09/07/20 17:33:58  krbtgt/MYREALM@MYREALM
	renew until 09/14/20 16:33:58
	
# 其他
kinit -l 3600
kinit -l 5:00
kinit -l 30m
kinit -l "10d 0h 0m 0s"

# 同时制定有效期和 renew 最大可更新时间
kinit -l 10h -r 5d my_principal
```

### 使用 kdestroy 销毁票据

```shell
$ kdestroy
```

# 常见问题

## 无法使用 kadmin 创建用户

报错：Operation requires ``add'' privilege

    sudo kadmin -p root/admin -q "addprinc -randkey futeng/host128138@MYREALM"
    Authenticating as principal root/admin with password.
    Password for root/admin@MYREALM:
    WARNING: no policy specified for futeng/host128138@MYREALM; defaulting to no policy
    add_principal: Operation requires ``add'' privilege while creating "futeng/host128138@MYREALM".

原因：需要开启用户权限，此处为 root/admin 用户实际无 add 权限

解决：

1、修改 acl 权限 vim /var/kerberos/krb5kdc/kadm5.acl

    # 默认为下面
    # */admin@EXAMPLE.COM     *
    # 修改为实际领域名称，本例为 MYREALM
    */admin@MYREALM     *

2、重启服务

    $ sudo service kadmin restart
    $ sudo service krb5kdc restart

3、再次执行即可

##  无法访问受 Kerberos 包含的页面

参考：[How to Configure Browsers for Kerberos Authentication](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_sg_browser_access_kerberos_protected_url.html#topic_6_2__section_cg5_gwv_ls)

建议使用火狐浏览器访问，配置最为简单。

解决：

```shell
# 浏览器标签页，输入：about:config
# 搜索：network.negotiate-auth.trusted-uris
# 值输入要访问的地址：，逗号分割
```

![image-20200911145456626](https://tva1.sinaimg.cn/large/e6c9d24ely1h39a3bt55vj21z40cqtay.jpg)

# 参考资料

## man klist

```shell
NAME
       klist - list cached Kerberos tickets
SYNOPSIS
       klist [-e] [[-c] [-l] [-A] [-f] [-s] [-a [-n]]] [-C] [-k [-t] [-K]] [-V] [cache_name|keytab_name]
DESCRIPTION
       klist lists the Kerberos principal and Kerberos tickets held in a credentials cache, or the keys held in a keytab file.
OPTIONS
       -e     Displays the encryption types of the session key and the ticket for each credential in the credential cache, or each key in the keytab file.
       -l     If a cache collection is available, displays a table summarizing the caches present in the collection.
       -A     If a cache collection is available, displays the contents of all of the caches in the collection.
       -c     List tickets held in a credentials cache. This is the default if neither -c nor -k is specified.
       -f     Shows the flags present in the credentials, using the following abbreviations:
                 F    Forwardable
                 f    forwarded
                 P    Proxiable
                 p    proxy
                 D    postDateable
                 d    postdated
                 R    Renewable
                 I    Initial
                 i    invalid
                 H    Hardware authenticated
                 A    preAuthenticated
                 T    Transit policy checked
                 O    Okay as delegate
                 a    anonymous
       -s     Causes klist to run silently (produce no output).  klist will exit with status 1 if the credentials cache cannot be read or is expired, and with status 0 otherwise.
       -a     Display list of addresses in credentials.
       -n     Show numeric addresses instead of reverse-resolving addresses.
       -C     List configuration data that has been stored in the credentials cache when klist encounters it.  By default, configuration data is not listed.
       -k     List keys held in a keytab file.
       -i     In combination with -k, defaults to using the default client keytab instead of the default acceptor keytab, if no name is given.
       -t     Display the time entry timestamps for each keytab entry in the keytab file.
       -K     Display the value of the encryption key in each keytab entry in the keytab file.
       -V     Display the Kerberos version number and exit.
       If  cache_name or keytab_name is not specified, klist will display the credentials in the default credentials cache or keytab file as appropriate.  If the KRB5CCNAME environment variable is
       set, its value is used to locate the default ticket cache.
ENVIRONMENT
       See kerberos(7) for a description of Kerberos environment variables.
FILES
       FILE:/tmp/krb5cc_%{uid}
             Default location of Kerberos 5 credentials cache
       FILE:/etc/krb5.keytab
              Default location for the local host's keytab file.
SEE ALSO
       kinit(1), kdestroy(1), kerberos(7)
AUTHOR
       MIT
COPYRIGHT
       1985-2017, MIT
```

## man kinit

```shell
KINIT(1)                                                                                    MIT Kerberos                                                                                    NAME
       kinit - obtain and cache Kerberos ticket-granting ticket
SYNOPSIS
       kinit [-V] [-l lifetime] [-s start_time] [-r renewable_life] [-p | -P] [-f | -F] [-a] [-A] [-C] [-E] [-v] [-R] [-k [-t keytab_file]] [-c cache_name] [-n] [-S service_name] [-I input_ccache]
       [-T armor_ccache] [-X attribute[=value]] [principal]
DESCRIPTION
       kinit obtains and caches an initial ticket-granting ticket for principal.  If principal is absent, kinit chooses an appropriate principal name based on existing credential cache contents or
       the local username of the user invoking kinit.  Some options modify the choice of principal name.
OPTIONS
       -V     display verbose output.
       -l lifetime
              (duration string.)  Requests a ticket with the lifetime lifetime.
              For example, kinit -l 5:30 or kinit -l 5h30m.
              If the -l option is not specified, the default ticket lifetime (configured by each site) is used.  Specifying a ticket lifetime longer than the maximum ticket lifetime (configured by
              each site) will not override the configured maximum ticket lifetime.
       -s start_time
              (duration string.)  Requests a postdated ticket.  Postdated tickets are issued with the invalid flag set, and need to be resubmitted to the KDC for validation before use.
              start_time specifies the duration of the delay before the ticket can become valid.
       -r renewable_life
              (duration string.)  Requests renewable tickets, with a total lifetime of renewable_life.
       -f     requests forwardable tickets.
       -F     requests non-forwardable tickets.
       -p     requests proxiable tickets.
       -P     requests non-proxiable tickets.
       -a     requests tickets restricted to the host's local address[es].
       -A     requests tickets not restricted by address.
       -C     requests canonicalization of the principal name, and allows the KDC to reply with a different client principal from the one requested.
       -E     treats the principal name as an enterprise name (implies the -C option).
       -v     requests that the ticket-granting ticket in the cache (with the invalid flag set) be passed to the KDC for validation.  If the ticket is within its requested time range, the cache is
              replaced with the validated ticket.
       -R     requests renewal of the ticket-granting ticket.  Note that an expired ticket cannot be renewed, even if the ticket is still within its renewable life.
              Note  that  renewable  tickets  that have expired as reported by klist(1) may sometimes be renewed using this option, because the KDC applies a grace period to account for client-KDC
              clock skew.  See krb5.conf(5) clockskew setting.
       -k [-i | -t keytab_file]
              requests a ticket, obtained from a key in the local host's keytab.  The location of the keytab may be specified with the -t keytab_file option, or with the -i option to  specify  the
              use  of  the  default  client keytab; otherwise the default keytab will be used.  By default, a host ticket for the local host is requested, but any principal may be specified.  On a
              KDC, the special keytab location KDB: can be used to indicate that kinit should open the KDC database and look up the key directly.  This permits an administrator to  obtain  tickets
              as any principal that supports authentication based on the key.

       -n     Requests anonymous processing.  Two types of anonymous principals are supported.

              For  fully  anonymous Kerberos, configure pkinit on the KDC and configure pkinit_anchors in the client's krb5.conf(5).  Then use the -n option with a principal of the form @REALM (an
              empty principal name followed by the at-sign and a realm name).  If permitted by the KDC, an anonymous ticket will be returned.
              A second form of anonymous tickets is supported; these realm-exposed tickets hide the identity of the client but not the client's realm.  For this mode, use kinit -n  with  a  normal
              principal name.  If supported by the KDC, the principal (but not realm) will be replaced by the anonymous principal.
              As of release 1.8, the MIT Kerberos KDC only supports fully anonymous operation.
       -I input_ccache
          Specifies  the  name  of  a credentials cache that already contains a ticket.  When obtaining that ticket, if information about how that ticket was obtained was also stored to the cache,
          that information will be used to affect how new credentials are obtained, including preselecting the same methods of authenticating to the KDC.
       -T armor_ccache
              Specifies the name of a credentials cache that already contains a ticket.  If supported by the KDC, this cache will be used  to  armor  the  request,  preventing  offline  dictionary
              attacks and allowing the use of additional preauthentication mechanisms.  Armoring also makes sure that the response from the KDC is not modified in transit.
       -c cache_name
              use cache_name as the Kerberos 5 credentials (ticket) cache location.  If this option is not used, the default cache location is used.
              The  default cache location may vary between systems.  If the KRB5CCNAME environment variable is set, its value is used to locate the default cache.  If a principal name is specified
              and the type of the default cache supports a collection (such as the DIR type), an existing cache containing credentials for the principal is selected or a new  one  is  created  and
              becomes the new primary cache.  Otherwise, any existing contents of the default cache are destroyed by kinit.
       -S service_name
              specify an alternate service name to use when getting initial tickets.
       -X attribute[=value]
              specify  a pre-authentication attribute and value to be interpreted by pre-authentication modules.  The acceptable attribute and value values vary from module to module.  This option
              may be specified multiple times to specify multiple attributes.  If no value is specified, it is assumed to be "yes".
              The following attributes are recognized by the PKINIT pre-authentication mechanism:
              X509_user_identity=value
                     specify where to find user's X509 identity information
              X509_anchors=value
                     specify where to find trusted X509 anchor information
              flag_RSA_PROTOCOL[=yes]
                     specify use of RSA, rather than the default Diffie-Hellman protocol
ENVIRONMENT
       See kerberos(7) for a description of Kerberos environment variables.
FILES
       FILE:/tmp/krb5cc_%{uid}
              default location of Kerberos 5 credentials cache
       FILE:/etc/krb5.keytab
              default location for the local host's keytab.
SEE ALSO
       klist(1), kdestroy(1), kerberos(7)
AUTHOR
       MIT
COPYRIGHT
       1985-2017, MIT
1.15.1
```

## man kadmin

支持的操作：

    add_principal, addprinc, ank
                             Add principal
    delete_principal, delprinc
                             Delete principal
    modify_principal, modprinc
                             Modify principal
    rename_principal, renprinc
                             Rename principal
    change_password, cpw     Change password
    get_principal, getprinc  Get principal
    list_principals, listprincs, get_principals, getprincs
                             List principals
    add_policy, addpol       Add policy
    modify_policy, modpol    Modify policy
    delete_policy, delpol    Delete policy
    get_policy, getpol       Get policy
    list_policies, listpols, get_policies, getpols
                             List policies
    get_privs, getprivs      Get privileges
    ktadd, xst               Add entry(s) to a keytab
    ktremove, ktrem          Remove entry(s) from a keytab
    lock                     Lock database exclusively (use with extreme caution!)
    unlock                   Release exclusive database lock
    purgekeys                Purge previously retained old keys from a principal
    get_strings, getstrs     Show string attributes on a principal
    set_string, setstr       Set a string attribute on a principal
    del_string, delstr       Delete a string attribute on a principal
    list_requests, lr, ?     List available requests.
    quit, exit, q            Exit program.



## 其他参考

- 参考《[Ticket management](http://web.mit.edu/kerberos/krb5-latest/doc/user/tkt_mgmt.html#ticket-management)》
- [轻松迁移到带有 Kerberos-5 支持的 OpenAFS](https://www.oschina.net/question/12_8782)
- [Supported date and time formats](https://web.mit.edu/kerberos/krb5-1.12/doc/basic/date_format.html#getdate)
- [kadmin](https://web.mit.edu/kerberos/krb5-1.12/doc/admin/admin_commands/kadmin_local.html)

# 更新记录

- 2020-09-07 16:02 | Teng Fu 

