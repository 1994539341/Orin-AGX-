# Orin AGX 硬件填坑

硬件底板设计修改部分Pin，软件填坑记录 Jetpack 5.1.2 r35.4.1

硬件修改部分

1. eth0 reset		D56-->F56	PY01-->PQ01
2. eth0 interrupt	C57-->H52	PY03-->PN01
3. 添加一路SPI2
4. 添加两路Can

## 环境搭建

采用物理机搭建，CPU为i5-10，内存保证在8G以上即可，硬盘为512G，如果是虚拟机建议至少分200G左右

参考连接：[【NVIDIA Jetpack5.x】Jetson AGX Orin内核、设备树更新指南_jetson orin nx内核设备树更新-CSDN博客](https://blog.csdn.net/qq_31985307/article/details/131499050)

[Jetson AGX Orin刷机教程，奶奶看完都说会了！-CSDN博客](https://blog.csdn.net/weixin_53776054/article/details/128552701)

采用链接2的安装方式，其余操作均按照1进行

Jetpack5.1.2 r35.4.1下载地址[Jetson Linux 35.4.1 |NVIDIA 开发者](https://developer.nvidia.com/embedded/jetson-linux-r3541)

### 搭建中遇到的坑

1、在SDKmanager中没有自己所需的Jetpack版本需添加 --archived-version参数，如果无法启动先用root启动一次，再切换回一般用户

`sdkmanager --archived-version`

2、下载后需进行编译，再解压完编译器后，建议将环境变量添加到.bashrc中

```
`export CROSS_COMPILE=$HOME/l4t-gcc/bin/aarch64-buildroot-linux-gnu-`
`export CROSS_COMPILE_AARCH64_PATH=$HOME/l4t-gcc`
`export CROSS_COMPILE_AARCH64=$HOME/l4t-gcc/bin/aarch64-buildroot-linux-gnu-`
`export LOCALVERSION="-tegra"`
```

3、在5.1.3中拷贝内核编译产物中如下命令其实烧写的时候是无法加载dtb的

 `sudo cp -r $HOME/kernel_output/arch/arm64/boot/dts/nvidia <path-to>/Linux_for_Tegra/kernel/dtb/`

在flashlog中可以看到，两个copy路径是不一致的

`Copy /home/ayxx/OrinWork/Linux_for_Tegra/kernel/dtb/tegra234-p3701-0005-p3737-0000.dtb to /home/ayxx/OrinWork/Linux_for_Tegra/kernel/dtb/tegra234-p3701-0005-p3737-0000.dtb.rec`

所以应将指令修改为

`sudo cp -r $HOME/kernel_output/arch/arm64/boot/dts/nvidia* <path-to>/Linux_for_Tegra/kernel/dtb/`

4、烧写过程中请耐心等待，即使是烧写完了也不要动电源，否则可能变砖

救砖：[(1 封私信) Jetson Agx Orin刷机及ubuntu系统使用 - 知乎](https://zhuanlan.zhihu.com/p/632052753)

[(1 封私信) Jetson AGX Orin使用SDK Manager刷Jetpack 5.1.1的踩坑记录 - 知乎](https://zhuanlan.zhihu.com/p/640769844)

## 修改ETH1（已测试通过）

```
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
```

在如上路径下此文件中将PY01修改为PQ01，PY03修改为PN01，如果是出场设置可以在gpioinfo里面看到PQ01是被regulator@106占用的，在源码中解除占用或替换即可

