#  

# FreeSwitch 1.10 源码安装

参考官方文档：`https://freeswitch.org/confluence/display/FREESWITCH/CentOS+7+and+RHEL+7`

`https://blog.csdn.net/tidehc/article/details/86593130#mod_avlib_70`

[TOC]

## 一、环境

- 操作系统：`CentOS 7.6.1801`
- FreeSwitch版本：`v1.10`

## 二、初始设置

1. 关闭防火墙
2. 关闭 SELinux

## 三、安装依赖软件包

```bash
# yum install -y https://files.freeswitch.org/repo/yum/centos-release/freeswitch-release-repo-0-1.noarch.rpm epel-release	--> 安装依赖软件包的 yum 源

# yum install alsa-lib-devel autoconf automake bison broadvoice-devel codec2-devel e2fsprogs-devel erlang flite-devel g722_1-devel gdbm-devel gnutls-devel ilbc2-devel lame-devel ldns-devel libcurl-devel libdb4-devel libedit-devel libjpeg-turbo-devel libks libmemcached-devel libogg-devel libshout-devel libsilk-devel libsndfile-devel libtheora-devel libtiff-devel libtool libvorbis-devel libxml2-devel libyuv-devel lua-devel mpg123-devel net-snmp-devel opus-devel perl-ExtUtils-Embed portaudio-devel  python-devel signalwire-client-c soundtouch-devel speex-devel sqlite-devel yasm libatomic gcc-c++ libuuid-devel unixODBC-devel





# yum install -y yum-plugin-ovl centos-release-scl rpmdevtools yum-utils git

# yum-builddep -y freeswitch	--> 安装 freeswitch 需要的依赖

# yum install -y devtoolset-4-gcc*
```

## 四、mod_av

要支持 H264 视频通话，需要单独安装

mod_av 依赖 libav，libav 需要 x264 lib 才能支持 H264，x264 依赖 nasm(系统自带的版本太低，需要大于 2.13 版才行)，所以安装顺序是 nasm -> x264 -> libav

```bash
1. 安装 nasm
# cd /usr/local/src
# wget https://www.nasm.us/pub/nasm/releasebuilds/2.14.02/nasm-2.14.02.tar.xz
# tar xf nasm-2.14.02.tar.xz
# cd nasm-2.14.02
# ./configure
# make
# make install

2. 安装 x264
# wget ftp://88.191.250.2/pub/x264/snapshots/last_x264.tar.bz2
# tar xf last_x264.tar.bz2
# cd x264-snapshot-20190813-2245
# ./configure --enable-shared --enable-static --disable-opencl
# make
# make install
# ln -sf /usr/local/lib/pkgconfig/x264.pc /usr/lib64/pkgconfig/x264.pc

3. 安装 libav
# wget https://libav.org/releases/libav-12.3.tar.xz
# tar xf libav-12.3.tar.xz
# cd libav-12.3
进入 libav 源码目录下, 将 libavcodec/libx264.c 文件里面的 "x264_bit_depth" 全部替换为 "X264_BIT_DEPTH"，否则编译会报错。
# ./configure --enable-shared --enable-libx264 --enable-gpl
# make
# make install
# ln -sf /usr/local/lib/pkgconfig/libavcodec.pc  /usr/lib64/pkgconfig/libavcodec.pc
# ln -sf /usr/local/lib/pkgconfig/libavdevice.pc  /usr/lib64/pkgconfig/libavdevice.pc
# ln -sf /usr/local/lib/pkgconfig/libavfilter.pc  /usr/lib64/pkgconfig/libavfilter.pc
# ln -sf /usr/local/lib/pkgconfig/libavformat.pc  /usr/lib64/pkgconfig/libavformat.pc
# ln -sf /usr/local/lib/pkgconfig/libavresample.pc  /usr/lib64/pkgconfig/libavresample.pc
# ln -sf /usr/local/lib/pkgconfig/libavutil.pc  /usr/lib64/pkgconfig/libavutil.pc
# ln -sf /usr/local/lib/pkgconfig/libswscale.pc  /usr/lib64/pkgconfig/libswscale.pc

将 /usr/local/lib 加入系统库文件搜索路径
# cat /etc/ld.so.conf.d/locallib.conf 
/usr/local/lib
# ldconfig
```

## 五、mod_signalwire

安装顺序： libatomic -> cmake -> libks -> signalwire

```shell
1. 安装 libatomic
# yum install libatomic -y

2. 安装高版本 cmake，系统自带的版本太低
# wget https://github.com/Kitware/CMake/releases/download/v3.15.2/cmake-3.15.2.tar.gz
# tar xf cmake-3.15.2.tar.gz
# cd cmake-3.15.2
# ./bootstrap 
# gmake
# make install

3. 安装 libks
# git clone https://github.com/signalwire/libks.git
# cd libks
# cmake .
# make
# make install

4. 安装 signalwire-c
# git clone https://github.com/signalwire/signalwire-c.git
# cd signalwire-c
# cmake .
# make
# make install
```

## 六、安装 FreeSwitch

```shell
# git clone -b v1.10 https://freeswitch.org/stash/scm/fs/freeswitch.git freeswitch
# cd freeswitch
# ./bootstrap.sh -j
修改源码目录中的 modules.conf 文件，取消 "#say/mod_say_zh" 这行的注释 "say/mod_say_zh"，使其支持中文语音，取消 "#applications/mod_distributor" 注释，使其支持多线路选线功能
# ./configure --prefix=/usr/local/freeswitch --enable-portable-binary --with-gnu-ld --with-python --with-erlang --with-openssl --enable-core-odbc-support --enable-zrtp
# make
# make -j install
# make -j cd-sounds-install
# make -j cd-moh-install
```

## 七、安装中文语音

freeswitch 默认不加载中文语音。需要在 freeswitch 的 src 中首先编译中文模块。

1. 在 configure 之前，编辑 modules.conf，取消 "#say/mod_say_zh" 这行的注释 "say/mod_say_zh"
2. 补救安装
   - `cd /usr/local/src/freeswitch/src/mod/say/mod_say_zh`
   - `make && make install`

修改 `/usr/local/freeswitch/etc/freeswitch/vars.xml`

```shell
 <X-PRE-PROCESS cmd="set" data="sound_prefix=$${sounds_dir}/en/us/callie"/> 修改为
 <X-PRE-PROCESS cmd="set" data="sound_prefix=$${sounds_dir}/zh/cn/link"/>
```

复制修改中文 say 的配置

```bash
1. # cd /usr/local/freeswitch/etc/freeswitch/lang
2. # cp -fr en/ zh
3. # cd zh
4. # mv en.xml zh.xml
5. 修改 zh.xml
<language name="en" say-module="en" sound-prefix="$${sound_prefix}" tts-engine="cepstral" tts-voice="callie"> 修改为
<language name="zh" say-module="zh" sound-prefix="$${sound_prefix}/zh/cn/link" tts-engine="mod_tts_commandline" tts-voice="link">
```

添加中文配置

```bash
1. # cd /usr/local/freeswitch/etc/freeswitch
2. 在 <section name="languages" description="Language Management"> 下新增一行
<X-PRE-PROCESS cmd="include" data="lang/zh/*.xml"/>
```

打开自动加载中文模块的配置

```shell
1. # cd /usr/local/freeswitch/etc/freeswitch/autoload_configs/modules.conf.xml
2. 取消 <load module="mod_say_zh"/> 这行的注释
```

将中文语音包上传到 `/usr/local/freeswitch/share/freeswitch/sounds/zh/cn/link` 目录下

## 八、开启视频通话功能

1. 编辑 `/usr/local/freeswitch/etc/freeswitch/vars.xml`

   ```bash
   <X-PRE-PROCESS cmd="set" data="global_codec_prefs=OPUS,G722,PCMU,PCMA,H264,VP8"/>  
   <X-PRE-PROCESS cmd="set" data="outbound_codec_prefs=OPUS,G722,PCMU,PCMA,H264,VP8"/>
   ```

2. 编辑 `/usr/local/freeswitch/etc/freeswitch/sip_profiles/internal.xml`

   ```shell
   将 <param name="inbound-proxy-media" value="true"/> 这行注释去掉
   ```

3. 加载 mod_av 模块

   启动 freeswitch 后，在 fs_cli 控制台中输入：`load mod_av`

   开启启动 freeswitch 自动加载 mod_av 方法：

   修改`/usr/local/freeswitch/etc/freeswitch/autoload_configs/modules.conf.xml` 文件

   取消注释 `<load module="mod_av"/>`

## 九、IP 地址设置

修改`/usr/local/freeswitch/etc/freeswitch/vars.xml`

```bash
<X-PRE-PROCESS cmd="stun-set" data="external_sip_ip=x.x.x.x"/>
<X-PRE-PROCESS cmd="stun-set" data="external_rtp_ip=x.x.x.x"/>
设置为公网 IP
```

## 十、 freeswitch.service 设置

```shell
# vim /usr/lib/systemd/system/freeswitch.service
[Unit]
Description=FreeSWITCH
After=syslog.target network.target
#After=postgresql.service postgresql-9.3.service postgresql-9.4.service mysqld.service httpd.service

[Service]
User=freeswitch
#EnvironmentFile=-/etc/sysconfig/freeswitch
# RuntimeDirectory is not yet supported in CentOS 7. A workaround is to use /etc/tmpfiles.d/freeswitch.conf
#RuntimeDirectory=/run/freeswitch
#RuntimeDirectoryMode=0750
WorkingDirectory=/usr/local/freeswitch/var/run/freeswitch
ExecStart=/usr/local/freeswitch/bin/freeswitch -nc -nf $FREESWITCH_PARAMS
ExecReload=/usr/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

## 十一、fs_cli 连接设置

修改 `/usr/local/freeswitch/etc/freeswitch/autoload_configs/event_socket.conf.xml`

```shell
 <param name="listen-ip" value="::"/> 修改为
  <param name="listen-ip" value="127.0.0.1"/>
```

## 十二、连接 MySQL 数据库

**1、安装 MySQL 连接驱动**

``` bash
# wget https://dev.mysql.com/get/Downloads/Connector-ODBC/8.0/mysql-connector-odbc-8.0.17-1.el7.x86_64.rpm
# yum install mysql-connector-odbc-8.0.17-1.el7.x86_64.rpm
```

**2、编辑 `/etc/odbc.ini` 文件**

```ini
[passmanage_dev]
Description=MySQL SIP Databases
Driver=/usr/lib64/libmyodbc8w.so
SERVER=172.18.219.215
PORT=3307
DATABASE=passmanage_dev
OPTION=67108864
CHARSET=utf8mb4
USER=root
PASSWORD=sfm@2018
Threading=0
```

> 如果提示找不到 socket 文件，则在 `/etc/odbc.ini` 文件里面指定 socket 文件的位置，如：
>
> SOCKET=/data/mysql/3306/logs/mysql.sock

测试是否能够连通数据库：`isqv -v passmanage_dev`

**3、修改配置文件 **

FreeSwitch 默认使用 SQLite 数据库，这里我们将使用 MySQL 数据库来管理 FreeSwitch 中的账户，并实现账户之间相互拔打电话。



修改 `/usr/local/freeswitch/etc/freeswitch/autoload_configs/lua.conf.xml` 文件

```xml
在 <settings> 标签下添加如下内容：
<param name="xml-handler-script" value="/usr/local/freeswitch/share/freeswitch/scripts/gen_dir_user_xml.lua"/>
<param name="xml-handler-bindings" value="directory"/>
```



将 `gen_dir_user_xml.lua` 文件放到`lua.conf.xml` 文件中指定的目录下

```lua
gen_dir_user_xml.lua 内容如下：

freeswitch.consoleLog("NOTICE","lua take the users...\n");
-- gen_dir_user_xml.lua
-- example script for generating user directory XML

-- comment the following line for production:
--freeswitch.consoleLog("notice", "Debug from gen_dir_user_xml.lua, provided params:\n" .. params:serialize() .. "\n")

local req_domain = params:getHeader("domain")
local req_key    = params:getHeader("key")
local req_user   = params:getHeader("user")
local req_password   = params:getHeader("pass")

-- sql safe
local name = string.format("%q",req_user)
local sql_normal = [[select sip_passwd from sip where sip_account=]] .. name .. [[ limit 1;]]

local dbh = freeswitch.Dbh("passmanage_dev","root","xxx");
freeswitch.consoleLog("NOTICE","start connect DB...\r\n");
assert(dbh:connected());
dbh:query(sql_normal ,function(row)
 freeswitch.consoleLog("NOTICE",string.format("%s\n",row.sip_passwd))
 req_password=string.format("%s",row.sip_passwd)
end);
dbh:release();

--freeswitch.consoleLog("NOTICE","info:"..req_domain.."--"..req_key.."--"..req_user.."--"..req_password.."\n");

--assert (req_domain and req_key and req_user,
--"This example script only supports generating directory xml for a single user !\n")
if req_domain ~= nil and req_key~=nil and req_user~=nil then
 XML_STRING =
 [[<?xml version="1.0" encoding="UTF-8" standalone="no"?>
 <document type="freeswitch/xml">
  <section name="directory">
   <domain name="]]..req_domain..[[">
    <params>
     <param name="dial-string" value="{^^:sip_invite_domain=${dialed_domain}:presence_id=${dialed_user}@${dialed_domain}}${sofia_contact(*/${dialed_user}@${dialed_domain})},${verto_contact(${dialed_user}@${dialed_domain})}"/>
     <param name="jsonrpc-allowed-methods" value="verto"/>
     <param name="jsonrpc-allowed-event-channels" value="demo,conference,presence"/>
     </params>
    <groups>
     <group name="default">
     <users>
      <user id="]]..req_user..[[">
       <params>
        <param name="password" value="]]..req_password..[["/>
        <param name="vm-password" value="]]..req_password..[["/>
        <param name="user-agent-string" value="voip" />
       </params>
       <variables>
        <variable name="toll_allow" value="domestic,international,local"/>
        <variable name="accountcode" value="]]..req_user..[["/>
        <variable name="user_context" value="default"/>
        <variable name="directory-visible" value="true"/>
        <variable name="directory-exten-visible" value="true"/>
        <variable name="limit_max" value="15"/>
        <variable name="effective_caller_id_name" value="Extension ]]..req_user..[["/>
        <variable name="effective_caller_id_number" value="]]..req_user..[["/>
        <variable name="outbound_caller_id_name" value="$${outbound_caller_name}"/>
        <variable name="outbound_caller_id_number" value="$${outbound_caller_id}"/>
        <variable name="callgroup" value="techsupport"/>
       </variables>
      </user>
     </users>
     </group>
    </groups>
   </domain>
  </section>
 </document>]]
else
XML_STRING =
 [[<?xml version="1.0" encoding="UTF-8" standalone="no"?>
 <document type="freeswitch/xml">
  <section name="directory">
  </section>
 </document>]]
end

-- comment the following line for production:
freeswitch.consoleLog("notice", "Debug from gen_dir_user_xml.lua, generated XML:\n" .. XML_STRING .. "\n");
```



修改`/usr/local/freeswitch/etc/freeswitch/directory/default.xml` 文件

```xml
删除如下内容：
<!--group name="default">
  <users>
    <xX-PRE-PROCESS cmd="include" data="default/*.xml"/>
  </users>
</group-->
```

> 如果不想删除而使用注释方式的话，需要将`X-PRE-PROCESS` 这种格式破坏掉，比如将它改成`xX-PRE-PROCESS`不然无法注释这一行。



创建数据库表：

```sql
CREATE TABLE `sip` (
  `sip_id` int NOT NULL AUTO_INCREMENT,
  `sip_account` bigint(20) unsigned NOT NULL COMMENT 'SIP 帐号',
  `sip_passwd` varchar(20) NOT NULL DEFAULT '123.123' COMMENT 'SIP 密码',
  `sip_comment` varchar(50) DEFAULT NULL COMMENT '描述',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`sip_id`),
  UNIQUE KEY `uk_sip_account` (`sip_account`)
);
```

