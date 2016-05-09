ELK前端统计搭建
===

### 1、知识储备

* 前端请求js库：[alogs](https://github.com/fex-team/alogs)
* 日志收集：[Logstash](https://www.gitbook.com/book/chenryn/kibana-guide-cn/details)
* 日志数据存储及检索：[Elasticsearch](http://es.xiaoleilu.com/)
* 数据可视化：[Kibana4.x](https://www.gitbook.com/book/chenryn/kibana-guide-cn/details)

### 2、alogs简要介绍

alogs是由百度FEX开源的前端日志打点库，其原理是请求后端一个1x1的gif图片以使得后端打印日志方便之后的统计计算，除此之外，Alog修正了一些IE的兼容性问题并增加模块化。想要了解Alog的源码，可以通过其[简要解析文档](https://github.com/fex-team/alogs/blob/master/doc/alog%E4%BB%A3%E7%A0%81%E6%B5%85%E6%9E%90.md)进行了解，其[API](https://github.com/fex-team/alogs/blob/master/doc/API.md)及[example](https://github.com/fex-team/alogs/tree/master/examples)也可以在其项目中方便找到。

### 3、ELK相关搭建

#### 3.1、Logstash相关设置

##### 3.1.1、Logstash基本配置
在ELK的架构中通常有3个角色，Shipper、Broker及Indexer，这也是官方比较推荐的一种的架构方式，其原因在于Logstash的TCP默认最大只能创建20个事件，并且logstash的input-->filter-->output是阻塞进行的，但如果日志压力不大，则我们可以取消Broker只使用Shipper --> Indexer的架构方式。<br/>

由于我们只搭建前端的日志统计系统，那么一般情况下我们只需要收集nginx对应的access.log相关日志即可，对于alogs来说，其统计url大致如下所示：
```html
http://localhost:8080/t.gif?ts=3a&t=pageview&sid=inzo5jznsxd&ht=102&lt=103&drt=110
```
因此我们只需要解析querystring即可。<br/>

logstash解析nginx的access.log有多种方式，比较好的实践是设置log_fotmat，让nginx自己拼接出json字符串，例如：
```shell
log_format json '{"@timestamp":"$time_iso8601",'
                 '"host":"$server_addr",'
                 '"clientip":"$remote_addr",'
                 '"size":$body_bytes_sent,'
                 '"responsetime":$request_time,'
                 '"upstreamtime":"$upstream_response_time",'
                 '"upstreamhost":"$upstream_addr",'
                 '"http_host":"$host",'
                 '"url":"$uri",'
                 '"args": "$args",'
                 '"cookies":"$http_cookie",'
                 '"xff":"$http_x_forwarded_for",'
                 '"referer":"$http_referer",'
                 '"agent":"$http_user_agent",'
                 '"statggus":"$status"}';
```
这样，可以跳过logstash中的grok匹配等耗性能的阶段，只让logstash做对应的json解析操作，其配置类似于：
```shell
   input{
        file {
            path => "/var/log/nginx/*.log"
            type => "web"
            codec => json
        }
   }

   filter{
        if [args] {
            kv {
                prefix => "arg_"
                source => "args"
                field_split => "& "
                remove_field => ["args","url"]
            }
        }

        if [cookies] {
            kv {
                prefix => "cookie_"
                source => "cookies"
                field_split => "; "
                remove_field => ["cookies"]
            }
        }

        mutate {
           split => [ "upstreamtime", "," ]
        }

        mutate {
           convert => [ "upstreamtime", "float" ]
        }
   }

   output{
        stdout{
            codec => rubydebug
        }
   }
```

当前端尝试访问时http://127.0.0.1:8080?task=test&sid=1111 ，我们能够得到如下的结果：
```shell
{
      "@timestamp" => "2016-05-09T07:41:06.000Z",
            "host" => "127.0.0.1",
        "clientip" => "127.0.0.1",
            "size" => 612,
    "responsetime" => 0.0,
    "upstreamtime" => [
        [0] 0.0
    ],
    "upstreamhost" => "-",
       "http_host" => "127.0.0.1",
             "xff" => "-",
         "referer" => "-",
           "agent" => "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:46.0) Gecko/20100101 Firefox/46.0",
          "status" => "200",
        "@version" => "1",
            "path" => "/var/log/nginx/elk.access.log",
        "arg_task" => "test",
         "arg_sid" => "1111"
}
```

##### 3.1.2、Logstash增加Broker
由于logstash中TCP只会运行20个事例，并且input --> filter --> output是阻塞的（这也是headrtbeat检测插件的基础），因此为了提高日志收集的性能，我们需要使用到Broker，官方推荐的Broker为Redis，对于Shipper及Indexer的配置如下所示：
```shell
    # Shipper配置

    input{
        # 如上省略...
    }

    filter {
        # 如上省略...
    }

    output{
        redis{
            data_type => "list"
            key => "logstash-list"
            host => "127.0.0.1"
            port => 6379
        }
    }
```

```shell
    # Indexer配置
    input{
        redis{
            batch_count => 50 # 批量获取配置，默认为50条，可加大
            data_type => "list"
            key => "logstash-list"
            host => "127.0.0.1"
            port => 6379
            threads => 5
            codec => json
        }
    }

    output{
        elasticsearch{
            host => ["127.0.0.1"]
            protocol => "http"
            index => "logstash-%{type}-%{+YYYY.MM.dd}"
            document_type => "%{type}"
            workers => 5
            flush_size => 5000      # elasticsearch的bulk size，推荐以15m为基准，例如每条日志3k，则15*1024/3 ~= 5k
            idle_flush_time => 10   # 若10秒钟积累不到5000条，则也进行数据发送
            template_overwrite => true
        }
    }
```

##### 3.1.3、Logstash性能优化
* 直接进行json的编解码，减少groke等正则匹配
* 增大default template的index.refresh_interval
* 对not analyzed的字段进行doc value设置，防止fielddata的OOM

##### 3.1.4、Logstash压力测试
编写对应的conf文件
```shell
    input{
        generator{
            count => 10000000
            message => '{"@timestamp":"2016-05-09T00:52:21-07:00","host":"127.0.0.1","clientip":"127.0.0.1","size":612,"responsetime":0.000,"upstreamtime":"-","upstreamhost":"-","http_host":"127.0.0.1","url":"/index.html","args":"ts=3a&t=pageview&sid=inzo5jznsxd&ht=102&lt=103&drt=110","xff":"-","referer":"-","agent":"Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:46.0) Gecko/20100101 Firefox/46.0","status":"200"}'
            codec => json
        }
    }

    filter{
        if [args] {
            kv {
                prefix => "arg_"
                source => "args"
                field_split => "& "
                remove_field => ["args","url"]
            }
        }

        if [cookies] {
            kv {
                prefix => "cookie_"
                source => "cookies"
                field_split => "; "
                remove_field => ["cookies"]
            }
        }

        mutate {
           split => [ "upstreamtime", "," ]
        }

        mutate {
           convert => [ "upstreamtime", "float" ]
        }
    }

    output{
        stdout{
            codec => dots
        }
    }
```
执行对应的shell，等待速度稳定（例如12.5kiB/s等于12.5k event/s)
```shell
   > ./bin/logstash -f generator_dots.conf | pv -abt > /dev/null
   1.42MB 0:03:08 [7.75kB/s]B/s]
```

##### 3.1.5、Logstash监控
使用logstash-input-heartbeat进行心跳检测
```shell
    input{
        heartbeat{
            message => "epoch"
            interval => 60
            type => "heartbeat"
        }
    }

    output{
        if [type] == "heartbeat" {
            file{
                path => "/tmp/logstash-log/log-%{+YYYY.MM.dd}.log"
            }
        }else{

        }
    }
```

```shell
{"clock":1462785068,"host":"ubuntu","@version":"1","@timestamp":"2016-05-09T09:11:08.887Z","type":"heartbeat"}
```
然后通过clock和@timestamp进行时间对比就可以判断logstash是否有队列堵塞

##### 3.1.6、完整的Shipper配置
```shell
    input{
        heartbeat{
            message => "epoch"
            interval => 60
            type => "heartbeat"
        }

        file {
            path => ["/var/log/nginx/*.log"]
            type => "web"
            codec => json
        }
   }

   filter{
        if [type] == "web" {
            if [args] {
                kv {
                    prefix => "arg_"
                    source => "args"
                    field_split => "& "
                    remove_field => ["args","url"]
                }
            }

            if [cookies] {
                kv {
                    prefix => "cookie_"
                    source => "cookies"
                    field_split => "; "
                    remove_field => ["cookies"]
                }
            }

            mutate {
               split => [ "upstreamtime", "," ]
            }

            mutate {
               convert => [ "upstreamtime", "float" ]
            }
        }
   }

   output{
        if [type] == "heartbeat" {
            file{
                path => "/tmp/logstash-log/log-%{+YYYY.MM.dd}.log"
            }
        }else{
            redis{
                data_type => "list"
                key => "logstash-list"
                host => "127.0.0.1"
                port => 6379
            }
        }
   }
```

##### 3.1.7 Indexer完整配置
```shell
    input{
        heartbeat{
            message => "epoch"
            interval => 60
            type => "heartbeat"
        }

        redis{
            batch_count => 50 # 批量获取配置，默认为50条，可加大
            data_type => "list"
            key => "logstash-list"
            host => "127.0.0.1"
            port => 6379
            threads => 5
            codec => json
        }
    }

   output{
        if [type] == "heartbeat" {
            file{
                path => "/tmp/logstash-log/log-%{+YYYY.MM.dd}.log"
            }
        }else{
            elasticsearch{
                host => ["127.0.0.1"]
                protocol => "http"
                index => "logstash-%{type}-%{+YYYY.MM.dd}"
                document_type => "%{type}"
                workers => 5
                flush_size => 5000      # elasticsearch的bulk size，推荐以15m为基准，例如每条日志3k，则15*1024/3 ~= 5k
                idle_flush_time => 10   # 若10秒钟积累不到5000条，则也进行数据发送
                template_overwrite => true
            }
        }
   }
```

#### 3.2 Elasticsearch相关配置

Elasticsearch在一般应用中主要是用作搜索服务器，因此对于日志这种多写少读的场景，需要优化elasticsearch的写相关性能

* 增大refresh_interval：ELK默认1s及刷新memory buffer到磁盘生成segement，但是对于基本是写的日志统计来说我们不需要这么实时的刷新，因此一般情况下ELK需要增加refresh_interval，同时这样也可以减少归并线程的性能消耗
```shell
   curl -XPOST http://127.0.0.1:9200/logstash-2015.06.21/_setting -d '
    {"refresh_interval":"30s"}
   '
```
* 根据磁盘增大归并线程的限速：归并segement是相当耗时的过程，因此elasticsearch为了归并不影响到其他任务的进行，将速度限制到了20MB，这个速度对于写入量大和SSD来说明显偏少，建议提升到100+MB
```shell
    curl -XPOST http://127.0.0.1:9200/_cluster/settings -d '
    {
        "persistent": {
            "indices.store.throttle.max_bytes_per_sec" : "100mb"
        }
    }
   '
```

* 根据实际情况修改归并策略：
    1. index.merge.policy.floor_segment：默认2M，低于这个数值的segment不归并
    2. index.merge.policy.max_merge_at_once：默认依次最多归并10个segment
    3. index.merge.policy.max_merge_at_once_explicit：默认optimize时一次归并30个
    4. index.merge.policy.max_merged_segment：默认5G，大于这个不参与归并，optimize时除外

* 合理配置分片：在ELK，为了提高对应的写入性能，我们可以减少副本的使用（在不需要数据可靠性下甚至可以没有），并且为了防止冷热分片不均造成的集群崩溃，我们最好提前计算单个集群的分片数，例如：一个集群有2台机器，索引主分片10个，副本分片1个，那么，一个节点应该有4个分片（为了某台机器出现故障需要分片迁移，我们一般+1，设置为5）
```shell
    curl -XPOST http://127.0.0.1:9200/logstash-2015.06.21/_setting -d '
    {
        "index":{
            "routing.allocation.total_shards_per_node": "5"
        }
    }
   '
```

* 读写分离
```shell
    #1.在N台冷数据的节点配置中配置node.tag: stable
    #2.在M台热数据的节点配置中配置node.tag: hot
    #3.模板中控制对新建索引添加hot标签
    {
        "order":0,
        "template":"*",
        "setting":{
            "index.routing.allocation.require.tag":"hot"
        }
    }
    #4.cron任务，定时将tag更改为stable
    curl -XPUT http://127.0.0.1:9200/indexname/_setting -d'
        "index" :{
            "routing":{
                "allocation":{
                    "require":{
                        "tag":"stable"
                    }
                }
            }
        }
    '
```
这样，elasticsearch会自动将对应数据迁移到冷数据的N台机器上

* 定期使用curator工具进行optimize
```shell
    curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 5 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-nginx-
    curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 10 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-client- --exclude 'logstash-mweibo-client-2015.05.11'
    curator --timeout 36000 --host 10.0.0.100 delete indices --older-than 30 --time-unit days --timestring '%Y.%m.%d' --regex '^logstash-\d+'
    curator --timeout 36000 --host 10.0.0.100 close indices --older-than 7 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-
    curator --timeout 36000 --host 10.0.0.100 optimize --max_num_segments 1 indices --older-than 1 --newer-than 7 --time-unit days --timestring '%Y.%m.%d' --prefix logstash-
```