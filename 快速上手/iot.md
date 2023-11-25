
# ESP

## 乐鑫科技

### ESP 8266

* ESP01S：只有两组io针脚（8针），体积最小，需要对接8针烧录器。
* ESP12E/F/S：Flash大小不一样，只有针脚，烧录需要焊线。
* ESP8266nodeMCU：集成了usb烧录等模块，可以直接用arduino来烧录。

#### 烧录

* 使用 https://www.arduino.cc/ 烧录。Mac版本需要自行安装串口驱动，看芯片一般以下几种：
 * ch340/ch9120 https://www.wch.cn/
 * cp2102 https://www.silabs.com/
* 安装ESP8266 sdk，使 arduino 支持。
 * 首选项->附加开发版管理器地址，https://www.arduino.cn/package_esp8266com_index.json
 * 工具->开发版->开发版管理器->输入esp8266->选择版本->安装
* 安装点灯科技的库，这样可以使用点灯app进行操作 https://diandeng.tech/dev
 * SDK 放到 文稿/Arduino/libraries
 * 可看到 项目->加载库->Blinker
