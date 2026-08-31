# pcapsipdump (中文文档)

`pcapsipdump` 是一个用于抓取 SIP 会话（并在可用时包含关联的 RTP/RTCP/T.38 媒体流）并转储到磁盘的工具。其存储格式与 `tcpdump -w` 完全相同（标准 pcap 格式），但会**按 SIP 会话将数据包拆分为独立的文件**（即便同时存在数千个并发通话）。

此外，`pcapsipdump` 也可以用于将包含多个通话的“大容量” pcap 文件离线拆分为每个通话单独的 pcap 文件。

---

## 核心特性

- **按通话独立抓包**：每个 SIP 会话单独生成一个 `.pcap` 文件。
- **自动关联媒体流**：解析 SDP 协议，自动提取并抓取通话关联的 RTP、RTCP 及 T.38 媒体流。
- **结构化目录存储**：按时间层级组织输出文件（`年/月/日/时/年月日-时分秒-主叫-被叫-CallID.pcap`）。
- **多种过滤策略**：支持按电话号码（支持正则表达式）、T.38 传真负载、RTP 类型进行精准过滤。
- **离线拆包能力**：支持读取已有 pcap 文件并按通话进行拆分导出。

---

## 编译与安装

### 依赖环境

- `libpcap` 开发库（CentOS/RHEL: `libpcap-devel`，Debian/Ubuntu: `libpcap-dev`，macOS: 系统自带或 `brew install libpcap`）
- 标准 C++ 编译器（`g++` / `clang++`）与 `make`

### 编译命令

1. **默认编译**：
   ```bash
   make
   ```

2. **启用号码正则表达式过滤支持**：
   ```bash
   make DEFS=-DUSE_REGEXP
   ```

3. **编译调试版本（带 `-ggdb`）**：
   ```bash
   make pcapsipdump-debug
   ```

4. **清理构建产物**：
   ```bash
   make clean
   ```

5. **安装到系统**：
   ```bash
   make install [DESTDIR=/path]
   ```

---

## 使用方法

### 命令格式
```bash
pcapsipdump [-fpU] [-i <网卡名称>] [-r <pcap文件>] [-d <输出目录>] [-v 级别] [-R 过滤类型] [-n 号码] [-t] [-P <端口>]
```

### 参数说明

| 参数 | 说明 |
| :--- | :--- |
| `-i <interface>` | 指定实时监听的网络接口/网卡名称。 |
| `-r <file>` | 从指定的 pcap 文件中读取数据包（离线模式），而不是实时抓包。 |
| `-d <directory>` | 指定抓包文件的输出保存目录（默认路径：`/var/spool/pcapsipdump`）。 |
| `-f` | 前台运行（不 fork 到后台守护进程，不脱离控制终端）。 |
| `-p` | 不将网卡设置为混杂模式（Non-promiscuous mode）。 |
| `-U` | 启用数据包缓冲写入（Packet-buffered）。虽然性能稍低，但在文件写入未结束时即可随时打开读取且格式完整。 |
| `-P <port>`, `--port <port>` | 指定监听的 SIP 端口（默认值：`5060`）。 |
| `-v <level>` | 设置日志详细级别（数值越高输出越详细）。 |
| `-n <number>` | 号码过滤：仅记录主叫或被叫匹配指定号码的通话（若编译时指定 `DEFS=-DUSE_REGEXP`，此处支持正则）。 |
| `-t` | T.38 过滤：仅记录在 SDP 中声明包含 T.38 载荷的传真通话。 |
| `-R <filter>` | RTP 过滤模式，可选值包括：<br>• `rtp+rtcp`（默认）：同时记录 RTP 和 RTCP 数据包<br>• `rtp`：仅记录 RTP 数据包<br>• `rtpevent`：仅记录命名事件（DTMF等）<br>• `t38`：仅记录 T.38 流量<br>• `none`：不记录任何媒体流，仅记录 SIP 信令 |

---

## 典型示例

### 1. 实时监听并后台捕获所有 SIP/RTP 流量
```bash
pcapsipdump -i eth0 -d /var/spool/pcapsipdump
```

### 2. 指定自定义 SIP 端口与前台运行
```bash
pcapsipdump -f -i eth0 -P 5080 -d /tmp/sip_dumps
```

### 3. 过滤指定号码的通话

```bash
pcapsipdump -f -i eth0 -n "1001" -d /tmp/sip_dumps
```

### 4. 离线拆分已抓取的大 pcap 文件
```bash
pcapsipdump -r full_capture.pcap -d /tmp/split_calls
```

### 5. 仅记录 SIP 信令（不记录 RTP）
```bash
pcapsipdump -i eth0 -R none -d /var/spool/pcapsipdump
```

---

## 输出目录与文件命名格式

抓包文件会按照时间分层保存到 `-d` 指定的目录中，路径格式如下：
```text
<输出目录>/YYYY/MM/DD/HH/YYYYMMDD-HHMMSS-[主叫]-[被叫]-[Call-ID].pcap
```
例如：
```text
/var/spool/pcapsipdump/2026/09/01/14/20260901-143000-1001-1002-abc123xyz@192.168.1.1.pcap
```

---

## 操作系统包说明

- **RedHat / CentOS / Fedora**: 参见 [redhat/](redhat/) 目录下的 spec 文件和 sysconfig 配置。
- **Debian / Ubuntu**: 参见 [debian/](debian/) 目录下的 init 脚本和配置。
