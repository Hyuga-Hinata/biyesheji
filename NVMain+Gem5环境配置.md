# NVMain+Gem5环境配置

配置：Ubuntu16.04 、g++5、gcc5、gcov5、python2.7.12、pip20.3.3frompython2.7、scons3.1.2

为后面的人铺路吧。

可以说踩了很多坑，因为官网NVMain也没了，就比较麻烦，配置配对就没啥大问题。

主要参考文献：

https://blog.csdn.net/weixin_43914889/article/details/104629254

https://www.jianshu.com/p/af2efe552a11

https://github.com/yyuanye/Gem5-NVMain.git

```
sudo apt-get install mercurial scons swig m4 python-dev libgoogle-perftools-dev libprotobuf-dev

sudo apt install zlib1g
sudo apt install zlib1g-dev

git clone https://github.com.cnpmjs.org/yyuanye/Gem5-NVMain.git
```



## 1.NVMain

###g++/gcc配置：

参考文献：（感谢大佬们的帮助）

https://www.jianshu.com/p/7690bfbcee72

https://linuxize.com/post/how-to-install-gcc-compiler-on-ubuntu-18-04/

更改/etc/apt/sources.list,添加Ubuntu16.04的源，

```bash
sudo vim /etc/apt/sources.list

# deb cdrom:[Ubuntu 16.04 LTS _Xenial Xerus_ - Release amd64 (20160420.1)]/ xenial main restricted
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security multiverse
```

然后进行update，

```bash
sudo apt-get update
apt-cache policy gcc-5
sudo apt-get install gcc-5 g++-5
```

再进行

```bash
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 50 --slave /usr/bin/g++ g++ /usr/bin/g++-5 --slave /usr/bin/gcov gcov /usr/bin/gcov-5
sudo update-alternatives --config gcc
```

###python2.7.12安装

参考文献

https://www.jianshu.com/p/29e4a9d23706

```bash
sudo update-alternatives  --install /usr/bin/python python /usr/bin/python2.7 1
sudo update-alternatives  --install /usr/bin/python python /usr/bin/python3.5 2
sudo update-alternatives --config python
```

https://www.howtoing.com/install-python-2-7-on-ubuntu-and-linuxmint

```bash
cd /usr/src
sudo wget https://www.python.org/ftp/python/2.7.12/Python-2.7.12.tgz
sudo tar xzf Python-2.7.12.tgz
cd Python-2.7.12
sudo ./configure --prefix /usr/local/
sudo make install
```

python 2.7.18

```
sudo apt-get install python
```





###pip2

参考文献

https://linuxize.com/post/how-to-install-pip-on-ubuntu-20.04/

```bash
sudo add-apt-repository universe
curl https://bootstrap.pypa.io/get-pip.py --output get-pip.py
sudo python get-pip.py
sudo update-alternatives  --install /usr/bin/pip pip /usr/bin/pip2 1
sudo update-alternatives  --install /usr/bin/pip pip /usr/bin/pip3 2
sudo update-alternatives --config pip
```

### scons

记住此处的pip一定要是python2的pip

```bash
sudo python -m pip install scons
export PATH=$PATH:/home/hinata/.local/bin或者reboot
```



### 最后

```bash
cd NVMain
scons --build-type=fast

./nvmain.fast Config/PCM_ISSCC_2012_4GB.config Tests/Traces/hello_world.nvt 1000000
```



## 2.Gem5

### 下载

https://github.com/gem5/gem5/releases

```
sudo git clone https://github.com/gem5/gem5.git
```





```bash
pip install six
#pip install python-config
sudo apt install python2-dev
sudo apt install m4

scons build/X86/gem5.opt
```



### SE测试

```bash
./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello 
```



### FS测试

```bash
wget http://www.m5sim.org/dist/current/x86/x86-system.tar.bz2
wget http://www.m5sim.org/dist/current/m5_system_2.0b3.tar.bz2
tar vxfj x86-system.tar.bz2
mkdir fs-image
mv binaries fs-image
mv disks fs-image
tar vxfj m5_system_2.0b3.tar.bz2
cp m5_system_2.0b3/disks/linux-bigswap2.img fs-image/disks

export M5_PATH=$M5_PATH:/home/你的路径/fs-image
#export M5_PATH=$M5_PATH:/home/hinata/Downloads/NVM/fs-image
./build/X86/gem5.opt configs/example/fs.py --disk-image linux-x86.img --kernel x86_64-vmlinux-2.6.22.9

cd util/term/
make

./m5term localhost 3456
```



### 退出

```
m5 exit
```



## 3. NVMain和Gem5集成环境

### Gem5补丁

手动补丁：修改gem5/configs/common/Options.py，在addNoISAOptions(parser):函数下添加

```python
vim configs/common/Options.py

for arg in sys.argv:
    if arg[:9] == "--nvmain-":
        parser.add_option(arg, type="string", default="NULL", help="Set NVMain configuration value for a parameter")
```

![img](https://upload-images.jianshu.io/upload_images/1433829-522cf803897f9ec1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1155/format/webp)



### 编译命令

```bash
scons EXTRAS=../nvmain build/X86/gem5.opt
```



### 测试

```bash
export M5_PATH=$M5_PATH:/home/你的路径/fs-image
#export M5_PATH=$M5_PATH:/home/hinata/Gem5-NVMain/fs-image

./build/X86/gem5.opt configs/example/se.py -c tests/test-progs/hello/bin/x86/linux/hello --caches --l2cache --mem-type=NVMainMemory --nvmain-config=../nvmain/Config/PCM_ISSCC_2012_4GB.config
```



### 报错

####1.NameError: global name 'Transform' is not defined:

```bash
sudo vim nvmain/build/SConscript

添加
from gem5_scons import Transform
```

#### 2.SyntaxError: invalid syntax

```bash
修改为：
print("Setting %s to %s" % (param_name, param_value))
```

#### 3.TypeError : File  found where directory expected.

````bash
rm -r build

scons EXTRAS=../nvmain build/X86/gem5.opt
````





## ”意外“事件

### 1.无网络连接

参考文献：https://blog.csdn.net/SOPHIA16527/article/details/107360879

1、删除NetworkManager缓存文件

```bash
service NetworkManager stop
rm /var/lib/NetworkManager/NetworkManager.state
service NetworkManager start
```

2.修改`/etc/NetworkManager/NetworkManager.conf`

```bash
managed=true
```

3.重启NetworkManage

```bash
service network-manager restart
```



gem5-20.0.0.0

```
/home/hinata/Downloads/NVM/nvmain/Simulators/gem5/nvmain_mem.hh:176:5: error: 'BaseSlavePort' does not name a type
```



