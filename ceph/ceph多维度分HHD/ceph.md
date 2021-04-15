

## node节点连接rook-ceph集群

1、配置Ceph yum源

```
# 如果主机是centos7系统
[root@k8s-node-01 ~]# cat /etc/yum.repos.d/ceph.repo 
[ceph-centos7]
name=ceph
baseurl=https://mirrors.aliyun.com/ceph/rpm-octopus/el7/x86_64/
enabled=1
gpgcheck=0

# 如果主机是centos8系统
cat /etc/yum.repos.d/ceph.repo 
[ceph-centos8]
name=ceph
baseurl=https://mirrors.aliyun.com/ceph/rpm-octopus/el8/x86_64/
enabled=1
gpgcheck=0
```

2、安装ceph-common

`[root@k8s-node-01 ~]# yum -y install ceph-common`



## 对象存储

### Amazon S3 Tools

`yum install s3cmd -y`

```shell
# 查看所有用户
[root@k8s-node-01 ~]# radosgw-admin metadata list user
[
    "dashboard-admin",
    "ceph-user-2ZCR3j35",
    "rook-ceph-internal-s3-user-checker-286d98fc-5321-41c5-a7ae-49c2c8999885"
]

# 查看用户信息
[root@k8s-node-01 ~]# radosgw-admin user info --uid=dashboard-admin
{
    "user_id": "dashboard-admin",
    "display_name": "dashboard-admin",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "dashboard-admin",
            "access_key": "PKI0VMOVVKKOOQSYY479",
            "secret_key": "eGyuZR5RhNeCnLYysBqGcskeMTLifURJTpHymyHQ"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "system": "true",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

# 配置 s3cmd 配置文件
[root@k8s-node-01 ~]# s3cmd --configure

Enter new values or accept defaults in brackets with Enter.
Refer to user manual for detailed description of all options.

Access key and Secret key are your identifiers for Amazon S3. Leave them empty for using the env variables.
Access Key: PKI0VMOVVKKOOQSYY479
Secret Key: eGyuZR5RhNeCnLYysBqGcskeMTLifURJTpHymyHQ
Default Region [US]: 

Use "s3.amazonaws.com" for S3 Endpoint and not modify it to the target Amazon S3.
S3 Endpoint [s3.amazonaws.com]: 10.43.233.199:80
------------ rook-ceph-rgw-my-store  ClusterIP 10.43.233.199  80/TCP -------------

Use "%(bucket)s.s3.amazonaws.com" to the target Amazon S3. "%(bucket)s" and "%(location)s" vars can be used
if the target S3 system supports dns based buckets.
DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3.amazonaws.com]: 10.43.233.199:80/%(bucket)s               

Encryption password is used to protect your files from reading
by unauthorized persons while in transfer to S3
Encryption password: 
Path to GPG program [/usr/bin/gpg]: 

When using secure HTTPS protocol all communication with Amazon S3
servers is protected from 3rd party eavesdropping. This method is
slower than plain HTTP, and can only be proxied with Python 2.7 or newer
Use HTTPS protocol [Yes]: no

On some networks all internet access must go through a HTTP proxy.
Try setting it here if you can't connect to S3 directly
HTTP Proxy server name: 

New settings:
  Access Key: PKI0VMOVVKKOOQSYY479
  Secret Key: eGyuZR5RhNeCnLYysBqGcskeMTLifURJTpHymyHQ
  Default Region: US
  S3 Endpoint: 10.43.233.199:80
  DNS-style bucket+hostname:port template for accessing a bucket: 10.43.233.199:80/%(bucket)s
  Encryption password: 
  Path to GPG program: /usr/bin/gpg
  Use HTTPS protocol: False
  HTTP Proxy server name: 
  HTTP Proxy server port: 0

Test access with supplied credentials? [Y/n] y
Please wait, attempting to list all buckets...
Success. Your access key and secret key worked fine :-)

Now verifying that encryption works...
Not configured. Never mind.

Save settings? [y/N] y
Configuration saved to '/root/.s3cfg'

# 查看所有桶
[root@k8s-node-01 ~]# radosgw-admin bucket list
[
    "ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1",
    "rook-ceph-bucket-checker-286d98fc-5321-41c5-a7ae-49c2c8999885"
]


# 查看桶内对象
[root@k8s-node-01 ~]# radosgw-admin bucket list --bucket=ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1
[]

# 查看桶内对象
[root@k8s-node-01 ~]# s3cmd ls s3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1
ERROR: S3 error: 403 (SignatureDoesNotMatch)

# 修改认证
[root@k8s-node-01 ~]# vi /root/.s3cfg
···
signature_v2 = True
···

# 查看桶内对象
[root@k8s-node-01 ~]# s3cmd ls s3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1
[root@k8s-node-01 ~]# 

# 上传文件
[root@k8s-node-01 ~]# s3cmd put ./node.sh s3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1/node.sh
upload: './node.sh' -> 's3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1/node.sh'  [1 of 1]
 5894 of 5894   100% in    0s   356.64 KB/s  done

# 查看桶内对象
[root@k8s-node-01 ~]# s3cmd ls s3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1
2021-04-02 04:16         5894  s3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1/node.sh

# 删除桶内对象
[root@k8s-node-01 ~]# s3cmd del s3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1/node.sh   
delete: 's3://ceph-bkt-8310793d-b6b7-4dd4-86c3-afe9efe218c1/node.sh'



```



## 调整 CRUSH Map

### 1. 查看map信息

```sh
[root@k8s-node-01 ~]# ceph -s
  cluster:
    id:     687843e5-dfc7-4e6d-b9a1-6221f3922a7a
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 18h)
    mgr: a(active, since 18h)
    mds: myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
    osd: 6 osds: 6 up (since 16h), 6 in (since 16h)
    rgw: 2 daemons active (my.store.a, my.store.b)
 
  task status:
 
  data:
    pools:   11 pools, 177 pgs
    objects: 2.43k objects, 6.6 GiB
    usage:   26 GiB used, 424 GiB / 450 GiB avail
    pgs:     177 active+clean
 
  io:
    client:   1.3 KiB/s rd, 170 B/s wr, 2 op/s rd, 0 op/s wr

[root@k8s-node-01 ~]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
-1         0.34186  root default                                   
-9         0.04880      host k8s-node-04                           
 3    hdd  0.04880          osd.3             up   1.00000  1.00000
-3         0.09769      host k8s-node-05                           
 0    hdd  0.09769          osd.0             up   1.00000  1.00000
-5         0.09769      host k8s-node-06                           
 1    hdd  0.09769          osd.1             up   1.00000  1.00000
-7         0.09769      host k8s-node-07                           
 2    hdd  0.09769          osd.2             up   1.00000  1.00000
 4    hdd        0  osd.4                     up   1.00000  1.00000
 5    hdd        0  osd.5                     up   1.00000  1.00000
 
 # 查看crush map分布
[root@k8s-node-01 ~]# ceph osd crush tree
ID  CLASS  WEIGHT   TYPE NAME           
-1         0.34186  root default        
-9         0.04880      host k8s-node-04
 3    hdd  0.04880          osd.3       
-3         0.09769      host k8s-node-05
 0    hdd  0.09769          osd.0       
-5         0.09769      host k8s-node-06
 1    hdd  0.09769          osd.1       
-7         0.09769      host k8s-node-07
 2    hdd  0.09769          osd.2
 
 # 更详细的查看map信息
 [root@k8s-node-01 ~]# ceph osd crush dump
 {
    "devices": [      # 设备
        {
            "id": 0,
            "name": "osd.0",
            "class": "hdd"
        },
        {
            "id": 1,
            "name": "osd.1",
            "class": "hdd"
        },
        {
            "id": 2,
            "name": "osd.2",
            "class": "hdd"
        },
        {
            "id": 3,
            "name": "osd.3",
            "class": "hdd"
        },
        {
            "id": 4,
            "name": "osd.4",
            "class": "hdd"
        },
        {
            "id": 5,
            "name": "osd.5",
            "class": "hdd"
        }
    ],
    "types": [     # 类型，不同级别的保护机制
        {
            "type_id": 0,
            "name": "osd"
        },
        {
            "type_id": 1,
            "name": "host"
        },
        {
            "type_id": 2,
            "name": "chassis"
        },
        {
            "type_id": 3,
            "name": "rack"
        },
        {
            "type_id": 4,
            "name": "row"
        },
        {
            "type_id": 5,
            "name": "pdu"
        },
        {
            "type_id": 6,
            "name": "pod"
        },
        {
            "type_id": 7,
            "name": "room"
        },
        {
            "type_id": 8,
            "name": "datacenter"
        },
        {
            "type_id": 9,
            "name": "zone"
        },
        {
            "type_id": 10,
            "name": "region"
        },
        {
            "type_id": 11,
            "name": "root"
        }
    ],
    "buckets": [
        {
            "id": -1,
            "name": "default",
            "type_id": 11,
            "type_name": "root",
            "weight": 22404,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": -3,
                    "weight": 6402,
                    "pos": 0
                },
                {
                    "id": -5,
                    "weight": 6402,
                    "pos": 1
                },
                {
                    "id": -7,
                    "weight": 6402,
                    "pos": 2
                },
                {
                    "id": -9,
                    "weight": 3198,
                    "pos": 3
                }
            ]
        },
        {
            "id": -2,
            "name": "default~hdd",
            "type_id": 11,
            "type_name": "root",
            "weight": 22404,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": -4,
                    "weight": 6402,
                    "pos": 0
                },
                {
                    "id": -6,
                    "weight": 6402,
                    "pos": 1
                },
                {
                    "id": -8,
                    "weight": 6402,
                    "pos": 2
                },
                {
                    "id": -10,
                    "weight": 3198,
                    "pos": 3
                }
            ]
        },
        {
            "id": -3,
            "name": "k8s-node-05",
            "type_id": 1,
            "type_name": "host",
            "weight": 6402,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 0,
                    "weight": 6402,
                    "pos": 0
                }
            ]
        },
        {
            "id": -4,
            "name": "k8s-node-05~hdd",
            "type_id": 1,
            "type_name": "host",
            "weight": 6402,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 0,
                    "weight": 6402,
                    "pos": 0
                }
            ]
        },
        {
            "id": -5,
            "name": "k8s-node-06",
            "type_id": 1,
            "type_name": "host",
            "weight": 6402,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 1,
                    "weight": 6402,
                    "pos": 0
                }
            ]
        },
        {
            "id": -6,
            "name": "k8s-node-06~hdd",
            "type_id": 1,
            "type_name": "host",
            "weight": 6402,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 1,
                    "weight": 6402,
                    "pos": 0
                }
            ]
        },
        {
            "id": -7,
            "name": "k8s-node-07",
            "type_id": 1,
            "type_name": "host",
            "weight": 6402,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 2,
                    "weight": 6402,
                    "pos": 0
                }
            ]
        },
        {
            "id": -8,
            "name": "k8s-node-07~hdd",
            "type_id": 1,
            "type_name": "host",
            "weight": 6402,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 2,
                    "weight": 6402,
                    "pos": 0
                }
            ]
        },
        {
            "id": -9,
            "name": "k8s-node-04",
            "type_id": 1,
            "type_name": "host",
            "weight": 3198,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 3,
                    "weight": 3198,
                    "pos": 0
                }
            ]
        },
        {
            "id": -10,
            "name": "k8s-node-04~hdd",
            "type_id": 1,
            "type_name": "host",
            "weight": 3198,
            "alg": "straw2",
            "hash": "rjenkins1",
            "items": [
                {
                    "id": 3,
                    "weight": 3198,
                    "pos": 0
                }
            ]
        }
    ],
    "rules": [
        {
            "rule_id": 0,
            "rule_name": "replicated_rule",
            "ruleset": 0,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 1,
            "rule_name": "replicapool",
            "ruleset": 1,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 2,
            "rule_name": "myfs-metadata",
            "ruleset": 2,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 3,
            "rule_name": "myfs-data0",
            "ruleset": 3,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 4,
            "rule_name": "my-store.rgw.control",
            "ruleset": 4,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 5,
            "rule_name": "my-store.rgw.meta",
            "ruleset": 5,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 6,
            "rule_name": "my-store.rgw.log",
            "ruleset": 6,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 7,
            "rule_name": "my-store.rgw.buckets.index",
            "ruleset": 7,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 8,
            "rule_name": "my-store.rgw.buckets.non-ec",
            "ruleset": 8,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 9,
            "rule_name": ".rgw.root",
            "ruleset": 9,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        },
        {
            "rule_id": 10,
            "rule_name": "my-store.rgw.buckets.data",
            "ruleset": 10,
            "type": 1,
            "min_size": 1,
            "max_size": 10,
            "steps": [
                {
                    "op": "take",
                    "item": -1,
                    "item_name": "default"
                },
                {
                    "op": "chooseleaf_firstn",
                    "num": 0,
                    "type": "host"
                },
                {
                    "op": "emit"
                }
            ]
        }
    ],
    "tunables": {
        "choose_local_tries": 0,
        "choose_local_fallback_tries": 0,
        "choose_total_tries": 50,
        "chooseleaf_descend_once": 1,
        "chooseleaf_vary_r": 1,
        "chooseleaf_stable": 1,
        "straw_calc_version": 1,
        "allowed_bucket_algs": 54,
        "profile": "jewel",
        "optimal_tunables": 1,
        "legacy_tunables": 0,
        "minimum_required_version": "jewel",
        "require_feature_tunables": 1,
        "require_feature_tunables2": 1,
        "has_v2_rules": 0,
        "require_feature_tunables3": 1,
        "has_v3_rules": 0,
        "has_v4_buckets": 1,
        "require_feature_tunables5": 1,
        "has_v5_rules": 0
    },
    "choose_args": {}
}
    
# 查看当前pool写入osd的规则
[root@k8s-node-01 ~]# ceph osd crush rule ls
replicated_rule
replicapool
myfs-metadata
myfs-data0
my-store.rgw.control
my-store.rgw.meta
my-store.rgw.log
my-store.rgw.buckets.index
my-store.rgw.buckets.non-ec
.rgw.root
my-store.rgw.buckets.data

# 查看当前集群中的所有pool
[root@k8s-node-01 ~]# ceph osd lspools
1 device_health_metrics
2 replicapool
3 myfs-metadata
4 myfs-data0
5 my-store.rgw.control
6 my-store.rgw.meta
7 my-store.rgw.log
8 my-store.rgw.buckets.index
9 my-store.rgw.buckets.non-ec
10 .rgw.root
11 my-store.rgw.buckets.data

# 查看某一个pool的crush规则
[root@k8s-node-01 ~]# ceph osd pool get replicapool crush_rule
crush_rule: replicapool



```



### 2. 编译 CRUSH Map

要获取集群的 CRUSH Map，执行命令

```sh
[root@k8s-node-01 ~]# ceph osd getcrushmap -o crushmap20210413.bin
22

# crushmap.bin 是一个二进制文件
[root@k8s-node-01 ~]# file crushmap20210413.bin
crushmap20210413.bin: MS Windows icon resource - 16 icons, 11-colors
```

反编译 CRUSH Map

```sh


# 反编译 CRUSH Map
[root@k8s-node-01 ~]# crushtool -d crushmap20210413.bin -o crushmap20210413.conf

[root@k8s-node-01 ~]# cat crushmap20210413.conf
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable chooseleaf_stable 1
tunable straw_calc_version 1
tunable allowed_bucket_algs 54

# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root

# buckets
host k8s-node-05 {
        id -3           # do not change unnecessarily
        id -4 class hdd         # do not change unnecessarily
        # weight 0.098
        alg straw2
        hash 0  # rjenkins1
        item osd.0 weight 0.098
}
host k8s-node-06 {
        id -5           # do not change unnecessarily
        id -6 class hdd         # do not change unnecessarily
        # weight 0.098
        alg straw2
        hash 0  # rjenkins1
        item osd.1 weight 0.098
}
host k8s-node-07 {
        id -7           # do not change unnecessarily
        id -8 class hdd         # do not change unnecessarily
        # weight 0.098
        alg straw2
        hash 0  # rjenkins1
        item osd.2 weight 0.098
}
host k8s-node-04 {
        id -9           # do not change unnecessarily
        id -10 class hdd                # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.3 weight 0.049
}
root default {
        id -1           # do not change unnecessarily
        id -2 class hdd         # do not change unnecessarily
        # weight 0.342
        alg straw2
        hash 0  # rjenkins1
        item k8s-node-05 weight 0.098
        item k8s-node-06 weight 0.098
        item k8s-node-07 weight 0.098
        item k8s-node-04 weight 0.049
}

# rules
rule replicated_rule {
        id 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule replicapool {
        id 1
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-metadata {
        id 2
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-data0 {
        id 3
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.control {
        id 4
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.meta {
        id 5
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.log {
        id 6
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.index {
        id 7
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.non-ec {
        id 8
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule .rgw.root {
        id 9
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.data {
        id 10
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
# end crush map

```



编辑crush map

```
[root@k8s-node-01 ~]# cp crushmap20210413.conf crushmap20210413-new.conf && vi crushmap20210413-new.conf
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable chooseleaf_stable 1
tunable straw_calc_version 1
tunable allowed_bucket_algs 54

# devices
device 0 osd.0 class hdd
device 1 osd.1 class hdd
device 2 osd.2 class hdd
device 3 osd.3 class ssd
device 4 osd.4 class ssd
device 5 osd.5 class hdd

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 zone
type 10 region
type 11 root

# buckets
host k8s-node-05 {
        id -3           # do not change unnecessarily
        id -4 class hdd         # do not change unnecessarily
        # weight 0.098
        alg straw2
        hash 0  # rjenkins1
        item osd.0 weight 0.098
}
host k8s-node-06 {
        id -5           # do not change unnecessarily
        id -6 class hdd         # do not change unnecessarily
        # weight 0.098
        alg straw2
        hash 0  # rjenkins1
        item osd.1 weight 0.098
}
host k8s-node-07 {
        id -7           # do not change unnecessarily
        id -8 class hdd         # do not change unnecessarily
        # weight 0.098
        alg straw2
        hash 0  # rjenkins1
        item osd.2 weight 0.098
}
host k8s-node-04 {
        id -9           # do not change unnecessarily
        id -10 class ssd                # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.3 weight 0.049
}
host k8s-node-03 {
        id -11           # do not change unnecessarily
        id -12 class ssd                # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.4 weight 0.049
}
host k8s-node-02 {
        id -13           # do not change unnecessarily
        id -14 class hdd                # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.5 weight 0.049
}
root hdd {
        id -1           # do not change unnecessarily
        id -2 class hdd         # do not change unnecessarily
        # weight 0.342
        alg straw2
        hash 0  # rjenkins1
        item k8s-node-05 weight 0.098
        item k8s-node-06 weight 0.098
        item k8s-node-07 weight 0.098
        item k8s-node-02 weight 0.049
}
root ssd {
        # weight 0.342
        alg straw2
        hash 0  # rjenkins1
        item k8s-node-03 weight 0.049
        item k8s-node-04 weight 0.049
}
# rules
rule replicated_rule {
        id 0
        type replicated
        min_size 1
        max_size 10
        step take ssd
        step chooseleaf firstn 0 type host
        step emit
}
rule replicapool {
        id 1
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-metadata {
        id 2
        type replicated
        min_size 1
        max_size 10
        step take ssd
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-data0 {
        id 3
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.control {
        id 4
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.meta {
        id 5
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.log {
        id 6
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.index {
        id 7
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.non-ec {
        id 8
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule .rgw.root {
        id 9
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.data {
        id 10
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}
# end crush map
```

编译 CRUSH Map

```
[root@k8s-node-01 ~]# crushtool -c crushmap20210413-new.conf -o crushmap20210413-new.bin #把配置文件变成二进制

[root@k8s-node-01 ~]# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
-1         0.34186  root default                                   
-9         0.04880      host k8s-node-04                           
 3    hdd  0.04880          osd.3             up   1.00000  1.00000
-3         0.09769      host k8s-node-05                           
 0    hdd  0.09769          osd.0             up   1.00000  1.00000
-5         0.09769      host k8s-node-06                           
 1    hdd  0.09769          osd.1             up   1.00000  1.00000
-7         0.09769      host k8s-node-07                           
 2    hdd  0.09769          osd.2             up   1.00000  1.00000
 4    hdd        0  osd.4                     up   1.00000  1.00000
 5    hdd        0  osd.5                     up   1.00000  1.00000

[root@k8s-node-01 ~]# ceph -s
  cluster:
    id:     687843e5-dfc7-4e6d-b9a1-6221f3922a7a
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 19h)
    mgr: a(active, since 19h)
    mds: myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
    osd: 6 osds: 6 up (since 18h), 6 in (since 18h)
    rgw: 2 daemons active (my.store.a, my.store.b)
 
  task status:
 
  data:
    pools:   11 pools, 177 pgs
    objects: 2.45k objects, 6.7 GiB
    usage:   27 GiB used, 423 GiB / 450 GiB avail
    pgs:     177 active+clean
 
  io:
    client:   59 KiB/s rd, 28 op/s rd, 0 op/s wr

# 应用新规则
[root@k8s-node-01 ~]# ceph osd setcrushmap -i crushmap20210413-new.bin
23

[root@k8s-node-01 ~]# ceph osd tree
ID   CLASS  WEIGHT   TYPE NAME             STATUS  REWEIGHT  PRI-AFF
-15         0.09799  root ssd                                       
-11         0.04900      host k8s-node-03                           
  4    ssd  0.04900          osd.4             up   1.00000  1.00000
 -9         0.04900      host k8s-node-04                           
  3    ssd  0.04900          osd.3             up   1.00000  1.00000
 -1         0.34297  root hdd                                       
-13         0.04900      host k8s-node-02                           
  5    hdd  0.04900          osd.5             up   1.00000  1.00000
 -3         0.09799      host k8s-node-05                           
  0    hdd  0.09799          osd.0             up   1.00000  1.00000
 -5         0.09799      host k8s-node-06                           
  1    hdd  0.09799          osd.1             up   1.00000  1.00000
 -7         0.09799      host k8s-node-07                           
  2    hdd  0.09799          osd.2             up   1.00000  1.00000

[root@k8s-node-01 ~]# ceph -s
  cluster:
    id:     687843e5-dfc7-4e6d-b9a1-6221f3922a7a
    health: HEALTH_WARN
            Degraded data redundancy: 8525/7359 objects degraded (115.845%), 59 pgs degraded
 
  services:
    mon: 3 daemons, quorum a,b,c (age 19h)
    mgr: a(active, since 19h)
    mds: myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
    osd: 6 osds: 6 up (since 18h), 6 in (since 18h); 131 remapped pgs
    rgw: 2 daemons active (my.store.a, my.store.b)
 
  task status:
 
  data:
    pools:   11 pools, 177 pgs
    objects: 2.45k objects, 6.7 GiB
    usage:   27 GiB used, 423 GiB / 450 GiB avail
    pgs:     8525/7359 objects degraded (115.845%)
             900/7359 objects misplaced (12.230%)
             56 active+recovery_wait+undersized+degraded+remapped
             42 active+clean
             38 active+remapped+backfill_wait
             33 active+clean+remapped
             3  active+recovery_wait+degraded
             3  active+recovery_wait+remapped
             1  active+recovering
             1  active+recovering+undersized+remapped
 
  io:
    client:   1.3 KiB/s rd, 257 B/s wr, 2 op/s rd, 0 op/s wr
    recovery: 78 KiB/s, 1 keys/s, 14 objects/s

 [root@k8s-node-01 ~]# ceph -s
  cluster:
    id:     687843e5-dfc7-4e6d-b9a1-6221f3922a7a
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 20h)
    mgr: a(active, since 20h)
    mds: myfs:1 {0=myfs-a=up:active} 1 up:standby-replay
    osd: 6 osds: 6 up (since 18h), 6 in (since 18h); 33 remapped pgs
    rgw: 2 daemons active (my.store.a, my.store.b)
 
  task status:
 
  data:
    pools:   11 pools, 177 pgs
    objects: 2.45k objects, 6.7 GiB
    usage:   27 GiB used, 423 GiB / 450 GiB avail
    pgs:     29/7359 objects misplaced (0.394%)
             144 active+clean
             33  active+clean+remapped
 
  io:
    client:   5.9 KiB/s rd, 511 B/s wr, 6 op/s rd, 3 op/s wr
 
[root@k8s-node-01 ~]# ceph osd df 
ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA     OMAP     META      AVAIL    %USE  VAR   PGS  STATUS
 4    ssd  0.04900   1.00000   50 GiB  1.1 GiB   84 MiB      0 B     1 GiB   49 GiB  2.17  0.37   33      up
 3    ssd  0.04900   1.00000   50 GiB  1.1 GiB   84 MiB  2.9 MiB  1021 MiB   49 GiB  2.17  0.37   33      up
 5    hdd  0.04900   1.00000   50 GiB  4.3 GiB  3.3 GiB      0 B     1 GiB   46 GiB  8.53  1.44   88      up
 0    hdd  0.09799   1.00000  100 GiB  7.0 GiB  6.0 GiB  3.8 MiB  1020 MiB   93 GiB  6.96  1.18  139      up
 1    hdd  0.09799   1.00000  100 GiB  7.3 GiB  6.3 GiB  1.6 MiB  1022 MiB   93 GiB  7.30  1.24  129      up
 2    hdd  0.09799   1.00000  100 GiB  5.9 GiB  4.9 GiB  1.9 MiB  1022 MiB   94 GiB  5.89  1.00  109      up
                       TOTAL  450 GiB   27 GiB   21 GiB   10 MiB   6.0 GiB  423 GiB  5.90                   
MIN/MAX VAR: 0.37/1.44  STDDEV: 2.51
```



### 3. 定制CRUSH Map注意事项

- 在扩容和删除osd还有编辑curshmap的时候最好备份一个

- 初始化就规划好curshmap的规则，否则应用过程中改变的话，就会有大量的pg迁移

- 重启服务的时候就会自动修改curshmap，所以需要备份，也可以禁用在增加或者修改osd的时候自动分配curshmap 

  （osd crush update on start = false）



## 快照与克隆特性

### 安装控制器和CRD

```shell
# 安装CRD资源
~ git clone https://github.com/kubernetes-csi/external-snapshotter.git
~ kubectl apply -f external-snapshotter/client/config/crd/
customresourcedefinition.apiextensions.k8s.io/volumesnapshotclasses.snapshot.storage.k8s.io created
customresourcedefinition.apiextensions.k8s.io/volumesnapshotcontents.snapshot.storage.k8s.io created
customresourcedefinition.apiextensions.k8s.io/volumesnapshots.snapshot.storage.k8s.io created
~ kubectl get customresourcedefinitions.apiextensions.k8s.io|grep snapshot
volumesnapshotclasses.snapshot.storage.k8s.io                   2021-04-06T07:05:17Z
volumesnapshotcontents.snapshot.storage.k8s.io                  2021-04-06T07:05:17Z
volumesnapshots.snapshot.storage.k8s.io                         2021-04-06T07:05:17Z

# 安装控制器
~ kubectl apply -f external-snapshotter/deploy/kubernetes/snapshot-controller/ 
serviceaccount/snapshot-controller created
clusterrole.rbac.authorization.k8s.io/snapshot-controller-runner created
clusterrolebinding.rbac.authorization.k8s.io/snapshot-controller-role created
role.rbac.authorization.k8s.io/snapshot-controller-leaderelection created
rolebinding.rbac.authorization.k8s.io/snapshot-controller-leaderelection created
deployment.apps/snapshot-controller created

# 查看控制器pods
~ kubectl get pods -n kube-system -l app=snapshot-controller
NAME                                 READY   STATUS             RESTARTS   AGE
snapshot-controller-9f68fdd9-2t578   0/1     ImagePullBackOff   0          37s
snapshot-controller-9f68fdd9-5wnfc   0/1     ImagePullBackOff   0          37s

# 本地下载镜像
~ docker pull k8s.gcr.io/sig-storage/snapshot-controller:v4.0.0                                          
v4.0.0: Pulling from sig-storage/snapshot-controller
e59bd8947ac7: Pull complete 
0810b1707876: Pull complete 
Digest: sha256:00fcc441ea9f72899c25eed61d602272a2a58c5f0014332bdcb5ac24acef08e4
Status: Downloaded newer image for k8s.gcr.io/sig-storage/snapshot-controller:v4.0.0
k8s.gcr.io/sig-storage/snapshot-controller:v4.0.0

# 本地打包镜像
~ docker save -o snapshot-controller.tar k8s.gcr.io/sig-storage/snapshot-controller:v4.0.0

# 上传到node节点
~ scp snapshot-controller.tar root@106.14.4.32:/root/

[root@k8s-node-01 ~]# docker load -i snapshot-controller.tar
[root@k8s-node-01 ~]# for i in k8s-node-0{2..7};do scp snapshot-controller.tar $i:/root/;done
[root@k8s-node-01 ~]# for i in k8s-node-0{2..7};do ssh $i "docker image load -i /root/snapshot-controller.tar";done

~ kubectl get pods -n kube-system -l app=snapshot-controller
NAME                                 READY   STATUS    RESTARTS   AGE
snapshot-controller-9f68fdd9-2t578   1/1     Running   0          14m
snapshot-controller-9f68fdd9-5wnfc   1/1     Running   0          14m
```





