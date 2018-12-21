# 试剂系统通过IAP更新程序的评估报告

## 一. STM32F407的Flash分布表

​	STM32F407有1Mbytes的Flash存储空间，其中0-3号扇区为16Kbytes,4号扇区为64Kbytes,5-11号扇区为128Kbytes,具体的分布参照下表：

| 扇区      | 地址范围                  | 大小       | 作用               |
| --------- | ------------------------- | ---------- | ------------------ |
| Sector 0  | 0x0800 0000 - 0x0800 3FFF | 16 Kbytes  | 存储Bootloader代码 |
| Sector 1  | 0x0800 4000 - 0x0800 7FFF | 16 Kbytes  | 同上               |
| Sector 2  | 0x0800 8000 - 0x0800 BFFF | 16 Kbytes  | 存储APP执行代码    |
| Sector 3  | 0x0800 C000 - 0x0800 FFFF | 16 Kbytes  | 同上               |
| Sector 4  | 0x0801 0000 - 0x0801 FFFF | 64 Kbytes  | 同上               |
| Sector 5  | 0x0802 0000 - 0x0803 FFFF | 128 Kbytes | Flash参数存储区    |
| Sector6   | 0x0804 0000 - 0x0805 FFFF | 128 Kbytes | 存储APP待更新代码  |
| Sector 7  | 0x0806 0000 - 0x0807 FFFF | 128 Kbytes | 存储标志位信息     |
| ...       |                           |            | 预留               |
| Sector 11 | 0x080E 0000 - 0x080F FFFF | 128Kbytes  | 预留               |

Keil进行ROM空间配置的方法：

Bootloader:

![1545377606342](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\1545377606342.png)

App:

![1545377791685](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\1545377791685.png)



## 二. IAP更新逻辑详解

1. 流程图

```flow

start=>start: 开始
initialBoot=>operation: 初始化时钟和CAN1通信
selfTestingPased=>condition: (读标志位 == 0xAAAA?)|approved

debug=>operation: 跳转APP执行代码|invalid
jumpboot=>operation: 清除AAAA标志|invalid
jumpApp=>operation: 跳转App |ok
appwriteAAAA=>operation: App初始化时写入AAAA标志 |ok
submitTestingPased=>condition: 收到更新程序指令？|ok

cycleboot=>subroutine: 执行BootLoader程序|invalid
cycleapp=>subroutine: 执行用户程序|invalid
end=>end: 软件重启|future
start->initialBoot
initialBoot->selfTestingPased
selfTestingPased(yes)->jumpApp
jumpApp->appwriteAAAA
appwriteAAAA->submitTestingPased
submitTestingPased(yes)->jumpboot
selfTestingPased(no)->cycleboot
cycleboot->selfTestingPased
submitTestingPased(no)->cycleapp
cycleapp->submitTestingPased
jumpboot->end

```

1. 逻辑图

   ![1545379048214](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\1545379048214.png)
   ​        

   ​        Cortex-M内核规定，中断向量表开始的4个字节存放的是堆栈栈顶的地址，其后是中断向量表各中断服务程序的地址。当发生中断后程序通过查找该表得到相应的中断服务程序入口地址，然后再跳到相应的中断服务程序中执行，中断服务程序中最终调用用户实现的各函数。例如：main函数就是复位中断服务函数中调用的！



   ​        上电后从0x08000004处取出复位中断向量的地址，然后跳转到复位中断程序的入口(标号①所示)，在IAP中读取标志位信息，如果为AAAA，在跳转②处，如③进入APP程序的main函数执行代码，在执行App的main函数的过程中发生中断，则STM32强制将PC指针指回中断向量表处(标号④所示)，但由于我们设置了中断向量表偏移量为N+M，因此PC指针被强制跳转0x08000004+N+M处的中断向量表中，查找得到相应的中断函数地址，再跳转到相应新的中断服务函数，执行结束后返回到main函数中来。



   ​        需要注意的是，复位中断比较特殊。产生复位后，PC的值会被硬件强制置为0x08000004。因为，在发生复位后，负责中断向量偏移的寄存器VTOR变为了0，因此，复位后的中断就变为了0x08000004。而其他中断发生时，VTOR为已经设置好的中端向量表偏移。

## 三. 如何导出S19文件格式

​	由于SMART6000采用了ST和飞思卡尔这两种单片机，为实现可执行文件的格式的统一性，特将STM32的导出的APP文件由HEX文件更改为S19文件，应用IAP下载工具利用同样的算法，就可实现APP程序的下载，具体的操作如下：

1. 命名文件名为sj,并按下图勾选对应项目

![1545376551851](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\1545376551851.png)



1. 勾选Run#1 ,并在UserCommand中输入fromelf --m32combined --output=outfile.s19 .\Objects\sj.axf

![1545376582022](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\1545376582022.png)

1. 重新编译程序，在目标路径下就可以得到名字为“outfiles.s19” 的S19文件格式的文件了。

## 四. Bootloader跳转APP功能的实现

​	Bootloader中可通过以下代码可跳转至用户程序，其中USER_FLASH_FIRST_PAGE_ADDRESS为用户程序的首地址即 ：0x0800 8000。

```
    /* Check if valid stack address (RAM address) then jump to user application */
    if (((*(__IO uint32_t*)USER_FLASH_FIRST_PAGE_ADDRESS) & 0x2FFE0000 ) == 0x20000000)
    {
      /* Jump to user application */
      JumpAddress = *(__IO uint32_t*) (USER_FLASH_FIRST_PAGE_ADDRESS + 4);
      Jump_To_Application = (pFunction) JumpAddress;
      /* Initialize user application's Stack Pointer */
      __set_MSP(*(__IO uint32_t*) USER_FLASH_FIRST_PAGE_ADDRESS);
      Jump_To_Application();
    }
```

​	App初始化开始位置，需要重新定位中断向量表的首地址。 因为APP的起始地址为(0X08000000)+0x8000,故可通过以下代码实现中断向量表的重新定位：

```
SCB->VTOR = FLASH_BASE | 0x8000;/*中断向量表定位在FLASH_BASE(0X08000000)+0x8000处*/
```





 





















