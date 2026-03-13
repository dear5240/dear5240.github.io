# 3. tick_timer

tick_timer是一个简单的定时器，在该tick_timer初始化后，会以2ms的间隔起来一次中断，优先级为IRQ_TTICKTMR_IP.

具体tick_timer中断如下:

```
SET(interrupt(""))
void tick_timer_isr()
{
    /* 用户的timer函数不能加入到这里,加到tick_timer_loop */
    j32CPU(core_num())->TTMR_CON |= BIT(6);

    tick_timer_ram_loop();
    if (tick_timer_close) {
        if (tick_timer_close()) {
            return ;
        }
    }

    tick_timer_loop();
}
```

**在该中断里面主要分为两部分：**

​		1、tick_timer_ram_loop() ： 该函数里面需要全部放入ram中执行；2、tick_timer_loop() ： 该函数用户正常处理；

**设计原因为：**

​		相关底层驱动在操作flash时会关闭总中断，又想在这段时间里执行tick_timer中断

**那么就需要满足两个条件：**

​		1、关总中断时额外放开tick_timer中断不受影响。

​		2、执行的代码都放在ram中；（避免关闭了flash，中断又需要访问flash来执行代码导致卡死问题）

那么在系统底层驱动操作flash时，虽然关闭了总中断，但是放开了 **tick_timer** 中断， **tick_timer** 中断运行了 **tick_timer_ram_loop()** 部分后， **tick_timer_close()** 判断为真而不执行后面的 **tick_timer_loop()**；

这样用户可以flash关闭期间仍旧执行放在tick_timer的代码操作。

## 3.1. 注意事项

1. tick_timer实际是一个定时器触发中断的行为，所以用户在tick_timer函数里不要执行过长时间以免中断执行过久。
2. tick_timer中断的默认优先级为3，存在会被优先级比他高的中断打断 或者 被同级优先级的中断延迟起中断时间 的问题。

















