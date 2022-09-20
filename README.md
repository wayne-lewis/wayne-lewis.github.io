# 下载安装包
打开网址`https://www.elastic.co/cn/downloads/elasticsearch`，选择服务器下载对应版本，CentOS可以使用压缩包`.tar.gz`安装，也可以使用`rpm`安装（本地或远程），二者文件存放路径不同（包括配置文件，以及运行文件）。

# 用户权限检查及创建
> 如果需要Elasticsearch的机器上已经安装过老版本，用户和组不是`elastic`，需要新建用户和组`elastic`，否则可能有一些地方会报错，具体原文没有详细调查

* 查看是否有用户`elastic`
```shell
[root@centos-tc ~]# cut -d : -f 1 /etc/passwd | grep elastic
elastic
```

如果用户不存在，那么需要为Elasticsearch创建单独的用户
```shell
[root@centos-tc ~]# useradd -m elastic
```

* 查看是否有组`elastic`
```shell
[root@centos-tc ~]# cut -d : -f 1 /etc/group | grep elastic      
elastic
```

如果组不存在，那么需要为Elasticsearch创建单独的组（从安全性考虑，不建议使用root组）
```shell
[root@centos-tc ~]# groupadd elastic
```

# 安装
## rpm方式安装
```shell
[root@bigdata-jchj01 server]# rpm -ivh elasticsearch-7.17.5-x86_64.rpm 
warning: elasticsearch-7.17.5-x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID d88e42b4: NOKEY
Preparing...                ########################################### [100%]
Creating elasticsearch group... OK
Creating elasticsearch user... OK
   1:elasticsearch          ########################################### [100%]
### NOT starting on installation, please execute the following statements to configure elasticsearch service to start automatically using chkconfig
 sudo chkconfig --add elasticsearch
### You can start elasticsearch service by executing
 sudo service elasticsearch start
warning: usage of JAVA_HOME is deprecated, use ES_JAVA_HOME
Future versions of Elasticsearch will require Java 11; your Java version from [/server/jdk1.8.0_161/jre] does not meet this requirement. Consider switching to a distribution of Elasticsearch with a bundled JDK. If you are already using a distribution with a bundled JDK, ensure the JAVA_HOME environment variable is not set.
Created elasticsearch keystore in /etc/elasticsearch/elasticsearch.keystore
[root@bigdata-jchj01 server]# 
```
> 安装过程中提示的`ES_JAVA_HOME`的问题，在下面`启动`章节进行修改

安装后的配置文件目录：`/etc/elasticsearch/`

安装后的执行文件目录：`/usr/share/elasticsearch/`

## rpm卸载
* 查找安装的Elasticsearch信息
```shell
[root@bigdata-jchj01 elasticsearch]# rpm -qa | grep elastic
elasticsearch-7.17.5-1.x86_64
[root@bigdata-jchj01 elasticsearch]# 
```

* 删除指定名字的文件
```shell
[root@bigdata-jchj01 server]# rpm -e elasticsearch-7.17.5-1.x86_64
Stopping elasticsearch service... OK
```

## tar.gz方式安装
解压文件即可
```shell
[root@bigdata-jchj01 server]# tar -zxvf elasticsearch-7.17.5-linux-x86_64.tar.gz 
```

创建证书
```shell
[root@bigdata-jchj01 elasticsearch-7.17.5]# ./bin/elasticsearch-certutil cert -out ./config/elastic-certificates.p12 -pass ""
```

解压后目录说明：
```
bin： 下面存放着Es启动文件
config： 配置目录
data： 数据目录（默认，建议修改）
jdk、lib： Java运行环境以及依赖包
logs： 日志目录（默认，建议修改）
modules、plugins： 模块及插件目录
```
## tar.gz卸载
直接删除安装目录、日志文件目录、数据文件目录就完成了删除。**删除日志以及数据文件要谨慎！**

# 修改文件所有者信息
```shell
[root@bigdata-jchj01 share]#chown -R elastic:elastic /etc/elasticsearch
[root@bigdata-jchj01 share]#chown -R elastic:elastic /usr/share/elasticsearch
[root@bigdata-jchj01 share]#chown -R elastic:elastic /etc/sysconfig/elasticsearch
[root@bigdata-jchj01 share]#chown -R elastic:elastic /var/log/elasticsearch
[root@bigdata-jchj01 share]#chown -R elastic:elastic /var/lib/elasticsearch

参数说明：
elastic:elastic 用户名:组名
/etc/elasticsearch等: 目录名，配置文件中设置的路径需要手动创建
```

# 配置文件修改
`elasticsearch.yml`主要修改的配置如下：
```yaml
#JVM内存配置，默认4g，按现场服务器配置可以调整
-Xms4g
-Xmx4g
# 集群名称
cluster.name: sqyyptcluster
# 当前节点名称，同一个集群内不可以重复，单节点随便配置一个就可以
node.name: node-1
# 数据存放目录（按实际需求配置）
path.data: /var/lib/elasticsearch
# 日志存放目录（按实际需求配置）
path.logs: /var/log/elasticsearch
# 避免启动报错：ERROR: bootstrap checks failed system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk...
bootstrap.system_call_filter: false
# 绑定IP地址
network.host: 10.23.10.160
# 端口
http.port: 9200
# 节点IP配置，集群需要配置所有节点IP地址，和上面${network.host}配置匹配
discovery.seed_hosts: ["10.23.10.160"]
# 初始化一个新的集群时需要此配置来选举master
cluster.initial_master_nodes: ["node-1"]
# 配置X-Pack
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

`jvm.options`主要修改的配置如下：
> rpm安装方式会自动配置好，一般情况下不需要调整；tar.gz方式安装需要关注一下。

```
# 以下配置主要关注目录即可
-XX:HeapDumpPath=/var/lib/elasticsearch
-XX:ErrorFile=/var/log/elasticsearch/hs_err_pid%p.log
8:-Xloggc:/var/log/elasticsearch/gc.log
9-:-Xlog:gc*,gc+age=trace,safepoint:file=/var/log/elasticsearch/gc.log:utctime,pid,tags:filecount=32,filesize=64m
```

# Elasticsearch启动
> 由于Elasticsearch启动需要使用jvm，建议启动前先修改`elasticsearch-env`文件，在文件的最上面增加一行JVM配置

## 启动
```properties
# 具体目录按照实际修改
ES_JAVA_HOME="/usr/share/elasticsearch/jdk"
```

```shell
# 切换用户
[root@dzswjnr-fbtest elasticsearch]# su - elastic
# 进入运行文件根目录（以rpm安装方式为例）
[elastic@dzswjnr-fbtest ~]$ cd /usr/share/elasticsearch/
# 启动
[elastic@dzswjnr-fbtest elasticsearch]$ ./bin/elasticsearch -d
[1] 3574
[elastic@dzswjnr-fbtest elasticsearch]$ 
```

启动后查看日志，看到下面这行日志说明Elasticsearch已经启动成功
```
[2022-09-19T16:52:04,048][INFO ][o.e.c.r.a.AllocationService] [node-160] Cluster health status changed from [RED] to [GREEN] (reason: [shards started [[.ds-.logs-deprecation.elasticsearch-default-2022.08.03-000001][0]]]).
```

## 设置密码
> 系统第一次启动成功后，并不能直接访问（~~*未设置密码访问会报错*~~），因为设置了xpack，所以需要再第一次访问前设置密码。Elasticsearch提供了两种初始化密码的方式。
### 初始化密码
* 由Elasticsearch自动生成密码

```shell
[elastic@bigdata-jchj01 elasticsearch-7.17.5]$ ./bin/elasticsearch-setup-passwords auto
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
The passwords will be randomly generated and printed to the console.
Please confirm that you would like to continue [y/N]Y


Changed password for user apm_system
PASSWORD apm_system = o3mAQzZQbxzQER0Yo1Nl

Changed password for user kibana_system
PASSWORD kibana_system = KsOdiM53axKyaCpS2zQZ

Changed password for user kibana
PASSWORD kibana = KsOdiM53axKyaCpS2zQZ

Changed password for user logstash_system
PASSWORD logstash_system = V91VN9NUhOV8fQDu09Eb

Changed password for user beats_system
PASSWORD beats_system = mHQba6bt3WLY89Tp1RII

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = PZVjWeXKSHD4YE5APshp

Changed password for user elastic
PASSWORD elastic = 1Wir4pgwqgSRVrkK0Clv
```

* 手动为每个用户设置密码（交互式）

```shell
[elastic@bigdata-jchj01 elasticsearch-7.17.5]$ ./bin/elasticsearch-setup-passwords interactive
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]Y


Enter password for [elastic]: 
Reenter password for [elastic]: 
Enter password for [apm_system]: 
Reenter password for [apm_system]: 
Enter password for [kibana_system]: 
Reenter password for [kibana_system]: 
Enter password for [logstash_system]: 
Reenter password for [logstash_system]: 
Enter password for [beats_system]: 
Reenter password for [beats_system]: 
Enter password for [remote_monitoring_user]: 
Reenter password for [remote_monitoring_user]: 
12Changed password for user [apm_system]
3Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]
[elastic@bigdata-jchj01 elasticsearch-7.17.5]$ 
```

### 修改密码
通过REST API请求修改用户密码，前提是已知操作用户密码
```shell
$ curl -XPOST -u elastic 'http://10.23.13.207:9203/_security/user/elastic/_password' -H 'Content-Type: application/json' -d'{"password" : "elastic123456"}'
Enter host password for user 'elastic':
{}
```

* 其中`elastic123456`为**修改后密码**
* 控制台需要键入**修改前密码**进行验证`Enter host password for user 'elastic':`
* 输入**修改前密码**：`123456`

### 重置所有密码
> 重置所有用户的密码，就是要把保存密码的索引删除，之后重新初始化就可以了

1. 停止Elasticsearch
```shell
kill -15 {PID}
```
2. xpack取消
> 注释下面的配置或者把true改成false
```properties
http.cors.enabled: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

3. 启动，以无密码形式登录
```shell
./bin/elasticsearch -d 
```

4. 删除.security
```shell
# 首先查询索引名称，保存密码的索引的名称是.security-*
curl -XGET -u elastic:elastic123456 'http://10.23.13.207:9203/_cat/indices?v'
# 删除对应的索引
curl -XDELETE -u elastic:elastic123456 'http://10.23.13.207:9203/.security-7'
```

5. 停止Elasticsearch
```shell
kill -15 {PID}
```
6. xpack开启
> 打开下面的配置注释或者把false改成true
```properties
http.cors.enabled: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

# Elasticsearch关闭
停止服务可以使用`kill`命令

# 简单命令
> 为了演示说明，请参考实际情况修改命令
> * Elasticsearch地址: http://10.23.13.207:9203
> * 用户名: elastic
> * 密码: elastic123456

## 查看集群状态
```shell
$ curl -XGET -u elastic:elastic123456 'http://10.23.13.207:9203'
{
  "name" : "node-1",
  "cluster_name" : "sqyyptcluster",
  "cluster_uuid" : "4JnEHqugTBKXbAvbzCcB-w",
  "version" : {
    "number" : "7.17.5",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "8d61b4f7ddf931f219e3745f295ed2bbc50c8e84",
    "build_date" : "2022-06-23T21:57:28.736740635Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

## 查看集群健康信息
```shell
$ curl -XGET -u elastic:elastic123456 'http://10.23.13.207:9203/_cat/health?v'
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1663639376 02:02:56  sqyyptcluster green           1         1      3   3    0    0        0             0                  -                100.0%
```

## 查看索引信息
```shell
$ curl -XGET -u elastic:elastic123456 'http://10.23.13.207:9203/_cat/indices?v'
health status index       uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .security-7 6XwIbwJMRT6h27SRPV-tnQ   1   0          7            1     38.8kb         38.8kb
```

## 删除索引
> ***危险动作，谨慎操作***

```shell
$ curl -XDELETE -u elastic:elastic123456 'http://10.23.13.207:9203/sqyypt'
{"acknowledged":true}
# sqyypt是索引名或者别名
```

## 查询数据
```shell
$ curl -XGET -u elastic:elastic123456 'http://10.23.13.207:9203/.security-7/_search?pretty'
{
  "took" : 8,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 7,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : ".security-7",
        "_type" : "_doc",
        "_id" : "reserved-user-apm_system",
        "_score" : 1.0,
        "_source" : {
          "password" : "$2a$10$BvzkhYUy/mRYGk3HaVQ66u/i0ABIUuGGEhRPep6sdQmQ2NmVUpSl2",
          "enabled" : true,
          "type" : "reserved-user"
        }
      },
      {
        "_index" : ".security-7",
        "_type" : "_doc",
        "_id" : "reserved-user-elastic",
        "_score" : 1.0,
        "_source" : {
          "password" : "$2a$10$t7FhjxsvZ0GKhIvLIKx.u.sUmjGn8/D2dDgYsCZiAL.kHtfc0CvLm",
          "enabled" : true,
          "type" : "reserved-user"
        }
      }
    ]
  }
}
```

# 第三方客户端
提供一个第三方客户端<ElasticView>，用途如下：
> * 图形化查看索引状态
> * 图形化删除索引
> * 简单查询索引数据

官方网址：[ElasticView](https://github.com/1340691923/ElasticView)

源码地址：[ElasticView](http://www.elastic-view.cn/)

# 特殊说明
启动后日志中有如下报错直接忽略
```
[2022-09-19T16:51:58,610][ERROR][o.e.i.g.GeoIpDownloader  ] [node-160] exception during geoip databases update
java.net.UnknownHostException: geoip.elastic.co
```

# 常见问题

## 授权问题`Failed to authenticate user 'elastic'`
```shell
[elastic@bigdata-jchj01 elasticsearch]$ ./bin/elasticsearch-setup-passwords interactive  
[2022-09-19T17:52:02,352][INFO ][o.e.x.s.a.RealmsAuthenticator] [node-1] Authentication of [elastic] was terminated by realm [reserved] - failed to authenticate user [elastic]

Failed to authenticate user 'elastic' against http://10.23.13.207:9201/_security/_authenticate?pretty
Possible causes include:
 * The password for the 'elastic' user has already been changed on this cluster
 * Your elasticsearch node is running against a different keystore
   This tool used the keystore at /etc/elasticsearch/elasticsearch.keystore


ERROR: Failed to verify bootstrap password
[elastic@bigdata-jchj01 elasticsearch]$ 
```
问题处理流程：
1. 停止Elasticsearch
```shell
kill -15 {PID}
```
2. xpack取消
> 注释下面的配置或者把true改成false
```properties
http.cors.enabled: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```

3. 启动，以无密码形式登录
```shell
./bin/elasticsearch -d 
```

4. 删除.security
```shell
# 首先查询索引名称，保存密码的索引的名称是.security-*
curl -XGET -u elastic:elastic123456 'http://10.23.13.207:9203/_cat/indices?v'
# 删除对应的索引
curl -XDELETE -u elastic:elastic123456 'http://10.23.13.207:9203/.security-7'
```

5. 停止Elasticsearch
```shell
kill -15 {PID}
```
6. xpack开启
> 打开下面的配置注释或者把false改成true
```properties
http.cors.enabled: true
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
```
7. 删除elasticsearch/config目录下elasticsearch.keystore、elastic-certificates.p12
```shell
rm -rf {elastic_home}/config/elasticsearch.keystore
rm -rf {elastic_home}/config/elastic-certificates.p12
```

8. 生成新的证书
```shell
./bin/elasticsearch-certutil cert -out {elastic_home}/config/elastic-certificates.p12 -pass ""
```
如果是集群，需要把config/elastic-certificates.p12拷贝到其他节点的config目录下
9. es节点依次启动
```shell
./bin/elasticsearch -d 
```

10. 设置密码,等待cluster health is currently RED.变为green在下一步
> 详情查看初始化密码章节

## 启动报错`ClassNotFoundException`
报错信息如下：
```
[2022-09-20T09:08:55,036][WARN ][o.e.b.Natives            ] [node-160] JNA not found. native methods will be disabled.
java.lang.ClassNotFoundException: com.sun.jna.Native
        at jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641) ~[?:?]
        at jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:188) ~[?:?]
        at java.lang.ClassLoader.loadClass(ClassLoader.java:521) ~[?:?]
        at java.lang.Class.forName0(Native Method) ~[?:?]
        at java.lang.Class.forName(Class.java:383) ~[?:?]
        at java.lang.Class.forName(Class.java:376) ~[?:?]
        at org.elasticsearch.bootstrap.Natives.<clinit>(Natives.java:34) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.bootstrap.Bootstrap.initializeNatives(Bootstrap.java:106) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:183) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:434) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:169) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:160) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:77) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:112) [elasticsearch-cli-7.17.5.jar:7.17.5]
        at org.elasticsearch.cli.Command.main(Command.java:77) [elasticsearch-cli-7.17.5.jar:7.17.5]
        at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:125) [elasticsearch-7.17.5.jar:7.17.5]
        at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:80) [elasticsearch-7.17.5.jar:7.17.5]
[2022-09-20T09:08:55,068][WARN ][o.e.b.Natives            ] [node-160] cannot check if running as root because JNA is not available
[2022-09-20T09:08:55,072][WARN ][o.e.b.Natives            ] [node-160] cannot register console handler because JNA is not available
```

发生这类问题的主要原因是文件在解压过程中发生问题，导致lib目录下的jar包错误导致，重新解压出正确的jar文件替换就可以解决了


# kibana安装
## 下载
打开网址`https://www.elastic.co/cn/downloads/kibana`，选择服务器下载对应版本，CentOS可以使用压缩包`.tar.gz`安装，也可以使用`rpm`安装（本地或远程），二者文件存放路径不同（包括配置文件，以及运行文件）。

## 安装
```shell
[root@dzswjnr-fbtest server]# rpm -ivh kibana-7.17.5-x86_64.rpm 
warning: kibana-7.17.5-x86_64.rpm: Header V4 RSA/SHA512 Signature, key ID d88e42b4: NOKEY
Preparing...                ########################################### [100%]
        package kibana-7.17.5-1.x86_64 is already installed
[root@dzswjnr-fbtest server]# 
```
## 配置文件修改
修改配置文件`/etc/kibana/kibana.yml`
```yaml
# 可以访问的地址
server.publicBaseUrl: "http://10.23.13.207:5601"
# 定义一个名字
server.name: "kibana207"
# elasticsearch用户名，一般使用kibana专有用户名
elasticsearch.username: "kibana_system"
# elasticsearch密码
elasticsearch.password: "mRrxLwv1XoaMWrAsnNv4"
# pid文件存放路径
pid.file: /run/kibana/kibana.pid
# 日志存放路径
logging.dest:  "/var/log/kibana/kibana.log"
# 该值设为 true 时，禁止所有日志输出
logging.silent: false
# 该值设为 true 时，禁止除错误信息除外的所有日志输出
logging.quiet: false
# 设置为true记录所有事件，包括系统使用情况信息和所有请求。在Elastic Cloud Enterprise上受支持。
logging.verbose: true
# 设置此值可以更改Kibana界面语言
i18n.locale: "zh-CN"
```

# 启动
> kibana是用node写的，启动的时间比较长，需要大改5分钟左右，要耐心等待
```
[root@bigdata-jchj01 kibana]# ./bin/kibana --allow-root &
```

# 关闭
查找关键字`--allow-root`的进程，停止服务可以使用`kill`命令

## 简单使用
* 访问`http://10.23.13.207:5601`
* 页面提示输入用户名、密码，这个用户名和密码是访问Elasticsearch的用户名密码
* 常用页面有：
* * Management -> 开发工具：执行命令
* * Management -> Stack Management -> 索引管理页面: 此页面能查看当前的索引信息，管理索引等操作
