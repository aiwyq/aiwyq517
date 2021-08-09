
## Arduino指纹解锁门禁系统
支持按键`录入指纹`、`删除指纹`、`验证指纹`并驱动舵机完成开锁动作

在此基础上可添加`ESP8266WIFI`模块实现远程开锁、小爱同学语音控制等功能



## 效果图:
![](https://img-blog.csdnimg.cn/20210603113306129.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fpd3lxcQ==,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/763418ac971e486385dfe6b35d945709.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fpd3lxcQ==,size_16,color_FFFFFF,t_70#pic_center)
![csdn](https://img-blog.csdnimg.cn/20210603111950766.gif)


<br>

## 硬件清单


|`````——————————材料—————————— `````|`````——————————用途——————————````` |
| ------ | ------ |
| Arduino Nano |开发板 处理各种数据 |
| ESP8266 WIFI模块 |用于数据远程传输|
| AS608指纹模块 | 用于采集指纹 |
| MG90S舵机 | 用于带动门锁 |
|OLED0.96| 显示系统信息 |
|4x4矩阵按键| 操作交互 |
|USB电源模块| 为多级和开发板供电 |
|DHT11温度传感器| 采集环境温度与湿度 |
<br>
* 合计一百元(2020 12.29)

## Hardware connection：
![image](https://img-blog.csdnimg.cn/20210603112422449.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fpd3lxcQ==,size_16,color_FFFFFF,t_70)  <br>


## Procedure flow：
![image](https://img-blog.csdnimg.cn/a01ee9a0a1344d4db2b275144f1bbf0b.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fpd3lxcQ==,size_16,color_FFFFFF,t_70#pic_center)  <br>

The complete source code and font files are in the source code folder above

完整源码和font文件在上方Source Code文件夹中<br>
另附百度网盘链接: https://pan.baidu.com/s/1KsvAKneo3ZYYGctuGGUrmA <br>
提取码: hjpt <br>

 *************更新内容**************************************
 * 更新于2021年5月27日  B站这个视频终于10万播放量了
 * 更新一下新的代码叭：
 * 
 * 将之前的u8g2库更改为u8g库；
 * 功能不变的下精简优化了代码 减少体积30%
 * Arduino Nano ROM占用由99%降低至69%；
 * 优化系统里流畅度；
 * 提高系统稳定性；
 * --------------------------------------------------------
 * By: Zyb 
 * Bilibili: aiwyq
 * Coolapk:  aiwyq
 * VX /Phone: 17820488893
 *             2021 05.27
 **********************************************************       
 */


请提前装好以下库文件 `工具-管理库(Ctrl+Shift+I):`
`U8glib库` `DHT11库` `Adafruit_Fingerprint库`
`报错可联系我 欢迎交流意见`
* 不足之处还请各位指正



## 局部源码：
```c++
/*

#include <U8glib.h>                 //u8g库 用于0.96 OLED IIC显示器 修改于21年5.24 原为u8g库
#include <Adafruit_Fingerprint.h>   //AS608指纹库
#include<DHT.h>                     //温湿度传感  
#include "font.h"                   //调用同目录下的字库
DHT dht(7,DHT11);                   //温湿度data接脚
#ifdef U8X8_HAVE_HW_I2C
#endif
U8GLIB_SSD1306_128X64 u8g(U8G_I2C_OPT_NONE | U8G_I2C_OPT_DEV_0);  // I2C / TWI
#define x_coordinate 40
#define KEY1 2
#define KEY2 3
#define KEY3 4
#define KEY4 5
#define KEY5 6
SoftwareSerial mySerial(11,12);     //新建一个名为mySerial的软串口 并将11号引脚作为RX端 12号引脚作为TX端
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);
u16 q=1024,t,h;     //累计开门次数、温度、湿度
u8 key_num=0;

void key_init()
{
    pinMode(KEY1, INPUT_PULLUP);
    pinMode(KEY2, INPUT_PULLUP);
    pinMode(KEY3, INPUT_PULLUP);
    pinMode(KEY4, INPUT_PULLUP);
    pinMode(KEY5, INPUT_PULLUP);
}

u8 key_scan(u8 mode)
{
    static u8 key_up=1;         //按键按松开标志
    if(mode)key_up=1;           //支持连按
    if(key_up&&(digitalRead(KEY1)==0||digitalRead(KEY2)==0||
                digitalRead(KEY3)==0||digitalRead(KEY4)==0||digitalRead(KEY5)==0))
    {
        delay(10);
        key_up=0;
        if(digitalRead(KEY1)==0)return 1;
        else if(digitalRead(KEY2)==0)return 2;
        else if(digitalRead(KEY3)==0)return 3;
        else if(digitalRead(KEY4)==0)return 4;
        else if(digitalRead(KEY5)==0)return 5;
    }
    else if(digitalRead(KEY1)==1&&digitalRead(KEY2)==1&&
            digitalRead(KEY3)==1&&digitalRead(KEY4)==1&&digitalRead(KEY5)==1)
        key_up=1;
    return 0;   // 无按键按下
}


void MG90S()//MG90S舵机PWM脉冲
{
    for(int i=1; i<51; i++)
    {
        servopulse(75);
        delay(10);
    }
    delay(1444);
    for(int i=1; i<51; i++)
    {
        servopulse(10);
        delay(10);
    }
}

void servopulse(int angle)//MG90S舵机PWM脉冲
{
    int pulsewidth = (angle * 11) + 500;
    digitalWrite(9, HIGH);
    delayMicroseconds(pulsewidth);
    digitalWrite(9, LOW);
    delayMicroseconds(20000 - pulsewidth);
}


/**
@添加指纹
*/
void Add_FR()      
{  
    u8 i,ensure,processnum=0;    
    u8 ID_NUM=0;        
    char str2[10];
    while(1)
    {
        switch (processnum)
        {
        case 0:
            i++;
            u8g.firstPage();
            do
            {
                u8g.drawXBMP(32,24,64,16,State5);  /* 字串 请按手指   64x16  */
            }
            while(u8g.nextPage());
            ensure=finger.getImage();
            if(ensure==FINGERPRINT_OK)
            {
                ensure=finger.image2Tz(1);//生成特征
                if(ensure==FINGERPRINT_OK)
                {
                    u8g.firstPage();
                    do
                    {
                        u8g.drawXBMP(32,24,64,16,State6);  /* 字串 指纹正常  64x16  */
                    }
                    while(u8g.nextPage());
                    // Serial.println(" 000 is true");
                    i=0;
                    processnum=1;//跳到第二步
                }
                else {};
            }
            else {};
            break;

        case 1:
            i++;
            u8g.firstPage();
            do
            {
                u8g.drawXBMP(32,24,64,16,State7);  /* 字串 再按一次   64x16  */
            }
            while(u8g.nextPage());
            ensure=finger.getImage();
            if(ensure==FINGERPRINT_OK)
            {
                ensure=finger.image2Tz(2);//生成特征
                if(ensure==FINGERPRINT_OK)
                {
                    u8g.firstPage();
                    do
                    {
                        u8g.drawXBMP(32,24,64,16,State6);  /* 字串 指纹正常  64x16  */
                    }
                    while(u8g.nextPage());
                    i=0;
                    processnum=2;//跳到第三步
                }
                else {};
            }
            else {};
            break;

        case 2:
            u8g.firstPage();
            do
            {
                u8g.drawXBMP(32,24,64,16,State8);/* 字串 创建模板   64x16  */
            }
            while(u8g.nextPage());
            ensure=finger.createModel();
            if(ensure==FINGERPRINT_OK)
            {
                u8g.firstPage();
                do
                {
                    u8g.drawXBMP(16,24,96,16,State9);  /* 字串 模板创建成功   96x16  */
                }
                while(u8g.nextPage());
                processnum=3;//跳到第四步
            }
            else
            {
                u8g.firstPage();
                do
                {
                    u8g.drawXBMP(16,24,96,16,State10);  /* 字串 模板创建失败   96x16  */
                }
                while(u8g.nextPage());
                i=0;
                processnum=0;//跳回第一步
            }
            delay(500);
            break;
        case 3:

            u8g.firstPage();
            do
            {
                u8g.drawXBMP(1,0,128,48,State11);
                /* 字串  按K4加，按K2减 按K3保存 0=< ID <=99 128x48*/
                u8g.setFont(u8g_font_6x10); // 选择字体
                u8g.drawStr(40,62,"ID=00");
            }
            while(u8g.nextPage());

            while(key_num!=3)
            {
                key_num=key_scan(0);
                if(key_num==2)
                {
                    key_num=0;
                    if(ID_NUM>0)
                        ID_NUM--;
                    if(ID_NUM<10)
                        sprintf(str2,"ID=0%d",ID_NUM);
                    else
                        sprintf(str2,"ID=%d",ID_NUM);
                    u8g.firstPage();
                    do
                    {
                        u8g.setFont(u8g_font_6x10); // 选择字体
                        u8g.drawXBMP(1,0,128,48,State11);
                        u8g.drawStr(40,62,str2);
                    }
                    while(u8g.nextPage());
                }
                if(key_num==4)
                {
                    key_num=0;
                    if(ID_NUM<99)
                        ID_NUM++;
                    if(ID_NUM<10)
                        sprintf(str2,"ID=0%d",ID_NUM);
                    else
                        sprintf(str2,"ID=%d",ID_NUM);
                    u8g.firstPage();
                    do
                    {
                        u8g.setFont(u8g_font_6x10);
                        u8g.drawStr(40,62,str2);
                        u8g.drawXBMP(1,0,128,48,State11);
                    }
                    while(u8g.nextPage());
                }
            }
            key_num=0;
            ensure=finger.storeModel(ID_NUM);//储存模板
            if(ensure==0x00)
            {
                u8g.firstPage();
                do
                {
                    u8g.drawXBMP(16,24,96,16,State12);  /* 字串 录入指纹成功   96x16  */
                }
                while(u8g.nextPage());
                Serial.println("FR receive OK");
                delay(1500);
                return ;
            }
            else
            {
                processnum=0;
            }
            break;
        }
        delay(400);
        if(i==10)//超过5次没有按手指则退出
        {
            break;
        }
    }
}



void Press_FR()
{
    ...
}


void Del_FR()  
{
    ...
}


void MENU()
{
    u8g.firstPage();
    do
    {
        u8g.setFont(u8g_font_6x10); // 选择字体
        u8g.drawXBMP(4,0,112,16,State1);  //显示字模汉字   /* 竹401指纹门禁  128x16  */
        u8g.drawXBMP(40,16,64,16,State2);/* 字串 添加指纹   64x16  */
        u8g.drawXBMP(40,32,64,16,State3);/* 字串 删除指纹   64x16  */
        u8g.drawXBMP(40,48,64,16,State4);/* 字串 验证指纹   64x16  */
        u8g.drawStr(22,30,"K1");
        u8g.drawStr(22,46,"K2");
        u8g.drawStr(22,62,"K3");
    }
    while(u8g.nextPage());
    Serial.println("一切准备就绪....");
    Serial.println("By Zhongyanbin");
}

void setup()
{
    // put your setup code here, to run once:
    key_init();
    dht.begin();//开启温湿度
    u8g.begin();       //开启OLED通信
    Serial.begin(9600); //开启串口通信 波特率9600
    finger.begin(57600); //设置AS608波特率 57600
    MENU();
}

void loop()
{
   ...
}

```

![](https://img-blog.csdnimg.cn/20210603113315174.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fpd3lxcQ==,size_16,color_FFFFFF,t_70)
![](https://img-blog.csdnimg.cn/3cb86d868eef443f8ea94ec89b8bf6ac.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fpd3lxcQ==,size_16,color_FFFFFF,t_70#pic_center)
![](https://img-blog.csdnimg.cn/20210603113323755.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fpd3lxcQ==,size_16,color_FFFFFF,t_70)
