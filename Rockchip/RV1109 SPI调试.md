# RV1109 SPI调试

## 1.设备树

```
&spi0 {
    /delete-property/ pinctrl-1;
    max-freq = <50000000>;
    pinctrl-0 = <&spi0m1_clk &spi0m1_cs0n &spi0m1_cs1n &spi0m1_miso &spi0m1_mosi>;
    status = "okay";
    spi_test@00 {
        compatible = "rockchip,spidev";
        reg = <0>;
        spi-max-frequency = <5000000>;
    };
};

 驱动设备加载注册成功后，会出现类似这个名字的设备：/dev/spidev1.1
请参照 Documentation/spi/spidev_test.c
测试：
 echo 1 > /dev/spidev0.0
 时钟和数据线会有波形。
```

重新编译内核，启动后看日志是否加载成功，并且启动后有类似`/dev/spidev0.0`名称的设备。

## 2. 测试

### 2.1 使用shell测试

```
 echo 1 > /dev/spidev0.0
```

 时钟和数据线会有波形。

### 2.2 测试应用

```

#include "spi_test.h"
#include <stdint.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <getopt.h>
#include <fcntl.h>
#include <time.h>
#include <sys/ioctl.h>
#include <linux/ioctl.h>
#include <sys/stat.h>
#include <linux/types.h>
#include <linux/spi/spidev.h>


#define SPI_DEBUG 1

static const char *device = "/dev/spidev0.0";
static uint8_t mode = 3; /* SPI通信使用全双工，设置CPOL＝1，CPHA＝1。 */
static uint8_t bits = 8; /* 8bits读写，MSB first。*/
static uint32_t speed = 24 * 1000 * 1000;/* 设置12M传输速度 */
static uint16_t delay = 0;
int g_SPI_Fd;



/**********************************************************************************
* 功 能：打开设备 并初始化设备
* 入口参数 ：
* 出口参数：
* 返回值：0 表示已打开 0XF1 表示SPI已打开 其它出错
***********************************************************************************/
int spi_init(void)
{
    int fd;
    int ret = 0;

    // mode=SPI_MODE_0;

    if (g_SPI_Fd != 0) /* 设备已打开 */
    return 0xF1;

    fd = open(device, O_RDWR);
    if (fd < 0)
        printf("can't open device");
    else
        printf("SPI - Open Succeed. Start Init SPI...\n");

    g_SPI_Fd = fd;
    /*
    * spi mode
    */
    ret = ioctl(fd, SPI_IOC_WR_MODE, &mode);
    if (ret == -1)
        printf("can't set spi mode");


    ret = ioctl(fd, SPI_IOC_RD_MODE, &mode);
    if (ret == -1)
        printf("can't get spi mode");

    /*
    * bits per word
    */
    ret = ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    if (ret == -1)
        printf("can't set bits per word");


    ret = ioctl(fd, SPI_IOC_RD_BITS_PER_WORD, &bits);
    if (ret == -1)
        printf("can't get bits per word");

    /*
    * max speed hz
    */
    ret = ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);
    if (ret == -1)
        printf("can't set max speed hz");


    ret = ioctl(fd, SPI_IOC_RD_MAX_SPEED_HZ, &speed);
    if (ret == -1)
        printf("can't get max speed hz");


    printf("spi mode: %d\n", mode);
    printf("bits per word: %d\n", bits);
    printf("max speed: %d KHz (%d MHz)\n", speed / 1000, speed / 1000 / 1000);

    return ret;
}

/*************************************************************************************
* 功 能：自发自收测试程序
* 接收到的数据与发送的数据如果不一样 ，则失败
* 说明：
* 在硬件上需要把输入与输出引脚短跑
***********************************************************************************/
int spi_lookback_test(void)
{
    int ret, i;
    const int BufSize = 16;
    uint8_t tx[BufSize], rx[BufSize];


    bzero(rx, sizeof(rx));
    for (i = 0; i < BufSize; i++)
    tx[i] = i;

    printf("\nSPI - LookBack Mode Test...\n");
    ret = spi_transfer(tx, rx, BufSize);
    if (ret > 1)
    {
        ret = memcmp(tx, rx, BufSize);
        if (ret != 0)
        {
            printf("LookBack Mode Test error\n");
        //printf("error");
        }
        else
            printf("SPI - LookBack Mode OK\n");
    }

    return ret;
}


/***********************************************************************************
* 功 能：同步数据传输
* 入口参数 ：
* TxBuf -> 发送数据首地址
* len -> 交换数据的长度
* 出口参数：
* RxBuf -> 接收数据缓冲区
* 返回值：0 成功
***********************************************************************************/
int spi_transfer(const uint8_t *TxBuf, uint8_t *RxBuf, uint32_t len)
{
    int ret;
    int fd = g_SPI_Fd;
    
    struct spi_ioc_transfer tr ={
        .tx_buf = (unsigned long) TxBuf,
        .rx_buf = (unsigned long) RxBuf,
        .len = len,
        .speed_hz = speed,
        .delay_usecs = delay,
        .bits_per_word = bits,
        .cs_change = 0,
        .tx_nbits = 8,
        .rx_nbits = 8,
        .pad = 0,
    };

    ret = ioctl(fd, SPI_IOC_MESSAGE(1), &tr);
    if (ret < 1)
    printf("can't send spi message");
    else
    {
    #if SPI_DEBUG
        int i;
        printf("nsend spi message Succeed");
        printf("nSPI Send [Len:%d]: ", len);
        for (i = 0; i < len; i++)
        {
            if (i % 8 == 0)
            printf("nt");
            printf("0x%02X ", TxBuf[i]);
        }
        printf("n");


        printf("SPI Receive [len:%d]:", len);
        for (i = 0; i < len; i++)
        {
            if (i % 8 == 0)
            printf("nt");
            printf("0x%02X ", RxBuf[i]);
        }
        printf("n");
    #endif
    }
    return ret;
}


/**********************************************************************************
* 功 能：发送数据
* 入口参数 ：
* TxBuf -> 发送数据首地址
* len -> 发送与长度
* 返回值：0 成功
***********************************************************************************/
int spi_write(uint8_t *TxBuf, int len)
{
    int ret;
  
    ret = write(g_SPI_Fd, TxBuf, len);
    if (ret < 0)
    printf("SPI Write error\n");
#if SPI_DEBUG
    else
    {
        int i;
        printf("nSPI Write [Len:%d]: ", len);
        for (i = 0; i < len; i++)
        {
            if (i % 8 == 0)
            printf("nt");
            printf("0x%02X ", TxBuf[i]);
        }
        printf("\n");

    }
#endif

    return ret;
}


/**********************************************************************************
* 功 能：接收数据
* 出口参数：
* RxBuf -> 接收数据缓冲区
* rtn -> 接收到的长度
* 返回值：>=0 成功
***********************************************************************************/
int spi_read(uint8_t *RxBuf, int len)
{
    int ret;
    int fd = g_SPI_Fd;
    ret = read(fd, RxBuf, len);
    if (ret < 0)
    printf("SPI Read errorn\n");
    else
    {
    #if SPI_DEBUG
        int i;
        printf("SPI Read [len:%d]:", len);
        for (i = 0; i < len; i++)
        {
            if (i % 8 == 0)
            printf("nt");
            printf("0x%02X ", RxBuf[i]);
        }
        printf("\n");
    #endif
    }

    return ret;
}

int count = 0;
/*******************************************************************************
* 描述	    : 入口Main函数
*******************************************************************************/
int main(void) 
{      
    spi_init();
    uint8_t data[4] = "123";
    
    while(1) {
        spi_write(data, 3);
        sleep(1);
        count++;
        printf("count:%d\n", count);
    }
    return 0;
}

```

