# Milvus cloud 部署设计方案

## 总体原则
1. 为每个用户创建各自的 aws 实例，用户间不共享 aws 实例
2. 创建 aws 实例需要提供两个参数: 实例类型、磁盘大小； cpu、gpu、memory 由实例类型决定
3. 不支持动态更改 aws 实例类型
4. 直接对用户暴露 aws 实例
5. 用户必须首先设置访问白名单，才能访问 aws 上的 miluvs 实例
6. miluvs 客户端不需要做任何修改
7. username 放在 http 请求的 header 内

## 创建 miluvs 实例
`request`
```json
{
    "milvus-name": "search-cats",
    "milvus-type" : "M5",
    "disk-size" : 50
}
```

使用 `HTTP` 的返回码表示当前请求是否成功，返回码在 `HTTP` 请求的头部

`response` on success
```json
{
    "milvs-name" : "search-cats",
    "milvus-url" : "ec2-52-82-52-246.cn-northwest-1.compute.amazonaws.com.cn",
    "milvus-port" : "19530",
}
```

`response` on failed
```json
{
    "reason" : "the reason of failed",
    "description" : "can't create aws instances"
}
```

mongodb 中存储的数据
```json
{
    "username" : "milvus-test",
    "milvus-name" : "search-cats",
    "milvus-url" : "ec2-52-82-52-246.cn-northwest-1.compute.amazonaws.com.cn",
    "milvus-port" : "19530",
    "milvus-type" : "M5",
    "subnet-id" : "subnet-64b1090d",
    "security-groups" : [
        {
            "group-name" : "sg-base",
        },
        {
            "group-name" : "sg-miluvs-test"
        }
    ],
    "instances" :[
        {
            "node-type" : "master",
            "instance-id" : "i-00879b91cf475a597",
            "status" : "starting",
            "key-pair" : "zilliz-hz02",
            "public-ip" : "52.82.52.246",
            "private-ip" : "172.31.9.114",
        },
        {
            "node-type" : "worker",
            "instance-id" : "i-0102e93a4413df1ba",
            "status" : "loading",
            "key-pair" : "zilliz-hz02",
            "public-ip" : "3.12.47.200",
            "private-ip" : "172.31.9.115",
        }
    ]
}
```

后台工作：
1. 创建 aws ec2 实例
2. 每个 aws ec2 实例和 disk 至少包含 4 个 tag : "username", "milvus-name", "miluse-username", "node-type"
3. miluvs 集群的所有机器使用同一个 "subnet-id" 和 "security-groups"
4. "sg-base" 内部包含基础的安全规则，比如允许控制台机器远程登录
5. "sg-miluvs-test" 内部放置用户设置的，允许方位 miluvs 实例的机器 ip 地址白名单
6. instances.status 一共有四中状态: `starting`, 启动 aws 实例； `loading`, 安装 miluvs; `running`, 安装完成，可以对外提供服务; `failed` : 实例启动失败，需要被删除


## 删除 milvus 实例

`request`
```json
{
    "milvus-name": "search-cats",
}
```

使用 `HTTP` 的返回码表示当前请求是否成功，返回码在 `HTTP` 请求的头部

`response` on success
```json
{
    "milvs-name" : "search-cats",
    "milvus-url" : "ec2-52-82-52-246.cn-northwest-1.compute.amazonaws.com.cn",
    "milvus-port" : "19530",
}
```

`response` on failed
```json
{
    "reason" : "the reason of failed",
    "description" : "can't create aws instances"
}
```

## 设置白名单
白名单的规则放置在 "sg-miluvs-test" 内
`request`
```json
{
    "milvus-name": "search-cats",
    "ip-ranges" : [
        {
            "source-ip" : "115.236.166.154/32",
            "port-type" : "miluvs-port"
        }

        {
            "source-ip" : "115.236.166.154/32",
            "port-type" : "miluvs-em-port"
        }
    ]
}
```
"port-type" 设置向用户开放的端口，可选值为 "miluvs-port"

使用 `HTTP` 的返回码表示当前请求是否成功，返回码在 `HTTP` 请求的头部


`response` on success
```json
{
    "ip-ranges" : [
        {
            "source-ip" : "115.236.166.154/32",
            "port-type" : "miluvs-port"
        }

        {
            "source-ip" : "115.236.166.154/32",
            "port-type" : "miluvs-em-port"
        }
    ]
}
```

`response` on faied
```json
{
    "reason" : "the reason of failed",
    "description" : "can't create aws instances"
}
```

## 获得 ip 地址白名单

`request`
```json
{
    "milvs-name" : "search-cats"
}
```