---
title: "onvif+rtsp获取布控球视频流"
date: 2022-10-18T21:42:28+08:00
draft: false

---

# 背景

最近正在做UWB与视觉融合定位相关的项目，使用的是购买的成品布控球设备，机芯型号未知，向客服协商后同意开放机芯的onvif给我们获取视频流，以供后续开发使用。

![布控球](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/布控球.5zb78wj78ks0.webp)

# 配置

最终是要部署在带算力的边端开发板上面，目前身边的电脑跑着Windows系统，先调通下，后续移植到Linux开发板上面。代码语言优先使用Python，便于与后面目标检测相关的代码做集成，并且个人目前觉得Python是最好用的语言，执行性能的问题获取对于目前的我来说不是问题。

这款布控球个人理解应该是套了个壳子，机芯或许是海康威视或者大华等，他应该基于机芯做了防水防尘加密等等一系列符合施工现场标准的集成。

在接入网线后，查询发现云台显示屏上显示的IP地址与ipconfig中的不同。

![截图](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/截图.2rbjfm1squw0.webp)

第一步便是先更改下IP地址，保证机器与电脑处于统一网段内。

![截图](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/截图.5k5pl3vpbt40.webp)

确认后，查询ipconfig发现处于统一网段，这时候按照布控球的使用说明配置一系列IE浏览器的操作，输入后台地址便可以进入后台管理界面，检查摄像头正常后，下一步是重点，即通过onvif取流。

个人理解onvif应该是摄像头的一个管理控制协议，本身并不具备直接获取视频流的功能，操作应该如下：onvif配置布控球--获取rtsp视频流地址--opencv解码--显示

首先需要进行的是路由机芯的IP地址，如下图，配置页面系统里面这个应该是内置设备的，也就是机芯，我从外面访问的话应该还有一个开放的IP和端口？

![截图](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/截图.2hickvjd4f00.webp)

后来询问后知道是通过加路由实现，就是说设置一个静态路由，192.168.0.99 指向 192.168.253.64,其中后者为机芯的地址。管理员模式CMD下输入下面的语句即可添加临时的路由，重启后会消失，-p可以添加永久的。

```route add 192.168.253.64 mask 255.255.255.255 192.168.0.99```

![截图](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/截图.6ge3xah4w3w0.webp)

到此为止，配置结束。

# onvif+rtsp+opencv

Python中使用[python-onvif-zeep](https://github.com/FalkTannhaeuser/python-onvif-zeep)库，支持Python3进行onvif的控制，详细使用仓库都有，仓库没有Issue找，再次感叹Python太简洁强大了，大大缩短开发时间和周期。

安装完[python-onvif-zeep](https://github.com/FalkTannhaeuser/python-onvif-zeep)后，配置代码如下:

```python
from onvif import ONVIFCamera
import zeep

IP = '192.168.253.64'
USR = 'admin'
PSS = 'XXX'


def zeep_pythonvalue(self, xmlvalue):
    return xmlvalue

# 获取视频流地址
def geturi():
    # Get target profile
    zeep.xsd.simple.AnySimpleType.pythonvalue = zeep_pythonvalue
    mycam = ONVIFCamera(IP, 80, USR, PSS)
    media_service = mycam.create_media_service()
    profiles = media_service.GetProfiles()
    token = profiles[0].token
    obj = media_service.create_type('GetStreamUri')
    obj.ProfileToken = token
    obj.StreamSetup = {'Stream': 'RTP-Unicast', 'Transport': {'Protocol': 'RTSP'}}
    print(media_service.GetStreamUri(obj))


if __name__ == '__main__':
    geturi()
```

成功后便会打印出来对应的rtsp视频流地址：

![截图](https://cdn.staticaly.com/gh/zhendehanzi/Blog-Images-1@master/截图.o3hfoahvnk0.webp)

之后使用OpenCV直接进行读取显示即可，其中注意rtsp的格式，进行认证获取就好，用户名：密码对应前面图中后台管理中的，如果不知道需要向客服询问：

```python
import cv2

# 'rtsp://【用户名】:【密码】@【IP地址】:【端口号（rtsp协议默认554）】/h264/【视频通道（如ch2）】/main/av_stream')  # 获取视频流,opencv
cap = cv2.VideoCapture(
    'rtsp://admin:xxx@192.168.253.64:554/Stream/Live/101?transportmode=unicast&profile=ONFProfileToken_101')  #
# 获取视频流,opencv
print(cap)
ret, frame = cap.read()
while ret:
    ret, frame = cap.read()
    cv2.imshow("WIN_NAME", frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break
cv2.destroyAllWindows()
cap.release()

```

到此取流基本结束！
