# NVMain+Gem5环境配置

配置：Ubuntu16.04 、g++5、gcc5、gcov5、python2.7.12、pip20.3.3frompython2.7、scons3.1.2

为后面的人铺路吧。

可以说踩了很多坑，因为官网NVMain也没了，就比较麻烦，配置配对就没啥大问题。

主要参考文献：

https://blog.csdn.net/weixin_43914889/article/details/104629254

https://www.jianshu.com/p/af2efe552a11

https://github.com/yyuanye/Gem5-NVMain.git

gem5和NVMain都在这个github里面

```
git clone https://github.com.cnpmjs.org/yyuanye/Gem5-NVMain.git
```



## 1.NVMain

### scons

记住此处的pip一定要是python2的pip

```bash
sudo python -m pip install scons
export PATH=$PATH:/home/hinata/.local/bin或者reboot
```



### 然后

```bash
cd NVMain
scons --build-type=fast

./nvmain.fast Config/PCM_ISSCC_2012_4GB.config Tests/Traces/hello_world.nvt 1000000
```



## 2.Gem5

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
#export M5_PATH=$M5_PATH:/home/hinata/Gem5-NVMain/fs-image
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



### PARSEC 测试

1.下载PARSEC对应的PAL code文件, 复制到`fs-image/binaries`目录下

```bash
wget http://www.cs.utexas.edu/~parsec_m5/tsb_osfpal
```

2.下载PARSEC-2.1 Disk Image并解压到`fs-image/disks`目录中

```bash
wget http://www.cs.utexas.edu/~parsec_m5/x86root-parsec.img.bz2

bzip2 -d x86root-parsec.img.bz2
```

3.下载PARSEC script生成包，并解压到`benchmark`目录下

```bash
wget http://www.cs.utexas.edu/~parsec_m5/TR-09-32-parsec-2.1-alpha-files.tar.gz
tar zxvf TR-09-32-parsec-2.1-alpha-files.tar.gz
```

4.生成`script`

```bash
#./writescripts.pl <benchmark> <nthreads>
./writescripts.pl blackscholes 4
```

5.运行生成的脚本

```
export M5_PATH=$M5_PATH:/home/hinata/Gem5-NVMain/fs-image

./build/X86/gem5.opt ./configs/example/fs.py -n 2 --script=../benchmark/TR-09-32-parsec-2.1-alpha-files/blackscholes_4c_test.rcS --disk-image x86root-parsec.img --kernel x86_64-vmlinux-2.6.22.9.smp
```

6.连接测试

```
telnet localhost 3456
```



### FPC压缩方案（改进）

参考资料：https://www.jianshu.com/p/af2efe552a11

1.在nvmain/DataEncoders中创建FPCompress文件夹，然后创建：

```bash
cd nvmain/DataEncoders/FPCompress

vim FlipNWrite.h
```

```c++
/*******************************************************************************
* Copyright (c) 2012-2014, The Microsystems Design Labratory (MDL)
* Department of Computer Science and Engineering, The Pennsylvania State University
* All rights reserved.
* 
* This source code is part of NVMain - A cycle accurate timing, bit accurate
* energy simulator for both volatile (e.g., DRAM) and non-volatile memory
* (e.g., PCRAM). The source code is free and you can redistribute and/or
* modify it by providing that the following conditions are met:
* 
*  1) Redistributions of source code must retain the above copyright notice,
*     this list of conditions and the following disclaimer.
* 
*  2) Redistributions in binary form must reproduce the above copyright notice,
*     this list of conditions and the following disclaimer in the documentation
*     and/or other materials provided with the distribution.
* 
* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
* ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
* WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
* DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
* FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
* DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
* SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
* CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
* OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
* OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
* 
* Author list: 
*   Matt Poremba    ( Email: mrp5060 at psu dot edu 
*                     Website: http://www.cse.psu.edu/~poremba/ )
*******************************************************************************/

#ifndef __NVMAIN_FLIPNWRITE_H__
#define __NVMAIN_FLIPNWRITE_H__

#include "src/DataEncoder.h"
#include <set>

namespace NVM {

class FlipNWrite : public DataEncoder
{
  public:
    FlipNWrite( );
    ~FlipNWrite( );

    void SetConfig( Config *config, bool createChildren = true );

    ncycle_t Read( NVMainRequest *request );
    ncycle_t Write( NVMainRequest *request );

    void RegisterStats( );
    void CalculateStats( );

  private:
    std::set< uint64_t > flippedAddresses;
  
    uint64_t bitsFlipped;
    uint64_t bitCompareSwapWrites;
    double flipNWriteReduction;
    int fpSize;

    void InvertData( NVMDataBlock &data, uint64_t startBit, uint64_t endBit );
};

};

#endif
```



```bash
vim FlipNWrite.cpp
```

```c++
/*******************************************************************************
* Copyright (c) 2012-2014, The Microsystems Design Labratory (MDL)
* Department of Computer Science and Engineering, The Pennsylvania State University
* All rights reserved.
* 
* This source code is part of NVMain - A cycle accurate timing, bit accurate
* energy simulator for both volatile (e.g., DRAM) and non-volatile memory
* (e.g., PCRAM). The source code is free and you can redistribute and/or
* modify it by providing that the following conditions are met:
* 
*  1) Redistributions of source code must retain the above copyright notice,
*     this list of conditions and the following disclaimer.
* 
*  2) Redistributions in binary form must reproduce the above copyright notice,
*     this list of conditions and the following disclaimer in the documentation
*     and/or other materials provided with the distribution.
* 
* THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
* ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
* WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
* DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
* FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
* DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
* SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
* CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
* OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
* OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
* 
* Author list: 
*   Matt Poremba    ( Email: mrp5060 at psu dot edu 
*                     Website: http://www.cse.psu.edu/~poremba/ )
*******************************************************************************/

#include "DataEncoders/FlipNWrite/FlipNWrite.h"

#include <iostream>

using namespace NVM;

FlipNWrite::FlipNWrite( )
{
    flippedAddresses.clear( );

    /* Clear statistics */
    bitsFlipped = 0;
    bitCompareSwapWrites = 0;
}

FlipNWrite::~FlipNWrite( )
{
    /*
     *  Nothing to do here. We do not own the *config pointer, so
     *  don't delete that.
     */
}

void FlipNWrite::SetConfig( Config *config, bool /*createChildren*/ )
{
    Params *params = new Params( );
    params->SetParams( config );
    SetParams( params );

    /* Cache granularity size. */
    fpSize = config->GetValue( "FlipNWriteGranularity" );

    /* Some default size if the parameter is not specified */
    if( fpSize == -1 )
        fpSize = 32; 
}

void FlipNWrite::RegisterStats( )
{
    AddStat(bitsFlipped);
    AddStat(bitCompareSwapWrites);
    AddUnitStat(flipNWriteReduction, "%");
}

void FlipNWrite::InvertData( NVMDataBlock& data, uint64_t startBit, uint64_t endBit )
{
    uint64_t wordSize;
    int startByte, endByte;

    wordSize = p->BusWidth;
    wordSize *= p->tBURST * p->RATE;
    wordSize /= 8;

    startByte = (int)(startBit / 8);
    endByte = (int)((endBit - 1) / 8);

    for( int i = startByte; i <= endByte; i++ )
    {
        uint8_t originalByte = data.GetByte( i );
        uint8_t shiftByte = originalByte;
        uint8_t newByte = 0;

        for( int j = 0; j < 8; j++ )
        {
            uint64_t currentBit = i * 8 + j;
           
            if( currentBit < startBit || currentBit >= endBit )
            {
                shiftByte = static_cast<uint8_t>(shiftByte >> 1);
                continue;
            }

            if( !(shiftByte & 0x1) )
            {
                newByte = static_cast<uint8_t>(newByte | (1 << (7-j)));
            }

            shiftByte = static_cast<uint8_t>(shiftByte >> 1);
        }

        data.SetByte( i, newByte );
    }
}

ncycle_t FlipNWrite::Read( NVMainRequest* /*request*/ )
{
    ncycle_t rv = 0;

    // TODO: Add some energy here

    return rv;
}

ncycle_t FlipNWrite::Write( NVMainRequest *request ) 
{
    NVMDataBlock& newData = request->data;
    NVMDataBlock& oldData = request->oldData;
    NVMAddress address = request->address;

    /*
     *  The default life map is an stl map< uint64_t, uint64_t >. 
     *  You may map row and col to this map_key however you want.
     *  It is up to you to ensure there are no collisions here.
     */
    uint64_t row;
    uint64_t col;
    ncycle_t rv = 0;

    request->address.GetTranslatedAddress( &row, &col, NULL, NULL, NULL, NULL );

    /*
     *  If using the default life map, we can call the DecrementLife
     *  function which will check if the map_key already exists. If so,
     *  the life value is decremented (write count incremented). Otherwise 
     *  the map_key is inserted with a write count of 1.
     */
    uint64_t rowSize;
    uint64_t wordSize;
    uint64_t currentBit;
    uint64_t flipPartitions;
    uint64_t rowPartitions;
    int *modifyCount;

    wordSize = p->BusWidth;
    wordSize *= p->tBURST * p->RATE;
    wordSize /= 8;

    rowSize = p->COLS * wordSize;
    rowPartitions = ( rowSize * 8 ) / fpSize;
    
    flipPartitions = ( wordSize * 8 ) / fpSize; 

    modifyCount = new int[ flipPartitions ];

    /*
     *  Count the number of bits that are modified. If it is more than
     *  half, then we will invert the data then write.
     */
    for( uint64_t i = 0; i < flipPartitions; i++ )
        modifyCount[i] = 0;

    currentBit = 0;

    /* Get what is currently in the memory (i.e., if it was previously flipped, get the flipped data. */
    for( uint64_t i = 0; i < flipPartitions; i++ )
    {
        uint64_t curAddr = row * rowPartitions + col * flipPartitions + i;

        if( flippedAddresses.count( curAddr ) )
        {
            InvertData( oldData, i*fpSize, (i+1)*fpSize );
        }
    }

    /* Check each byte to see if it was modified */
    for( uint64_t i = 0; i < wordSize; ++i )
    {
        /*
         *  If no bytes have changed we can just continue. Yes, I know this
         *  will check the byte 8 times, but i'd rather not change the iter.
         */
        uint8_t oldByte, newByte;

        oldByte = oldData.GetByte( i );
        newByte = newData.GetByte( i );

        if( oldByte == newByte )
        {
            currentBit += 8;
            continue;
        }

        /*
         *  If the bytes are different, then at least one bit has changed.
         *  check each bit individually.
         */
        for( int j = 0; j < 8; j++ )
        {
            uint8_t oldBit, newBit;

            oldBit = ( oldByte >> j ) & 0x1;
            newBit = ( newByte >> j ) & 0x1;

            if( oldBit != newBit )
            {
                modifyCount[(int)(currentBit/fpSize)]++;
            }

            currentBit++;
        }
    }

    /*
     *  Flip any partitions as needed and mark them as inverted or not.
     */
    for( uint64_t i = 0; i < flipPartitions; i++ )
    {
        bitCompareSwapWrites += modifyCount[i];

        uint64_t curAddr = row * rowPartitions + col * flipPartitions + i;

        /* Invert if more than half of the bits are modified. */
        if( modifyCount[i] > (fpSize / 2) )
        {
            InvertData( newData, i*fpSize, (i+1)*fpSize );

            bitsFlipped += (fpSize - modifyCount[i]);

            /*
             *  Mark this address as flipped. If the data was already inverted, it
             *  should remain as inverted for the new data.
             */
            if( !flippedAddresses.count( curAddr ) )
            {
                flippedAddresses.insert( curAddr );
            }
        }
        else
        {
            /*
             *  This data is not inverted and should not be marked as such.
             */
            if( flippedAddresses.count( curAddr ) )
            {
                flippedAddresses.erase( curAddr );
            }

            bitsFlipped += modifyCount[i];
        }
    }

    delete modifyCount;
    
    return rv;
}

void FlipNWrite::CalculateStats( )
{
    if( bitCompareSwapWrites != 0 )
        flipNWriteReduction = (((double)bitsFlipped / (double)bitCompareSwapWrites)*100.0);
    else
        flipNWriteReduction = 100.0;
}
```

```bash
vim Sconscript
```

```
# Copyright (c) 2012-2013, The Microsystems Design Labratory (MDL)
# Department of Computer Science and Engineering, The Pennsylvania State University
# All rights reserved.
# 
# This source code is part of NVMain - A cycle accurate timing, bit accurate
# energy simulator for both volatile (e.g., DRAM) and non-volatile memory
# (e.g., PCRAM). The source code is free and you can redistribute and/or
# modify it by providing that the following conditions are met:
# 
#  1) Redistributions of source code must retain the above copyright notice,
#     this list of conditions and the following disclaimer.
# 
#  2) Redistributions in binary form must reproduce the above copyright notice,
#     this list of conditions and the following disclaimer in the documentation
#     and/or other materials provided with the distribution.
# 
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
# ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
# WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
# OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
# 
# Author list: 
#   Matt Poremba    ( Email: mrp5060 at psu dot edu 
#                     Website: http://www.cse.psu.edu/~poremba/ )

Import('*')

# Assume that this is a gem5 extras build if this is set.
if 'TARGET_ISA' in env and env['TARGET_ISA'] == 'no':
    Return()

if 'NVMAIN_BUILD' in env:
    NVMainSourceType('src', 'Backend Source')


NVMainSource('FlipNWrite.cpp')
```

修改DataEncoderFactory.cpp

```
#include "DataEncoders/FPCompress/FPCompress.h"

// DataEncoderFactory.cpp
DataEncoder *DataEncoderFactory::CreateDataEncoder( std::string encoderName )
{
    DataEncoder *encoder = NULL;

    if( encoderName == "default" ) encoder = new DataEncoder( );
    else if( encoderName == "FlipNWrite" ) encoder = new FlipNWrite( );
    else if( encoderName == "FPCompress") encoder = new FPCompress( );

    return encoder;
}
```

然后在nvmain/Config/PCM_MLC_example.config文件末尾加入

```
DataEncoder FPCompress
```

编译

```
scons --build-type=fast
```

测试

```
./nvmain.fast Config/PCM_MLC_example.config Tests/Traces/hello_world.nvt 1000000
```

这里我的数据和它的不一样，不知道是不是内核不同的原因。他的内核我电脑跑不了，会出error。
应该是nvmain的版本不同所导致，里面的hello_world.nvt不一样应该。



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



