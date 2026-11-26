# ESP32 Wi-Fi 中继

基于 ESP32（S3 / C3 / C6）的 Wi-Fi 中继：STA 连接上游路由器，SoftAP 给下游设备上网（NAPT），
附带网页配置、DNS 中继与自定义解析，以及一个同时支持 HTTP/2 和 HTTP/1.1 的 HTTPS 站点。

> 配置页没有登录，也不加密传输；Wi-Fi 密码与上传的私钥以明文存放在 Flash 中。
> 只建议在自己可控的网络中使用。

## 功能

- **Wi-Fi 中继**：APSTA 模式 + lwIP NAPT；上游断线自动重连
- **网页配置**（`http://192.168.4.1/`）
  - 上游 Wi-Fi：扫描附近网络（按信号强度排序）后点选
  - 本机热点：SSID、密码、最大客户端数；保存到 NVS，无需重启即生效
  - 运行状态与实时日志（WebSocket，连接时补发历史）
- **DNS 中继**：监听 `192.168.4.1:53`，转发到上游 DNS（客户端何时连入都能解析）
- **DNS 自定义解析**：最多 16 个域名，A 记录返回指定 IPv4（留空为 AP 地址），
  AAAA 等其他类型回空应答；未配置的域名转发上游
- **HTTPS 站点（端口 443）**
  - 自研组件 `components/h2_server`：esp-tls 服务端 + ALPN + [nghttp2](https://github.com/nghttp2/nghttp2)
  - 同时支持 HTTP/2 与 HTTP/1.1（持久连接、流水线）
  - 域名、证书链、私钥可在网页上修改，热替换，新连接生效
  - 客户端证书（mTLS）默认忽略
- **HTTP/2 自定义接口**：最多 8 条「路径 → JSON」
  - 匹配不区分方法与大小写，忽略结尾斜杠和查询串；路径在请求路径中任意位置整段出现即命中
    （设置 `/abc` 时 `/abc`、`/a/abc`、`/abc/x` 都命中），多条命中取最长
  - 应答：`Content-Type: application/json`、`X-Checksum: <md5(body)>`、`Cache-Control: no-store`，
    正文为压缩 JSON
  - 命中时实时日志高亮并弹出提示
- **板载灯**（WS2812，默认 GPIO48，`CONFIG_REPEATER_LED_GPIO`）：`GET /api/led` 查看；
  `POST /api/led` 带 `on=1|0` 或 `r=&g=&b=`（0-255），可带 `gpio=` 临时切换数据引脚；
  `blink=<秒>` 彩色闪烁（0 停止）。自定义接口命中时自动彩色闪烁 10 秒（闪烁中再次命中重新计时），结束后恢复原来的颜色
- **车机联网检测的本机应答**（概览页「车机专用功能」里的开关，默认开启，存 NVS，见 `main/car_probe.h`）：特斯拉车机连上热点后发
  `GET http://connman.vn.cloud.tesla.cn/online/status.html` 和 `HEAD http://<www.baidu.cn 的 IP>/`（后者每 10 分钟重复）。
  开启后这两个域名解析到 `203.0.113.80`（优先于 DNS 黑名单，低于自定义解析；车机不连内网地址，所以本机另挂了这个别名地址，
  文档保留网段、公网上不存在，本机的 80 / 443 服务在这个地址上同样可达），本机 80 端口回 200 + `X-ConnMan-Status: online`
  与 403（同百度的真实响应），上游没连上时车机也认为热点能上网。`POST /api/carprobe` 带 `enabled=1|0`，状态在 `/api/status` 的 `car_probe`
- **恢复语音（临时 TLS 透传）**（概览页「车机专用功能」里的「恢复语音」开关，默认关闭，窗口秒数可配，存 NVS，见 `main/h2_pass.h`）：站点域名被解析到本机后，
  个别接口必须由真实服务器处理（车机语音要用 `/mothership/vehicles/<VIN>/jwt_token` 换令牌）。开启后，HTTPS 站点收到命中该路径的请求时
  不应答并关闭连接，此后 N 秒（默认 3）内 443 端口的新连接原样转发到站点域名的真实服务器，窗口结束后恢复由本机处理；
  路径同时命中自定义接口时仍由本机应答。`GET / POST /api/h2pass`（`enabled=1|0&secs=1..60`）
- **车机浏览器打开配置页**：`esp.lan` 始终解析到 `203.0.113.80`（车机浏览器不连内网地址），连上热点后打开 `http://esp.lan/`；
  自定义解析里有同名条目时以自定义为准
- **SNTP 对时**：上游连上后自动对时（`ntp.aliyun.com`、`ntp.tencent.com`、`cn.pool.ntp.org`，北京时间），之后每小时同步一次；
  概览页显示设备时间，`/api/status` 的 `time` 为 Unix 秒（未同步时为 0）
- **配置备份**：导出 / 导入 YAML（Wi-Fi、DNS 自定义解析、HTTPS 站点、自定义接口）
- **记录页**（`http://192.168.4.1/monitor`，本次开机内的数据，只存内存）
  - 设备：接入热点的设备、IP、主机名、MAC、实时上下行速率、累计流量、DNS 查询数
  - 连接：每条连接的类型（按协议与端口）、域名（DNS 应答 / TLS SNI / HTTP Host 识别）、
    远端地址、实时速率、累计流量
  - DNS 记录：每条查询及结果（转发 / 本地应答 / 已拦截）与应答 IP
  - 黑名单（重启后清空）：域名及子域名不再解析（NXDOMAIN），已知 IP 与匹配的连接丢包，
    已建立的 TCP 连接向客户端发 RST 断开

## 硬件

**主目标板：ESP32-S3-DevKitC-1 N16R8**（`ESP32-S3-WROOM-1-N16R8`：16 MB flash、8 MB Octal PSRAM；
板载 WS2812 在 GPIO48，`CONFIG_REPEATER_LED_GPIO`）。芯片相关配置在 `sdkconfig.defaults.<芯片>`
（构建时自动叠加在通用的 `sdkconfig.defaults` 之后）；N16R8 也可用独立的
`sdkconfig.defaults.n16r8`（与 `sdkconfig.defaults.esp32s3` 对齐，便于单独构建目录）。

| 芯片 / 板型 | 默认配置 | 流量记录 |
|---|---|---|
| **ESP32-S3-DevKitC-1 N16R8（主路径）** | 16 MB flash（DIO）、8 MB octal PSRAM、240 MHz、LED GPIO48 | 默认关闭，可在记录页开启；记录表放 PSRAM |
| ESP32-C3 / C6 | 4 MB flash（DIO）、无 PSRAM、160 MHz、TLS 缓冲按需分配 | 默认关闭，可在记录页开启；记录表放内部 RAM（约 52KB），容量较小（连接 64 条、DNS 记录 128 条） |

C3 / C6 仅在编译层面验证过，尚未上板实测。其他 S3 模组（例如 8 MB flash + quad PSRAM）请按实际
改 `sdkconfig.defaults.esp32s3` 中的 Flash 大小与 `CONFIG_SPIRAM_MODE_*`。

DevKitC-1 串口：标「UART」的口走板载 USB-UART（默认控制台 UART0）；标「USB」的口是片内
USB Serial/JTAG（烧录可用；若要把控制台切到该口，在 menuconfig 里选
`CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG`）。Linux 上常见为 `/dev/ttyUSB*`（UART）或 `/dev/ttyACM*`（USB）。

流量记录关闭时，转发钩子直接放行，DNS 黑名单与记录页黑名单的 DNS 拦截照常生效，
但不再按连接丢包和断开 TCP 连接。是否开机开启由 `CONFIG_REPEATER_MONITOR_DEFAULT_ON` 决定。

若 Flash 支持 QIO，可改用 `CONFIG_ESPTOOLPY_FLASHMODE_QIO`（本仓库默认 DIO，兼容性更好）。

## 构建与烧录

需要 ESP-IDF v6.1.x。

### 主路径：ESP32-S3-DevKitC-1 N16R8

```bash
idf.py set-target esp32s3
idf.py build
idf.py -p <串口> flash monitor
```

`<串口>` 示例：Linux `/dev/ttyUSB0` 或 `/dev/ttyACM0`，macOS `/dev/cu.*`，Windows `COMx`。
只烧录并监视时可合并为：

```bash
idf.py -p <串口> flash monitor
```

1. （可选）准备 HTTPS 站点证书。证书不入库，构建时嵌入固件：

   ```
   main/certs2/www.aabbcc.com.fullchain.pem   # 证书链（PEM，含中间证书）
   main/certs2/www.aabbcc.com.key             # 私钥（PEM，未加密）
   ```

   **没有这两个文件也能编译**：构建时会用 openssl 在 `build/h2_selfsigned/` 生成一张自签证书
   （RSA-2048，有效期 10 年，域名为 `main/repeater.h` 中的 `H2_SITE_DEFAULT_DOMAIN`），
   并给出警告。自签证书在浏览器里会提示不受信任；需要 openssl（OpenSSL 或 LibreSSL 均可）。
   烧录后也可以在配置页换成自己的域名和证书。

2. 其他芯片用独立构建目录（`build_*/` 已在 `.gitignore` 中）：

   ```bash
   idf.py -B build_c3 -D SDKCONFIG=build_c3/sdkconfig -D IDF_TARGET=esp32c3 -p <串口> flash monitor
   idf.py -B build_c6 -D SDKCONFIG=build_c6/sdkconfig -D IDF_TARGET=esp32c6 -p <串口> flash monitor
   ```

   若要在独立目录显式叠 N16R8 板级默认（与主路径配置相同）：

   ```bash
   idf.py -B build_n16r8 -D SDKCONFIG=build_n16r8/sdkconfig \
          -D "SDKCONFIG_DEFAULTS=sdkconfig.defaults;sdkconfig.defaults.n16r8" \
          -D IDF_TARGET=esp32s3 -p <串口> flash monitor
   ```

   首次构建会通过组件管理器下载 `espressif/nghttp`（版本见 `dependencies.lock`）。

## 使用

1. 连接热点 `ESP32-Repeater`，默认密码 `12345678`（`sdkconfig.defaults` 中修改默认值）
2. 打开 `http://192.168.4.1/`，扫描并选择上游 Wi-Fi，填写密码后保存
3. 连上上游后，热点下的设备即可上网

恢复出厂默认值：`idf.py -p <串口> erase-flash` 后重新烧录。

### HTTPS 站点内置接口

DNS 中继默认把 `www.aabbcc.com` 解析到 `192.168.4.1`，连接热点后可直接访问：

| 路径 | 说明 |
|---|---|
| `/` | 说明页 |
| `/ping` | 协议（h2 / http/1.1）、流编号、客户端地址 |
| `/status` | 中继运行状态 |
| `/headers` | 回显请求头 |
| `/echo` | 原样回显请求体 |

```bash
curl --http2 https://www.aabbcc.com/ping
```

### 配置页 API（端口 80）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/status` | 运行状态 |
| GET / POST | `/api/config` | Wi-Fi 配置（GET 不返回密码） |
| GET | `/api/scan` | 扫描周边网络 |
| GET | `/api/log` | 历史日志（纯文本） |
| POST | `/api/restart` | 重启 |
| GET / POST | `/api/h2routes` | 自定义接口（`action=set&path=&json=` / `action=del&path=`） |
| GET / POST | `/api/dns` | DNS 自定义解析（`action=set&name=&ip=` / `action=del&name=`） |
| GET / POST | `/api/h2site` | HTTPS 站点（`action=set&domain=&cert=&key=` / `action=reset`），GET 不返回私钥 |
| POST | `/api/carprobe` | 车机联网检测的本机应答（`enabled=1|0`） |
| GET / POST | `/api/h2pass` | 恢复语音：临时 TLS 透传（`enabled=1|0&secs=1..60`） |
| GET | `/api/export` | 导出全部配置 JSON（含密码与私钥） |
| GET | `/ws` | 实时日志 WebSocket |
| GET | `/monitor` | 记录页 |
| GET | `/api/mon/live` | 设备、连接、钩子性能 |
| GET | `/api/mon/dns?since=<id>` | DNS 记录（增量） |
| GET / POST | `/api/mon/block` | 黑名单（`action=add&domain=` / `action=del&domain=`） |
| POST | `/api/mon/ctl` | `action=enable` / `disable` / `reset_perf` |
| GET | `/api/mon/bench` | 合成包压测转发钩子 |

POST 均为 `application/x-www-form-urlencoded`。

## 目录结构

```
components/h2_server/   HTTP/2 + HTTP/1.1 服务端组件（esp-tls + nghttp2）
main/
  wifi_repeater.c       Wi-Fi APSTA、NAPT、启动流程
  repeater_config.c     Wi-Fi 配置（NVS）
  dns_relay.c           DNS 中继与自定义解析
  car_probe.c           车机联网检测的本机应答开关
  h2_pass.c             HTTPS 站点的临时 TLS 透传
  napt_lowport.c        源端口 < 1024 的 UDP 经 NAPT 转发
  web_server.c          配置页 HTTP 服务与 API
  web_log.c             日志转发到 WebSocket
  h2_test.c             HTTPS 站点的路由与内置接口
  h2_routes.c           自定义接口（NVS）
  h2_site.c             站点域名与证书（NVS，热替换）
  monitor.c             记录与拦截（lwIP 转发钩子）
  lwip_hook/            注入 lwip 组件的钩子头文件
  www/index.html        配置页
  www/monitor.html      记录页
```

## 记录与拦截的实现与开销

- 通过 `LWIP_HOOK_IP4_CANFORWARD` 在转发路径（NAPT 转换之前）统计与丢包；
  钩子头文件经 `ESP_IDF_LWIP_HOOK_FILENAME` 只注入 lwip 组件（见 `main/CMakeLists.txt`）
- 设备上合成包压测（64 条连接、19200 个包）：每包耗时中位数约 7.5 µs，p99 约 9–12 µs
- 记录页右上角可关闭记录与拦截，钩子随即直接放行
- 只能看到元数据，HTTPS 内容不可见；共享 IP（CDN）上拉黑可能误伤，
  DoH 加 ECH 的连接无法识别域名

## 已知限制

- 配置页与 API 没有认证，热点与上游局域网内的设备都能访问和修改
- DNS 自定义解析只对使用 `192.168.4.1` 作为 DNS 的设备有效；
  开启私人 DNS / 安全 DNS（DoH、DoT）的设备会绕过
- 原生 USB 口在烧录后用串口工具读取时偶尔失去响应，需要重新插拔
