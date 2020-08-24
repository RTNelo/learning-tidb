# 准备环境

用 VirtualBox 开一个 Ubuntu Server 20.04 LTS，以保持一个干净的环境。
然后 `apt install build-essential` 感觉以后会用得上。

# 构建

## TiKV

参考 https://github.com/tikv/tikv/blob/master/CONTRIBUTING.md

### 安装依赖

#### rustup
参考 https://www.rust-lang.org/tools/install 来安装 rustup
`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

因为国内网络的问题，可以考虑使用代理。

#### 其他

`apt install git make cmake`

### 获取代码

首先获取已发布版本中最新的 `v4.0.4` 。

```
git clone https://github.com/tikv/tikv.git -b v4.0.4 -depth 1
cd tikv
```

根据文档的指导添加依赖（其实在安装 rustup 时已经一同装好了）

```
rustup component add rustfmt
rustup component add clippy
```

### 构建
`make build`


## TiDB

### 安装依赖
TiDB 依赖 Go 1.13 以上版本，官方源中的软件包正好满足要求
`apt install golang-go`

### 获取代码

首先获取已发布版本中最新的 `v4.0.4` 。

```
git clone https://github.com/pingcap/tidb.git -b v4.0.4 -depth 1
cd tidb
```

### 构建
`make`

## PD

### 安装依赖
除了文档中提到的依赖，PD 的构建其实还依赖了 unzip。
`apt install unzip`

### 获取代码

首先获取已发布版本中最新的 `v4.0.4` 。

```
git clone https://github.com/pingcap/pd.git -b v4.0.4 -depth 1
cd pd
```

### 构建
`make`

# 部署

## 启动
参考文档给出的命令（需要调整可执行文件的路径）

```
$PD_SERVER --name=pd1 \ 
                --data-dir=pd1 \ 
                --client-urls="http://127.0.0.1:2379" \ 
                --peer-urls="http://127.0.0.1:2380" \ 
                --initial-cluster="pd1=http://127.0.0.1:2380" \ 
                --log-file=pd1.log & 
$TIKV_SERVER ./bin/tikv-server --pd-endpoints="127.0.0.1:2379" \ 
                --addr="127.0.0.1:20160" \ 
                --data-dir=tikv1 \ 
                --log-file=tikv1.log & 
$TIKV_SERVER --pd-endpoints="127.0.0.1:2379" \ 
                --addr="127.0.0.1:20161" \ 
                --data-dir=tikv2 \ 
                --log-file=tikv2.log & 
$TIKV_SERVER --pd-endpoints="127.0.0.1:2379" \ 
                --addr="127.0.0.1:20162" \ 
                --data-dir=tikv3 \ 
                --log-file=tikv3.log &     
$TIDB_SERVER --store=tikv  --path="127.0.0.1:2379" --log-file=tidb1.log & 
```

## 测试
首先安装 mycli （比 mysql-client 好用）
`apt install mycli`

以后可以用 mycli 创建数据库和表。

# 源码修改

在 TiDB 启动事务时输出内容为 `hello transaction` 的日志。

## 寻找逻辑

参考 TiDB 的官方博文，了解 TiDB 的源码结构。
在代码中搜索 `Transaction` 字样，发现事务统一来自于

`func (s *tikvStore) Begin() (kv.Transaction, error)`

那么可以在函数中打印日志了：
``logutil.BgLogger().Info("hello transaction")

## 验证
观察日志，会发现 `hello transaction` 在客户端开启事务外，也会周期性地出现。
使用 debug 包打印调用盏，会发现 TiDB 在周期性地执行后台任务。
