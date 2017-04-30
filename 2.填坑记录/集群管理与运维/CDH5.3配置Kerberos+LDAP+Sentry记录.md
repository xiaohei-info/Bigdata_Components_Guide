## 系统环境说明

- 操作系统：Centos6.5
- CDH版本：5.3
- JDK版本：1.7
- 操作用户：root
- Kerberos版本：1.10.3   
- LDAP版本：2.4.40   
- Sentry版本：1.4   

**集群配置**

- 机器数量：5   
	- 内存：64G   
	- 硬盘：10T   
	- CPU核心数：24   

- 运行的服务：HDFS、Yarn、HBase、Hive、Sqoop2、Impala、Zookeeper、Hue、Sentry、Oozie、CM   
- 各服务主进程内存划分：1G（默认值）
- 各节点可分配的CPU核心数：24
- 各节点可分配的内存：6-8G
- 各容器可使用的最大内存：7-8G
- MapReduce任务分配的内存：1G   

**节点规划**

|124|123|122|121|120|
|---|---|---|---|---|
|NN+DN|SNN+DN|DN|DN|DN|
|HM+HR|HR+HTS+HRS|HR|HR|HR|
|NM|RM+NM|NM|NM|NM|
|HiveM+Server2|Hive Server2||||
|ImpalaC+ImpalaS|Impala|Impala|Impala|Impala|
|||zk|zk|zk|
|CM*5||||||

各节点之间可以通过ssh免密码登录

通过CM来帮助快速对集群集成Kerberos的身份认证、LDAP的账号认证和Sentry的角色授权（CM安装的集群推荐使用）     
通过YUM安装的CDH方式可以参考：http://blog.javachen.com/2014/11/04/config-kerberos-in-cdh-hdfs.html

**使用自动化脚本之前确保脚本的目录结构（shell目录）不变！**

## Kerberos

### 安装

hadoop-10-0-8-124作为Kerberos主节点安装服务：

```shell
yum install krb5-server -y
```

检查shell脚本目录下的master和slaves文件的主机名是否正确

其他子节点安装krb5-devel、krb5-workstation ：

```shell
./tool.sh -r "yum install krb5-devel krb5-workstation -y"
```

### 修改配置文件

kdc服务器包含三个配置文件：

```
# 集群上所有节点都有这个文件而且内容同步
/etc/krb5.conf
# 主服务器上的kdc配置
/var/kerberos/krb5kdc/kdc.conf
# 能够不直接访问 KDC 控制台而从 Kerberos 数据库添加和删除主体，需要添加配置
/var/kerberos/krb5kdc/kadm5.acl
```

修改/etc/krb5.conf为以下内容

```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log
[libdefaults]
 default_realm = XIAOHEI.INFO
 dns_lookup_kdc = false
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 default_tgs_enctypes = rc4-hmac
 default_tkt_enctypes = rc4-hmac
 permitted_enctypes = rc4-hmac
 clockskew = 120
 udp_preference_limit = 1
[realms]
 XIAOHEI.INFO = {
 kdc = hadoop-10-0-8-124
 admin_server = hadoop-10-0-8-124
 }
[domain_realm]
 .XIAOHEI.INFO = XIAOHEI.INFO
 XIAOHEI.INFO = XIAOHEI.INFO
```

配置项说明：

- [logging]：日志输出设置     
- [libdefaults]：连接的默认配置   
	- default_realm：Kerberos应用程序的默认领域，所有的principal都将带有这个领域标志   
	- ticket_lifetime： 表明凭证生效的时限，一般为24小时   
	- renew_lifetime： 表明凭证最长可以被延期的时限，一般为一个礼拜。当凭证过期之后，对安全认证的服务的后续访问则会失败   
	- clockskew：时钟偏差是不完全符合主机系统时钟的票据时戳的容差，超过此容差将不接受此票据。通常，将时钟扭斜设置为 300 秒（5 分钟）。这意味着从服务器的角度看，票证的时间戳与它的偏差可以是在前后 5 分钟内   
	- udp_preference_limit= 1：禁止使用 udp 可以防止一个 Hadoop 中的错误
- [realms]：列举使用的 realm    
	- kdc：代表要 kdc 的位置。格式是 机器:端口   
	- admin_server：代表 admin 的位置。格式是 机器:端口   
	- default_domain：代表默认的域名   
- [domain_realm]：域名到realm的关系

修改/var/kerberos/krb5kdc/kdc.conf

```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 XIAOHEI.INFO = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  max_renewable_life = 7d
  max_life = 1d
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
  default_principal_flags = +renewable, +forwardable
 }
```

配置项说明：

- kdcdefaults：kdc相关配置，这里只设置了端口信息   
- realms：realms的配置
	- XIAOHEI.INFO：设定的realms领域   
	- master_key_type：和 supported_enctypes 默认使用 aes256-cts。JAVA 使用 aes256-cts 验证方式需要安装 JCE 包   
	- acl_file：标注了 admin 的用户权限，文件格式是：Kerberos_principal permissions [target_principal] [restrictions]   
	- supported_enctypes：支持的校验方式   
	- admin_keytab：KDC 进行校验的 keytab   

安装JCE包需要到[Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files for JDK/JRE 7](http://www.oracle.com/technetwork/java/embedded/embedded-se/downloads/jce-7-download-432124.html)这个网址下载，解压之后到$JAVA_HOME/jre/lib/security目录

创建/var/kerberos/krb5kdc/kadm5.acl，内容为：

```
*/admin@XIAOHEI.INFO	*
```

配置文件详细的说明参考官网

### 创建Kerberos数据库

```shell
# 保存路径为/var/kerberos/krb5kdc 如果需要重建数据库，将该目录下的principal相关的文件删除即可
kdb5_util create -r XIAOHEI.INFO -s
```

-r指定配置的realm领域名

出现 Loading random data 的时候另开个终端执行点消耗CPU的命令如 cat /dev/sda > /dev/urandom 可以加快随机数采集   
该命令会在 /var/kerberos/krb5kdc/ 目录下创建 principal 数据库

### 启动服务

在主节点上执行：

```shell
chkconfig --level 35 krb5kdc on
chkconfig --level 35 kadmin on
service krb5kdc start
service kadmin start
```

### 创建Kerberos管理员

```shell
# 需要设置两次密码，当前设置为root
kadmin.local -q "addprinc xiaohei/admin"
```

principal的名字的第二部分是admin,那么该principal就拥有administrative privileges   
**这个账号将会被CDH用来生成其他用户/服务的principal**

### 测试Kerberos

```shell
# 列出Kerberos中的所有认证用户，即principals
kadmin.local -q "list_principals"
# 添加认证用户，需要输入密码
kadmin.local -q "addprinc user1"
# 使用该用户登录，获取身份认证，需要输入密码
kinit user1
# 查看当前用户的认证信息ticket
klist
# 更新ticket
kinit -R
# 销毁当前的ticket
kdestroy
# 删除认证用户
kadmin.local -q "delprinc user1"
```

抽取密钥并将其储存在本地 keytab 文件 /etc/krb5.keytab 中。这个文件由超级用户拥有，所以必须是 root 用户才能在 kadmin shell 中执行以下命令：

```shell
kadmin.local -q "ktadd kadmin/admin"
# 查看生成的keytab
klist -k /etc/krb5.keytab
```

## CDH启用Kerberos

在CM的界面上点击启用Kerberos
启用的时候需要确认几个事情：

> 1.KDC已经安装好并且正在运行   
> 2.将KDC配置为允许renewable tickets with non-zerolifetime   
	- 在之前修改kdc.conf文件的时候已经添加了kdc_tcp_ports、max_life和max_renewable_life这个三个选项   
> 3.在Cloudera Manager Server上安装openldap-clients    
> 4.为Cloudera Manager创建一个principal，使其能够有权限在KDC中创建其他的principals，就是上面创建的Kerberos管理员账号

确定完了之后点击continue，进入下一页进行配置，要注意的是：这里的『Kerberos Encryption Types』必须跟KDC实际支持的加密类型匹配（即kdc.conf中的值）   
这里使用了默认的aes256-cts

**注意，这里的『Kerberos Encryption Types』必须和/etc/krb5.conf中的default_tgs_enctypes、default_tkt_enctypes和permitted_enctypes三个选项的值对应起来，不然会出现集群服务无法认证通过的情况！**   
**异常信息：javax.security.auth.login.LoginException: No supported encryption types listed in default_tkt_enctypes**

点击continue，进入下一页，这一页中可以不勾选『Manage krb5.conf through Cloudera Manager』   
注意，如果勾选了这个选项就可以通过CM的管理界面来部署krb5.conf，**但是实际操作过程中发现有些配置仍然需要手动修改该文件并同步**   

点击continue，进入下一页，输入Cloudera Manager Principal的管理员账号和密码，**注意输入账号的时候要使用@前要使用全称，xiaohei/admin**

点击continue，进入下一页，导入KDC Account Manager Credentials

点击continue，进入下一页，restart cluster并且enable Kerberos

之后CM会自动重启集群服务，启动之后会会提示Kerberos已启用

这个过程中CM会自动在Kerberos的数据库中创建各个节点中各个账户对应的principle

可以使用

```shell
kadmin.local -q "list_principals"
```

来查看，格式为username/hostname@XIAOHEI.INFO，例如hdfs/hadoop-10-0-8-124@XIAOHEI.INFO   

在CM上启用Kerberos的过程中，CM会自动做以下的事情：

> 1.集群中有多少个节点，每个账户都会生成对应个数的principal   
> 2.为每个对应的principal创建keytab   
> 3.部署keytab文件到指定的节点中   
> 4.在每个服务的配置文件中加入有关Kerberos的配置

**其中包括Zookeeper服务所需要的jaas.conf和keytab文件都会自动设定并读取，如果用户仍然手动修改了Zookeeper的服务，要确保这两个文件的路径和内容正确性**

keytab是包含principals和加密principal key的文件   
keytab文件对于每个host是唯一的，因为key中包含hostname   
keytab文件用于不需要人工交互和保存纯文本密码，实现到kerberos上验证一个主机上的principal

启用之后访问集群的所有资源都需要使用相应的账号来访问，否则会无法通过Kerberos的authenticatin

### 创建HDFS超级用户

此时直接用CM生成的principal访问HDFS会失败，因为那些自动生成的principal的密码是随机的，用户并不知道   
而通过命令行的方式访问HDFS需要先使用kinit来登录并获得ticket   
所以使用kinit hdfs/hadoop-10-0-8-124@XIAOHEI.INFO需要输入密码的时候无法继续   

用户可以通过创建一个hdfs@XIAOHEI.INFO的principal并记住密码从命令行中访问HDFS

```shell
# 需要输入两遍密码
kadmin.local -q "addprinc hdfs"
```

先使用

```shell
kinit hdfs@XIAOHEI.INFO
```

登录之后就可以通过认证并访问HDFS   
默认hdfs用户是超级用户

### 为每个用户创建principal

当集群运行Kerberos后，每一个Hadoop user都必须有一个principal或者keytab来获取Kerberos credentials   
（即使用密码的方式或者使用keytab验证的方式）   
这样才能访问集群并使用Hadoop的服务   
也就是说，如果Hadoop集群存在一个名为hdfs@XIAOHEI.INFO的principal   
那么在集群的每一个节点上应该存在一个名为hdfs的Linux用户   
同时，在HDFS中的目录/user要存在相应的用户目录（即/user/hdfs），且该目录的owner和group都要是hdfs

一般来说，Hadoop user会对应集群中的每个服务   
即一个服务对应一个user   
例如impala服务对应用户impala

检查shell脚本目录下的hadoop-user文件的用户名是否正确

```shell
./tool.sh -g
```

**至此，集群上的服务都启用了Kerberos的安全认证**

### 确认Kerberos在集群上正常工作

**1.确认HDFS可以正常使用**

登录到某一个节点后，切换到hdfs用户，然后用kinit来获取credentials
现在用'hadoop dfs -ls /'应该能正常输出结果

用kdestroy销毁credentials后，再使用hadoop dfs -ls /会发现报错

**2.确认可以正常提交MapReduce job**

获取了hdfs的证书后，提交一个PI程序，如果能正常提交并成功运行，则说明Kerberized Hadoop cluster在正常工作

### 集群集成Kerberos过程中遇到的坑

#### hdfs用户提交mr作业无法运行

**异常信息：**   
INFO mapreduce.Job: Job job_1442654915965_0002 failed with state FAILED due to: Application application_1442654915965_0002 failed 2 times due to AM Container for appattempt_1442654915965_0002_000002 exited with exitCode: -1000 due to: Application application_1442654915965_0002 initialization failed (exitCode=255) with output: **Requested user hdfs is not whitelisted and has id 496,which is below the minimum allowed 1000**

**原因：**   
Linux user 的 user id 要大于等于1000，否则会无法提交Job   
例如，如果以hdfs（id为490）的身份提交一个job，就会看到以上的错误信息

**解决方法：**   
> 1.使用命令 usermod -u <new-user-id> <user>修改一个用户的user id   
> 2.修改Clouder关于这个该项的设置，Yarn->配置->min.user.id修改为合适的值，当前为0

#### 提交mr作业时可以运行但是有错误信息

**异常信息：**   
**ERROR hdfs.KeyProviderCache: Could not find uri with key [dfs.encryption.key.provider.uri] to create a keyProvider !!**

**原因：**   
不明，不影响作业的执行，据说是一个BUG

#### 运行任何hadoop命令都会失败

**异常信息：**   
WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException: **GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]**

ls: Failed on local exception: java.io.IOException: javax.security.sasl.SaslException: **GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]; Host Details : local host is: "hadoop-10-0-8-124/10.0.8.124"; destination host is: "10.0.8.124":8020;**

**原因：**   
之前的Kerberos数据库中有手动创建的principal的账号和CM自动生成的相冲突导致keytab和数据库中的principal不匹配   

**解决方法：**

尝试重新合成hdfs的keytab之后可以使用hadoop命令   

重新Kerberos创建数据库，并统一通过CM来自动生成各个节点的principal   
或者统一使用Kerberos命令手动创建各个节点的principal

如果还是没有解决问题，尝试逐项检查一下配置：

> * 检查操作时的身份，例如是否是用hdfs身份操作的；
> * 检查是否已经获得了credentials：kinit hdfs@XIAOHEI.INFO;
> * 尝试删除credentials并重新获取：destroy => kinit
> * tickets是否是renewable，检查 kdc.conf 的配置；
> * 检查是否安装了JCE Policy File，这可以通过Cloudera的Kerberos Inspector来检查；

#### hdfs用户被禁止运行 YARN container

**异常信息：**   
INFO mapreduce.Job: Job job_1442722429197_0001 failed with state FAILED due to: Application application_1442722429197_0001 failed 2 times due to AM Container for appattempt_1442722429197_0001_000002 exited with exitCode: -1000 due to: Application application_1442722429197_0001 initialization failed (exitCode=255) with output: **Requested user hdfs is banned**

**原因：**   
yarn的设置中将hdfs用户禁用了

**解决方法：**   
修改Clouder关于这个该项的设置，Yarn->配置->banned.users 将hdfs用户移除

#### CM界面HDFS、HBase等服务的Canary检查不通过

**异常信息：**

- HDFS的"HDFS Canary 运行状况检查"
- HBase的"活动 Master 运行状况检查"
- Oozie的"Web 度量收集"
- Hive的"Hive Metastore Canary 运行状况检查"

以上在CM中出现警报

**原因：**

mgmt没有得到Kerberos的认证信息

**解决方案：**

重启mgmt服务获得keytab

**异常信息：**

重启之后zk出现ZooKeeper 服务 canary 因未知原因失败

**解决方案：**

- mgmt可能清空了ticket的缓存，需要重新获取ticket
- kinit -R
- 无法执行的话先获得hdfs@XIAOHEI.INFO的ticket再次执行即可

其他常见问题，参考[Troubleshooting Authentication Issues](http://www.cloudera.com/documentation/enterprise/5-3-x/topics/cm_sg_sec_troubleshooting.html)

#### YARN job运行时无法创建缓存目录

**异常信息：**   
main : user is hdfs
main : requested yarn user is hdfs
Can't create directory /data/data/yarn/nm/usercache/hdfs/appcache/application_1442724165689_0005 - Permission denied

**原因：**   
该缓存目录在集群进入Kerberos状态前就已经存在了。例如当我们还没为集群Kerberos支持的时候，就用该用户跑过YARN应用

**解决方法：**   
在每一个NodeManager节点上删除该用户的缓存目录，对于用户hdfs，是/data/data/yarn/nm/usercache/hdfs

#### 个别节点无法通过Kerberos验证

**异常信息：**   
在子节点上使用hdfs principal登录获得ticket之后访问hdfs出现异常：
WARN security.UserGroupInformation: PriviledgedActionException as:hdfs (auth:KERBEROS) cause:javax.security.sasl.SaslException: **GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]**

**原因：**   
可能是该节点的配置有问题

**解决方法：**   
检查集群每个节点的Kerberos配置
Cloudera Manager => 管理 => Kerberos => 安全性检查器 => (等待检测结果···) => 显示检查结果   
根据检查的结果采取不同的措施

## LDAP安装

### 配置

hadoop-10-0-8-124作为LDAP服务器

```shell
yum install db4 db4-utils db4-devel cyrus-sasl* krb5-server-ldap -y
yum install openldap openldap-servers openldap-clients openldap-devel compat-openldap -y
```

更新配置库：

```shell
rm -rf /var/lib/ldap/*
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown -R ldap.ldap /var/lib/ldap
```

配置数据存储位置即目录：/etc/openldap/slapd.d   
尽量不要直接编辑改目录的文件，建议使用ldapadd,ldapdelete,ldapmodify等命令来修改

默认配置文件保存在 /etc/openldap/slapd.d，将其备份：

```shell
cp -rf /etc/openldap/slapd.d /etc/openldap/slapd.d.bak
```

添加一些基本配置，并引入 kerberos 和 openldap 的 schema：

```shell
cp /usr/share/doc/krb5-server-ldap-1.10.3/kerberos.schema /etc/openldap/schema/
touch /etc/openldap/slapd.conf
echo "include /etc/openldap/schema/corba.schema include /etc/openldap/schema/core.schema include /etc/openldap/schema/cosine.schema include /etc/openldap/schema/duaconf.schema include /etc/openldap/schema/dyngroup.schema include /etc/openldap/schema/inetorgperson.schema include /etc/openldap/schema/java.schema include /etc/openldap/schema/misc.schema include /etc/openldap/schema/nis.schema include /etc/openldap/schema/openldap.schema include /etc/openldap/schema/ppolicy.schema include /etc/openldap/schema/collective.schema include /etc/openldap/schema/kerberos.schema" > /etc/openldap/slapd.conf
echo -e "pidfile /var/run/openldap/slapd.pid\nargsfile /var/run/openldap/slapd.args" >> /etc/openldap/slapd.conf
# 更新slapd.d
slaptest -f /etc/openldap/slapd.conf -F /etc/openldap/slapd.d
chown -R ldap:ldap /etc/openldap/slapd.d && chmod -R 700 /etc/openldap/slapd.d
```

### 启动服务

```shell
chkconfig --add slapd chkconfig --level 345 slapd on /etc/init.d/slapd start
```

查看状态，验证服务端口：

```shell
ps aux | grep slapd | grep -v grep
netstat -tunlp | grep :389
```

### LDAP集成Kerberos

为了使Kerberos能够绑定到OpenLDAP服务器，需要创建一个管理员用户和一个principal，并生成keytab文件   
设置该文件的权限为LDAP服务运行用户可读（一般为ldap）：

```shell
kadmin.local -q "addprinc ldapadmin@XIAOHEI.INFO"
kadmin.local -q "addprinc -randkey ldap/hadoop-10-0-8-124@XIAOHEI.INFO"
kadmin.local -q "ktadd -k /etc/openldap/ldap.keytab ldap/hadoop-10-0-8-124@XIAOHEI.INFO"

chown ldap:ldap /etc/openldap/ldap.keytab && chmod 640 /etc/openldap/ldap.keytab
```

使用ldapadmin用户测试：

```shell
kinit ldapadmin
```

确保LDAP启动时使用上一步中创建的keytab文件，在/etc/sysconfig/ldap增加KRB5_KTNAME配置：

```shell
export KRB5_KTNAME=/etc/openldap/ldap.keytab
```

重启slapd服务

### 创建LDAP的数据库

进入到/etc/openldap/slapd.d目录，查看/etc/openldap/slapd.d/cn\=config/olcDatabase={2}bdb.ldif   
可以看到一些默认的配置，例如：

```
olcRootDN: cn=Manager,dc=my-domain,dc=com 
olcRootPW: secret 
olcSuffix: dc=my-domain,dc=com
```

建立modify.ldif文件，内容如下：

```
dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=xiaohei,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
replace: olcRootDN
# 只有该用户有写权限
olcRootDN: uid=ldapadmin,ou=people,dc=xiaohei,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
add: olcRootPW
# root用户的密码
olcRootPW: root

dn: cn=config
changetype: modify
add: olcAuthzRegexp
olcAuthzRegexp: uid=([^,]*),cn=GSSAPI,cn=auth uid=$1,ou=people,dc=xiaohei,dc=com

dn: olcDatabase={2}bdb,cn=config
changetype: modify
add: olcAccess
# 所有人可读
olcAccess: {0}to dn.base="" by * read
# 只有ldapadmin用户可以写
olcAccess: {1}to * by dn="uid=ldapadmin,ou=people,dc=xiaohei,dc=com" write by * read
```

使用下面命令导入更新上面的这三个配置：

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f modify.ldif
```

**更新配置过程中出现错误：additional info: modify/add: olcRootPW: no equality matching rule，修改modify.ldif中对应选项的add为replace即可**

这时候数据库没有数据，你可以手动编写ldif文件来导入一些用户和组   
或者使用migrationtools工具来生成ldif模板   
创建setup.ldif文件如下：

```
dn: dc=xiaohei,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: xiaohei com
dc: xiaohei

dn: ou=people,dc=xiaohei,dc=com
objectclass: organizationalUnit
ou: people
description: Users

dn: ou=group,dc=xiaohei,dc=com
objectClass: organizationalUnit
ou: group

dn: uid=ldapadmin,ou=people,dc=xiaohei,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: LDAP admin account
uid: ldapadmin
sn: ldapadmin
uidNumber: 1001
gidNumber: 100
homeDirectory: /home/ldap
loginShell: /bin/bash
```

使用以下命令进行导入：

```shell
# -w为之前指定的密码
ldapadd -x -D "uid=ldapadmin,ou=people,dc=xiaohei,dc=com" -w root -f setup.ldif
```

### 导入linux系统用户

安装migrationtools工具：

```shell
yum install migrationtools -y
```

利用迁移工具生成模板，先修改默认的配置：

```shell
vim /usr/share/migrationtools/migrate_common.ph

#71行默认的dns域名
DEFAULT_MAIL_DOMAIN = "XIAOHEI.INFO"; 
#74行默认的base 
DEFAULT_BASE = "dc=xiaohei,dc=com";
```

生成模板文件：

```shell
/usr/share/migrationtools/migrate_base.pl > /opt/base.ldif
```

可以修改该文件，然后执行导入命令：

```shell
ldapadd -x -D "uid=ldapadmin,ou=people,dc=xiaohei,dc=com" -w root -f /opt/base.ldif
```

将当前节点上的用户导入到 ldap 中，可以有选择的导入指定的用户：

```shell
# 先添加用户 
useradd test
# 查找系统上的 test用户 
grep -E "test" /etc/passwd >/opt/passwd.txt 
/usr/share/migrationtools/migrate_passwd.pl /opt/passwd.txt /opt/passwd.ldif 
ldapadd -x -D "uid=ldapadmin,ou=people,dc=xiaohei,dc=com" -w root -f /opt/passwd.ldif
```

将用户组导入到 ldap 中：

```shell
# 生成用户组的 ldif 文件，然后导入到 ldap 
grep -E "test" /etc/group >/opt/group.txt 
/usr/share/migrationtools/migrate_group.pl /opt/group.txt /opt/group.ldif 
ldapadd -x -D "uid=ldapadmin,ou=people,dc=xiaohei,dc=com" -w root -f /opt/group.ldif
```

### LDAP客户端配置

在其他子节点上安装：

```shell
./tool.sh -r "yum install openldap-clients -y"
```

修改/etc/openldap/ldap.conf以下两个配置

```
BASE    dc=xiaohei,dc=com
URI    ldap://hadoop-10-0-8-124
```

然后，运行下面命令测试：

```shell
# 会报异常
ldapsearch -b 'dc=xiaohei,dc=com'
# 重新获取 ticket
kinit ldapadmin/admin
# 没有报错
ldapsearch -b 'dc=xiaohei,dc=com'

```

### Hive集成LDAP

**注意，Cloudera文档中描述，Hive的LDAP是Kerberos的替代，不能同时启用，如果同时启用将会出现以下异常：**   

- Hue界面中无法连接Hive，错误提示：Bad status: 3 (Unsupported mechanism type GSSAPI)   
- beeline中使用Kerberos认证出现同上的错误

有以上情况时，将LDAP的配置移除即可解决   

Kerberos和LDAP的区别：   

- 如果使用 Kerberos 身份验证，Thrift 客户端和 HiveServer2 以及 HiveServer2 和安全 HDFS 之间都支持身份验证   
- 如果使用 LDAP 身份验证，仅在 Thrift 客户端和 HiveServer2 之间支持身份验证

在hive-site.xml中加入以下配置：

```xml
<property>
  <name>hive.server2.authentication</name>
  <value>LDAP</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.url</name>
  <value>ldap://hadoop-10-0-8-124</value>
</property>
<property>
  <name>hive.server2.authentication.ldap.baseDN</name>
  <value>ou=people,dc=xiaohei,dc=com</value>
</property>
```

重启Hive和Yarn服务

进入beeline测试：

```shell
beeline --verbose=true
beeline> !connect jdbc:hive2://hadoop-10-0-8-124:10000/default
# 输入账号密码，即hive/hive
```

### 配置Impala集成LDAP

**Impala中可以同时使用Kerberos+LDAP的认证方式，所以在已经启用Kerberos的情况下启用LDAP可以正常工作**

在Impala配置页中：   
> * 启用 LDAP 身份验证选项设置为true
> * 启用 LDAP TLS 选项设置为true
> * Impala 命令行参数高级配置代码段（安全阀）中添加-ldap_baseDN=ou=people,dc=xiaohei,dc=com

或者手动添加/etc/default/impala中的IMPALA_SERVER_ARGS参数：

```
-enable_ldap_auth=true \
-ldap_uri=ldaps://hadoop-10-0-8-124 \
-ldap_baseDN=ou=people,dc=xiaohei,dc=com
```

重启Impala服务

使用impala-shell测试LDAP账号：

```shell
impala-shell -l -u test
# 输入test账号密码
```

使用beeline测试LDAP账号：

```shell
beeline -u "jdbc:hive2://hadoop-10-0-8-124:21050/default;" -n test -p test
```

### Hue集成LDAP

在Hue中配置LDAP可以让Hue直接使用LDAP所管理的账号而不必在Hue中重新管理

在Hue的配置页面中修改   

- 身份验证后端/backend为desktop.auth.backend.LdapBackend   
- 登录时创建 LDAP 用户/create_users_on_login 设置为True   
- 使用搜索绑定身份验证/search_bind_authentication 设置为False

有两种方法可以通过 Hue 使用目录服务进行身份验证：   

- 搜索绑定   
- 直接绑定

这里将使用直接绑定的方式，关于搜索绑定请参考Cloudera的文档说明   
以上的配置将在登录Hue的时候自动创建默认情况下 Hue 中不存在的用户

直接绑定将用于身份验证的直接绑定机制将使用登录时提供的用户名和密码绑定到 LDAP 服务器   
使用直接绑定要设置一下两个配置的其中之一：   

- nt_domain：仅限与 Active Directory 一起使用   
- ldap_username_pattern：为在身份验证时最终将发送到目录服务的 DN 提供了一个模板。<username> 参数将被替换为登录时提供的用户名。
默认值："uid=<username>,ou=People,dc=mycompany,dc=com"

在nt_domain未指定的情况下将使用ldap_username_pattern配置值进行LDAP账号检索   
在Hue的配置页面中设置LDAP 用户名模式/ldap_username_pattern 为uid=<username>,ou=People,dc=xiaohei,dc=com   
**注意：ldap_username_pattern的格式要正确，不应该为 cn=<username>,dc=example,dc=com 的形式，否则会造成使用LDAP账号登录Hue的时候用户名或者密码错误的信息**

配置完成之后重启Hue服务即可完成，之后可以通过管理员账号在Hue的用户管理中导入/同步LDAP账号和组   
但是出现了一下的异常：   
**使用默认的配置无法导入/同步LDAP账号和组，服务器500错误**   
待解决

### LDAP相关知识

有关LDAP的用户简称（cn，dn），命令行使用可以参考：   
http://www.zhukun.net/archives/7980

## Sentry安装

可以直接在CM服务管理上添加Sentry服务，设置sentry-store的配置即可，参考：   
http://www.cloudera.com/content/www/zh-CN/documentation/enterprise/5-3-x/topics/sg_sentry_service_install.html

### Hive集成Sentry

修改HDFS上hive.metastore.warehouse.dir路径的权限：

```shell
# 需要先使用Kerberos账号通过认证
hadoop fs -chmod -R 771 /user/hive/warehouse
hadoop fs -chown -R hive:hive /user/hive/warehouse
```

在CM上转入Hive的配置页面：   
> * 取消勾选 HiveServer2 启用模拟属性   
> * 设置Sentry 服务属性为 Sentry

在hive-site.xml中加入以下配置：

```xml
<property>
   <name>hive.security.authorization.task.factory</name>
   <value>org.apache.sentry.binding.hive.SentryHiveAuthorizationTaskFactoryImpl</value>
</property>
<property>
   <name>hive.server2.session.hook</name>
   <value>org.apache.sentry.binding.hive.HiveAuthzBindingSessionHook</value>
</property>
<property>
   <name>hive.security.authorization.task.factory</name>
   <value>org.apache.sentry.binding.hive.SentryHiveAuthorizationTaskFactoryImpl</value>
</property>
<property>
    <name>hive.metastore.client.impl</name>
    <value>org.apache.sentry.binding.metastore.SentryHiveMetaStoreClient</value>
</property>
<property>  
    <name>hive.metastore.pre.event.listeners</name>  
    <value>org.apache.sentry.binding.metastore.MetastoreAuthzBinding</value>  
</property>
<property>
    <name>hive.metastore.event.listeners</name>  
    <value>org.apache.sentry.binding.metastore.SentryMetastorePostEventListener</value>  
</property>
```

这里需要注意的是，cloudera官方文档中还需要添加hive.sentry.conf.url属性来指定hive使用的sentry-site.xml文件的路径   
实际操作中并不需要，因为CM会使用自动生成的sentry-site.xml文件，添加配置指定的话也可以，但是要确保：

> 1.sentry-site.xml文件内容正确   
> 2.文件路径可以被CM搜索到   

否则CM的检测选项将会警报
转入Yarn配置页面，确保允许的系统用户属性包含 hive 用户

在Hive的配置页面搜索sentry-site.xml添加以下配置：

```xml
<property>
  <name>sentry.hive.testing.mode</name>
  <value>true</value>
</property>
<property>
  <name>sentry.service.client.server.rpc-connection-timeout</name>
  <value>200000</value>
</property>
```

搜索绕过 Sentry 授权用户/sentry.metastore.service.users选项修改为：hive,impala,hue,hdfs   
搜索用于 Sentry 授权的服务器名称/hive.sentry.server选项修改为：server1   
搜索Sentry 用户至组映射类/hive.sentry.provider确定该值为默认值

以上的三个选项将会被作用到Hive使用的sentry-site.xml文件中

因为之前通过CM启用Kerberos的时候默认Sentry已经采用Kerberos模式了，所以以下的配置不需要再次添加

```xml
<property>
    <name>sentry.service.security.mode</name>
    <value>kerberos</value>
</property>
<property>
  <name>sentry.service.server.principal</name>
    <value>sentry/_HOST@XIAOHEI.INFO</value>
</property>
<property>
    <name>sentry.service.server.keytab</name>
    <value>/etc/sentry/conf/sentry.keytab</value>
</property>
<property>
  <name>sentry.service.client.server.rpc-port</name>
  <value>8038</value>
</property>
<property>
  <name>sentry.service.client.server.rpc-address</name>
  <value>hadoop-10-0-8-124</value>
</property>
```

重启Hive服务

### 使用Sentry的角色授权机制

此时通过Kerberos的hive账户可以在集群上以hive shell的形式进入hive中创建数据库和表

通过beeline连接Hive Server2来创建角色和组：

```shell
# 通过Kerberos认证进入beeline，出现了未知的错误，状态为close，输出信息1，待查看日志解决
beeline -u "jdbc:hive2://hadoop-10-0-8-124:10001/default;principal=hive/@XIAOHEI.INFO"
# 通过LDAP的hive账号进入beeline，hive默认为admin权限
beeline -u "jdbc:hive2://hadoop-10-0-8-124:10000/default;" -n hive -p hive
```

使用LDAP账号进入beeline测试的时候   
如果出现错误：**Error: Could not open connection to jdbc:hive2://hadoop-10-0-8-124:10000/default;: Peer indicated failure: PLAIN auth failed: Error validating LDAP user (state=08S01,code=0)**

可能是test用户密码错误，修改即可：

```
# 注意，这里-D 后面的参数管理员DN是在/etc/openldap/slapd.conf中指定的rootdn
# 不确定格式和值的话使用 cat /etc/openldap/slapd.conf | grep rootdn 查看一下
ldappasswd -H ldap://hadoop-10-0-8-124 -x -D "cn=root,dc=xiaohei,dc=com" -W "uid=test,ou=People,dc=xiaohei,dc=com" -S
```

创建角色和授权

```
create role admin_role;
GRANT ALL ON SERVER server1 TO ROLE admin_role;
GRANT ROLE admin_role TO GROUP admin;
GRANT ROLE admin_role TO GROUP hive;

create role test_role;
GRANT ALL ON DATABASE filtered TO ROLE test_role;
GRANT ROLE test_role TO GROUP test;
```

- 设置/创建角色：admin_role和test_role，create role role_name
- 设置角色权限：
    - grant all on server server1 to role admin_role：将服务器server1上的所有权限给admin_role角色
    - grant all on database filtered to role test_role：将数据库filtered的所有权限给test_role角色
- 设置/创建用户组：使用LDAP即Linux用户和用户组的关系
- 角色到用户组的映射：
    - grant role admin_role to group admin/hive：admin和hive用户组将拥有角色admin_role的权限
    - grant role test_role to group test：test用户组将拥有角色test_role的权限

之后使用test账号（使用Kerberos或者LDAP二者之一，看使用Hive使用的是哪种验证）进入beeline进行操作时，该账号只能操作授权部分的内容

### Impala集成Sentry

配置Impala之前确保Hive已经集成了Sentry服务，因为Impala的授权是继承自Hive的，在Hive中所做的授权动作将会自动影响到Imapala

在Impala配置页面中设置 Sentry 服务属性为 Sentry   

为Impala的命令启动添加配置：-server_name=server1

重启Impala服务

至此，可以直接通过认证的LDAP账号登录Hue平台操作对应的Hive和Impala表，账号的权限信息和命令行中使用beeline测试效果一致

### 添加访问集群的新用户

通过Kerberos进行集群之间的相互认证，集群上命令行身份认证   
通过LDAP进行客户端的账号密码认证
通过Sentry对不同角色进行权限控制

环境集成完毕

添加能够访问集群的新用户步骤：   
- 创建对应的linux用户，**并设置正确的组**   
- 添加该用户Kerberos的principal   
- 将该用户导入LDAP，并设置密码   
- 设置Sentry的授权

用户组的设置关系到Sentry的授权，所以确保正确

操作LDAP的时候如果出现错误：
**ldap_bind: Invalid credentials (49)**

原因可能是：

> 1.管理员DN或者用户DN错误   
> 2.管理员密码错误

解决方式参考：
http://www.zhukun.net/archives/7969

## shell脚本

```shell
#!/bin/bash

getopts :r:g opt
case $opt in
r)
master=`cat master`
slaves=(`cat slaves`)

for slave in ${slaves[*]}
do
  echo "========$slave==========="
  if ssh $slave $OPTARG
  then
    echo "===Job done on $slave!==="
  fi
  echo
done
;;
g)
users=(`cat hadoop-users`)
for user in ${users[*]}
do
  if cat /etc/passwd | grep $user
  then
    if ls /home/$user > /dev/null 2> /dev/null
    then
      if echo -e "$user\n$user" | kadmin.local -q "addprinc $user" > /dev/null 2> /dev/null
      then
        echo "$user's principal added succeed!"
      else
        echo "$user'principal maybe already exists!"
      fi
    else
      echo "$user doesn't have a linux home dir!"
    fi
  else
    echo "$user doesn't exists in linux user list!"
  fi
done
;;
*)
echo "Useage:"
echo "-r cmd:the cmd run on remote hosts"
echo "-g    :generate kerberos principal for hadoop users"
;;
esac
```


作者：[@小黑](http://www.xiaohei.info)