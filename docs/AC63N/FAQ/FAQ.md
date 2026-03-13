# FAQ

## 1. 关于低功耗唤醒问题

### 1. shutoff

也叫软关机，该模式功耗为2uA，基本所有的芯片都是这个功耗。该模式下RAM是会掉电的，退出该模式后所有外设需要重新初始化

#### a. 进入条件

调用spple_power_event_to_user(POWER_EVENT_POWER_SOFTOFF);函数
该函数是AC630N_bt_data_transfer_sdk_release_v2.4.0版本的写法，其他低版本的sdk没有该函数请参考240版本实现一下。
**注意！！！**
spple_set_soft_poweroff
power_set_soft_poweroff
这两个函数只能在app_core任务中调用，不可以在中断、系统定时器、软件定时器、其他任务中调用

#### b. 唤醒条件

- 按键IO中断唤醒
- lp内置触摸按键唤醒
  部分芯片支持ac637、ac638
- rtc定时器唤醒

### 2. powerdown

  也叫低功耗、该模式RAM是不掉电的、但是硬件外设是会掉电的例如（uart、 硬件定时器等）。进入前芯片会保存寄存器的状态、被唤醒后芯片会从之前的状态继续运行。硬件外设不需要重新初始化。

#### a、进入条件

首先需要使能低功耗模块

![img](img/101.png)

程序不能控制芯片主动进低功耗，但是程序可以控制芯片不要进低功耗。那么反过来如果我们的程序进不了低功耗就可以查一下REGISTER_LP_TARGET注册的函数是不是有地方没有返回1；

```
//代码任意地方添加此段代码,全局修改此变量,底层会自动获取判断
u8 can_enter_lp = 0; //置1 可进睡眠，置0 不可进睡眠
static u8 custom_idle_query(void)
{
    if(can_enter_lp){
        return 1;
    }else{
        return 0;
    }
}
REGISTER_LP_TARGET(custom_lp_target) = {
    .name = "custom_lp",
    .is_idle = custom_idle_query,
};
```

#### b、唤醒条件

- 系统定时器唤醒

```
u16 sys_timer_add(void *priv, void (*func)(void *priv), u32 msec);
u16 sys_timeout_add(void *priv, void (*func)(void *priv), u32 msec);
```

- 按键IO中断唤醒
- lp内置触摸按键唤醒
  部分芯片支持ac637、ac638
- rtc定时器唤醒

#### c、进出低功耗回调函数

![img](img/102.png)

进入低功耗的时候会将所有的IO设置为高阻态，如果想维持原先io状态需要在close_gpio函数中调用port_protect函数

![img](img/103.png)

有打印<>和[]就表示有正常进低功耗了'<'和'>'是应用层的打印，'['和']'是库里面的打印，

#### d、低功耗参数配置

```
struct low_power_param {
    u8 osc_type;    //低功耗下时钟的选择：内部OSC_TYPE_LR时钟、外部OSC_TYPE_BT_OSC蓝牙时钟（需要btosc_disable=1）
    u32 btosc_hz;    //低功耗下的时钟频率
    u8  delay_us;    //提供给低功耗模块的延时(不需要需修改)
    u8  config;      //是否开启低功耗模块
    u8  btosc_disable;    //低功耗下是否维持外部晶振

    u8 vddiom_lev;    //工作模式下的vddio电压等级
    u8 vddiow_lev;    //低功耗模式下的vddio电压等级
    u8 pd_wdvdd_lev;    //wdvdd的电压等级
    u8 lpctmu_en;        //是否支持内置触摸按键唤醒
    u8 vddio_keep;       //低功耗下vddio是否跟正常模式下保持一致

    u32 osc_delay_us;    //晶振启动后延时多少个时钟周期退出低功耗模式

    u8 vd13_cap_en;    //有BT_AVDD引脚电容,可以置1,否则要配0
    u8 rtc_clk;        //低功耗下rtc时钟是否维持
    u8 light_sleep_attribute;

};
```

![img](img/104.png)

