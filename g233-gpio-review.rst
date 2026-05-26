G233 GPIO 实现复盘：从能过测试到更稳的设备模型
=====================================================

背景
----

这份笔记记录 Day2 GPIO 实验完成后的代码 review 结论，以及随后对
``hw/riscv/g233_feat/g233_gpio.c`` 的重构建议和修改理由。

目标不是否定原实现。原实现已经能覆盖 ``test-gpio-basic`` 和
``test-gpio-int`` 的主路径：MMIO 读写、``IN = OUT & DIR``、W1C、edge
rising、level high、PLIC IRQ 2 都能串起来。

这次优化关注的是：代码能过当前测试之后，如何减少后续做 PWM、WDT、
SPI 时遇到的状态混淆和副作用失控。

对之前代码的锐评
----------------

1. ``g233_update_state()`` 做得太多
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

之前的 ``g233_update_state()`` 同时负责：

* 计算当前 GPIO pin 电平；
* 更新 ``GPIO_IN``；
* 判断 edge interrupt；
* 判断 level interrupt；
* 修改 ``GPIO_IS``；
* 调用 ``qemu_set_irq()`` 把 IRQ 投影到 PLIC。

这种写法的优点是短、直接、容易一次写出来。缺点也明显：它把“状态计算”
和“副作用输出”混在一起了。后面如果一个测试失败，很难第一眼判断问题是
寄存器值错了、边沿判断错了，还是 IRQ 线没拉对。

更好的工程拆法是：

.. code-block:: text

   g233_gpio_level()
   -> 只计算当前 pin 电平

   g233_gpio_eval_interrupt()
   -> 根据寄存器和 pin 电平更新中断状态

   g233_gpio_update_irq()
   -> 只把 GPIO_IS / GPIO_IE 投影成 qemu_set_irq()

这个结构和 Day2 白皮书里的心智模型一致：

.. code-block:: text

   QTest
   -> GPIO MemoryRegion
   -> G233GPIOState
   -> update_irq()
   -> qemu_set_irq()
   -> PLIC pending/claim/complete

2. ``GPIO_IS`` 同时承载 edge 和 level，短期能过，长期危险
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

之前的实现只有一个 ``s->is``，edge sticky 状态和 level 当前状态都写进它。

当前测试只覆盖比较固定的路径：

* edge rising 触发；
* level high 触发后拉低；
* W1C 清 edge；
* PLIC IRQ 2 pending/claim/complete。

所以单个 ``s->is`` 足够通过测试。

但工程上有一个隐患：如果运行中切换 ``GPIO_TRIG`` 或 ``GPIO_POL``，旧的
level 状态可能残留在 ``s->is`` 里，被误当成 edge sticky 状态。也就是说，
它不是“当前真实硬件状态”，而是“各种来源混在一起后的结果”。

这类问题通常不会在第一批 happy path 测试里暴露，但后面一加边界测试就会
很难查。

3. W1C 逻辑是对的，但语义还可以更清晰
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

之前这段是正确的：

.. code-block:: c

   s->is &= ~value;

它表达的是 write-1-to-clear：写 1 的 bit 被清掉，写 0 的 bit 保持不变。

但如果 ``s->is`` 里混着 edge 和 level，两种状态的清除语义就不够直观。
edge 是 sticky，guest 写 1 清除后应该保持清除；level 是当前电平决定的，
如果电平仍然有效，清完后马上又可能被重算置位。

把二者拆开后，W1C 的行为更容易解释：

* ``edge_is``：事件发生后保持，直到 W1C；
* ``level_is``：由当前电平实时重算，W1C 只能清一次，若条件仍满足会再次出现。

4. MMIO access size 可以更明确
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

之前只设置了 ``.impl.min_access_size/max_access_size = 4``。

当前 qtest 都用 ``qtest_readl/qtest_writel``，所以没有问题。但从工程习惯看，
可以同时设置 ``.valid.min_access_size/max_access_size = 4``，明确这个设备只
接受 32-bit MMIO 访问。

这不是当前实验成败的关键，但属于“让模型边界更清楚”的小优化。

5. include guard 使用双下划线不推荐
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

之前头文件使用 ``__G233_GPIO_H__``。双下划线形式通常属于编译器/标准库保留
命名空间。实验里不会炸，但代码风格上更建议用：

.. code-block:: c

   #ifndef G233_GPIO_H
   #define G233_GPIO_H

本次修改建议和实际改动
----------------------

1. 拆分 GPIO 内部流程
~~~~~~~~~~~~~~~~~~~~~

本次把之前的“一坨 update”拆成三个函数：

.. code-block:: c

   static uint32_t g233_gpio_level(G233GPIOState *s);
   static void g233_gpio_eval_interrupt(G233GPIOState *s);
   static void g233_gpio_update_irq(G233GPIOState *s);

这样每个函数职责更清楚：

* ``g233_gpio_level()``：只回答“当前 pin 电平是多少”；
* ``g233_gpio_eval_interrupt()``：只回答“这些电平变化会不会产生中断状态”；
* ``g233_gpio_update_irq()``：只回答“当前中断状态是否需要拉高 IRQ 线”。

这对后续外设也有借鉴意义。WDT 可以拆成 ``update_counter()``、
``update_status()``、``update_irq()``；SPI 可以拆成 ``do_transfer()``、
``update_status()``、``update_irq()``。

2. 增加 ``edge_is`` 和 ``level_is``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

头文件新增：

.. code-block:: c

   uint32_t edge_is;
   uint32_t level_is;

然后统一由：

.. code-block:: c

   s->is = s->edge_is | s->level_is;

生成 guest 看到的 ``GPIO_IS``。

这样 ``GPIO_IS`` 仍然是对外寄存器，行为不变；内部实现则能区分：

* 哪些 bit 是 sticky edge 事件；
* 哪些 bit 是实时 level 状态。

3. W1C 清理两个来源，再重新评估
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

写 ``GPIO_IS`` 时现在做：

.. code-block:: c

   s->edge_is &= ~value;
   s->level_is &= ~value;
   g233_gpio_eval_interrupt(s);

这表示 guest 先清设备侧状态，然后设备再根据当前电平重新评估。

如果是 edge 中断，清完就清掉了；如果是 level 中断且电平仍有效，它会再次
出现。这更接近真实硬件里的 level interrupt 直觉。

4. ``update_irq()`` 只负责 IRQ 投影
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

现在 ``g233_gpio_update_irq()`` 的语义很窄：

.. code-block:: c

   s->is = s->edge_is | s->level_is;
   qemu_set_irq(s->irq, (s->is & s->ie) != 0);

它不判断边沿，不计算电平，只把寄存器状态投影成 IRQ 线。

这点很重要：GPIO 设备不应该直接操作 PLIC pending，也不应该理解
claim/complete。GPIO 只负责 ``qemu_set_irq()``，PLIC 负责自己的
pending/claim/complete。

5. 保留训练营最小模型，不提前复杂化
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当前仍然保留：

.. code-block:: c

   uint32_t external_in = 0;

原因是 Day2 测试没有外部设备驱动 GPIO 输入 pin。现在把外部输入先建模成
0，输出模式下通过 ``OUT & DIR`` 回读 ``IN``，已经足够覆盖训练营要求。

不要为了“真实 GPIO”提前引入多 bank、外部输入线、qdev GPIO out array。
这些会增加复杂度，但对当前 SoC 模块并不产生收益。

修改后的心智模型
----------------

.. code-block:: text

   qtest_writel(GPIO_OUT, value)
   -> g233_gpio_write()
   -> s->out = value
   -> g233_gpio_eval_interrupt()
      -> g233_gpio_level()
      -> 更新 s->in
      -> edge: 计算 0->1 / 1->0，写入 edge_is
      -> level: 按当前电平重算 level_is
   -> g233_gpio_update_irq()
      -> s->is = edge_is | level_is
      -> qemu_set_irq(s->irq, (s->is & s->ie) != 0)
   -> PLIC 看到 IRQ 2

清中断时：

.. code-block:: text

   qtest_writel(GPIO_IS, 1)
   -> 清 edge_is / level_is 对应 bit
   -> 重新评估当前电平
   -> 重新计算 GPIO_IS
   -> 重新拉高或拉低 IRQ 线

PLIC 侧仍然是另一层：

.. code-block:: text

   GPIO qemu_set_irq()
   -> PLIC pending
   -> read PLIC_CLAIM 得到 IRQ 2
   -> write PLIC_CLAIM 完成 IRQ 2

对后续实验的提醒
----------------

PWM
~~~

PWM 很容易写成“寄存器一写就马上出结果”。建议提前拆：

* 配置寄存器；
* 计数器状态；
* compare/period 判断；
* IRQ 更新。

不要把 timer callback、寄存器写、副作用全塞进一个函数。

WDT
~~~

WDT 的核心状态通常包括 counter、timeout flag、enable、IRQ enable。

建议从一开始就区分：

* timeout 原因位；
* IRQ enable；
* timer 是否正在跑；
* reset 或 interrupt 是哪种动作。

这和 GPIO 拆 ``edge_is/level_is`` 是同一个思想：状态来源分开，最后再汇总。

SPI
~~~

SPI 后面会涉及 TX/RX、CS、overrun、JEDEC response。最容易错的是状态位和
副作用顺序。

建议把 SPI 拆成：

* 写 TX 触发一次 transfer；
* transfer 修改 RX/status；
* status 决定是否 IRQ；
* CS 只控制设备选择，不要和数据状态混在一起。

一句话结论
----------

原实现是合格的 Day2 最小解；这次重构把它往“可维护设备模型”推进了一步。
关键不是多写代码，而是把状态来源和副作用边界拆清楚：edge 是 sticky，
level 是实时状态，IRQ 是状态的投影，PLIC 是外部中断控制器。
