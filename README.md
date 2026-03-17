Orin AGX 硬件填坑
硬件底板设计修改部分Pin，软件填坑记录 Jetpack 5.1.2 r35.4.1

硬件修改部分

eth0 reset		D56-->F56	PY01-->PQ01

eth0 interrupt	C57-->H52	PY03-->PN01

添加一路SPI2

添加两路Can

测试两路串口

环境搭建
采用物理机搭建，CPU为i5-10，内存保证在8G以上即可，硬盘为512G，如果是虚拟机建议至少分200G左右

参考连接：【NVIDIA Jetpack5.x】Jetson AGX Orin内核、设备树更新指南_jetson orin nx内核设备树更新-CSDN博客

Jetson AGX Orin刷机教程，奶奶看完都说会了！-CSDN博客

采用链接2的安装方式，其余操作均按照1进行

Jetpack5.1.2 r35.4.1下载地址Jetson Linux 35.4.1 |NVIDIA 开发者

搭建中遇到的坑
1、在SDKmanager中没有自己所需的Jetpack版本需添加 --archived-version参数，如果无法启动先用root启动一次，再切换回一般用户

sdkmanager --archived-version

2、下载后需进行编译，再解压完编译器后，建议将环境变量添加到.bashrc中

`export CROSS_COMPILE=$HOME/l4t-gcc/bin/aarch64-buildroot-linux-gnu-`
`export CROSS_COMPILE_AARCH64_PATH=$HOME/l4t-gcc`
`export CROSS_COMPILE_AARCH64=$HOME/l4t-gcc/bin/aarch64-buildroot-linux-gnu-`
`export LOCALVERSION="-tegra"`
3、在5.1.3中拷贝内核编译产物中如下命令其实烧写的时候是无法加载dtb的

 sudo cp -r $HOME/kernel_output/arch/arm64/boot/dts/nvidia <path-to>/Linux_for_Tegra/kernel/dtb/

在flashlog中可以看到，两个copy路径是不一致的

Copy /home/ayxx/OrinWork/Linux_for_Tegra/kernel/dtb/tegra234-p3701-0005-p3737-0000.dtb to /home/ayxx/OrinWork/Linux_for_Tegra/kernel/dtb/tegra234-p3701-0005-p3737-0000.dtb.rec

所以应将指令修改为

sudo cp -r $HOME/kernel_output/arch/arm64/boot/dts/nvidia* <path-to>/Linux_for_Tegra/kernel/dtb/

4、烧写过程中请耐心等待，即使是烧写完了也不要动电源，否则可能变砖

救砖：(1 封私信) Jetson Agx Orin刷机及ubuntu系统使用 - 知乎

(1 封私信) Jetson AGX Orin使用SDK Manager刷Jetpack 5.1.1的踩坑记录 - 知乎

修改ETH1（已测试通过）
`ayxx@ayxx:~$ cat OrinWork/Linux_for_Tegra/sources/hardware/nvidia/platform/t23x/concord/kernel-dts/cvb/tegra234-ethernet-3737-0000.dtsi`
`/*`

 * `Copyright (c) 2021, NVIDIA CORPORATION.  All rights reserved.`
   `*`
 * `This program is free software; you can redistribute it and/or modify it`
 * `under the terms and conditions of the GNU General Public License,`
 * `version 2, as published by the Free Software Foundation.`
   `*`
 * `This program is distributed in the hope it will be useful, but WITHOUT`
 * `ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or`
 * `FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License for`
 * `more details.`
   `*`
 * `You should have received a copy of the GNU General Public License`
 * `along with this program.  If not, see <http://www.gnu.org/licenses/>.`
   `*/`

`#include <dt-bindings/gpio/tegra234-gpio.h>`

`/ {`
        `/* MGBE - A */
        ethernet@6810000 {
                status = "okay";
                nvidia,mac-addr-idx = <0>;
                nvidia,max-platform-mtu = <16383>;
                /* 1=enable, 0=disable */
                nvidia,pause_frames = <1>;
                phy-handle = <&mgbe0_aqr113c_phy>;
                phy-mode = "10gbase-r";
                /* 0:XFI 10G, 1:XFI 5G, 2:USXGMII 10G, 3:USXGMII 5G */`
                `nvidia,phy-iface-mode = <0>;`
                `nvidia,phy-reset-gpio = <&tegra_main_gpio TEGRA234_MAIN_GPIO(Q, 1) 0>;`

                mdio {
                        compatible = "nvidia,eqos-mdio";
                        #address-cells = <1>;
                        #size-cells = <0>;
    
                        mgbe0_aqr113c_phy: ethernet_phy@0 {
                                compatible = "ethernet-phy-ieee802.3-c45";
                                reg = <0x0>;
                                nvidia,phy-rst-pdelay-msec = <150>; /* msec */
                                nvidia,phy-rst-duration-usec = <221000>; /* usec */
                                interrupt-parent = <&tegra_main_gpio>;
                                interrupts = <TEGRA234_MAIN_GPIO(N, 1) IRQ_TYPE_LEVEL_LOW>;
                        };
                };
        };

`};`
在如上路径下此文件中将PY01修改为PQ01，PY03修改为PN01，如果是出场设置可以在gpioinfo里面看到PQ01是被regulator@106占用的，在源码中解除占用或替换即可

串口测试
1、板卡有两路422，首先依照原理图连接线

image-20260316145151349

在此图中

原理图注脚	RS422
RS422_A	RX+
RS422_B	RX-
RS422_Y	TX+
RS422_Z	TX-
2、短接接好线后插入插座，进入cutecom，未安装请按照下列教程安装，已安装跳到步骤3

使用 cutecom 操作UART，cutecom 是一个跨平台的串口终端程序，它提供了一个简洁直观的图形用户界面，允许用户通过串口接口发送和接收数据。运行以下命令以安装 cutecom

sudo apt install cutecom
3、安装完成后在终端输入如下指令启动cutecom

sudo cutecom
串口设备号分别为ttyTHS0和ttyTHS1，在device中选择串口后点击open即可开启，在input中输入字符可以在上方（发送窗口）下方（接收窗口）同时看到信息即可

添加SPI测试
1、在Orin_Jetson_Series_Pinmux_Config_Template-v2.1中按如下修改pinmux

image-20260129141337117

2、生成三个dtsi后替换gpio.dtsi和pinmux.dtsi

路径如下 

OrinWork/Linux_for_Tegra/bootloader/tegra234-mb1-bct-gpio-p3701-0000-a04.dtsi

Linux_for_Tegra/bootloader/t186ref/BCT/tegra234-mb1-bct-pinmux-p3701-0000-a04.dtsi
在flashlog中也可以看到路径

copying pinmux_config(/home/ayxx/OrinWork/Linux_for_Tegra/bootloader/t186ref/BCT/tegra234-mb1-bct-pinmux-p3701-0000-a04.dtsi)... done.
官方对于Orin_Jetson_Series_Pinmux_Config_Template-v2.1的使用说明

Jetson AGX Orin 平台适配与升级——NVIDIA Jetson Linux 开发者指南

3、在tegra234-p3737-0000-a04.dtsi中将spi@c260000节点status修改为okay，烧录（SPI2）

4、打开SPI0，在Jetson-io.py中把spi勾上即可，顺手把can也开一下，路径是，执行如下命令

sudo /opt/nvidia/jetson-io/jetson-io.py
5、由于我们打开了SPI1，但是没有添加设备，需要反编译出来，在c260000中添加如下内容，添加一个spi@0，效果如下

spi@c260000 {
        iommus = <0x03 0x04>;
        #address-cells = <0x01>;
        dma-coherent;
        clock-names = "spi\0pll_p\0osc";
        nvidia,clk-parents = "pll_p\0osc";
        resets = <0x02 0x5c>;
        interrupts = <0x00 0x25 0x04>;
        clocks = <0x02 0x88 0x02 0x5e 0x02 0x5b>;
        #size-cells = <0x00>;
        spi-max-frequency = <0x3dfd240>;
        dma-names = "rx\0tx";
        compatible = "nvidia,tegra186-spi";
        status = "okay";
        reg = <0x00 0xc260000 0x00 0x10000>;
        phandle = <0x390>;
        dmas = <0x05 0x10 0x05 0x10>;
        reset-names = "spi";

        spi@0 {
            compatible = "tegra-spidev";
            reg = <0x00>;
            spi-max-frequency = <0x2faf080>;
            controller-data {
                nvidia,enable-hw-based-cs;
                nvidia,tx-clk-tap-delay = <0x00>;
                nvidia,rx-clk-tap-delay = <0x10>;
            };
        };

        prod-settings {
            #prod-cells = <0x04>;

            prod {
                prod = <0x00 0x194 0x80000000 0x00>;
            };
        };
    };
保险起见可以顺便加下pinmux，虽然在之前在源码里就加了pinmux的内容

        exp-header-pinmux {
            phandle = <0x561>;

            hdr40-pin24 {
                nvidia,tristate = <0x00>;
                nvidia,function = "spi1";
                nvidia,enable-input = <0x01>;
                nvidia,pins = "spi1_cs0_pz6";
            };

            hdr40-pin29 {
                nvidia,tristate = <0x01>;
                nvidia,function = "can0";
                nvidia,enable-input = <0x01>;
                nvidia,pins = "can0_din_paa1";
            };

            hdr40-pin19 {
                nvidia,tristate = <0x00>;
                nvidia,function = "spi1";
                nvidia,enable-input = <0x01>;
                nvidia,pins = "spi1_mosi_pz5";
            };

            hdr40-pin37 {
                nvidia,tristate = <0x01>;
                nvidia,function = "can1";
                nvidia,enable-input = <0x01>;
                nvidia,pins = "can1_din_paa3";
            };

            hdr40-pin33 {
                nvidia,tristate = <0x00>;
                nvidia,function = "can1";
                nvidia,enable-input = <0x00>;
                nvidia,pins = "can1_dout_paa2";
            };

            hdr40-pin23 {
                nvidia,tristate = <0x00>;
                nvidia,function = "spi1";
                nvidia,enable-input = <0x01>;
                nvidia,pins = "spi1_sck_pz3";
            };

            hdr40-pin31 {
                nvidia,tristate = <0x00>;
                nvidia,function = "can0";
                nvidia,enable-input = <0x00>;
                nvidia,pins = "can0_dout_paa0";
            };

            hdr40-pin21 {
                nvidia,tristate = <0x00>;
                nvidia,function = "spi1";
                nvidia,enable-input = <0x01>;
                nvidia,pins = "spi1_miso_pz4";
            };

            hdr40-pin26 {
                nvidia,tristate = <0x00>;
                nvidia,function = "spi1";
                nvidia,enable-input = <0x01>;
                nvidia,pins = "spi1_cs1_pz7";
            };
            spi2_sck_pcc0 {             //以下为添加内容，按道理是源码中加了这里不用加了，因为之前没有排查到regulator的pin脚占用，于是添加此内容
                nvidia,pins = "spi2_sck_pcc0";
                nvidia,function = "spi2";
                nvidia,tristate = <0x00>;
                nvidia,enable-input = <0x01>;
            };

            spi2_miso_pcc1 {
                nvidia,pins = "spi2_miso_pcc1";
                nvidia,function = "spi2";
                nvidia,tristate = <0x00>;
                nvidia,enable-input = <0x01>;
            };

            spi2_mosi_pcc2 {
                nvidia,pins = "spi2_mosi_pcc2";
                nvidia,function = "spi2";
                nvidia,tristate = <0x00>;
                nvidia,enable-input = <0x01>;
            };

            spi2_cs0_pcc3 {
                nvidia,pins = "spi2_cs0_pcc3";
                nvidia,function = "spi2";
                nvidia,tristate = <0x00>;
                nvidia,enable-input = <0x01>;
            };
        };
6、但是，我们的pcc.02被regulator@116占用，所以需要将regulator@116的gpio注释掉，ps应该是和电源有关，修改后目前不影响，之后再编译后替换文件

image-20260317153158866

    regulator@116 {
        regulator-max-microvolt = <0x1b7740>;
        //gpio = <0x06 0x12 0x00>;
        enable-active-high;
        regulator-min-microvolt = <0x1b7740>;
        regulator-name = "dsi-vdd-1v8-bl-en";
        compatible = "regulator-fixed";
        reg = <0x74>;
        phandle = <0x52b>;
        vin-supply = <0x09>;
    };
根据exlinux.config可知需要替换的FDT文件为JetsonIO标签下的内容（用了jetson-io.py之后，设备树的位置就会改变为，LABLE也会改变，从primary变为JetsonIO）替换成功后重启

反编译、编译和替换指令为

dtc -I fs -O dts /proc/device-tree > my_current_system.dts      //反编译
dtc -I dts -O dtb my_current_system.dts -o modified_system.dtb  //编译
sudo mv modified_system.dtb /boot/kernel_tegra234-p3701-0005-p3737-0000-user-custom.dtb
image-20260317153621546

7、安装并使用spidevtest

git clone https://github.com/rm-hull/spidev-test                    //下载spidevtest
cd spidev-test
gcc spidev_test.c -o spidev_test                                    //编译
使用前需要probe spi的驱动，按原理图接好线，MOSI和MISO对接即可。

image-20260317155241030

sudo modprobe spidev
./spidev_test -D /dev/spidev0.0 -s 100000 -p "\x11\x22\x33" -v      //运行
./spidev_test -D /dev/spidev1.0 -s 100000 -p "\x11\x22\x33" -v      //运行
image-20260317154502561

PS：添加一个SPI@3230000的，应该也是在这里添加设备，往下可以看到c260000的节点，文件名为tegra234-p3737-0000-a04.dtsi，因保交付所以不深入研究，放在此处参考

    spi@3230000{ /* SPI3 in 40 pin conn */
        status = "okay";
        spi@0 { /* chip select 0 */
            compatible = "tegra-spidev";
            reg = <0x0>;
            spi-max-frequency = <50000000>;
            controller-data {
                nvidia,enable-hw-based-cs;
                nvidia,rx-clk-tap-delay = <0x10>;
                nvidia,tx-clk-tap-delay = <0x0>;
            };
        };
        spi@1 { /* chips select 1 */
            compatible = "tegra-spidev";
            reg = <0x1>;
            spi-max-frequency = <50000000>;
            controller-data {
                nvidia,enable-hw-based-cs;
                nvidia,rx-clk-tap-delay = <0x10>;
                nvidia,tx-clk-tap-delay = <0x0>;
            };
        };
    };
串口和spi测试参考链接

五、其他外设 | 控元科技（广州）有限公司

After enabling spi1 in the device tree, the self-loopback test of spidev_test failed - Jetson Systems / Jetson AGX Orin - NVIDIA Developer Forums

CAN配置和测试
1、在jetson-io.py中打开can，并按照原理图接线6<->4、2<->9

image-20260317155435340

sudo /opt/nvidia/jetson-io/jetson-io.py
2、挂载can内核

ayxx@tegra-ubuntu:~/spidev-test$ sudo modprobe can
ayxx@tegra-ubuntu:~/spidev-test$ sudo modprobe can_raw
ayxx@tegra-ubuntu:~/spidev-test$ sudo modprobe mttcan
3、can属性设置

sudo ip link set down can0
sudo ip link set can0 type can bitrate 500000
sudo ip link set up can0
sudo ip link set down can1
sudo ip link set can1 type can bitrate 500000
sudo ip link set up can1
4、安装并使用工具

sudo apt-get install can-utils
cansend can0 0E2#05.00.00.00.00.00.00.00  #cansend can0/1 [can_id]#[八字节数据]
cangen -v can0  #随机发送
candump can0 #接受can帧
can0收can1发

image-20260317160457846

can1收can0发

image-20260317160601155

测试通过

ETH2杂谈
Orin AGX，硬件还添加了一个网口，型号为yt8521，不展示原理图，随便贴点东西吧

1、Pinmux，还有一个reset脚就不截了，设为gpio即可，看硬件设计是什么了

image-20260317160810254

2、设备树修改部分，在tegra234-ethernet-3737-0000.dtsi中添加节点，添加到6810000下面，最外面的大括号里面，东西具体什么含义查AI，变量名含义应该很好找了，也可以看下doc，变量名和内容也解释的很清楚

 /* EQOS */
    ethernet@2310000 {
        status = "okay";
        compatible = "nvidia,tegra234-eqos", "snps,dwmac-5.10a";
        reg = <0x0 0x02310000 0x0 0x10000>,    /* EQOS Base Register */
              <0x0 0x023D0000 0x0 0x10000>,    /* MACSEC Base Register */
              <0x0 0x02300000 0x0 0x10000>;    /* HV Base Register */
        reg-names = "mac", "macsec-base", "hypervisor";
        phy-handle = <&yt_8521phy>;
        phy-mode = "rgmii-id";

        nvidia,mac-addr-idx = <1>;
        nvidia,mdio_addr = <1>;

        nvidia,phy-reset-gpio = <&tegra_main_gpio TEGRA234_MAIN_GPIO(AC, 1) 1>;

        mdio {
            compatible = "nvidia,eqos-mdio";
            #address-cells = <1>;
            #size-cells = <0>;

            yt_8521phy: ethernet-phy@1 {
                compatible = "motorcomm,yt8521","ethernet-phy-id0000011a","ethernet-phy-ieee802.3-c22";
                reg = <0x1>;
                motorcomm,rx-delay-sel = <2>;
                motorcomm,tx-delay-sel = <2>;
                nvidia,phy-rst-pdelay-msec = <224>;
                nvidia,phy-rst-duration-usec = <10000>;
            };
        };
    };
3、驱动的话各个厂商不同，但基本都是在内核的menuconfig里面添加

参考链接

https://forums.developer.nvidia.com/t/rgmii-ethernet-on-orin-agx-cvb/264796

https://forums.developer.nvidia.com/t/jetson-agx-orin-r36-4-3-ti-dp83867-phy-rgmii-not-working/359999

https://docs.nvidia.com/jetson/archives/r35.4.1/DeveloperGuide/text/HR/JetsonModuleAdaptationAndBringUp/JetsonAgxOrinSeries.html?highlight=universal#for-rgmii
