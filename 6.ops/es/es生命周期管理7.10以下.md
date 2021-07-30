
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

### 1）创建一个索引策略
```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
//rollover前距离索引的创建时间最大为7天(这里测试1m)
            "max_age": "1m",
//rollover前索引的最大大小不超过10G
            "max_size": "10G",
//rollover前索引的最大文档数不超过1000000个
            "max_docs": 1000000
          }
        }
      },
      "warm": {
//rollover之后进入warm阶段的时间不小于10天
        "min_age": "2d",
        "actions": {
          "forcemerge": {
//强制分片merge到segment为1
            "max_num_segments": 1
          },
          "shrink": {
//收缩分片数为1
            "number_of_shards": 1
          },
          "allocate": {
//副本数为1
            "number_of_replicas": 1,
            "require": {
//分配到warm 节点，ES可根据机器资源配置不同类型的节点
              "type": "warm"
            }
          }
        }
      },
      "cold": {
//rollover之后进入cold阶段的时间不小于30天
        "min_age": "3d",
        "actions": {
          "allocate": {
//副本数为0
            "number_of_replicas": 0,
            "require": {
//分配到cold 节点，ES可根据机器资源配置不同类型的节点
              "type": "cold"
            }
          }
        }
      },
      "delete": {
//rollover之后进入cold阶段的时间不小于60天
        "min_age": "4d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```
去注释完整策略
```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age" : "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "10G",
            "max_docs": 1000
          }
        }
      },
      "warm": {
        "min_age": "2d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          },
          "allocate": {
            "number_of_replicas": 1,
            "require": {
              "type": "warm"
            }
          }
        }
      },
      "cold": {
        "min_age": "3d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0,
            "require": {
              "type": "cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "4d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### 2）创建一个索引模版，使用指定的索引策略
```json
PUT _template/my_template
{
//模版匹配的索引名以"logs-"开头
  "index_patterns": ["mylogs-*"],
  "settings": {
//索引分片数为2
    "number_of_shards": 1,
//索引副本数为1 
    "number_of_replicas": 1,
//索引使用的索引策略为my_policy
    "index.lifecycle.name": "my_policy",
//定义了我们需要indexing的node的属性是hot
    "index.routing.allocation.require.data": "hot",
//索引rollover后切换的索引别名为 mylogs
    "index.lifecycle.rollover_alias": "mylogs"    
  }
}
```

### 3) 创建一个符合上述索引模版的索引
```json
PUT mylogs-000001
    {
      "aliases": {
        //别名为 mylogs
        "mylogs": {
          //允许索引被写入数据
          "is_write_index": true
        }
      }
    }
```
- 当发生rollover时，老索引的别名mylogs将被去掉，新创建的索引别名为mylogs，同时索引名自动在索引名上自增，变为 mylogs-000002
- 这里为啥要用索引模版来关联索引和索引策略呢？因为如果在创建索引时不通过模版指定索引策略，当发生rollover时，新的索引并不会继承原来索引的索引策略

ES检测索引的索引策略是否该生效的时间默认为10min，可通过修改以下配置：
```json
PUT _cluster/settings
{
  "transient": {
    "indices.lifecycle.poll_interval": "3s" 
  }
}
```

## 常用命令
```sh
# 查看集群node状态
GET _cat/nodes?v

# 检查集群node属性
GET _cat/nodeattrs?v&s=name

# 查看所有索引
GET _cat/indices?v

# 查看索引
GET _cat/indices?v
GET _cat/indices/*rabbit*?v
GET _cat/indices/*mylogs*?v

# 查看分片
GET _cat/shards?v
GET _cat/shards/*rabbit*?v
GET _cat/shards/*mylogs*?v
```


[阿里云参考文档](https://help.aliyun.com/document_detail/178307.html)
https://help.aliyun.com/document_detail/178307.html

https://elasticstack.blog.csdn.net/article/details/110816948