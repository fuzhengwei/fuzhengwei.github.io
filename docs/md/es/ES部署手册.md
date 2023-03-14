#### linux部署
```java
1.获取安装包，linux可以直接通过wget命令下载，这样会快不少
wget https://mirrors.huaweicloud.com/elasticsearch/7.6.2/elasticsearch-7.6.2-linux-x86_64.tar.gz

2.修改jvm配置 es官网推荐xms和xms是机器内存的50%
vim ./config/jvm.options

3.新建usr来启动es，es使用root用户进行启动会报错
useradd elastic
chown -R elastic:elastic [elastic_path]
su elastic

4.启动es -d表示后台启动
bash ./bin/elasticsearch -d

5.验证
curl http://localhost:9200

```


#### 异常解决
以下配置如无特殊说明均配置在conf/elasticsearch.y m l中

default discovery settings are unsuitable for production use; at least one of \[discovery.seed\_hosts, discovery.seed\_providers, cluster.initial\_master\_nodes\] must be configured

```java
es7.6.2下单机启动配置2不需要设置，不知道是否和单机启动有关
discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]
```
外网无法访问

```java
network.host: 0.0.0.0 
http.port: 9200

vim  /etc/security/limits.conf 中
* soft nofile 65536
* hard nofile 65536

vim  /etc/sysctl.conf中
vm.max_map_count=655360

// 加载参数
sysctl -p
```


#### IK分词器安装
```Plain Text
https://github.com/medcl/elasticsearch-analysis-ik/releases
根据es版本去选择对应的ik分词器
将压缩包放到elasticsearch-5.2.0\plugins\下解压缩,然后重启es(liunux下需要在插件下新建文件夹（例如ik）然后解压缩)
```


#### es-head安装
```javascript
#获取安装包
wget https://codeload.github.com/mobz/elasticsearch-head/zip/refs/tags/v5.0.0 
#安装
npm install 
#后台发布
npm run start & ps aux | grep start
#es设置跨域
http.cors.enabled: true  // 允许跨域访问
http.cors.allow-origin: "*"  // 所有请求链接都能访问
```
