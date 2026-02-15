# 基于 ESP32-P4 EV Board（ESP-IDF v5.5）的局域网 RTSP 图传系统实现指南

> 目标：在同一局域网内，通过 ESP32-P4 官方开发板采集摄像头画面并通过 RTSP（RTP/UDP）推流到 VLC/ffplay。

---

## 1. 方案目标与边界

- **开发框架**：ESP-IDF `v5.5`
- **板卡**：ESP32-P4 EV Board（官方）
- **网络环境**：LAN（AP/路由器下）
- **视频编码策略**：优先使用 **MJPEG**（开发门槛低、与 RTSP 兼容好）
- **推流模式**：板端实现 RTSP Server，PC/手机作为 RTSP Client 拉流

> 为什么先上 MJPEG：
>
> 1) ESP 侧实现复杂度低；2) 避免在初版中引入高复杂编码器；3) VLC/ffplay 原生支持 `RTP/JPEG`。

---

## 2. 总体架构

```text
+----------------------------- ESP32-P4 --------------------------------+
|                                                                       |
|  Camera Driver  --->  Frame RingBuffer  --->  RTP Packetizer (JPEG)   |
|        |                       |                          |            |
|        +---- sensor cfg        +---- producer/consumer    +---- UDP   |
|                                                                       |
|  RTSP Control Plane (TCP 554)  <----->  Client (VLC/ffplay)           |
|   - OPTIONS                                                            |
|   - DESCRIBE (SDP)                                                     |
|   - SETUP (Transport)                                                   |
|   - PLAY / TEARDOWN                                                     |
|                                                                       |
|  Wi-Fi STA + mDNS (optional)                                           |
+-----------------------------------------------------------------------+
```

### 模块拆分

1. **camera_capture**：采集 JPEG 帧（建议 QVGA~VGA 起步）。
2. **rtsp_server**：处理 RTSP 命令，管理会话、返回 SDP。
3. **rtp_jpeg**：将一帧 JPEG 拆分为多个 RTP 包并发送。
4. **stream_task**：按帧率调度发送，维护 sequence/timestamp。
5. **net_service**：Wi-Fi 连接、IP 获取、mDNS 广播。

---

## 3. 工程目录建议

```text
esp32p4_rtsp/
├─ CMakeLists.txt
├─ sdkconfig.defaults
├─ main/
│  ├─ CMakeLists.txt
│  ├─ app_main.c
│  ├─ camera_capture.c
│  ├─ camera_capture.h
│  ├─ rtsp_server.c
│  ├─ rtsp_server.h
│  ├─ rtp_jpeg.c
│  └─ rtp_jpeg.h
└─ components/
   └─ (可选) 官方 camera/sensor 组件
```

---

## 4. 依赖与配置（ESP-IDF v5.5）

### 4.1 组件建议

- ESP-IDF 自带：`lwip`, `freertos`, `esp_timer`, `nvs_flash`, `esp_wifi`
- 官方开源组件：
  - 摄像头驱动（依据 P4 EV 板传感器接口选择）
  - 如果使用官方 BSP，请把对应 EV board BSP 引入工程（`idf_component.yml`）

### 4.2 sdkconfig 关键项

建议在 `menuconfig` 中确认：

- `Component config -> Wi-Fi`：启用 STA
- `Component config -> LWIP`：启用 UDP/TCP
- `Component config -> mbedTLS`：默认即可（RTSP 明文无需 TLS）
- `Partition Table`：给应用足够空间（含图像缓存）

---

## 5. 关键代码实现

> 下面代码是可落地的“最小可运行架构”。你需要按实际摄像头型号补齐 sensor 初始化细节。

### 5.1 `main/app_main.c`

```c
#include <stdio.h>
#include "nvs_flash.h"
#include "esp_log.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "esp_wifi.h"
#include "camera_capture.h"
#include "rtsp_server.h"

static const char *TAG = "main";

static void wifi_init_sta(const char *ssid, const char *pass)
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    wifi_config_t wifi_config = {0};
    snprintf((char *)wifi_config.sta.ssid, sizeof(wifi_config.sta.ssid), "%s", ssid);
    snprintf((char *)wifi_config.sta.password, sizeof(wifi_config.sta.password), "%s", pass);
    wifi_config.sta.threshold.authmode = WIFI_AUTH_WPA2_PSK;

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
    ESP_ERROR_CHECK(esp_wifi_connect());
}

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());

    wifi_init_sta("YOUR_SSID", "YOUR_PASSWORD");

    camera_capture_config_t cam_cfg = {
        .width = 640,
        .height = 480,
        .fps = 15,
        .jpeg_quality = 20,
    };
    ESP_ERROR_CHECK(camera_capture_init(&cam_cfg));

    rtsp_server_config_t rtsp_cfg = {
        .rtsp_port = 554,
        .rtp_port_base = 5004,
        .stream_name = "live",
        .fps = cam_cfg.fps,
    };
    ESP_ERROR_CHECK(rtsp_server_start(&rtsp_cfg));

    ESP_LOGI(TAG, "RTSP URL: rtsp://<board_ip>/live");
}
```

### 5.2 `main/camera_capture.h`

```c
#pragma once
#include <stdint.h>
#include "esp_err.h"

typedef struct {
    int width;
    int height;
    int fps;
    int jpeg_quality;
} camera_capture_config_t;

typedef struct {
    uint8_t *data;
    uint32_t len;
    uint64_t pts_us;
} camera_frame_t;

esp_err_t camera_capture_init(const camera_capture_config_t *cfg);
esp_err_t camera_capture_get_frame(camera_frame_t *frame, uint32_t timeout_ms);
void camera_capture_return_frame(camera_frame_t *frame);
```

### 5.3 `main/rtp_jpeg.h`

```c
#pragma once
#include <stdint.h>
#include <stddef.h>

int rtp_send_jpeg_frame(int sock,
                        const uint8_t *jpeg,
                        size_t jpeg_len,
                        uint16_t *seq,
                        uint32_t timestamp,
                        uint32_t ssrc,
                        const char *client_ip,
                        uint16_t client_rtp_port);
```

### 5.4 `main/rtp_jpeg.c`（核心分包）

```c
#include "rtp_jpeg.h"
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define RTP_VERSION 2
#define RTP_HDR_LEN 12
#define JPEG_HDR_LEN 8
#define MTU_PAYLOAD 1400

static void build_rtp_header(uint8_t *h, uint16_t seq, uint32_t ts, uint32_t ssrc, int marker)
{
    h[0] = (RTP_VERSION << 6);
    h[1] = (marker ? 0x80 : 0x00) | 26; // PT=26 JPEG
    h[2] = (seq >> 8) & 0xFF;
    h[3] = seq & 0xFF;
    h[4] = (ts >> 24) & 0xFF;
    h[5] = (ts >> 16) & 0xFF;
    h[6] = (ts >> 8) & 0xFF;
    h[7] = ts & 0xFF;
    h[8] = (ssrc >> 24) & 0xFF;
    h[9] = (ssrc >> 16) & 0xFF;
    h[10] = (ssrc >> 8) & 0xFF;
    h[11] = ssrc & 0xFF;
}

int rtp_send_jpeg_frame(int sock,
                        const uint8_t *jpeg,
                        size_t jpeg_len,
                        uint16_t *seq,
                        uint32_t timestamp,
                        uint32_t ssrc,
                        const char *client_ip,
                        uint16_t client_rtp_port)
{
    struct sockaddr_in dst = {0};
    dst.sin_family = AF_INET;
    dst.sin_port = htons(client_rtp_port);
    inet_pton(AF_INET, client_ip, &dst.sin_addr);

    size_t offset = 0;
    uint8_t packet[1500];

    while (offset < jpeg_len) {
        size_t chunk = jpeg_len - offset;
        size_t max_chunk = MTU_PAYLOAD - RTP_HDR_LEN - JPEG_HDR_LEN;
        if (chunk > max_chunk) chunk = max_chunk;
        int marker = (offset + chunk >= jpeg_len) ? 1 : 0;

        build_rtp_header(packet, (*seq)++, timestamp, ssrc, marker);

        uint8_t *jh = packet + RTP_HDR_LEN;
        memset(jh, 0, JPEG_HDR_LEN);
        // Type-specific(1)=0, Fragment offset(3)
        jh[1] = (offset >> 16) & 0xFF;
        jh[2] = (offset >> 8) & 0xFF;
        jh[3] = offset & 0xFF;
        jh[4] = 1;   // type, 根据采样模式可调整
        jh[5] = 255; // q
        jh[6] = 0;   // width/8, 可选在 SDP 里约束
        jh[7] = 0;   // height/8

        memcpy(packet + RTP_HDR_LEN + JPEG_HDR_LEN, jpeg + offset, chunk);

        ssize_t sent = sendto(sock,
                              packet,
                              RTP_HDR_LEN + JPEG_HDR_LEN + chunk,
                              0,
                              (struct sockaddr *)&dst,
                              sizeof(dst));
        if (sent < 0) return -1;

        offset += chunk;
    }

    return 0;
}
```

### 5.5 `main/rtsp_server.h`

```c
#pragma once
#include <stdint.h>
#include "esp_err.h"

typedef struct {
    uint16_t rtsp_port;
    uint16_t rtp_port_base;
    const char *stream_name;
    int fps;
} rtsp_server_config_t;

esp_err_t rtsp_server_start(const rtsp_server_config_t *cfg);
```

### 5.6 `main/rtsp_server.c`（最小 RTSP 状态机）

```c
#include "rtsp_server.h"
#include "camera_capture.h"
#include "rtp_jpeg.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

static const char *TAG = "rtsp";

typedef struct {
    int playing;
    int rtsp_fd;
    int rtp_fd;
    char client_ip[32];
    uint16_t client_rtp_port;
    uint16_t seq;
    uint32_t ts;
    uint32_t ssrc;
    int fps;
    char session_id[16];
    char stream_name[32];
} rtsp_session_t;

static void rtsp_reply(int fd, const char *cseq, const char *extra, const char *body)
{
    char resp[1024];
    int body_len = body ? (int)strlen(body) : 0;
    int n = snprintf(resp, sizeof(resp),
                     "RTSP/1.0 200 OK\r\n"
                     "CSeq: %s\r\n"
                     "%s"
                     "%s"
                     "\r\n",
                     cseq ? cseq : "1",
                     extra ? extra : "",
                     body ? body : "");
    if (body_len > 0) {
        (void)n;
    }
    send(fd, resp, strlen(resp), 0);
}

static void stream_task(void *arg)
{
    rtsp_session_t *s = (rtsp_session_t *)arg;
    const uint32_t clock_rate = 90000;
    const uint32_t ts_step = clock_rate / s->fps;

    while (1) {
        if (!s->playing) {
            vTaskDelay(pdMS_TO_TICKS(10));
            continue;
        }

        camera_frame_t f = {0};
        if (camera_capture_get_frame(&f, 100) == 0) {
            if (f.data && f.len > 0) {
                (void)rtp_send_jpeg_frame(s->rtp_fd, f.data, f.len,
                                          &s->seq, s->ts, s->ssrc,
                                          s->client_ip, s->client_rtp_port);
                s->ts += ts_step;
            }
            camera_capture_return_frame(&f);
        }
    }
}

static void handle_client(int fd, struct sockaddr_in *peer, const rtsp_server_config_t *cfg)
{
    char req[2048];
    rtsp_session_t s = {0};
    s.rtsp_fd = fd;
    s.rtp_fd = socket(AF_INET, SOCK_DGRAM, 0);
    s.seq = 1000;
    s.ts = 0;
    s.ssrc = 0x22334455;
    s.fps = cfg->fps;
    snprintf(s.session_id, sizeof(s.session_id), "12345678");
    snprintf(s.stream_name, sizeof(s.stream_name), "%s", cfg->stream_name);
    inet_ntop(AF_INET, &peer->sin_addr, s.client_ip, sizeof(s.client_ip));

    xTaskCreate(stream_task, "stream_task", 8192, &s, 5, NULL);

    while (1) {
        int len = recv(fd, req, sizeof(req) - 1, 0);
        if (len <= 0) break;
        req[len] = '\0';

        char cseq[32] = "1";
        char *pcseq = strstr(req, "CSeq:");
        if (pcseq) sscanf(pcseq, "CSeq: %31s", cseq);

        if (strstr(req, "OPTIONS")) {
            rtsp_reply(fd, cseq,
                       "Public: OPTIONS, DESCRIBE, SETUP, PLAY, TEARDOWN\r\n",
                       NULL);
        } else if (strstr(req, "DESCRIBE")) {
            char sdp[512];
            snprintf(sdp, sizeof(sdp),
                     "v=0\r\n"
                     "o=- 0 0 IN IP4 %s\r\n"
                     "s=ESP32P4 Stream\r\n"
                     "t=0 0\r\n"
                     "a=control:*\r\n"
                     "m=video 0 RTP/AVP 26\r\n"
                     "a=rtpmap:26 JPEG/90000\r\n"
                     "a=control:track1\r\n",
                     s.client_ip);

            char hdr[256];
            snprintf(hdr, sizeof(hdr),
                     "Content-Base: rtsp://%s/%s/\r\n"
                     "Content-Type: application/sdp\r\n"
                     "Content-Length: %d\r\n",
                     s.client_ip, s.stream_name, (int)strlen(sdp));

            rtsp_reply(fd, cseq, hdr, sdp);
        } else if (strstr(req, "SETUP")) {
            char *t = strstr(req, "client_port=");
            if (t) {
                int p1 = 0, p2 = 0;
                sscanf(t, "client_port=%d-%d", &p1, &p2);
                s.client_rtp_port = (uint16_t)p1;
            }
            char hdr[256];
            snprintf(hdr, sizeof(hdr),
                     "Transport: RTP/AVP;unicast;client_port=%u-%u;server_port=%u-%u\r\n"
                     "Session: %s\r\n",
                     s.client_rtp_port, (uint16_t)(s.client_rtp_port + 1),
                     cfg->rtp_port_base, (uint16_t)(cfg->rtp_port_base + 1),
                     s.session_id);
            rtsp_reply(fd, cseq, hdr, NULL);
        } else if (strstr(req, "PLAY")) {
            s.playing = 1;
            char hdr[128];
            snprintf(hdr, sizeof(hdr),
                     "Session: %s\r\n"
                     "RTP-Info: url=rtsp://%s/%s/track1;seq=%u;rtptime=%u\r\n",
                     s.session_id, s.client_ip, s.stream_name, s.seq, s.ts);
            rtsp_reply(fd, cseq, hdr, NULL);
        } else if (strstr(req, "TEARDOWN")) {
            s.playing = 0;
            rtsp_reply(fd, cseq, "Session: 12345678\r\n", NULL);
            break;
        }
    }

    close(s.rtp_fd);
    close(fd);
}

static void rtsp_task(void *arg)
{
    const rtsp_server_config_t *cfg = (const rtsp_server_config_t *)arg;

    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(cfg->rtsp_port);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(listen_fd, (struct sockaddr *)&addr, sizeof(addr));
    listen(listen_fd, 2);

    ESP_LOGI(TAG, "RTSP listening on %u", cfg->rtsp_port);

    while (1) {
        struct sockaddr_in peer = {0};
        socklen_t plen = sizeof(peer);
        int fd = accept(listen_fd, (struct sockaddr *)&peer, &plen);
        if (fd >= 0) {
            handle_client(fd, &peer, cfg);
        }
    }
}

esp_err_t rtsp_server_start(const rtsp_server_config_t *cfg)
{
    static rtsp_server_config_t g_cfg;
    g_cfg = *cfg;
    xTaskCreate(rtsp_task, "rtsp_task", 8192, &g_cfg, 5, NULL);
    return ESP_OK;
}
```

---

## 6. 性能与稳定性优化建议

1. **双缓冲/环形缓冲**：camera 与 rtp sender 解耦，避免阻塞。
2. **PSRAM 利用**：帧缓存放 PSRAM，避免内部 RAM 紧张。
3. **帧率自适应**：当 UDP send 失败率上升时，降分辨率/降 fps。
4. **GOP / 编码策略**：若后续引入 H.264，需补充 SPS/PPS 与 SDP fmtp。
5. **多客户端策略**：初版建议单客户端，后续可扩展 session list。

---

## 7. 编译与烧录（ESP-IDF v5.5）

```bash
idf.py set-target esp32p4
idf.py menuconfig
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

拉流验证：

```bash
ffplay -fflags nobuffer -flags low_delay rtsp://<board_ip>/live
# 或
vlc rtsp://<board_ip>/live
```

---

## 8. 你可以直接落地的迭代路线

- **M1（本回答实现）**：MJPEG over RTSP 单路推流。
- **M2**：增加 RTCP、丢包统计、码率与帧率动态调整。
- **M3**：接入硬件编码（若芯片/SDK路径支持），切换到 H.264 RTSP。
- **M4**：增加 Web 配置页（Wi-Fi、分辨率、帧率、码率）+ OTA。

---

## 9. 常见坑位（ESP 侧）

1. **MTU 分包错误**导致 VLC 花屏或无法播放。
2. **RTSP 响应头不规范**（特别是 `CSeq`、`Session`、`Transport`）。
3. **timestamp 递增不稳定**造成画面卡顿。
4. **摄像头输出不是 JPEG**时需先做 JPEG 编码再 RTP。
5. **Wi-Fi 省电模式**影响实时性（建议关闭省电）。

---

## 10. 结论

按上面的架构，你可以在 ESP-IDF v5.5 上快速构建一个“可拉流、可演示、可迭代”的局域网 RTSP 图传系统。先用 MJPEG 跑通链路，稳定后再升级编码与控制面能力，是风险最低、工程效率最高的路径。
