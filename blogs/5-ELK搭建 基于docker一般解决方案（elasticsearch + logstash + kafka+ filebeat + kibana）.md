#ELK搭建 基于docker一般解决方案（elasticsearch + logstash + kafka+ filebeat + kibana）
> 当我们有多台服务器上的服务日志需要管理的时候，这时候一台台ssh上去看，着实不方便，所以我们急需一个集中管理查看的方案，这时候可以考虑elk（使用docker纯粹为了环境搭建方便且后续管理着想）

## 一、首先让我们看一下主要的架构图
![ELK 架构图.png](https://raw.githubusercontent.com/bzylt/blogs/master/images/5/ELK%20架构.png)

- 可以看见整体的架构
1. 原始的日志，也就是我们的软件产生的日志，一般位于linux的磁盘里面
2. filebeat，一个轻量级的日志收集软件，可以直接把日志output到file、kafka、logstash、es等地方
3. kafka，不用多介绍，这是一个消息队列，主要作用在于 削峰填谷，还有消息的暂存（默认7天）
4. logstash， 日志收集/过滤格式化，其实这里主要起到一个连接kafka和es的作用，直接写es，量大容易出问题
5. kibana，这个node应用主要用于友好界面展示es的数据，方便我们分析和聚合
- 接下来我们先按图运行单节点版（后续再改多节点集群）的该架构

## 二、基础准备
1. 准备centos7.x服务器，云服务器、本机、vmware 虚拟机都可以
> 这里我是用的VMware的虚拟机（网上找的绿色版）
2. 待启动后，安装docker，然后启动docker服务
```shell
# 先创建目录
mkdir ~/elk
mkdir ~/log_producer
cd elk
sudo yum update
sudo yum install docker 
sudo systemctl start docker
```
## 三、使用docker启动elasticsearch 
1. 启动命令
```
docker run -d -p 0.0.0.0:9200:9200 -p 9300:9300 --name elasticsearch -e "discovery.type=single-node" elasticsearch:7.5.2
```
2. 浏览器访问[http://ip:9200/](http://ip:9200/) 看见类似如下即说明启动成功了
```json
{
  "name" : "b79d7d5cada9",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "8sgM5FFzTzCdQuvGu7fuig",
  "version" : {
    "number" : "7.5.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "8bec50e1e0ad29dad5653712cf3bb580cd1afcdf",
    "build_date" : "2020-01-15T12:11:52.313576Z",
    "build_snapshot" : false,
    "lucene_version" : "8.3.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
3. es的管理工具 [https://github.com/mobz/elasticsearch-head#running-with-built-in-server](https://github.com/mobz/elasticsearch-head#running-with-built-in-server) ， 这里直接用现成的[Chrome 插件]([https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm/](https://chrome.google.com/webstore/detail/elasticsearch-head/ffmkiejjmecolpfloofpjologoblkegm/)
)就可以，成功后可以看见如下图：
![es head.jpg](https://raw.githubusercontent.com/bzylt/blogs/master/images/5/es%20head.jpg)
- 然后输入es的地址即可


## 四、启动kibana
1. 启动命令
```shell
#一般内部使用的话，单个kibana就足够了
docker run -d -p 5601:5601 --link elasticsearch -e ELASTICSEARCH_URL=http://elasticsearch:9200 kibana:7.5.2
```
- 等待一会儿  访问[http://ip:5601](http://ip:5601/) ， 应该可以访问kibana的web界面了![kibana home.jpg](https://raw.githubusercontent.com/bzylt/blogs/master/images/5/kibana%20home.jpg)

## 五、然后启动kafka单节点
> 由于kafka是依赖zookeeper的，所以，我们先启动zookeeper，不过由于没官方的docker镜像，所以就选择维护比较频繁的社区镜像 wurstmeister/zookeeper、wurstmeister/kafka
1.  zookeeper启动命令
```shell
docker run --name zookeeper -p 2181:2181 wurstmeister/zookeeper
```
2. kafka启动命令，这里使用kafka 这个host，主要是为了外网（其他的机器）可以访问
```
docker run -p 9092:9092 --name kafka --link zookeeper --add-host kafka:127.0.0.1 -e \
KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 -e \
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 -e \
KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka
```
3. kafka数据查看
```
# 交互式进入kafka的docker容器
docker exec -it kafka bash
# 然后查看所有的topic
bin/kafka-topics.sh --list --zookeeper zookeeper:2181
# 然后 根据topicName 消费数据。就能看见当前topic的所有数据了
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic {topicName} --from-beginning
```

## 六、启动logstash
1. 修改 /elk/logstash.conf，内容为：
```yml
input {
#    beats {
#       port => "5044"
#	client_inactivity_timeout => 300
#    }
     kafka {
        # 这个kafka的host在启动docker的时候指定
        bootstrap_servers => "kafka:9092"
        #此属性会将当前topic、offset、group、partition等信息也带到message中
        decorate_events => true
        consumer_threads => 5
        auto_offset_reset => "earliest"
        # 日志的主题
        topics => ["log.filebeat.topic","processlog.filebeat.topic"]
     }
}
filter{
        mutate{
            add_field => {
                #将切割后的第一位数据放入自定义的“index”字段中
                "topic" => "%{[@metadata][kafka][topic]}"
            }
            #remove_field => ["message"]
        }
        json {
            #假设数据通过grok预处理，将result内容捕获
            source => "message"
        }
        mutate{
            remove_field => ["message"]
        }
}
output {
    elasticsearch {
        hosts => ["elasticsearch:9200"]
	ilm_enabled => "false"
	index => "cloudnode-%{[topic]}-%{+yyyy.MM.dd}"
    }
}
```

2. 启动命令, 这里修改了kafka这个host的地址，实际为kafka的宿主机ip，原因在于连接kafka后，回去获取metadata，其中的连接地址就是kafka:9092，所以这里弄好映射
```shell
docker run --rm -it -p 5044:5044 --add-host kafka:192.168.117.132 --name logstash \
--link elasticsearch \
-v ~/elk/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
 docker.elastic.co/logstash/logstash:7.5.2
```
## 七、filebeat，日志文件收集器
> 日志文件收集器，可以定义output push到kafka
1. 定义/elk/filebeat.xml ,输入以下内容：
```yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    # 日志文件在容器里面的路径，实际需要挂载宿主机的一个真实的日志目录
    - /home/logs/*.log
  json.keys_under_root: true
  json.message_key: log

setup.ilm.enabled: false
setup.template.name: "cloud"
setup.template.pattern: "cloud-*"

output.kafka:
  hosts: ["192.168.117.132:9092"]
  topic: log.filebeat.topic
  partition.round_robin:
    reachable_only: false
  
  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
  retry.max: 3
```
2. 启动命令：
```
docker run --rm -it --name filebeat --add-host kafka:192.168.117.132 \
-v ~/elk/filebeat.yml:/usr/share/filebeat/filebeat.yml \
-v ~/log_producer/log/:/home/logs/ \
docker.elastic.co/beats/filebeat:7.5.2

```

## 八、测试
1. 我们可以手动在~/log_producer/log下产生日志
```shell
mkdir /log_producer/log
vim /log_producer/log/test1.log
```
- 输入以下内容：
```json
{"startTime":"2020-02-10T07:43:44.049Z","categoryName":"main","data":["test message","1015"],"level":{"level":20000,"levelStr":"INFO","colour":"green"},"context":{},"pid":107082}
{"startTime":"2020-02-10T07:43:46.391Z","categoryName":"main","data":["test message","1016"],"level":{"level":20000,"levelStr":"INFO","colour":"green"},"context":{},"pid":107082}
{"startTime":"2020-02-10T07:43:48.392Z","categoryName":"main","data":["test message","1017"],"level":{"level":20000,"levelStr":"INFO","colour":"green"},"context":{},"pid":107082}
{"startTime":"2020-02-10T07:43:50.394Z","categoryName":"main","data":["test message","1018"],"level":{"level":20000,"levelStr":"INFO","colour":"green"},"context":{},"pid":107082}
{"startTime":"2020-02-10T07:43:52.397Z","categoryName":"main","data":["test message","1019"],"level":{"level":20000,"levelStr":"INFO","colour":"green"},"context":{},"pid":107082}
{"startTime":"2020-02-10T07:43:54.399Z","categoryName":"main","data":["test message","1020"],"level":{"level":20000,"levelStr":"INFO","colour":"green"},"context":{},"pid":107082}
{"startTime":"2020-02-10T07:43:56.402Z","categoryName":"main","data":["test message","1021"],"level":{"level":20000,"levelStr":"INFO","colour":"green"},"context":{},"pid":107082}
```
- 默认等filebeat 30s后应该就会按架构图那样的数据流程走了，也就是可以在kibana的Management中的es的
Index Manageme 里面看见 cloud-logstash-xxxx 这样的index
