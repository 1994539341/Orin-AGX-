# Orin-AGX-
硬件底板设计修改部分Pin，软件填坑记录 JetPack 5.1.2,r35.4.1

硬件修改部分

Orin AGX eth0 1.reset D56-->F56
                               
                 PY01-->PQ01
                          
              2.interrupt C57-->H52
                                      
                PY03--PN01
          
                添加一路SPI2
          
                添加两路Can

# 环境搭建
采用搭建物理机配置为i510系，内存保证在8G以上即可，参考
https://blog.csdn.net/qq_31985307/article/details/131499050

1、环境搭建上遇到的坑

