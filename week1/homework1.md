# Homework 1

## 基本信息

- Week 1
- 日期：2020 年 8 月 14 日
- 姓名：何田丰
- 环境：macOS Catalina

## 题目描述

本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境:

- 1 TiDB
- 1 PD
- 3 TiKV

改写后

- 使得 TiDB 启动事务时，会打一个 “hello transaction” 的日志

## 第一步：源码编译

下载 TiDB 源码并编译(已安装 go, make)

```bash
git clone https://github.com/pingcap/tidb.git
cd tidb
make
```
**注：本机不明原因报错：*
```
flag provided but not defined: -L/usr/local/opt/llvm/lib
```
*将环境变量 LDFLAGS 从*
```
LDFLAGS=-L/usr/local/opt/llvm/lib
```
*改为(-L 后增加空格)*
```
LDFLAGS=-L /usr/local/opt/llvm/lib
```
*即可成功编译*

下载 TiKV 源码并编译(已安装 rustup, cmake, make)

```bash
git clone https://github.com/tikv/tikv.git
rustup component add rustfmt
rustup component add clippy
cd tikv
make build
```

下载 PD 源码并编译(已安装 go, make)

```bash
git clone https://github.com/pingcap/pd.git
cd pd
make
```

## 第二步：运行集群

运行 1 个 PD

```bash
cd pd
./bin/pd-server --name=pd1 \
                --data-dir=pd1 \
                --client-urls="http://127.0.0.1:2379" \
                --peer-urls="http://127.0.0.1:2380" \
                --initial-cluster="pd1=http://127.0.0.1:2380" \
                --log-file=pd1.log
```

调整系统最大可以打开的文件数量，TiKV 实例要求该数量不小于 82920

```bash
sudo launchctl limit maxfiles 82920
```

运行 3 个 TiKV

```bash
cd tikv
./target/debug/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20160" \
                --data-dir=tikv1 \
                --log-file=tikv1.log

./target/debug/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20161" \
                --data-dir=tikv2 \
                --log-file=tikv2.log

./target/debug/tikv-server --pd-endpoints="127.0.0.1:2379" \
                --addr="127.0.0.1:20162" \
                --data-dir=tikv3 \
                --log-file=tikv3.log
```

运行 1 个 TiDB

```bash
cd tidb
./bin/tidb-server --store=tikv --path="127.0.0.1:2379"
```

用 mysql 客户端连接集群

```bash
mysql -h 127.0.0.1 -P 4000 -u root -D test
```

测试查询

```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| INFORMATION_SCHEMA |
| METRICS_SCHEMA     |
| PERFORMANCE_SCHEMA |
| mysql              |
| test               |
+--------------------+
5 rows in set (0.00 sec)

```

## 第三步：修改 Transaction

打开 kv/txn.go，在函数 RunInNewTxn 函数中加入

```go
func RunInNewTxn(store Storage, retryable bool, f func(txn Transaction) error) error {
	logutil.BgLogger().Info("hello transaction")
    ...
}
```

重启 TiDB 实例，即可看到结果

```bash
...
[2020/08/14 18:03:56.878 +08:00] [INFO] [server.go:235] ["server is running MySQL protocol"] [addr=0.0.0.0:4000]
[2020/08/14 18:03:56.878 +08:00] [INFO] [http_status.go:82] ["for status and metrics report"] ["listening on addr"=0.0.0.0:10080]
[2020/08/14 18:03:57.435 +08:00] [INFO] [txn.go:30] ["hello transaction"]
[2020/08/14 18:03:57.435 +08:00] [INFO] [txn.go:30] ["hello transaction"]
[2020/08/14 18:03:58.435 +08:00] [INFO] [txn.go:30] ["hello transaction"]
[2020/08/14 18:03:58.435 +08:00] [INFO] [txn.go:30] ["hello transaction"]
...
```
