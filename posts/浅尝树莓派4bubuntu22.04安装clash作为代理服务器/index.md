# 浅尝树莓派4B(Ubuntu22.04)安装Clash作为代理服务器


&lt;!--more--&gt;



## 0 前言

树莓派（[Raspberry Pi](https://www.raspberrypi.com/)）是一种基于ARM架构的单板计算机，仅有银行卡大小，本意是提供一种低成本的计算机学习硬件。但前几年由于各种原因，价格一路高歌猛进，让人望而却步。最近逛淘宝发现，树莓派价格差不多回归了正常架构，于是果断下手买一个尝尝。

&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309091517163.jpg&#34; alt=&#34;1694243814807的副本&#34; style=&#34;zoom:50%;&#34; /&gt;



## 1 安装 Ubuntu22.04

到手的是 树莓派4B 4G内存版，因为内存足够，准备安装带GUI界面的 Ubuntu Desktop 22.04 。



### 1.1 烧录操作系统

树莓派的存储是在一张SD卡上，所以需要通过 [Raspberry Pi Imager](https://www.raspberrypi.com/software/) 将操作系统烧录到SD卡中。

不需要提前下载镜像，直接可以在工具中选择系统，然后选中目标SD卡，点击烧录。

```txt
&gt; Other general-purpose OS
  &gt; Ubuntu
    &gt; Ubuntu Desktop 22.04.3 LTS (64-bit)
```

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309091548861.png&#34; alt=&#34;截屏2023-09-09 15.48.11&#34; style=&#34;zoom:50%;&#34; /&gt;&lt;/div&gt;



烧录完成后，将SD卡插入树莓派，通电启动即可。系统安装后，即可通过 micro HMDI 连接到显示设备，并使用键盘、鼠标进行操作。首次启动会引导设置语言时区、用户名主机名等信息，不在此赘述。



### 1.2 Ubuntu软件源修改

树莓派切换镜像源的方式和一般Ubuntu设备并无不同，唯一需要注意的是 **要使用 ubuntu-ports 镜像**，这里面才有arm64的源。

我们使用[清华镜像站](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu-ports/)，页面比较友好，可以直接在页面切换选项并产生配置内容。

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309091610222.png&#34; style=&#34;zoom:50%;&#34; /&gt;&lt;/div&gt;

备份原 `sources.list`，将清华源信息覆写到 `sources.list`，执行 `sudo apt update ` 更新软件包信息。



### 1.3 固定网络IP

为了方便ssh登录，以及作为网络代理服务器，需要设置一个固定的IP地址。

编辑netplan配置文件  `/etc/netplan/01-network-manager-all.yaml`。

```yaml
network:
  version: 2
  renderer: NetworkManager
  # 无线网络配置
  wifis:
    # 无线网卡 wlan0
    wlan0:
      # 关闭dhcp
      dhcp4: false
      # 固定IP， /24 用来表示掩码信息
      addresses: [192.168.31.250/24] 
      optional: true
      # 配置项 gateway4 在新版本中已弃用，通过routes配置网关
      routes:
        - to: default
          via: 192.168.31.1
      # DNS服务
      nameservers:
        addresses: [114.114.114.114]
      # 配置WiFi信息
      access-points:
        &#34;WiFi名称&#34;:
          password: &#39;WiFi密码&#39;
```

配置修改完成后，使用 `sudo netplan try` 测试网络配置，无异常即可使用 `sudo netplan apply` 应用配置。



### 1.4 安装 openssh-server

配置完成后，发现无法ssh访问，发现默认没有安装 openssh-server，通过apt安装。安装后即可远程ssh访问。

```shell
sudo apt install openssh-server
```



## 2 Clash

[Clash](https://github.com/Dreamacro/clash) 是一个基于规则的代理工具，主要的用途不多说了，如果你知道这个软件，就说明你大概率有这个需求。这个安装方式也不限于树莓派设备，只要是运行Linux系统的设备，都可以



这次之所以选择带GUI的Ubuntu，很大一部分原因就是想要通过图形化界面，比较方便的配置Clash。



### 2.1 Clash for Windows

[Clash for Windows](https://github.com/Fndroid/clash_for_windows_pkg/releases)  是比较好用的一个 Clash GUI 客户端。

***&lt;font color=&#39;red&#39;&gt;但是！它这个名字太具有迷惑性了！&lt;/font&gt;***

***&lt;font color=&#39;red&#39;&gt;它其实 支持 Windows系统！也支持 Linux！还支持 macOS ！&lt;/font&gt;***

从GitHub release页面下载，`Clash.for.Windows-0.20.34-arm64-linux.tar.gz`，从它提供的各种安装包也能看出支持多平台

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309091901250.png&#34; alt=&#34;截屏2023-09-09 18.58.02&#34; style=&#34;zoom:55%;&#34; /&gt;&lt;/div&gt;

下载后解压，运行 `cfw`，即可出现GUI界面，然后开始进行配置。

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309091910073.png&#34; alt=&#34;clashforwin-0&#34; style=&#34;zoom:100%;&#34;  /&gt;&lt;/div&gt;

1. 在左侧 **General 菜单**中，**打开 Allow LAN开关，允许局域网连接**。这样不止为本机提供网络代理，也能为局域网其他设备提供代理。
2. 在左侧 **Profiles 菜单**中，**将机场提供的订阅地址填入输入框，下载得到代理规则**。config.yaml为默认配置，sub为订阅到的规则。

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309091928725.png&#34; alt=&#34;202309091916624&#34; style=&#34;zoom:100%;&#34; /&gt;&lt;/div&gt;

到这为止，Clash for Windows就已经准备完毕了，随时可以使用了。



### 2.2 Clash(无图形界面)

如是是无图形界面的情况，可以通过命令安装Clash。

下载Clash，注意版本，这里以树莓派使用的ARM版本为例

```shell
mkdir ~/clash &amp;&amp; cd ~/clash
wget https://github.com/Dreamacro/clash/releases/download/v1.18.0/clash-linux-armv7-v1.18.0.gz
```

解压后重命名

```shell
gunzip clash-linux-armv7-v1.18.0.gz
mv clash-linux-armv7-v1.18.0 clash
```

移动clash到指定位置，并赋予执行权限

```shell
sudo cp clash /usr/local/bin
sudo chmod &#43;x /usr/local/bin/clash
```

初始化执行，自动生成config.yaml 和 Country.mmdb

```shell
sudo /usr/local/bin/clash -d /etc/clash/
# 输出
# INFO[0000] Can&#39;t find config, create a initial config file 
# INFO[0000] Can&#39;t find MMDB, start download              
# INFO[0003] inbound mixed://127.0.0.1:7890 create success. 
```

ctrl&#43;c中断执行，**修改config.yaml**。这个就替换成你机场Clash订阅地址返回的yaml代理站点内容。

**&lt;font color=&#39;red&#39;&gt;需要注意的是，在config.yaml中有几个配置，可能需要自己修改一下，因为机场提供的值可能不是你想要的&lt;/font&gt;**

1. **关于端口**

   代理端口分为 `http代理端口（port）`、`socks代理端口（socks-port）` 和  `混合代理端口 （mixed-port）`。这里建议直接将`混合代理端口 （mixed-port）`设置为7890，其他两项设为0。这样不管是http、socks都是用同一个7890端口，更加方便

2. **关于外部控制**

   Clash是支持通过 RESTful 接口来对Clash进行功能配置的，目前市面上的GUI客户端配置管理也是基于这个方式。

   config.yaml中有一项配置 `external-controller`，修改为 `0.0.0.0:9090` 这样方便我从局域网其他设备对其进行控制。

   还有一项配置 `secret` 主要用于 RESTful 控制接口的身份认证密码。如果是局域网不对外的话，可暂时不配。

3. **clash.razord.top**

   对于没有GUI的Clash，可以在其他设备上通过这个网站，获得一个控制的Dashboard。

   网站打开时会让填写 Host、端口、秘钥，填写后就可以借助这个网页，对无图形界面设备上安装的Clash进行配置。具体原理的话就是页面发起配置对应功能的http请求。



代理配置完后，创建systemd 配置文件 `/etc/systemd/system/clash.service`，让我们的Clash能够开机自启。

```ini
[Unit]
Description=Clash 守护进程, Go 语言实现的基于规则的代理.
After=network-online.target

[Service]
Type=simple
Restart=always
ExecStart=/usr/local/bin/clash -d /etc/clash

[Install]
WantedBy=multi-user.target
```

重新加载 systemd，设置开机启动 和 立即启动

```shell
sudo systemctl daemon-reload
# 设置系统启动时启动clash
sudo systemctl enable clash
# 立即启动clash
sudo systemctl start clash
# 查看clash运行状态
sudo systemctl status clash
```

到这为止，Clash就已经准备完毕了，随时可以使用了。



### 2.3 使用方式

&gt; 如果你使用的设备安装了SSR、V2ray或者Clash，那你可以直接在软件界面上操作，通过界面提供的功能来开启系统代理。

这里主要是介绍一下没有安装代理软件的设备，如何借助局域网内其他已安装的代理设备(即代理服务器)，实现网络代理功能。



#### 2.3.1 Windows 10 设备

在 Window 10 设置 &gt; 网络和Internet &gt; 代理 中，打开【使用代理服务器】开关，地址填树莓派IP地址，端口为Clash使用的7890端口。

也可以设置对 本地地址(localhost)、常见的局域网网段(192.168.x.x) 不使用代理服务器。

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309101055135.png&#34; alt=&#34;VD7~4$~CB7W10@9NIW6HVFH&#34; style=&#34;zoom: 70%;&#34; /&gt;&lt;/div&gt;



#### 2.3.2 手机、iPad等无线设备

进入 无线局域网设置(WLAN设置)，点击已连接的无线网络，将网络详情页面拉到底，有HTTP代理设置，填入树莓派IP地址，Clash的7890端口保存即可。

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309101116742.jpg&#34; alt=&#34;MTXX_PT20230910_111545372&#34; style=&#34;zoom:15%;&#34; /&gt;&lt;/div&gt;



#### 2.3.3 Ubuntu/Linux

如果是带有GUI界面的操作系统，直接在界面上配置，以Ubuntu为例，在 Settings &gt; Network &gt; Network Proxy 中，将代理配置中填入树莓派IP地址和Clash的7890端口保存即可

&lt;div align=center&gt;&lt;img src=&#34;https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/blog/202309101147300.png&#34; alt=&#34;截屏2023-09-10 11.47.19&#34; style=&#34;zoom:60%;&#34; /&gt;&lt;/div&gt;

如果是没有界面的Linux系统，可以通过设置环境变量的方式设置

```shell
export https_proxy=http://192.168.31.250:7890 
export http_proxy=http://192.168.31.250:7890
export all_proxy=socks5://192.168.31.250:7890
```



#### 2.3.4 按需配置

总的来说，我个人并不喜欢为系统配置全局网络代理，我更喜欢**给有需要的程序配置网络代理**。举个几个例子：



**容器运行时containerd**

使用containerd时，有些镜像国内无法拉取，而且国内没有厂商同步这些镜像，这就可以为containerd配置代理服务器，来帮助拉取这些镜像。

```shell
sudo mkdir -p /etc/systemd/system/containerd.service.d/

cat &lt;&lt; EOF | sudo tee -a /etc/systemd/system/containerd.service.d/http-proxy.conf
[Service]
Environment=&#34;HTTP_PROXY=192.168.31.250:7890&#34;
Environment=&#34;HTTPS_PROXY=192.168.31.250:7890&#34;
EOF

sudo systemctl restart containerd
```



**proxychains4**

这是一个很简单、好用的代理工工具。你只需要通过apt安装

```shell
sudo apt install proxychains4
```

安装后修改配置文件  `/etc/proxychains.config`，将文件最后的代理地址改为代理服务器的地址，比如 `socks5 192.168.31.250 7890`即可。

使用方法的话，只需要在原本命令前添加 proxychains4 即可

```shell
proxychains4 wget xxx
proxychains4 curl xxx
proxychains4 git clone xxx
```



---

> 作者:   
> URL: https://dingyufan.github.io/posts/%E6%B5%85%E5%B0%9D%E6%A0%91%E8%8E%93%E6%B4%BE4bubuntu22.04%E5%AE%89%E8%A3%85clash%E4%BD%9C%E4%B8%BA%E4%BB%A3%E7%90%86%E6%9C%8D%E5%8A%A1%E5%99%A8/  

