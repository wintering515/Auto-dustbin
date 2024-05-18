# Auto-dustbin


# 智能垃圾桶项目recommend

#本项目只完成基础功能，进阶功能还在更新

## 1.配件清单

基础配件：STM32F103系列开发板，HC-SR04超声波模块，SG90舵机模块

进阶配件：JDY31蓝牙模块，096寸OLED显示屏

## 2.实现功能：

基础功能：是用超声波模块检测距离，当距离达到阈值，控制舵机打开垃圾桶

进阶功能：提供按键手动控制打开垃圾桶功能，使用蓝牙模块远程控制垃圾桶打开功能，使用OLED显示屏显示当前垃圾桶状态和容量

## 3.项目主要步骤

### 链接开发板基础配置部分：

1）打开仿真

![1716023581301](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716023581301.png)

2）打开时钟

![1716023631034](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716023631034.png)

3）设备数HCLK改成72

![1716023677920](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716023677920.png)

4）工程保存名字，路径，IDE改成MDK-ARM

![1716023744128](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716023744128.png)

5）舵机DWM用TIM1模块完成修改clock source&channel&configuration（需计算）里的PSC和Counter period

`72000000/PSC/1000=频率50HZ（0.02s=20ms周期）`

![1716023931976](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716023931976.png)

6）超声波模块先连接双边缘引脚

先把GPIO model改成rising/falling（双边缘）

然后再把改PB3（GPIO_Output)/PB4(GPIO_EXIT4)

![1716024359664](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716024359664.png)

7）开启定时器3和4来设置超声波部分的控制

![1716025420822](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716025420822.png)

![1716025431080](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716025431080.png)

### 编程部分：

#### A. main.c:

1）控制舵机开关：

```c
HAL_TIM_PWM_Start(&htim1,TIM_CHANNEL_1);//open PWN channel
TIM1->CCR1=50;//turn on -90，50HZ means every 20ms turn on -90
HAL_Delay(1000);//delay function
TIM1->CCR1=100;//turn on -45
HAL_Delay(1000);
TIM1->CCR1=150;//turn on 0
HAL_Delay(1000);
TIM1->CCR1=200;//turn on 45
HAL_Delay(1000);
TIM1->CCR1=250;//turn on 90
```

2）完成driver.c&driver.h之后在main.c里去调用两个函数

```c
#include "driver_sr04.h"
```

3）完成步骤2）后定义超声波以10ms发送一次

#假如到达50ms后更新然后发送超声波信号

#定义count变量让操作别太快（2ms更新一次操作）

#当小于10的时候就可以进行打开垃圾桶的操作否则关闭

```c
int count = 0；
while (1)
  {
		int SR04_tick =0;//define a varible to be sent every ten milliseconds
		
		if(HAL_GetTick() - SR04_tick > 50)//when the time arrive 50ms
		{
			SR04_tick = HAL_GetTick();//update time
			SR04_Tirgger();//sent the signal about driver
		  if(++count == 40)//make the funtion unquickly
			{
				count = 0; 
				
				if(GetDistance()<10)//when the distance less than 10
				{
					TIM1->CCR1=250;//turn on 90
				}
				else 
				{
					TIM1->CCR1=150;//turn on 0
				}
				
			}
		
		}
		
```



#新建文件夹放超声波模块，文件夹里新建文件.c/.h

在keil5里的test1文件夹右键manage program item里添加group“driver”，双击driver将新建的.c添加进去，在option for target “test1”里的C/C++里include paths添加driver路径![1716024787892](C:\Users\winter\AppData\Roaming\Typora\typora-user-images\1716024787892.png)

#### B. driver.c:（触发信号—>模块内部发出信号—>输出回响信号）

#链接头文件

#声明两个引脚（定时器3和定时器4）

#定义一个距离变量distance_cm

#微秒之间的延时函数：打开和关闭定时器3

#触发信号的延时函数

#外部中断（时间是低电平-高电平）

【测试距离=（高电平时间*声速（340m/s））/2】（注意单位转换）

【高电平开启定时器4开始计时用if函数来进行时间的增加

#回调函数返回距离



```c
#include "driver_sr04.h"

extern TIM_HandleTypeDef htim3;//statement time3
extern TIM_HandleTypeDef htim4;//statement time4

uint32_t distance_cm = 0;//statement distance

void Delay_us(uint16_t us)//set the timer function between microseconds
{
	uint16_t differ = 0xffff - 5;//initialize time3
	__HAL_TIM_SET_COUNTER(&htim3,differ);
	HAL_TIM_Base_Start(&htim3);//turn on time3
	
while(differ < 0xffff - 5)
{
	differ = __HAL_TIM_GET_COUNTER(&htim3);

}
HAL_TIM_Base_Stop(&htim3);//turn off time3

}

void SR04_Tirgger(void)
{
	Tirg_ON;
	Delay_us(10);//delay function
	Tirg_OFF;
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)//callback function for external interrupt
{
	static int count = 0;//define an autonomous variable
	
if(GPIO_Pin == GPIO_PIN_4)
{
	if(HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_4))
	{
		HAL_TIM_Base_Start(&htim4);//start counting when the level is high
		__HAL_TIM_SetCounter(&htim4, 0);//empty timer4
	}

	else if(HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_4) == 0)//when the level is low
	{
		HAL_TIM_Base_Stop(&htim4);//turn off timer4
		count = __HAL_TIM_GetCounter(&htim4);
		distance_cm = count * 340 / 2 *0.0000001 * 100;//formula
	}
}

}
uint32_t GetDistance(void)//return void
{
	return distance_cm;
}`
```



#### c. driver.h:

#找到格式

#链接main

#定义开启和关闭

#定义函数触发信号

#回调函数：返回距离



```c
#ifndef __DRIVER_SR04_H__
#define __DRIVER_SR04_H__

#include "main.h"

#define Tirg_ON HAL_GPIO_WritePin(GPIOB,GPIO_PIN_3,GPIO_PIN_SET)
#define Tirg_OFF HAL_GPIO_WritePin(GPIOB,GPIO_PIN_3,GPIO_PIN_SET)

void SR04_Tirgger(void);//touch signal

uint32_t GetDistance(void);//return void 

#endif
```

