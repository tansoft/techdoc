## misc
- GPIO4脚可作为+5v输入，再找个地接上，可作为外部电源供电
- GPIO2脚接的是屏幕的5v
- 萤石摄像头流：

```
添加管理员用户：
打开老版本的萤石云PC端软件（注意一定要是老版本，新版本没这个选项，本人的版本号是2.2.4），点击左下角的“设备管理”，在弹出的窗口中找到X5C，点击“高级配置”
在弹出的菜单中点击“用户”→“添加”，在弹出的新菜单中填入密码、IP地址和X5C的MAC地址，填好之后点击“应用”。这里要多说一句，X5C的IP地址一定要设置为静态IP，不能是DHCP自动获取，否则IP一变就要重新配置。
ffmpeg -rtsp_transport tcp -re -i rtsp://用户名:密码@IP地址:554/MPEG-4/ch1/main/av_stream -pred 1 -q:v 2
```

## misclink
- https://post.smzdm.com/p/561232/ 米家智能扫地机器人接入树莓派Domoticz

## HomeXX
- HomeAssistant 是一个程序，是智能家居的平台。它有一个界面，就是我们输入ip地址后看到的，可以用于控制智能设备。你所能看到的那些界面，就是HomeAssistant的界面，它可以集中化接入DIY设备和市面上的很多设备（譬如小米全家桶、博联系设备、亚马逊echo、飞利浦HUE、奔驰、特斯拉等汽车.......）
- HomeKit 是苹果设备的“家庭”程序，是一个 iOS App，这个用过iPhone的应该都知道了，可以通过Siri控制相应的智能设备，但是仅限于iOS10以上版本的苹果设备使用。
- Homebridge 是把非原生HomeKit支持的设备虚拟成HomeKit设备的程序，使这些设备可以被HomeKit控制，它只有命令行界面。
- homeassistant-homebridge 是一个打通 HomeAssistant 和 Homebridge的桥，类似于n鹊桥，这个我们上篇已经安装了智能家居DIY老司机手把手带你hassbian个性化配置 。
- habridge 是可以把我们做好的HomeAssistant中的设备虚拟成另一些类型的设备的程序，以便接入智能音箱，有web界面，通过智能音箱中文来控制智能设备，这个我们上篇也已经安装了。
- Hassbian 和 Hass.io都是集成了HomeAssistant的系统镜像，不同的是Hass.io是 HomeAssistant 官方为树莓派用户专门准备的傻瓜化系统，可以避免初期繁琐的环境搭建和后期添加功能时的手动操作，使小白也能轻松地享受智能家居的乐趣。但是其傻瓜化、封闭的特性也会造成后期操作的不便，因此建议及早换回 HomeAssistant，所以这也是为什么我介绍树莓派安装Hassbian的原因，当然了最好的还是docker安装HomeAssistant，这个进阶操作，暂时不讨论。
- hassdocker 镜像 lroguet/rpi-home-assistant

## Raspbian
### 烧录
- Mac下使用 https://www.balena.io/etcher/ 进行烧录
- 需要在根目录下放置空文件ssh，以启动ssh服务
- 连接网线并通过路由器找到ip连接，pi/raspberry，也可以通过```arp -a ```发现设备ip

- sudo raspi-config 进行基础配置，Advanced Options里可以启动vnc
- 直接在 /etc/wpa_supplicant/wpa_supplicant.conf 增加wifi设置不知道是否可行

```
network={
ssid="first_wifi"
psk="pass"
priority=5
}

network={
ssid="secord_wifi"
psk="pass"
priority=4
}
```

### 国内源

```
nano /etc/apt/sources.list
#注释掉原来的源
deb http://mirrors.aliyun.com/raspbian/raspbian/ jessie main non-free contrib
deb-src http://mirrors.aliyun.com/raspbian/raspbian/ jessie main non-free contrib
#Ctrl+O，回车，Ctrl + X，保存退出
sudo apt-get update && apt-get upgrade -y
```

### 中文字库和输入法
```
sudo apt-get install ttf-wqy-microhei ttf-wqy-zenhei xfonts-wqy
sudo apt-get install scim-pinyin
```

### 支持exFAT和NTFS
```
sudo apt-get install exfat-fuse fuse-utils ntfs-3g
```

### nodejs
```
curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### docker
注意，支持运行的是arm镜像

```
curl -sSL https://get.docker.com | sh
#提权，docker执行不需要sudo
sudo usermod -aG docker $USER
```
方法二：

```
#安装HTTPS所依赖的包
sudo apt-get install apt-transport-https ca-certificates software-properties-common
#添加Docker的GPG key
curl -fsSL https://yum.dockerproject.org/gpg | sudo apt-key add -
#验证key id:
apt-key fingerprint 58118E89F3A912897C070ADBF76221572C52609D
#设置稳定的repository:
sudo add-apt-repository \
       "deb https://apt.dockerproject.org/repo/ \
       raspbian-$(lsb_release -cs) \
       main"
----------------------
#如果 add-apt-repository 命令遇到问题，可以尝试将下面这行添加到树莓派软件源 sources.list，操作如下：
sudo nano /etc/apt/sources.list 添加一行
deb https://apt.dockerproject.org/repo/ raspbian-RELEASE main
----------------------
sudo apt-get update
sudo apt-get -y install docker-engine
#重启 systemctl 守护进程
sudo systemctl daemon-reload
#设置 Docker 开机启动
sudo systemctl enable docker
#开启 Docker 服务
sudo systemctl start docker
```

### docker portainer
Docker 图形化界面 portainer

```
docker pull registry.docker-cn.com/portainer/portainer:linux-arm-1.14.0
//or
docker pull portainer/portainer

mkdir -p ~/portaniner/data
docker run -d --name portainer --restart unless-stopped -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v ~/portaniner/data:/data portainer/portainer

//or
#创建 portainer 容器
docker volume create portainer_data
#运行 portainer
docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

### 配置移动硬盘自动休眠
```
sudo apt-get install hdparm
sudo nano /etc/rc.local
#在fi后和exit 0前插入一行命令：hdparm -S 60 /dev/sdb1  &
#立即休眠sdb1硬盘
sudo hdparm -Y /dev/sdb1
#在60/12=5min后休眠硬盘
sudo hdparm -S 60 /dev/sdb1  
```

### 支持airplay音箱
```
sudo apt-get install autoconf automake libtool                                                                                 sudo apt-get install libltdl-dev libao-dev libavahi-compat-libdnssd-dev                                             sudo apt-get install avahi-daemon
git clone https://github.com/juhovh/shairplay.git
cd shairplay
./autogen.sh
./configure
sudo make install

shairplay -a Shairplay

#调整音量
alsamixer

```

如果你遇到音箱有莫名噪音的问题，除了使用共地滤波器解决一下之外，还可以尝试更改 Audio 的 PWM 模式，修改为图中样子（没有的话就新增），之后重启一下就好了。

### 开机启动
```
cd /etc/init.d/
sudo touch shairplay
sudo nano shairplay

# Source function library.
. /lib/lsb/init-functions

DAEMON="/usr/local/bin/shairplay"
DAEMON_ARGS="-a Wohnzimmer"  # 这里的 Wohnzimmer 可以替换成你想要的音箱名称
AIRPORT_KEY_DIR="/usr/local/share/shairplay"

[ -x $binary ] || exit 0

RETVAL=0

start() {
 echo -n "Starting shairplay: "
 start-stop-daemon --start --quiet --chdir $AIRPORT_KEY_DIR \
 --exec "$DAEMON" -b --oknodo -- $DAEMON_ARGS
 log_end_msg $?
}

stop() {
 echo -n "Shutting down shairplay: "
 start-stop-daemon --stop --quiet --exec "$DAEMON" \
 --retry 1 --oknodo
 log_end_msg $?
}

restart() {
 stop
 sleep 1
 start
}

case "$1" in
 start)
 start
 ;;
 stop)
 stop
 ;;
 status)
 status shairplay
 ;;
 restart)
 restart
 ;;
 *)
 echo "Usage: $0 {start|stop|status|restart}"
 ;;
esac
exit 0


chmod +x /etc/init.d/shairplay
update-rc.d shairplay defaults
sudo mkdir /usr/local/share/shairplay
sudo cp shairplay/airport.key /usr/local/share/shairplay

#确认已经安装screen
nano /etc/rc.local

# Don't run multiple instances - start just one screen, named "shairplay":
[[ $(screen -list | grep shairplay) == '' ]] &&
 screen -dmS shairplay sh
# Keep shairplay perpetually running. When it crashes, we can just SIGKILL it, and it comes back:
[[ $(ps aux | grep -v grep | grep pts | grep '/usr/bin/shairplay') == '' ]] &&
 screen -S shairplay -p 0 -X stuff "while true; do /usr/bin/shairplay --apname=Airamaplay --ao_devicename=default; sleep 2s; done
"

```

### 界面程序开机自动启动
```
#新建autostart目录，在此目录下的命令会随开机自动启动
mkdir ~/.config/autostart
nano ~/.config/autostart/browserAuto.desktop

[Desktop Entry]
Type=Application
Exec=/usr/bin/chromium-browser --kiosk ~/pi-info/index.html
Hidden=false
X-GNOME-Autostart-enabled=true
Name=AutoBrowser
```

### 禁用sleep模式
```
sudo nano /etc/lightdm/lightdm.conf
#[SeatDefaults]里加入
xserver-command=X -s 0 dpms
```

### 全屏时隐藏鼠标
```
sudo apt-get install unclutter
nano ~/.config/autostart/unclutterAuto.desktop
[Desktop Entry]
Type=Application
Exec=unclutter -idle 0.1
```

### firefox
chromium虽说蛮好的(3B自带)，可有时候会弹出错误信息或直接崩溃掉，推荐用firefox替代
```
sudo apt-get install iceweasel
```

### crontab
```
sudo crontab -e
#每天重启一下浏览器
1 3 */3 * * killall firefox-esr
2 3 */3 * * export DISPLAY=:0 && firefox /home/pi/pi-info/index.html
```

### 蓝牙
```
sudo apt-get install libglib2.0-dev libdbus-1-dev libical-dev libreadline-dev libudev-dev
cd /home/pi
wget http://www.kernel.org/pub/linux/bluetooth/bluez-5.44.tar.gz
tar -xvf bluez-5.44.tar.gz
cd bluez-5.44
sudo ./configure--prefix=/usr --sysconfdir=/etc --localstatedir=/var --enable-tools --disable-test --disable-systemd --enable-deprecated
sudo make all
sudo apt-get install python-bluez python-requests
sudo cp attrib/gatttool /usr/bin/
export PATH=$PATH:~/bluez-5.44/attrib/
#操作
cd ~/bluez-5.44
sudo tools/btmgmt le on
sudo tools/btmgmt connectable on
sudo tools/btmgmt power on
sudo hciconfig hci0 down
sudo hciconfig hci0 up
hciconfig
sudo hcitool lescan
```

### i2c
```
#注意Advance Settings里面允许i2c开启
apt-get install i2c-tools
#安装好后运行，发现i2c上的设备
i2cdetect -y 1 #树莓派1应该是改成0
```

### 人脸识别
https://github.com/hirohe/facerec-python

```
sudo apt-get install build-essential cmake pkg-config python-dev libgtk2.0-dev libgtk2.0 zlib1g-dev libpng-dev libjpeg-dev libtiff-dev libjasper-dev libavcodec-dev swig unzip
#启用v4l2
sudo nano /etc/modules
# 增加一行记录
bcm2835-v4l2
# 重启后可以找到/dev/video0
# 编译v4l2-util
apt-get install autoconf gettext libtool libjpeg8 libjpeg8-dev
git clone git://git.linuxtv.org/v4l-utils.git
cd v4l-utils/
sudo ./bootstrap.sh
./configure
make
sudo make install
#编译OpenCV 2.4.9
wget https://jaist.dl.sourceforge.net/project/opencvlibrary/opencv-unix/2.4.9/opencv-2.4.9.zip
unzip opencv-2.4.9.zip
cd opencv-2.4.9/
cmake -DCMAKE_BUILD_TYPE=RELEASE -DCMAKE_INSTALL_PREFIX=/usr/local -DBUILD_PERF_TESTS=OFF -DBUILD_opencv_gpu=OFF -DBUILD_opencv_ocl=OFF
# 要使OpenCV开启对v4l2的支持 cmake之后要有以下输出
# V4L/V4L2: Using libv4l (ver 1.13.0)
sudo make
sudo make install
#安装PyQt4
sudo apt-get install python-qt4
```

### opencv
```
sudo apt-get install build-essential
sudo apt-get install cmake git libgtk2.0-dev pkg-config libavcodec-devlibavformat-dev libswscale-dev
#可选包
sudo apt-get install python-devpython-numpy libtbb2 libtbb-dev libjpeg-dev libpng-dev libtiff-devlibjasper-dev libdc1394-22-dev
wget -O opencv-2.4.13.zip http://sourceforge.net/projects/opencvlibrary/files/opencv-unix/2.4.13/opencv-2.4.13.zip/download
unzip opencv-2.4.13.zip
cd ~/opencv
mkdir release
cd release
cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=/usr/local ..
make
sudo make install
#安装约需要6小时
sudo ldconfig
#查看版本
pkg-config --modversion opencv
#测试
cd ~/opencv-2.4.13/release/bin
./opencv_test_core
```

### opencv for python
```
sudo rpi-update
sudo apt-get install libopencv-dev libpng3 libdc1394-22-dev build-essential libpnglite-dev libdc1394-22 libavformat-dev zlib1g-dbg libdc1394-utils x264 zlib1g libv4l-0 v4l-utils zlib1g-dev libv4l-dev ffmpeg pngtools libpython2.7 libcv2.4 libtiff4-dev python-dev libcvaux2.3 libtiff4 python2.7-dev libhighgui2.4 libtiffxx0c2 libgtk2.0-dev python-opencv libtiff-tools libpngwriter0-dev opencv-doc libjpeg8 libpngwriter0c2 libcv-dev libjpeg8-dev libswscale-dev libcvaux-dev libjpeg8-dbg libjpeg-dev libhighgui-dev libavcodec-dev libwebp-dev python-numpy libavcodec53 libpng-dev python-scipy libavformat53 libtiff5-dev python-matplotlib libgstreamer0.10-0-dbg libjasper-dev python-pandas libgstreamer0.10-0 libopenexr-dev python-nose libgstreamer0.10-dev libgdal-dev libeigen3-dev libxine1-ffmpeg python-tk libgtkglext1-dev libxine-dev python3-dev libpng12-0 libxine1-bin python3-tk libpng12-dev libunicap2 python3-numpy libpng++-dev libunicap2-dev
sudo apt-get install python-opencv
```

### nextcloud
使用snap来安装
```
sudo snap install nextcloud
#启用https，使用lets-encrypt免费证书
sudo /snap/bin/nextcloud.enable-https lets-encrypt
#启用自签名https
sudo /snap/bin/nextcloud.enable-https self-signed
#使用已有https证书
sudo /snap/bin/nextcloud.enable-https custom 证书 私钥 证书链
```

### 修改竖屏显示
```
nano /boot/config.txt
hdmi_cvt=800 480 60 6
hdmi_group=2
hdmi_mode=87
# 设置屏幕旋转角度
display_rotate=3
```

### 系统自带程序
- alsamixer 声音管理

### misc tools
sudo apt-get install mplayer
sudo apt-get install mpg123

## centos
### img烧录
```
#Minimal表示最小化(无GUI)，RaspberryPI表示树莓派定制版。
http://mirror.centos.org/altarch/7/isos/armhfp/
xz -d  /Users/holmesian/Downloads/CentOS-Userland-7-armv7hl-RaspberryPI-Minimal-1804-sda.raw.xz
diskutil list
# e.g. /dev/diskN
diskutil unmountDisk /dev/diskN
sudo dd if=/path/to/downloaded.img of=/dev/rdiskN bs=1m 
/dev/rdiskN 比 /dev/diskN 写入更快
如果1m报错，尝试1M
sudo sync
#只支持pi2 pi3
xzcat CentOS-Userland-7-armv7hl-sda.raw.xz | sudo dd of=/dev/rdiskN bs=4m
sudo sync
diskutil eject /dev/diskN
```

### 初始化
```
root / centos
#SD卡分区扩展
touch /.rootfs-repartition
systemctl reboot
#免sudo
vi sudo
root ALL=(ALL) NOPASSWD: ALL
#增加源，鉴于两个都是没有质保的，且支持armhfp的CentOS源是在是太少，请需要的自己选择
vi /etc/yum.repos.d/epel.repo
[epel]
name=Epel rebuild for armhfp
baseurl=https://armv7.dev.centos.org/repodir/epel-pass-1/
enabled=1
gpgcheck=0
#其实是一个基于fedora aarch64版的源，所以可以将epel.repo的内容改为上海交大的源：
[epel]
name=Extra Packages for Enterprise Linux 7
baseurl=http://ftp.sjtu.edu.cn/fedora/epel/7/aarch64/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
#test yum
yum install -y gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel
#python3 gevent
yum install python34 python34-devel python34-pip
pip3 install -U setuptools
pip3 install -U pip
pip3 install gevent
#wifi支持
curl --location https://github.com/RPi-Distro/firmware-nonfree/raw/54bab3d6a6d43239c71d26464e6e10e5067ffea7/brcm80211/brcm/brcmfmac43430-sdio.bin > /usr/lib/firmware/brcm/brcmfmac43430-sdio.bin
curl --location https://github.com/RPi-Distro/firmware-nonfree/raw/54bab3d6a6d43239c71d26464e6e10e5067ffea7/brcm80211/brcm/brcmfmac43430-sdio.txt > /usr/lib/firmware/brcm/brcmfmac43430-sdio.txt
reboot 
#输入下列命令，用来查看WiFi和连接WiFi
nmcli  d
#查看周围的wifi
nmcli  d  wifi
#连接wifi
nmcli d wifi connect yourSSID password 'yourpassword'  #查看wlan0的状态
nmcli d  show wlan0
#设置网络配置信息，0000是wifi的名字
vi etc/sysconfig/network-script/ifcfg-0000
BOOTPROTO=static              #静态IP
IPADDR=192.168.0.160       #IP地址
GATEWAY=192.168.1.1    #默认网关
NETMASK=255.255.255.0  #子网掩码
#更新系统
yum update && yum upgrade -y
#修改时区
yum install -y ntp
systemctl enable ntpd 
systemctl start ntpd
cp /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
```

### 编译工具
```
yum group install “Development Tools”
yum install make
git clone git://git.drogon.net/wiringPi
./build
gcc -Wall -o blink blink.c -lwiringPi
```

