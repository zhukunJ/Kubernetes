
## 1、Index Lifecycle Management 中可以用的 Action 有哪些
- Allocate
> 可指定一个索引的副本数，用于warm, cold阶段
> 在热阶段不允许进行分配操作。索引的初始分配必须手动完成或通过 索引模板完成
- Delete
> 永久删除索引
- Force merge
> 可触发一个索引分片的segment merge，同时释放掉被删除文档的占用空间。用于Warm阶段
- Freeze
> 冻结索引以最大程度地减少其内存占用量
- Migrate
> 将索引分片移动到与当前ILM阶段相对应的数据层
- Read only
> 阻止对索引的写操作
- Rollover
> 当写入索引达到了一定的大小，文档数量或创建时间时，Rollover可创建一个新的写入索引，将旧的写入索引的别名去掉，并把别名赋给新的写入索引。所以便可以通过切换别名控制写入的索引是谁。它可用于Hot阶段
- Searchable snapshot
> 为配置的存储库中的托管索引拍摄快照，并将其作为可搜索快照安装
- Set priority
> 降低索引在生命周期中的优先级，以确保首先恢复热索引
- Shrink
> 减少一个索引的主分片数，可用于Warm阶段。需要注意的是当shink完成后索引名会由原来的index-name 变为 shrink-index-name
- Unfollow
> 将关注者索引转换为常规索引。在过渡，缩小或可搜索快照操作之前自动执行
- Wait for snapshot
> 删除索引之前，请确保快照已存在


## 2、每个阶段可以进行的Action 有哪些

- Hot：索引可写入，也可查询。
- Warm：索引不可写入，但可查询。
- Cold：索引不可写入，但很少被查询，查询的慢点也可接受。
- Delete：索引可被安全的删除

|ILM的阶段	| 此阶段中可以执行的操作有什么  |
| -------- | ------------------------ |
| Hot	| Force merge, Rollover, Set priority, Unfollow |
| Warm	| Allocate, Force merge, Read only, Set priority, Shrink, Unfollow|
| Cold	| Allocate, Freeze, Set priority, Unfollow |
| Delete| Delete, Wait for snapshot |

## 3、设置索引策略

### 1.使用自带logs策略
```json
// 查看集群node状态
GET _cat/nodes?v
ip          heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.42.1.76            61          66   4    1.25    0.92     1.02 m         *      elasticsearch-master-0
10.42.6.225           35          69   3    1.92    1.27     1.08 cdfhstw   -      elasticsearch-datacold-0
10.42.6.224           30          69   3    1.92    1.27     1.08 cdfhstw   -      elasticsearch-datawarm-0
10.42.2.146           57          68   4    3.18    2.87     2.03 cdfhstw   -      elasticsearch-datawarm-1
10.42.4.227           64          68   3    0.53    0.91     1.39 cdfhstw   -      elasticsearch-datahot-1
10.42.1.77            42          69   3    1.25    0.92     1.02 cdfhstw   -      elasticsearch-datahot-0
10.42.3.190           34          66   3    1.15    0.93     0.72 -         -      elasticsearch-client-0

// 检查集群node属性
GET _cat/nodeattrs?v&s=name
node                     host        ip          attr            value
elasticsearch-client-0   10.42.3.190 10.42.3.190 xpack.installed true
elasticsearch-client-0   10.42.3.190 10.42.3.190 transform.node  false
elasticsearch-datacold-0 10.42.6.225 10.42.6.225 xpack.installed true
elasticsearch-datacold-0 10.42.6.225 10.42.6.225 box_type        cold
elasticsearch-datacold-0 10.42.6.225 10.42.6.225 transform.node  true
elasticsearch-datahot-0  10.42.1.77  10.42.1.77  xpack.installed true
elasticsearch-datahot-0  10.42.1.77  10.42.1.77  box_type        hot
elasticsearch-datahot-0  10.42.1.77  10.42.1.77  transform.node  true
elasticsearch-datahot-1  10.42.4.227 10.42.4.227 xpack.installed true
elasticsearch-datahot-1  10.42.4.227 10.42.4.227 box_type        hot
elasticsearch-datahot-1  10.42.4.227 10.42.4.227 transform.node  true
elasticsearch-datawarm-0 10.42.6.224 10.42.6.224 xpack.installed true
elasticsearch-datawarm-0 10.42.6.224 10.42.6.224 box_type        warm
elasticsearch-datawarm-0 10.42.6.224 10.42.6.224 transform.node  true
elasticsearch-datawarm-1 10.42.2.146 10.42.2.146 xpack.installed true
elasticsearch-datawarm-1 10.42.2.146 10.42.2.146 box_type        warm
elasticsearch-datawarm-1 10.42.2.146 10.42.2.146 transform.node  true
elasticsearch-master-0   10.42.1.76  10.42.1.76  xpack.installed true
elasticsearch-master-0   10.42.1.76  10.42.1.76  transform.node  false

// 查看索引
GET _cat/indices?v
GET _cat/indices/*rabbit*?v
GET _cat/indices/*mylogs*?v

// 查看分片
GET _cat/shards?v
GET _cat/shards/*rabbit*?v
GET _cat/shards/*mylogs*?v
```
### 2.修改策略生效时间
```json
// ES检测索引策略是否该生效的时间默认为10min，可通过修改以下配置
GET _cluster/settings
PUT _cluster/settings
{
  "transient": {
    "indices.lifecycle.poll_interval": "3s" 
  }
}
```
### 3.创建组件模板
- 目的是为了让生命周期从hot开始
```json
// 组件模板
PUT _component_template/mylogs-settings
{
  "template" : {
    "settings" : {
      "index" : {
        "lifecycle" : {
          "name" : "mylogs"
        },
        "codec" : "best_compression",
        "routing" : {
          "allocation" : {
            "require" : {
              "box_type" : "hot"
            }
          }
        },
        "query" : {
          "default_field" : [
            "message"
          ]
        }
      }
    }
  },
  "_meta" : {
    "description" : "default settings for the logs index template installed by x-pack",
    "managed" : true
  }
}
```
### 4.clone默认的索引模板
- 大于7.10的新版本，一定要在默认的索引模板页面点点点克隆
- 模板名称为 mylogs，匹配类型为 mylogs-*

### 5.创建新索引策略
```json
PUT _ilm/policy/mypolicy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age" : "0ms",
        "actions": {
          "rollover": {
            "max_age": "1m",
            "max_size": "10gb",
            "max_docs": 10000000
          },
          "set_priority" : {
            "priority" : 100
          }
        }
      },
      "warm": {
        "min_age": "3m",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          },
          "readonly" : { },
          "set_priority" : {
            "priority" : 50
          },
          "allocate": {
            "number_of_replicas": 1,
            "include" : { },
            "exclude" : { },
            "require": {
              "type" : "warm"
            }
          }
        }
      },
      "cold": {
        "min_age": "5m",
        "actions": {
          "allocate": {
            "number_of_replicas": 0,
            "include" : { },
            "exclude" : { },
            "require": {
              "type" : "cold"
            }
          },
          "freeze" : { },
          "set_priority" : {
            "priority" : 10
          }
        }
      },
      "delete": {
        "min_age": "10m",
        "actions": {
          "delete": {
            "delete_searchable_snapshot" : true
          }
        }
      }
    }
  }
}
```
```sh
# http://127.0.0.1:5601/app/management/data/index_lifecycle_management/policies/edit/logs 
在默认的策略下关联 
Warm phase -> Data allocation -> Data tier options -> {Custom}
                                 Select a node attribute -> {box_type:warm}

Cold phase -> Data allocation -> Data tier options -> {Custom}
                                 Select a node attribute -> {box_type:cold}
```
### 5.关联索引策略和索引模板
>- http://127.0.0.1:5601/app/management/data/index_lifecycle_management/policies
>- 在索引生命周期页面关联 mypolicy 策略和 mylogs 索引模板

### 6.配置logstash
```sh
  logstash.conf: |
    input {
      kafka {
        bootstrap_servers => "kafka-headless:9092"
        topics_pattern => "logs.*"
        codec => "json"
        client_id => "logs"
        auto_offset_reset => "latest"
      }
      kafka {
        bootstrap_servers => "kafka-headless:9092"
        topics_pattern => "mylogs.*"
        codec => "json"
        client_id => "mylogs"
        auto_offset_reset => "latest"
      }
    }
    filter {
      mutate {
       remove_field => ["input","beat","prospector","@version","offset"]
      }
    }
    output {
      if [client_id] == "mylogs" {
        elasticsearch {
          hosts => ["elasticsearch-client:9200"]
          index => "%{[topic]}-%{+YYYY-MM-dd}"
          user => "elastic"
          password => "elastic666"
          ilm_enabled => false
          action => "create"
          ilm_policy => "mypolicy"
        }
      } else {
        elasticsearch {
          hosts => ["elasticsearch-client:9200"]
          index => "%{[topic]}-%{+YYYY-MM-dd}"
          user => "elastic"
          password => "elastic666"
          ilm_enabled => false
          action => "create"
        }
      }
    }

```

### 7.查看
```json
index                                          shard prirep state   docs store ip          node
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000006 0     p      STARTED    0  230b 10.42.6.225 elasticsearch-datacold-0
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000007 0     p      STARTED    0  230b 10.42.6.225 elasticsearch-datacold-0
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000008 0     p      STARTED    0  208b 10.42.2.146 elasticsearch-datawarm-1
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000008 0     r      STARTED    0  208b 10.42.6.224 elasticsearch-datawarm-0
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000009 0     p      STARTED    0  208b 10.42.1.77  elasticsearch-datahot-0
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000009 0     r      STARTED    0  208b 10.42.4.227 elasticsearch-datahot-1
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000010 0     p      STARTED    0  208b 10.42.1.77  elasticsearch-datahot-0
.ds-mylogs-nginx1-2021-04-23-2021.04.23-000010 0     r      STARTED    0  208b 10.42.4.227 elasticsearch-datahot-1


index                                          shard prirep state   docs store ip          node
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000014 0     p      STARTED    0  208b 10.42.1.77  elasticsearch-datahot-0
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000014 0     r      STARTED    0  208b 10.42.4.227 elasticsearch-datahot-1
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000013 0     p      STARTED    0  208b 10.42.1.77  elasticsearch-datahot-0
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000013 0     r      STARTED    0  208b 10.42.4.227 elasticsearch-datahot-1
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000012 0     p      STARTED    0  208b 10.42.1.77  elasticsearch-datahot-0
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000012 0     r      STARTED    0  208b 10.42.4.227 elasticsearch-datahot-1
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000011 0     p      STARTED    0  208b 10.42.2.146 elasticsearch-datawarm-1
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000011 0     r      STARTED    0  208b 10.42.6.224 elasticsearch-datawarm-0
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000010 0     p      STARTED    0  230b 10.42.6.225 elasticsearch-datacold-0
.ds-logs-rabbitmq-2021-04-23-2021.04.23-000009 0     p      STARTED    0  230b 10.42.6.225 elasticsearch-datacold-0

```