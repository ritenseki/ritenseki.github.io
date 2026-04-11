+++
date = '2026-04-11T23:35:24+08:00'
draft = false
title = 'Minecraft Server Jamming Because of an Analytics Mod'
+++

## 设备

- Rainyun 云服务器
- Better MC 5 整合包 Neoforge Version

## 现象

在 `/stop` 服务器的时候会在某个环节卡住。

```log
[15:44:19] [Server thread/INFO] [minecraft/MinecraftServer]: Stopping the server
[15:44:19] [Server thread/INFO] [co.ya.al.ma.PluginManager//]: Deregistering server plugin data...
[15:44:19] [Server thread/INFO] [co.ya.al.ma.PluginManager//]: Deregistering server plugin data finished
[15:45:20] [spark-async-sampler-worker-thread/WARN] [spark/]: Timed out waiting for world statistics
[15:46:20] [spark-async-sampler-worker-thread/WARN] [spark/]: Timed out waiting for world statistics
[15:47:20] [spark-async-sampler-worker-thread/WARN] [spark/]: Timed out waiting for world statistics
```

## 排查过程

作为新手腐竹的我选择寻求了 LLM 的帮助。

一开始，据上面的 log ，初步的结论是 Spark 的性能检测模块卡住了；

后面试着 disabled Spark 依旧没能解决问题；

多次重现后我从 MobaXterm 的性能监视快捷窗口发现 CPU 占用不是很高，我觉得不是死锁一类的 bug ，LLM 认为可能是线程阻塞；

于是用 jstack 生成了 dump.

```bash
# 找到 Java 进程 PID
jps -l | grep server.jar
# 或
pgrep -f "server.jar"

# 生成线程 dump（假设 PID 是 12345）
jstack 12345 > threaddump.txt

# 查看卡住的线程
grep -A 5 "BLOCKED\|WAITING\|TIMED_WAITING" threaddump.txt | head -50
```

## 核心原因 & 修复

Server 线程被一个 HTTP 请求阻塞住了。

```log
"Server thread" #87 [246711] ... RUNNABLE
	at toni.packanalytics.PackAnalytics.sendKeepAliveRequest(packanalytics@1.0.5/PackAnalytics.java:169)
	at toni.packanalytics.PackAnalytics.lambda$onInitialize$1(packanalytics@1.0.5/PackAnalytics.java:105)
	at net.fabricmc.fabric.api.event.lifecycle.v1.ServerLifecycleEvents.lambda$static$4(...)
	at net.neoforged.neoforge.server.ServerLifecycleHooks.handleServerStopping(...)
```

PackAnalytics 模组在服务器停止时发送 "keep alive" 请求，但这个 HTTP 请求阻塞了主线程。

这是一个**同步网络请求**，服务器停止时卡住等待网络响应。PackAnalytics 在服务器停止时发 HTTP 请求，同步等待响应，导致主线程阻塞。

这个 mod 看起来是用来统计 Better MC 整合包下载安装量的，对游戏内容无影响，所以选择 disabled。

## 技术栈积累

服务器的主线程就叫 **"Server thread"**，所有游戏逻辑都在这里跑。stop 命令也是这个线程执行的。

```
"Server thread" #87 [246711] prio=5 os_prio=0 cpu=5389.62ms elapsed=95.80s tid=0x00007ca80d001db0 nid=246711 runnable
```

### 调用栈 Stack Trace

线程状态下面的 **at ...** 列表，就是从最底层到最上层的调用链。

```
at java.lang.Thread.run(java.base@25.0.2/Thread.java:1474)
```
→ 线程启动入口

```
at net.minecraft.server.MinecraftServer.lambda$spin$2(minecraft@1.21.1/MinecraftServer.java:267)
at net.minecraft.server.MinecraftServer.runServer(minecraft@1.21.1/MinecraftServer.java:724)
```
→ Minecraft 服务器主循环

```
at net.neoforged.neoforge.server.ServerLifecycleHooks.handleServerStopping(...)
```
→ **NeoForge 的"服务器正在停止"事件**

```
at net.fabricmc.fabric.api.event.lifecycle.v1.ServerLifecycleEvents...
```
→ **Fabric API 的事件转发**（NeoForge 用了 Fabric 兼容层）

```
at toni.packanalytics.PackAnalytics.lambda$onInitialize$1(packanalytics@1.0.5/PackAnalytics.java:105)
at toni.packanalytics.PackAnalytics.sendKeepAliveRequest(packanalytics@1.0.5/PackAnalytics.java:169)
```
→ **PackAnalytics 模组发送 keep alive 请求**

```
at sun.net.www.protocol.https.HttpsURLConnectionImpl.getResponseCode(...)
at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(...)
at sun.nio.ch.NioSocketImpl.read(...)
at sun.nio.ch.SocketDispatcher.read0(java.base@25.0.2/Native Method)
```
→ **Java 原生网络 I/O，正在等服务器响应**

### 阻塞点

关键特征：

| 特征 | 说明 |
|-----|------|
| `RUNNABLE` 状态 | 线程没在 sleep，是真的在"跑" |
| `SocketDispatcher.read0` | 卡在底层 socket 读操作 |
| `HttpsURLConnection.getResponseCode` | 正在等 HTTP 响应 |
| `elapsed=95.80s` | 这个线程已经跑了 95 秒，说明 stop 后一直卡在这里 |

### 对比正常线程 & 问题线程

#### 正常
```
"Timer hack thread" #50 ... TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(...)
```
→ 明确标记 `sleeping`，线程主动休息

#### 问题
```
"Server thread" #87 ... RUNNABLE
	at sun.nio.ch.SocketDispatcher.read0(...)
```
→ 标记 `RUNNABLE`，但卡在 I/O，这是**阻塞式网络请求**

### 快速定位技巧

> 以后看 dump，直接搜这几个关键词：

```bash
# 1. 找 Server thread
grep -A 20 '"Server thread"' threaddump.txt

# 2. 找 BLOCKED 或 RUNNABLE 但卡在 I/O 的
grep -B 2 -A 10 "java.net\|sun.nio\|Socket" threaddump.txt

# 3. 找模组（看包名）
grep "toni\|packanalytics\|c2me\|modernfix" threaddump.txt
```

## 总结判断流程

1. 找到 "Server thread"
2. 从下往上读调用栈
3. 找到第一个非 Minecraft/Forge/Java 标准的包名
4. 看是否在执行 I/O 操作（网络、文件）
5. 最终定位

本次案例：
```
Server thread → handleServerStopping → PackAnalytics.sendKeepAliveRequest → HttpsURLConnection.getResponseCode → Socket read (卡住！)
```
