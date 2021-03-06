内存超分配 
-----
#####透明页面共享  
+ KSM  
[KSM]是Linux内核提供的一个功能，主要用于合并完全相同的内存页面从而提高内存使用率。  
由于KVM虚拟机是Linux的一个进程，所以可以很方便的使用到该特性。从而提供内存超配的能力。  
  - 确认KSM是否开启：  
```shell
KSM /boot/config-`uname –r`  
``` 
  - 内核参数目录：/sys/kernel/mm/ksm  
  - 默认KSM可以监控2000个页面：  
```shell 
cat /sys/kernel/mm/KSM/max_kernel_pages   
2000 
```
  - KSM需要程序使用madvise函数申请内存。确认KVM编译包含了该能力。   
``` c
#ifdef MADV_MERGEABLE
        madvise(new_block->host, size, MADV_MERGEABLE);
#endif
```


+ UKSM   
UKSM是国人对KSM的改进。主要有透明的全页面扫描（KSM需要使用madvise函数申请的内存才会合并）。改进的扫描采样算法（部分Hash取样）。以及对不同内存区域采用不同的扫描频率等改进方式。
UKSM以内核patch方式存在，需要重新编译内核。  
控制接口在 /sys/kernel/mm/uksm中，参数如下：   

文件名          |说明                       |  
----------------|---------------------------|  
run             |控制uksmd启动，默认启动。0为关闭|  
cpu_governor    |通过cat cpu_governor可以看到几种可选的模式和当前激活的模式，比如 [full] medium low quiet，这个说明当前采用的是全速扫描。这个几个扫描级别基本对应了 90%以上，50%，20% 和 < 1%的最大CPU占用控制以及对应的一些其它高级参数的微调。例：echo "medium" > cpu_governor|  
max_cpu_percentage|在已经使用了某个具体的cpu使用模式下，你可以进一步精确限制最大的cpu占用百分比。例：（最大为35%的最高CPU占用）：echo 35 > max_cpu_percentage|  

高级控制接口：   

文件名          |说明                       |  
----------------|---------------------------|  
cpu_ratios      |每个扫描级别的CPU占用率，以万分之一作为单位，或者以最高CPU占用作为参考。例（cat cpu_ratios）： 50 75 MAX/4 MAX/1这个说明我当前的四个扫描级别的速度分别是0.5%，0.75%，最大扫描速度（max_cpu_percentage）的1/4和最大扫描速度，大部分时间在没有大量冗余内存的时候，uksmd对CPU的占用在 0.5%左右，top就看不出来了，我自己已经很满意了。|
eval_intervals|四个扫描级别的采样周期大小，以毫秒为单位。如果是0的话，表明不对其采样，扫描完毕为止。这一般用在最高的扫描级别，即我们已经非常确信这里面有很多冗余页面。|
abundant_threshold|一个区域有多少百分数的有效冗余页面，uksmd才会拔高它的扫描级别，缺省是10, 即只有当一个区域被发现超过10%的页面是冗余的时候，我们才会加速它的扫描。|

用户只读参数：   

文件名          |说明                       |  
----------------|---------------------------|  
|full_scans  |采样覆盖全部内存区域的次数。|
|hash_strength|  自适应增量采样算法的当前的强度，值越低，那么当前合并的速度将会越快。|
|pages_scanned |当前已经扫描了多少页面。|
|pages_shared |有多少个不同的物理页面处在被共享的状态。|
|pages_sharing |有多少个虚拟页面指向以上的共享页面，差不多可以等效看作你节省的内存。|
|pages_unshared |有多少个不同的物理页面未处在被共享的状态。|
|sleep_times  |uksmd已经休眠了多少次。|


#####内存气泡  
内存气泡的原理非常简单，通过在guest中申请内存来进行guest内的释放，供其他虚拟机使用。  
+ libvirt   

xml文件：  
```xml
  <devices>
    ......
    <memballoon model='virtio'/>
    ......
  </devices>
```

设置（虚拟机分配内存-气球内存）大小，大小可以带单位，默认单位为k。
```shell
virsh setmem DOMAIN_NAME SIZE
例如：
virsh setmem win7 2G
```

没有查看（虚拟机分配内存-气球内存）大小的virsh命令，但是可以使用：  
```shell
# virsh qemu-monitor-command --hmp win7 'info balloon'
balloon: actual=4096
```

+ qemu命令  

ballon设备：  
```xml
-device virtio-balloon-pci,id=DEVICE_ID
```
查看虚拟机实际内存：  
```xml
(qemu) info balloon
```
释放内存(设置气球大小,释放需要一个过程）：
```xml
(qemu) balloon SIZE
```

+ 其他  
关于[自动气泡]，从多个层次都可以实现。比如hypervisor层或者更上层的管理系统实现。

#####页面交换  
qemu-kvm虚拟机作为系统的进程存在，所使用的内存可以利用系统的swap能力进行换出。  
不需要特别设置。

#####页面压缩  
未见支持。  

[KSM]:https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/chap-KSM.html
[自动气泡]:http://lists.nongnu.org/archive/html/qemu-devel/2013-05/msg01295.html
