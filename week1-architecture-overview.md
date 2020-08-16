# TIDB架构总览

本周主要讲述了TIDB生态圈、社区相关运营组成结构以及tidb的压测数据、TIKV架构。其中TICDC对接消息队列是我感兴趣的一部分，后续有计划对TICDC的消息队列生态做出贡献，有兴趣的同学欢迎一起交流。

下面是官方的一个架构图:
![architecture](https://github.com/pingcap/tidb/blob/master/docs/architecture.png)

学习了这一节课程可以对TIDB的全局情况有一个大体的了解，B站传送门：[](https://www.bilibili.com/video/BV17K411T7Kd)

TIDB的总体架构用一句话总结来说的话就是：PD调度、TIDB计算、TIKV存储。

目前TIKV是CNCF孵化项目，目前正在走毕业投票流程，期待TIKV的毕业。

# Homework

本次作业描述是这样的：本地下载TIDB，TIKV，PD源代码，改写源代码并编译部署以下环境：
1. 1 TIDB
2. 1 PD
3. 3 TIKV

也就是需要部署一个最小集群，1个TIDB节点，1个PD节点，3个TIKV节点，那就从编译源代码开始了解TIDB吧!

## TIDB编译部署

1. 下载源代码并且编译

首先是下载源代码
`git clone https://github.com/pingcap/tidb.git`

首先官方Repo中没有提到如何去编译TIDB，那就看看源代码库里面有什么内容。

下载后可以看到在源代码目录中有Makefile的编译文件，那自然是直接运行`make`命令先看看是否能够编译通过。

```
[root@normal11 tidb]# make
CGO_ENABLED=1 GO111MODULE=on go build  -tags codes  -ldflags '-X "github.com/pingcap/parser/mysql.TiDBReleaseVersion=v4.0.0-beta.2-948-g148f2d4-dirty" -X "github.com/pingcap/tidb/util/versioninfo.TiDBBuildTS=2020-08-16 06:24:53" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitHash=148f2d456bdc531bb212028f8499ece141e20401" -X "github.com/pingcap/tidb/util/versioninfo.TiDBGitBranch=master" -X "github.com/pingcap/tidb/util/versioninfo.TiDBEdition=Community" ' -o bin/tidb-server tidb-server/main.go

Build TiDB Server successfully!
```

编译好了，在显示的编译内容中看到是编译到了`bin`目录下的`tidb-server`二进制文件,`-o bin/tidb-server`.

可以执行命令的help查看下都有什么参数可以设置：
```
[root@normal11 tidb]# ./bin/tidb-server -h
Usage of ./bin/tidb-server:
  -L string
        log level: info, debug, warn, error, fatal (default "info")
  -P string
        tidb server port (default "4000")
  -V    print version information and exit (default false)
...
  -path string
        tidb storage path (default "/tmp/tidb")
  ...
  -store string
        registered store name, [tikv, mocktikv] (default "mocktikv")
  ...
```

这里主要列了几个常用的，其中store提示默认会使用mocktikv,其实目前默认是使用`unistore`,命令行的提示还没有更新,这几天会计划贡献,如果其他同学看到有兴趣的也可以尝试一下,代码就在`main.go`当中。

我们主要会用到`path`和`store`,在使用tikv进行存储时需要将`store`指定为`tikv`并且`path`参数配置为`tikv`的地址。


我这里的启动参数是这样的：
``
./bin/tidb-server -P 4001 -status 10081 -store tikv -path 192.168.1.11:3379
``

由于需要TIKV的地址,这个部署会在TIKV启动后再进行部署。

2. 改代码使得在TIDB启动事务时，会打印一个"hello transaction"的日志

在看事务相关代码后发现事务的提交是在`txn.go`文件中的Commit(ctx context.Context)方法，再继续往下看发现是调用到了`2pc.go`的execute(ctx context.Context) 方法，所以我选择在这个方法的第一行打印了"hello transaction"。（本周作业）

```
func (c *twoPhaseCommitter) execute(ctx context.Context) (err error) {
	logutil.Logger(ctx).Info("hello transaction")
    ...
```

## PD编译部署

1. 下载源代码并且编译
`git clone https://github.com/pingcap/pd.git`

一样的，看到源码库中有Makefile文件,首先尝试下直接使用`make`进行编译。

编译的时候会发现一直卡在一个文件的下载中(如果网络科学上网应该没问题)：
```
go mod tidy
./scripts/embed-dashboard-ui.sh
+ Clean up existing asset file
+ Fetch TiDB Dashboard Go module
  - TiDB Dashboard directory: /root/gopath/pkg/mod/github.com/pingcap-incubator/tidb-dashboard@v0.0.0-20200807020752-01f0abe88e93
+ Create asset cache directory
+ Discover TiDB Dashboard release version
  - TiDB Dashboard release version: 2020.08.07.1
+ Check embedded assets exists in cache
  - Cached archive does not exist
  - Download pre-built embedded assets from GitHub release
  - Download https://github.com/pingcap-incubator/tidb-dashboard/releases/download/v2020.08.07.1/embedded-assets-golang.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   639  100   639    0     0    120      0  0:00:05  0:00:05 --:--:--   149
  1 8027k    1  101k    0     0   3976      0  0:34:27  0:00:26  0:34:01 11531
```

可以看到这是在进行dashboard相关的工作,但是本次不需要dashboard,所以看看是否能够跳过dashboard。

在Makefile中直接搜索``dashboard``可以找到下面这两段:
```

ifeq ($(DASHBOARD), 0)
	BUILD_TAGS += without_dashboard
else
	BUILD_CGO_ENABLED := 1
endif

...

ifneq ($(DASHBOARD), 0)
	# Note: LDFLAGS must be evaluated lazily for these scripts to work correctly
	LDFLAGS += -X "github.com/pingcap-incubator/tidb-dashboard/pkg/utils/version.InternalVersion=$(shell scripts/describe-dashboard.sh internal-version)"
	LDFLAGS += -X "github.com/pingcap-incubator/tidb-dashboard/pkg/utils/version.Standalone=No"
	LDFLAGS += -X "github.com/pingcap-incubator/tidb-dashboard/pkg/utils/version.PDVersion=$(shell git describe --tags --dirty)"
	LDFLAGS += -X "github.com/pingcap-incubator/tidb-dashboard/pkg/utils/version.BuildTime=$(shell date -u '+%Y-%m-%d %I:%M:%S')"
	LDFLAGS += -X "github.com/pingcap-incubator/tidb-dashboard/pkg/utils/version.BuildGitHash=$(shell scripts/describe-dashboard.sh git-hash)"
endif
```

看起来是``DASHBOARD``环境变量为0则不进行dashboard的编译工作，来尝试一下：
```
[root@normal11 pd]# DASHBOARD=0 make
...
go mod tidy
./scripts/embed-dashboard-ui.sh
+ Skip TiDB Dashboard
CGO_ENABLED=0 go build  -gcflags '' -ldflags '-X "github.com/pingcap/pd/v4/server/versioninfo.PDReleaseVersion=v4.0.0-rc.2-134-g68d2ba6" -X "github.com/pingcap/pd/v4/server/versioninfo.PDBuildTS=2020-08-16 06:45:41" -X "github.com/pingcap/pd/v4/server/versioninfo.PDGitHash=68d2ba6b0e82f17231f5af8700b822a5f7eb4c98" -X "github.com/pingcap/pd/v4/server/versioninfo.PDGitBranch=master" -X "github.com/pingcap/pd/v4/server/versioninfo.PDEdition=Community"' -tags " swagger_server without_dashboard" -o bin/pd-server cmd/pd-server/main.go
CGO_ENABLED=0 go build -gcflags '' -ldflags '-X "github.com/pingcap/pd/v4/server/versioninfo.PDReleaseVersion=v4.0.0-rc.2-134-g68d2ba6" -X "github.com/pingcap/pd/v4/server/versioninfo.PDBuildTS=2020-08-16 06:45:50" -X "github.com/pingcap/pd/v4/server/versioninfo.PDGitHash=68d2ba6b0e82f17231f5af8700b822a5f7eb4c98" -X "github.com/pingcap/pd/v4/server/versioninfo.PDGitBranch=master" -X "github.com/pingcap/pd/v4/server/versioninfo.PDEdition=Community"' -o bin/pd-ctl tools/pd-ctl/main.go
CGO_ENABLED=0 go build -gcflags '' -ldflags '-X "github.com/pingcap/pd/v4/server/versioninfo.PDReleaseVersion=v4.0.0-rc.2-134-g68d2ba6" -X "github.com/pingcap/pd/v4/server/versioninfo.PDBuildTS=2020-08-16 06:45:54" -X "github.com/pingcap/pd/v4/server/versioninfo.PDGitHash=68d2ba6b0e82f17231f5af8700b822a5f7eb4c98" -X "github.com/pingcap/pd/v4/server/versioninfo.PDGitBranch=master" -X "github.com/pingcap/pd/v4/server/versioninfo.PDEdition=Community"' -o bin/pd-recover tools/pd-recover/main.go
```

编译成功了，看看都编译了什么内容：
```
[root@normal11 pd]# ls bin
pd-ctl  pd-recover  pd-server
```

可以看到有三个二进制文件，我们需要的是`pd-server`.

一样的 可以执行下help看看有什么参数,这里就不在演示了。

我这里使用的启动参数是:

```
 ./bin/pd-server -client-urls http://192.168.1.11:3379 -data-dir ./data
```

## TIKV编译部署 

TIKV的编译算是最不容易搞定的部分了，主要是两部分：
1. Rust
2. git子模块(网络慢)

由于之前有过Rust的Hello World经验，编译Rust对我来说问题不大，git子模块慢还没有找到什么好办法，只能等待。

1. 下载源代码并编译

和之前的一样，看到Makefile首先执行make看看有没有问题。

在编译的过程中回去拉取git 子模块的代码，这部分很慢，我就不再演示了(不想再经历)。

```
[root@normal11 tikv]# make 
cargo build --release --no-default-features --features " jemalloc portable sse protobuf-codec"
   Compiling libc v0.2.66
   Compiling cfg-if v0.1.10
   Compiling log v0.4.8
   ...
   Building [===========================================>           ] 431/531: procfs, backtrace-sys(bui..
```

最下面一行会提示当前编译的进度，你可以根据这个进度来计算自己可以喝几杯咖啡，这就是这个进度条的用处。

编译好后看看都有什么产物：
```
[root@normal11 tikv]# ls target/release/
build  deps  examples  incremental  libcmd.d  libcmd.rlib  tikv-ctl  tikv-ctl.d  tikv-server  tikv-server.d
```

可以看到有`tikv-ctl`和`tikv-server`,我们部署需要的是`tikv-server`.

惯例，首先看一下help了解下都有什么参数可以设置。就不再掩饰了。 展示下我部署3个节点的tikv的启动参数：
```
target/debug/tikv-server --pd-endpoints http://192.168.1.11:3379 -A 192.168.1.11:20161 -f tikv.log --data-dir data

target/release/tikv-server --pd-endpoints http://192.168.1.11:3379 --status-addr 192.1.1.11:20152 -A 192.168.1.11:20162 -f tikv2.log --data-dir data2 

target/release/tikv-server --pd-endpoints http://192.168.1.11:3379  --status-addr 192.8.1.11:20153 -A 192.168.1.11:20163 -f tikv3.log --data-dir data3
```


由于我没有使用dashboard，需要看TIKV集群的信息可以通过`pd-ctl`查看:
```
[root@normal11 tikv]# ../pd/bin/pd-ctl store  -u http://192.168.1.11:3379 
{
  "count": 3,
  "stores": [
    {
      "store": {
        "id": 1,
        "address": "192.168.1.11:20161",
        "version": "4.1.0-alpha",
        "status_address": "127.0.0.1:20180",
        "git_hash": "df2fab45b19b5d067ff55ff5074819c50eef3ea2",
        "start_timestamp": 1597565907,
        "deploy_path": "/root/repo/tikv/target/release",
        "last_heartbeat": 1597567039153289831,
        "state_name": "Up"
      },
      "status": {
        "capacity": "49.98GiB",
        "available": "10.43GiB",
        "used_size": "32.56MiB",
        "leader_count": 14,
        "leader_weight": 1,
        "leader_score": 14,
        "leader_size": 14,
        "region_count": 22,
        "region_weight": 1,
        "region_score": 979224935.3087239,
        "region_size": 22,
        "start_ts": "2020-08-16T16:18:27+08:00",
        "last_heartbeat_ts": "2020-08-16T16:37:19.153289831+08:00",
        "uptime": "18m52.153289831s"
      }
    },
    {
      "store": {
        "id": 46,
        "address": "192.168.1.11:20163",
        "version": "4.1.0-alpha",
        "status_address": "192.168.1.11:20153",
        "git_hash": "df2fab45b19b5d067ff55ff5074819c50eef3ea2",
        "start_timestamp": 1597566768,
        "deploy_path": "/root/repo/tikv/target/release",
        "last_heartbeat": 1597567038217172506,
        "state_name": "Up"
      },
      "status": {
        "capacity": "49.98GiB",
        "available": "10.43GiB",
        "used_size": "31.5MiB",
        "leader_count": 4,
        "leader_weight": 1,
        "leader_score": 4,
        "leader_size": 4,
        "region_count": 22,
        "region_weight": 1,
        "region_score": 979224925.9906679,
        "region_size": 22,
        "start_ts": "2020-08-16T16:32:48+08:00",
        "last_heartbeat_ts": "2020-08-16T16:37:18.217172506+08:00",
        "uptime": "4m30.217172506s"
      }
    },
    {
      "store": {
        "id": 47,
        "address": "192.168.1.11:20162",
        "version": "4.1.0-alpha",
        "status_address": "192.168.1.11:20152",
        "git_hash": "df2fab45b19b5d067ff55ff5074819c50eef3ea2",
        "start_timestamp": 1597566769,
        "deploy_path": "/root/repo/tikv/target/release",
        "last_heartbeat": 1597567039620646027,
        "state_name": "Up"
      },
      "status": {
        "capacity": "49.98GiB",
        "available": "10.3GiB",
        "used_size": "31.65MiB",
        "leader_count": 4,
        "leader_weight": 1,
        "leader_score": 4,
        "leader_size": 4,
        "region_count": 22,
        "region_weight": 1,
        "region_score": 1008811475.6201195,
        "region_size": 22,
        "start_ts": "2020-08-16T16:32:49+08:00",
        "last_heartbeat_ts": "2020-08-16T16:37:19.620646027+08:00",
        "uptime": "4m30.620646027s"
      }
    }
  ]
}

```

可以看到三个节点都是`UP`的状态,接下来开始测试一下事务的内容：

我创建了一个`person`的表,一个name字段和money字段.

[table](https://res.cloudinary.com/lyp/image/upload/v1597567480/tidb/table.jpg)

插入一行数据查看事务的效果：

执行的语句如下：
```
begin;
insert into person (name,money) values ('tidb',10);
commit;
```

会在执行commit;的时候打印一句`hello transaction`.

```
[2020/08/16 16:48:57.336 +08:00] [INFO] [2pc.go:726] ["hello transaction"] [conn=2]
```

以上就是本次作业的效果：
1. 搭建一个最小的集群
2. 修改源代码，在提交事务的时候打印一个`hello transaction`

其实这里打印的地方可能是不对的，这是在提交事务的时候才会打印，也就是执行`commit;`的时候，而作业内容是`开始事务时`也就是`begin;`时,不过我没有更多时间再看了，所以提交的作业就写在了`execute`时打印. :)

欢迎感兴趣的同学一起交流，
我的知乎地址是：[LanLiang](https://www.zhihu.com/people/liangyuanpeng),后续会更新在发布的文章中。