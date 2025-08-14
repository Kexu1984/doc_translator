你是精通嵌入式的工程师，对芯片硬件的实现原理和驱动程序非常了解。这是一个通过python实现的硬件模拟器框架，这个框架用于模拟一个Soc/MCU系统，内部包含了bus，dma以及其他的外设.

我需要创建一个硬件模块框架，该框架用python（python3）实现，并且运行在linux环境上，现有如下需求：
1. 这个框架内应该至少包含几个主要的组件：Top model，bus model和device model；
2. Top model用于组合所有如下描述的组件，要求包含一个bus，一个memory model以及至少一个DMA device model
3. bus model主要用于模拟总线的行为
4. device model就是用于模拟各类外设的行为，例如：uart，crc，dma，memory等等
5. 应该包含一个公共的trace class，用于记录top/bus以及各个device的访问情况
通过组合上述的这些组件，我就可以生成一个近似完整的MCU模拟框架！！！

关于bus model，我有如下需求：
1. bus model应该具备管理device model的能力，例如："add_device"和"remove_device"；这两个函数用于向bus model去添加设备，并且也应该能够将bus自身的实例传递给device，以使得对应的device可以调用bus的接口
2. 每一个device都应该至少包含如下信息：地址空间，bus在添加对应的设备之前应该先检查当前准备添加的device的地址空间是否与已有的配置冲突
3. bus应该包含一个全局锁，去管理来自外部（c语言程序）的多线程访问请求
4. bus应该对外提供两个接口：read和write：
4.1）read接口需要识别参数：“master_id”（每一个device都应该有一个独立的master id）
4.2）read接口需要识别参数：“address”，如果address不再任何已注册的设备地址空间中，则返回response error；如果找到了匹配的地址空间，就继续调用找到的device.read接口将read request向下分发出去
4.3）write接口与read接口类似，区别就是write用于派发write request
4.4）read接口应该返回设备读到的内容，write接口应该返回设备写操作的状态；

关于device model，我有如下需求：
1. 此处应该有一个公共类：base class，任何具体device模型都应该继承这个base class
1.1）base class应该包含如下基础函数：init，read，write，register_irq_callback；
1.2）read和write通常对应设备寄存器的读写操作，也就是最终会调用到下面2.1中所说的register manager的read_callback和write_callback
1.3）如果对应设备是内存模型，那么read和write就对应内存的读写操作，此时就不需要read_callback和write_callback
1.4）init通常做如下操作：寄存器的添加，状态设置等；
1.5）register_irq_callback函数的作用：外部如果实现了发送中断的功能，就可以通过register_irq_callback将其中的send irq的函数传递给此设备；
2. 此处应该有一个公共类：register manager class，任何具体device模型都应该包含此类用于描述设备的所有寄存器
2.1） register manager class中应该包括一个函数：“add register”，用于添加一个寄存器，函数原型如下：
===================
    def define_register(self, offset: int, name: str, register_type: RegisterType = RegisterType.READ_WRITE,
                       reset_value: int = 0, mask: int = 0xFFFFFFFF,
                       read_callback: Optional[Callable[[self, int, int], int]] = None,
                       write_callback: Optional[Callable[[self, int, int], None]] = None) -> None
offset: register的偏移地址
read_callback: 如果有些寄存器的读取行为会触发一些额外的操作，那么对应的寄存器就需要注册此read_callback，例如：有些状态寄存器具有“读清零”的功能，那么当外部准备读此寄存器时，read_callback里应该就把对应的状态位清零；
write_callback：如果有些寄存器的写行为会触发一些额外的操作，那么对应的寄存器就需要注册此write_callback，例如：一些控制寄存器的使能位如果在写操作时被设置了，那么就相当于触发此设备开始工作，此时write_callback中就应该加上设备工作的具体流程
read_callback和write_callback应该能够拿到当前设备的所有寄存器，因为有些寄存器的读写会影响其他寄存器的值
===================
3. device_model应该能够被多次实例化以表示多个相同的设备，例如：一个Soc中可能存在多个uart单元
4. device_model如果具备触发中断的能力，那么应该在对应设备的某个job完成之后通过send irq的callback去对外发送中断；
5. device_model如果具备访问DMA的能力，那么也应该支持dma interface这个类，用于主动操作dma设备；
6. 内存的读写可以支持非4字节的宽度，所以内存的read和write应该额外加上一个参数：width;所以，其他device的read/write的width参数默认是4bytes
7. 对于能够连接外设的device（例如：uart，spi，can等）来说，还需要集成一个IO Interface类，该类具备如下说明：
7.1）支持Input和Output功能用于支持设备的数据输入和输出功能：已知c driver代码只能操作寄存器实现设备的Output功能，例如：驱动往uart的TX寄存器写数据后，uart就可以发送数据到外部；
7.2）对于Input而言，device需要具备connect接口，用于表示“存在外设连接上本设备”，如果是要求当前device output功能的connect，就要求外部设备创建thread等待output的数据，如果是要求当前device input功能的connect，此时就需要device去创建一个thread用于实时的获取外部设备随时可能发送来的数据；
7.3）无论输入还是输出都是至少两个参数：data和width，分别表示输入输出的数据和宽度；


关于top model，我有如下需求：
- 支持输入参数选定config.yaml/config.json用于描述当前系统中包含哪些组件；
- "top model"应该首先初始化其环境，然后再根据config文件依次创建组件：bus，memory，device等，然后再根据前面所说，将device添加到bus上
- 我也需要一个test model，test model用于做和top model一样的事，区别是test model创建的device比较少，同时也会通过对地址的读写操作实现各设备寄存器的访问，从而实现设备的操作流程
- 一个完整的测试通信链路至少要包含："test_model"->"bus_model"->"device_model"->"register manager"，同时能够得到正确的返回结果；
- 我可以提供一个例子：在test model中通过配置crc模块和dma模块实现dma的mem2peri模式的功能，要求输入memory的内容后，crc能够通过dma自动算出最终结果，并与预期结果相符；同时也可以直接配置dma实现其基础的mem2mem的内存拷贝功能
- test model中也需要验证各个model的trace功能

关于trace class，我有如下需求：
1. trace功能做成一个公共的类，使得top/bus/device各个组件都支持此trace功能；
2. trace支持按照模块去开关，这样可以节省空间，同时便于分析指定模块的功能；
3. 现有trace信息中的timestamp可读性差，优化成标准时间
4. trace信息需要保存在文件中：所有trace信息需要记录在一个文件内，也就是说trace buffer是共用一块，统一由trace manager去管理；

其他注意事项：
- 每一个模块应该生成一个对应的README文件，里面至少包含各个寄存器的描述信息