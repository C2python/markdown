#Fio(Flexible I/O Tester)#
-----------------------------------
Fio是专门用来测量IOPS(I/O per second)，用来对硬件进行压力测试和验证。支持多种不同的I/O引擎。支持gnuplot绘图。
Fio的安装：
>  
`yum install libaio-devel`  
`tar -zxvf fio-2.2.5.tar.gz`  
`cd fio-2.2.5`  
`make`  
`make install`

Fio的运行：  
1. 通过命令行的方式进行运行  
参数较少时使用。    
测试如下：  




2. 通过job文件进行运行  
\>`$fio job_file`    
job_file文件格式：  
