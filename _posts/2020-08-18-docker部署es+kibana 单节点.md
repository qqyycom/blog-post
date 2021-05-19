# docker部署es+kibana 单节点



1. 创建挂载的目录

```bash
mkdir -p /data/elasticsearch/config
mkdir -p /data/elasticsearch/data
echo "http.host: 0.0.0.0" >> /mydata/elasticsearch/config/elasticsearch.yml
```

2. 创建并启动容器

```bash
docker run --name elasticsearch -p 9200:9200 -p 9300:9300  -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms2g -Xmx2g" -v /data/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /data/elasticsearch/data:/usr/share/elasticsearch/data -v /data/elasticsearch/plugins:/usr/share/elasticsearch/plugins -v /data/elasticsearch/log:/usr/share/elasticsearch/logs -d elasticsearch:7.8.1

其中elasticsearch.yml是挂载的配置文件，data是挂载的数据，plugins是es的插件，如ik，而数据挂载需要权限，需要设置data文件的权限为可读可写,需要下边的指令。
chmod -R 777 要修改的路径

-e "discovery.type=single-node" 设置为单节点
特别注意：
-e ES_JAVA_OPTS="-Xms256m -Xmx256m" \ 测试环境下，设置ES的初始内存和最大内存，否则导致过大启动不了ES

```

问题:

Likely root cause: java.nio.file.AccessDeniedException  挂载文件目录的权限不够

chmod 777

3. 启动kibana

```bash
docker pull kibana:7.8.1

docker run --name kibana -e ELASTICSEARCH_HOSTS=http://自己的IP地址:9200 -p 5601:5601 -d kibana:7.8.1
//docker run --name kibana -e ELASTICSEARCH_URL=http://自己的IP地址:9200 -p 5601:5601 -d kibana:7.6.2

进入容器修改相应内容
server.port: 5601
server.host: 0.0.0.0
elasticsearch.hosts: [ "http://自己的IP地址:9200" ]
i18n.locale: "Zh-CN"

然后访问页面
http://自己的IP地址:5601/app/kibana

```

docker inspect --format '{{ .NetworkSettings.IPAddress }}' 容器id  查看容器真实ip