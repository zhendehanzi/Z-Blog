---
title: "20元打造一款智能摄像头"
date: 2023-05-27T09:53:44+08:00
draft: False
---

![ESP-CAM-IPEX](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/ESP-CAM-IPEX.1yhc2tvmiqsg.webp)

## 前言

机缘巧合，接触到了一款性价比爆炸的硬件——ESP-CAM，一款搭载了OV2640摄像头和ESP32-S的物联网模块，某宝20多元的售价，其中，ESP32板载支持Wi-Fi、蓝牙，双核240MHZ的MCU，还有4MB的PSRAM，拿来做无线摄像头再合适不过，OV2640还有200万像素，这么顶的配置还要什么自行车？

![ESP-CAM](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/ESP-CAM.1bvq6rxaz21s.webp)

于是，回归本文主题，通过ESP-CAM进行无线UDP串流，上位机接收，再通过Opencv-python完成机器视觉的功能，力求实现简洁、易用并可以高度定制化的智能监控。

> ***全部代码均附在文中***

## 硬件核心固件

视频传输的核心是需要稳定流畅，且满足机器视觉处理。这样看来使用UDP协议进行串流再合适不过。

UDP延迟低、不需要建立连接，占用资源也低。

唯一问题是因为MTU（最大传输单元）的限制，接收到的数据包是一副完整图像的若干块，需要自行完成对于UDP图像帧数据包的辨识与拼接组合。

这么看起来，万事有一利必有一弊，本项目经过实验，UDP利远远大于弊，没错是远远大于。经过调试优化后的固件在CIF分辨率(400x296)下的传输可以稳定达到60FPS（1s传输60帧图片），在VGA分辨率（640x480）下可以稳定在30FPS。再提高分辨率的话，大概会将帧率降低在15FPS左右

![30FPS](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/30FPS.31x361h9mgk0.webp)

![监控图](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/监控图.4emmsy166oa0.webp)

30FPS下的帧率和分辨率已经基本可以满足功能实现！

现有的一些网上的ESP-CAM推流例程，大多是基于Arduino的开发，这种开发非常简单也是ESP原生的优势，不得不高呼国产单片机之光。

但是，本项目还是选择ESP-IDF进行开发，原因也简单:

* 嘿嘿嘿，就是玩儿！（啪啪打脸👏），为了学习！
* 言归正传，Arduino的底层其实还是IDF，属于高级的API封装，浪费资源；
* IDF灵活性高，有很多IDF的API，Arduino没有；
* IDF原生支持的FreeRTOS开发，用起来也不会比Arduino复杂太多，并且容易上手

综上所述，介绍下这部分代码

***main.c***是主程序代码，包含初始化Wi-Fi、开启camera的任务。

***camera.c***是摄像头相关固件，包含OV2640引脚与图像配置，兼容主流的几款ESP摄像头

***Kconfig.projbuild*** 是预编译的sdkconfig文件，使用方法同IDF官方例程序

其中, ESP使用的是STA模式，sdkconfig中需要首先配置Wi-Fi的SSID和密码（仅支持2.4GHZ频段的Wi-Fi，5GHZ不支持），并且需要注意在sdkconfig中开启PSRAM。

***main.c***

```c
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_system.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_log.h"
#include "nvs_flash.h"

#include "lwip/err.h"
#include "lwip/sys.h"

/* The examples use WiFi configuration that you can set via project configuration menu

   If you'd rather not, just change the below entries to strings with
   the config you want - ie #define EXAMPLE_WIFI_SSID "mywifissid"
*/
#define EXAMPLE_ESP_WIFI_SSID      CONFIG_ESP_WIFI_SSID
#define EXAMPLE_ESP_WIFI_PASS      CONFIG_ESP_WIFI_PASSWORD
#define EXAMPLE_ESP_MAXIMUM_RETRY  CONFIG_ESP_MAXIMUM_RETRY

/* FreeRTOS event group to signal when we are connected*/
static EventGroupHandle_t s_wifi_event_group;

/* The event group allows multiple bits for each event, but we only care about two events:
 * - we are connected to the AP with an IP
 * - we failed to connect after the maximum amount of retries */
#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1

static const char *TAG = "main";

static int s_retry_num = 0;

static void event_handler(void* arg, esp_event_base_t event_base,
                                int32_t event_id, void* event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        if (s_retry_num < EXAMPLE_ESP_MAXIMUM_RETRY) {
            esp_wifi_connect();
            s_retry_num++;
            ESP_LOGI(TAG, "retry to connect to the AP");
        } else {
            xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
        }
        ESP_LOGI(TAG,"connect to the AP fail");
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t* event = (ip_event_got_ip_t*) event_data;
        ESP_LOGI(TAG, "got ip:" IPSTR, IP2STR(&event->ip_info.ip));
        s_retry_num = 0;
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}

void wifi_init_sta(void)
{
    s_wifi_event_group = xEventGroupCreate();

    ESP_ERROR_CHECK(esp_netif_init());

    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    esp_event_handler_instance_t instance_any_id;
    esp_event_handler_instance_t instance_got_ip;
    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT,
                                                        ESP_EVENT_ANY_ID,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_any_id));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT,
                                                        IP_EVENT_STA_GOT_IP,
                                                        &event_handler,
                                                        NULL,
                                                        &instance_got_ip));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .password = EXAMPLE_ESP_WIFI_PASS,
            /* Setting a password implies station will connect to all security modes including WEP/WPA.
             * However these modes are deprecated and not advisable to be used. Incase your Access point
             * doesn't support WPA2, these mode can be enabled by commenting below line */
	     .threshold.authmode = WIFI_AUTH_WPA2_PSK,

            .pmf_cfg = {
                .capable = true,
                .required = false
            },
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA) );
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config) );
    ESP_ERROR_CHECK(esp_wifi_start() );

    ESP_LOGI(TAG, "wifi_init_sta finished.");

    /* Waiting until either the connection is established (WIFI_CONNECTED_BIT) or connection failed for the maximum
     * number of re-tries (WIFI_FAIL_BIT). The bits are set by event_handler() (see above) */
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
            pdFALSE,
            pdFALSE,
            portMAX_DELAY);

    /* xEventGroupWaitBits() returns the bits before the call returned, hence we can test which event actually
     * happened. */
    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG, "connected to ap SSID:%s password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGI(TAG, "Failed to connect to SSID:%s, password:%s",
                 EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS);
    } else {
        ESP_LOGE(TAG, "UNEXPECTED EVENT");
    }

    /* The event will not be processed after unregister */
    ESP_ERROR_CHECK(esp_event_handler_instance_unregister(IP_EVENT, IP_EVENT_STA_GOT_IP, instance_got_ip));
    ESP_ERROR_CHECK(esp_event_handler_instance_unregister(WIFI_EVENT, ESP_EVENT_ANY_ID, instance_any_id));
    vEventGroupDelete(s_wifi_event_group);
}

esp_err_t init_camera(void);
void start_camera(void);

void app_main(void)
{
    //Initialize NVS
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
      ESP_ERROR_CHECK(nvs_flash_erase());
      ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    ESP_LOGI(TAG, "Init Camera");
    init_camera();

    ESP_LOGI(TAG, "ESP_WIFI_MODE_STA");
    wifi_init_sta();

    ESP_LOGI(TAG, "Start Camera");
    start_camera();
}
```

***camera.c***

```c
#include <esp_log.h>
#include <esp_system.h>
#include <nvs_flash.h>
#include <sys/param.h>
#include <string.h>

#include "lwip/sockets.h"
#include "lwip/netdb.h"
#include "lwip/api.h"
#include "errno.h"

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#include "esp_timer.h"
#include "esp_camera.h"


// support IDF 5.x
#ifndef portTICK_RATE_MS
#define portTICK_RATE_MS portTICK_PERIOD_MS
#endif


#define CAMERA_MODEL_AI_THINKER

// ESPEYE-S3
#if defined(CAMERA_MODEL_ESPEYE_S3)
  #define PWDN_GPIO_NUM -1
  #define RESET_GPIO_NUM -1
  #define XCLK_GPIO_NUM 15
  #define SIOD_GPIO_NUM 4
  #define SIOC_GPIO_NUM 5

  #define Y9_GPIO_NUM 16
  #define Y8_GPIO_NUM 17
  #define Y7_GPIO_NUM 18
  #define Y6_GPIO_NUM 12
  #define Y5_GPIO_NUM 10
  #define Y4_GPIO_NUM 8
  #define Y3_GPIO_NUM 9
  #define Y2_GPIO_NUM 11
  #define VSYNC_GPIO_NUM 6
  #define HREF_GPIO_NUM 7
  #define PCLK_GPIO_NUM 13


#elif defined(CAMERA_MODEL_AI_THINKER)
  #define PWDN_GPIO_NUM     32
  #define RESET_GPIO_NUM    -1
  #define XCLK_GPIO_NUM      0
  #define SIOD_GPIO_NUM     26
  #define SIOC_GPIO_NUM     27
  
  #define Y9_GPIO_NUM       35
  #define Y8_GPIO_NUM       34
  #define Y7_GPIO_NUM       39
  #define Y6_GPIO_NUM       36
  #define Y5_GPIO_NUM       21
  #define Y4_GPIO_NUM       19
  #define Y3_GPIO_NUM       18
  #define Y2_GPIO_NUM        5
  #define VSYNC_GPIO_NUM    25
  #define HREF_GPIO_NUM     23
  #define PCLK_GPIO_NUM     22

#elif defined(CAMERA_MODEL_ESPEYE)
  #define PWDN_GPIO_NUM     -1
  #define RESET_GPIO_NUM    -1
  #define XCLK_GPIO_NUM      4
  #define SIOD_GPIO_NUM     18
  #define SIOC_GPIO_NUM     23
  
  #define Y9_GPIO_NUM       36
  #define Y8_GPIO_NUM       37
  #define Y7_GPIO_NUM       38
  #define Y6_GPIO_NUM       39
  #define Y5_GPIO_NUM       35
  #define Y4_GPIO_NUM       14
  #define Y3_GPIO_NUM       13
  #define Y2_GPIO_NUM       34
  #define VSYNC_GPIO_NUM    5
  #define HREF_GPIO_NUM     27
  #define PCLK_GPIO_NUM     25

#else
  #error "Camera model not selected"
#endif

static const char *TAG = "camera";

esp_err_t init_camera(void)
{
    camera_config_t config;
    config.ledc_channel = LEDC_CHANNEL_0;
    config.ledc_timer = LEDC_TIMER_0;
    config.pin_d0 = Y2_GPIO_NUM;
    config.pin_d1 = Y3_GPIO_NUM;
    config.pin_d2 = Y4_GPIO_NUM;
    config.pin_d3 = Y5_GPIO_NUM;
    config.pin_d4 = Y6_GPIO_NUM;
    config.pin_d5 = Y7_GPIO_NUM;
    config.pin_d6 = Y8_GPIO_NUM;
    config.pin_d7 = Y9_GPIO_NUM;
    config.pin_xclk = XCLK_GPIO_NUM;
    config.pin_pclk = PCLK_GPIO_NUM;
    config.pin_vsync = VSYNC_GPIO_NUM;
    config.pin_href = HREF_GPIO_NUM;
    config.pin_sscb_sda = SIOD_GPIO_NUM;
    config.pin_sscb_scl = SIOC_GPIO_NUM;
    config.pin_pwdn = PWDN_GPIO_NUM;
    config.pin_reset = RESET_GPIO_NUM;
    config.xclk_freq_hz = 24000000;
    // config.xclk_freq_hz = 36000000;
    config.pixel_format = PIXFORMAT_JPEG;
    // config.frame_size = FRAMESIZE_QVGA; // 320 * 240 / 5 = 15,360 < 32,768
    // config.frame_size = FRAMESIZE_CIF; // 400 * 296 / 5 = 23,680 < 32,768
    config.frame_size = FRAMESIZE_VGA; // 480 * 320 / 5 = 30,720 < 32,768
    config.jpeg_quality = 10;
    config.fb_count = 2;
    config.fb_location = CAMERA_FB_IN_PSRAM;
    // config.grab_mode = CAMERA_GRAB_WHEN_EMPTY;
    config.grab_mode = CAMERA_GRAB_LATEST;
    config.fragment_mode = true;
    config.zero_padding = false;

    // camera init
    esp_err_t err = esp_camera_init(&config);


    if (err != ESP_OK)
    {
        ESP_LOGE(TAG, "Camera Init Failed error 0x%x", err);
        return err;
    }

    sensor_t *s = esp_camera_sensor_get();

    // drop down frame size for higher initial frame rate
    s->set_framesize(s, FRAMESIZE_VGA);
    // s->set_framesize(s, FRAMESIZE_QVGA);
    // s->set_framesize(s, FRAMESIZE_VGA);
    // s->set_framesize(s, FRAMESIZE_XGA);
    // s->set_framesize(s, FRAMESIZE_HD);
    // s->set_framesize(s, FRAMESIZE_SXGA);

    return ESP_OK;
}

#define USE_NETCONN 1
#if USE_NETCONN
struct netconn *camera_conn;
struct ip_addr peer_addr;
#else
static int camera_sock;
static struct sockaddr_in senderinfo;
#endif

static void camera_tx(void *param)
{
    size_t packet_max = 32768;
#if USE_NETCONN
    struct netbuf *txbuf = netbuf_new();
#else
    senderinfo.sin_port = htons(55556);
#endif

    uint32_t frame_count = 0;
    uint32_t frame_size_sum = 0;
    int64_t start = esp_timer_get_time();

    while (1)
    {
        camera_fb_t *fb = esp_camera_fb_get();
        if (!fb)
        {
            ESP_LOGE(TAG, "Camera capture failed");
            continue;
        }

        uint8_t *buf = fb->buf;
        size_t total = fb->len;
        size_t send = 0;

        // ESP_LOGI(TAG, "send %d", total);

        for (; send < total;)
        {
            size_t txlen = total - send;
            if (txlen > packet_max)
            {
                txlen = packet_max;
            }

#if USE_NETCONN
            err_t err = netbuf_ref(txbuf, &buf[send], txlen);
            if (err == ERR_OK)
            {
                err = netconn_sendto(camera_conn, txbuf, &peer_addr, 55556);
                if (err == ERR_OK)
                {
                    send += txlen;
                }
                else if (err == ERR_MEM)
                {
                    ESP_LOGW(TAG, "netconn_sendto ERR_MEM");
                    vTaskDelay(pdMS_TO_TICKS(10));
                }
                else
                {
                    ESP_LOGE(TAG, "netconn_sendto err %d", err);
                    break;
                }
            }
            else
            {
                ESP_LOGE(TAG, "netbuf_ref err %d", err);
                break;
            }
#else
            int err = sendto(camera_sock, &buf[send], txlen, 0, (struct sockaddr *)&senderinfo, sizeof(senderinfo));
            if (err < 0)
            {
                ESP_LOGE(TAG, "Error occurred during sending: errno %d", errno);
                if (errno == ENOMEM)
                {
                    vTaskDelay(pdMS_TO_TICKS(10));
                }
                else
                {
                    send = total;
                }
            }
            else if (err == 0)
            {
                vTaskDelay(pdMS_TO_TICKS(10));
            }
            else
            {
                send += err;
            }
#endif
        }

        esp_camera_fb_return(fb);

        if (buf[0] == 0xff && buf[1] == 0xd8 && buf[2] == 0xff)
        {
            frame_count++;
        }
        frame_size_sum += total;

        int64_t end = esp_timer_get_time();
        int64_t pasttime = end - start;
        if (pasttime > 1000000) //如果pasttime超过1秒
        {
            start = end;
            float adj = 1000000.0 / (float)pasttime;
            float mbps = (frame_size_sum * 8.0 * adj) / 1024.0 / 1024.0;
            ESP_LOGI(TAG, "%.1f fps %.3f Mbps", frame_count * adj, mbps);
            frame_count = 0;
            frame_size_sum = 0;
        }

        vTaskDelay(pdMS_TO_TICKS(1));
    }
}

void start_camera(void)
{
#if USE_NETCONN
    camera_conn = netconn_new(NETCONN_UDP);
    if (camera_conn == NULL)
    {
        ESP_LOGE(TAG, "netconn_new err");
        return;
    }

    err_t err = netconn_bind(camera_conn, IPADDR_ANY, 55555);
    if (err != ERR_OK)
    {
        ESP_LOGE(TAG, "netconn_bind err %d", err);
        netconn_delete(camera_conn);
        return;
    }

    ESP_LOGI(TAG, "Wait a trigger...");
    while (1)
    {
        struct netbuf *rxbuf;
        err = netconn_recv(camera_conn, &rxbuf);
        if (err == ERR_OK)
        {
            uint8_t *data;
            u16_t len;
            netbuf_data(rxbuf, (void **)&data, &len);
            if (len)
            {
                ESP_LOGI(TAG, "netconn_recv %d", len);
                if (data[0] == 0x55)
                {
                    peer_addr = *netbuf_fromaddr(rxbuf);
                    ESP_LOGI(TAG, "peer %x", peer_addr.u_addr.ip4.addr);

                    ESP_LOGI(TAG, "Trigged!");
                    netbuf_delete(rxbuf);
                    break;
                }
            }
            netbuf_delete(rxbuf);
        }
        vTaskDelay(pdMS_TO_TICKS(10));
    }
#else
    camera_sock = socket(AF_INET, SOCK_DGRAM, 0);
    if (camera_sock < 0)
    {
        ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
        return;
    }

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(55555);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(camera_sock, (struct sockaddr *)&addr, sizeof(addr));

    socklen_t addrlen = sizeof(senderinfo);
    char buf[128];
    ESP_LOGI(TAG, "Wait a trigger...");
    while (1)
    {
        int n = recvfrom(camera_sock, buf, sizeof(buf) - 1, 0, (struct sockaddr *)&senderinfo, &addrlen);
        if (n > 0 && buf[0] == 0x55)
        {
            break;
        }
        ESP_LOGI(TAG, "recvfrom %d", n);
    }
#endif
    xTaskCreatePinnedToCore(&camera_tx, "camera_tx", 4096, NULL, 10, NULL, tskNO_AFFINITY);
}

```

***Kconfig.projbuild*** 

```c
menu "Example Configuration"

    config ESP_WIFI_SSID
        string "WiFi SSID"
        default "myssid"
        help
            SSID (network name) for the example to connect to.

    config ESP_WIFI_PASSWORD
        string "WiFi Password"
        default "mypassword"
        help
            WiFi password (WPA or WPA2) for the example to use.

    config ESP_MAXIMUM_RETRY
        int "Maximum retry"
        default 5
        help
            Set the Maximum retry to avoid station reconnecting to the AP unlimited when the AP is really inexistent.
endmenu

```



> **固件由ESP-IDF5.0测试开发**

## 机器视觉

ESP-CAM尽管在单片机中性能不俗，但是拿来跑机器视觉必备的OpenCV还是有点弱，况且，通过UDP把稳定的视频流传回来，怎么折腾可以有各种玩儿法。

比如，自从ChatGPT掀起的**AI**浪潮，既然拿到了视频流不得随便跑点深度学习的AI模型，比如下面，图片随便从网上截的，加入了骨骼点识别模型，可以精准进行骨骼点的识别，再配合人体行为识别算法，等等一系列组合拳，就可以实现很多脑洞大开的玩儿法。

![骨骼点识别](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/骨骼点识别.4m57nfr6iug0.webp)

​									（图片来源于网络，效果为AlphaPose模型加持）

跑题了，这边提供一个简单但是好用的算法，使用到的实现机器视觉的方法，也就是O pen CV与深度学习无关，目的是为了兼容像树莓派、旧PC之类的设备也可以轻松运行，以利用空闲计算机设备的算力，因此要用最简单的方式实现能令人满意的效果。

先看一段随手截取的GIF👇

![20230527-09'25'26202352914134233](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/20230527-09'25'26202352914134233.5hob00ogtro0.gif)

实现效果总结下：

* 主题是居家环境，那么主要就是入侵检测功能，当画面中有物体移动，实时进行标记
* 自动截取移动的物体图片，保存在指定目录下，配合MQTT可以快速实现一键消息推送
* 24h的不间断智能监控预警，并定时清除旧文件，释放磁盘内存

![20230527-08'25'26](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/20230527-08'25'26.1fwrjck1d3ls.webp)

自动截取移动物体并保存

![QQ20230528-201836@2x](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/QQ20230528-201836@2x.5h38fm7zk2o0.webp)

24h视频的不间断保存

![QQ20230528-202236@2x](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/QQ20230528-202236@2x.5d1swlw5smg0.webp)

视频处理过程中，加入一些模糊化、扩张、缩减等处理，目的其实就是要对画面内容进行降噪，否则光线变化，以及图像噪声的会使得效果大打折扣，处理后便可以突显主体，这些步骤与其中的参数都要自己调整。

在判断jpeg图片的完整性中，用文件头和文件尾便可以进行巧妙的判断，比如通过👇的0xff0xd80xff就可以识别jpeg图片包的开始，同理可以识别包尾，然后按顺序一拼接，图片帧就出来了。

```text
Start Marker  | JFIF Marker | Header Length | Identifier
 
0xff, 0xd8    | 0xff, 0xe0  |    2-bytes    | "JFIF\0"
```

```python
import cv2
import numpy as np
import socket
import os
import threading
import queue
import time
import traceback
import datetime

frame_q = queue.Queue()
runing = True


def is_dark_environment(frame, threshold=100):
    # 将帧转换为灰度图像
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)

    # 计算灰度图像的平均亮度
    average_brightness = int(gray.mean())

    # 判断平均亮度是否低于阈值
    if average_brightness < threshold:
        return True
    else:
        return False


def keep_latest_files(folder_path, num_to_keep):
    """保留指定数量的最新文件。

    Args:
        folder_path (str): 文件夹路径。
        num_to_keep (int): 要保留的文件数量。

    Returns:
        None
    """
    # 遍历文件夹
    files = os.listdir(folder_path)

    # 获取每个文件的创建时间
    file_times = []
    for file in files:
        file_path = os.path.join(folder_path, file)
        file_time = os.path.getctime(file_path)
        file_times.append((file_path, file_time))

    # 按创建时间排序
    file_times.sort(key=lambda x: x[1])

    # 删除最早保存的文件
    num_to_delete = len(file_times) - num_to_keep  # 要删除的文件数量
    for i in range(num_to_delete):
        file_path = file_times[i][0]
        os.remove(file_path)


def udp_recv(listen_addr, target_addr):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(1)
    sock.bind((listen_addr, 55556))

    print('Start Streaming...')

    chunks = b''
    while runing:
        try:
            msg, address = sock.recvfrom(2 ** 16)
        except Exception as e:
            # print('sock.recvfrom',e)
            sock.sendto(b'\x55', (target_addr, 55555))
            continue
        soi = msg.find(b'\xff\xd8\xff')  # 返回第一次出现的位置，没有匹配返回-1
        eoi = msg.rfind(b'\xff\xd9')  # 返回最后一次出现的位置
        # print(time.perf_counter(), len(msg), soi, eoi, msg[:2], msg[-2:])
        if soi >= 0:
            if chunks.startswith(b'\xff\xd8\xff'):
                if eoi >= 0:
                    chunks += msg[:eoi + 2]
                    # print(time.perf_counter(), "Complete picture")
                    eoi = -1
                else:
                    chunks += msg[:soi]
                    # print(time.perf_counter(), "Incomplete picture")
                try:
                    frame_q.put(chunks, timeout=1)
                except Exception as e:
                    print(e)
            chunks = msg[soi:]
        else:
            chunks += msg
        if eoi >= 0:
            eob = len(chunks) - len(msg) + eoi + 2
            if chunks.startswith(b'\xff\xd8\xff'):
                byte_frame = chunks[:eob]
                # print(time.perf_counter(), "Complete picture")
                try:
                    frame_q.put(byte_frame, timeout=1)
                except Exception as e:
                    print(e)
            else:
                print(time.perf_counter(), "Invalid picture")
            chunks = chunks[eob:]
    sock.close()
    print('Stop Streaming')


def main():
    global runing

    thread = threading.Thread(target=udp_recv, args=(listen_ip, target_ip))
    thread.start()

    winname = 'frame'
    if if_fullscreen and if_videoshow:
        cv2.namedWindow(winname, cv2.WINDOW_NORMAL)
        cv2.setWindowProperty(winname, cv2.WND_PROP_FULLSCREEN, cv2.WINDOW_FULLSCREEN)
    writer = None
    img = None
    next_write = 0
    inital_flag = True
    start_file_time = time.time()
    while True:
        now = datetime.datetime.now()
        current_time = time.time()

        try:
            if if_write:
                if not frame_q.empty():
                    img = None
            else:
                img = None
            if img is None:
                while True:
                    byte_frame = frame_q.get(block=True, timeout=1)
                    if frame_q.empty() or grab_all:
                        break
                    # print(time.perf_counter(), 'Skip picture')
                # print(time.perf_counter(), 'Decode picture')
                img = cv2.imdecode(np.frombuffer(byte_frame, dtype=np.uint8), 1)

            if inital_flag:
                # 初始化平均影像
                avg = cv2.blur(img, (4, 4))
                avg_float = np.float32(avg)
                inital_flag = False
                continue
            # 判断是否到达指定时间间隔
            if current_time - start_file_time >= file_detect_interval:
                # 执行文件删除
                keep_latest_files("videos_record", keep_recordvideo_nums)  # 删除早期的视频视频
                keep_latest_files("motion_record", keep_motionjpg_nums)  # 删除早期的图像文件
                print("执行文件清理！")
                # 更新起始时间戳
                start_file_time = current_time
            # 模糊处理
            blur = cv2.blur(img, (4, 4))

            # 计算目前影格与平均影像的差异值
            diff = cv2.absdiff(avg, blur)

            # 将图片转为灰阶
            gray = cv2.cvtColor(diff, cv2.COLOR_BGR2GRAY)

            # 筛选出变动程度大于门槛值的区域
            ret, thresh = cv2.threshold(gray, 25, 255, cv2.THRESH_BINARY)

            # 使用型态转换函数去除杂讯
            kernel = np.ones((5, 5), np.uint8)
            thresh = cv2.morphologyEx(thresh, cv2.MORPH_OPEN, kernel, iterations=2)
            thresh = cv2.morphologyEx(thresh, cv2.MORPH_CLOSE, kernel, iterations=2)

            # 产生等高线
            cnts, cntImg = cv2.findContours(thresh.copy(), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
            img_copy = img.copy()
            for c in cnts:
                # 忽略太小的区域
                if cv2.contourArea(c) < 2500:
                    continue

                # 计算等高线的外框范围
                (x, y, w, h) = cv2.boundingRect(c)

                # 画出外框
                cv2.rectangle(img_copy, (x, y), (x + w, y + h), (0, 255, 0), 2)

                # 侦测到物体，可以自己加上处理的程式码在这里...
                nowtime = now.strftime("%Y%m%d-%H'%M'%S")
                motion_record_path = "motion_record" + "/" + nowtime + ".jpg"
                cv2.imwrite(motion_record_path, img_copy)
                print(f" {nowtime} ,saved motion warning image!!!")

            # 画出等高线（除错用）
            cv2.drawContours(img_copy, cnts, - 1, (0, 255, 255), 2)

            font = cv2.FONT_HERSHEY_SIMPLEX
            position = (1, 50)  # 左上角坐标
            font_size = 0.7
            color = (255, 255, 255)  # 白色
            thickness = 2
            date_once = now.strftime("%Y-%m-%d") + " " + now.strftime("%H:%M:%S")
            cv2.putText(img_copy, date_once, position, font, font_size, color, thickness)

            if if_write:
                if writer is None:
                    fourcc = cv2.VideoWriter_fourcc('m', 'p', '4', 'v')  # 编码参数
                    write_path = "videos_record" + "/" + now.strftime("%Y%m%d-%H'%M'%S") + ".mp4"
                    writer = cv2.VideoWriter(write_path, fourcc, set_fps, (img_copy.shape[1], img_copy.shape[0]))  #
                    start_write_time = time.time()
                if time.perf_counter() > next_write:
                    next_write += 1 / set_fps
                    # print(time.perf_counter(), 'Write picture')
                    writer.write(img_copy)
                if time.time() - start_write_time > 1 * 60 * 60:
                    writer.release()
                    writer = None
            if if_videoshow:
                # 显示侦测结果影像
                cv2.imshow(' frame ', img_copy)

                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break

            # 更新平均影像
            cv2.accumulateWeighted(blur, avg_float, 0.01)
            avg = cv2.convertScaleAbs(avg_float)

        except queue.Empty as e:
            pass
        except Exception as e:
            print(traceback.format_exc())
        except KeyboardInterrupt as e:
            print('KeyboardInterrupt')
            break
    if writer:
        writer.release()
    print('Waiting for recv thread to end')
    runing = True
    thread.join()
    cv2.destroyAllWindows()


if __name__ == "__main__":
    listen_ip = "192.168.x.xxx" # 上位机IP地址
    target_ip = "192.168.x.xxx" # ESP-CAM IP地址
    if_videoshow = True # 是否开始实时显示
    if_fullscreen = False # 是否全屏显示
    if_write = True # 是否开启写入
    set_fps = 30  # 视频文件写入的FPS
    grab_all = False  # False是取队列所有
    """
        文件存储相关配置
    """
    file_detect_interval = 1 * 60 * 60  # 定时检查清理文件夹内容
    keep_recordvideo_nums = 7 * 24  # 保留记录视频的个数，1h1个
    keep_motionjpg_nums = 10000  # 保留移动侦测图像的数量

    main()

```



## 文末

ESP-CAM模块建议改成外置IPEX天线进行传输，板载的PCB天线受干扰很大，而且具有指向性，被遮挡太多会影响视频的传输，而外置天线无论是增益还是稳定性都提高太多。

![20190811110257471](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/20190811110257471.76rw59wshec0.webp)

---

***致敬***

科幻起源于想象，会预见未来，流浪地球2中550W超级计算机，说不定就会在若干年后成为现实，这个时代最大的魔法似乎就是即将实现的科学技术。

墨菲定律并非指的是那些变坏的事情必会发生…...而是指那些能够发生的事情，就会发生。

这次的ESP模块经过若干天测试，可以连续稳定的不间断运行，这也完成了本文的主题，简单好用，并且有那么一点点的智能，具有极强的可扩展性，👀。



