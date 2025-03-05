# PA

# 基础设施

- preliminary

	- -Wall -Werror
	- fault–(test or printf)—>error–(assert(0))—>failure
	- make ARCH=riscv32-nemu ALL= dummy gdb

- trace

	-  function pointer           

		  `int (*fpr)(int, int);`                 

		  `void add(int a,int b);`   

		  `fpr= &add; `//&可有可无 

		  `fpr(2,3);`

	-  指针数组(example)

		`char *iringbuf[3] = {“abc”,”def”,“ghi”};` 

		`char *p0 = “abc” = iringbuf[0] = *iringbuf;`   `char **p0 = &p0 = iringbuf;`   

		`char *p1 = “def” = iringbuf[1] = *(iringbuf+1);`   `char *p1 = &p1 = iringbuf+1;`   

		`char *p2 = “ghi” = iringbuf[2] = *(iringbuf+2);`  `char *p2 = &p2 = iringbuf+2;`      

- difftest

	指令级行为验证 –> nemu & spike接受相同的输入，观测行为是否相同

	cpu_exec()主循环中调用difftest_step(s->pc,dnpc)  

	init_difftest()  DUT的guest memory, 寄存器状态拷贝到REF中，REF和DUT处于相同的状态.

	ref_difftest_exec(1)   difftest_exec(1)  REF执行相同的指令

	ref_difftest_regcpy(&ref_r, DIFFTEST_TO_DUT) 读出REF中的寄存器 

	extern “C”   make a function name in C++ have C linkage

	void *dlsym(void *handle, const char *symbol)，ref_so_file中定义difftest_regcpy函数

​       先声明函数指针void (*ref_difftest_regcpy)(void *dut, bool direction) = NULL;

​       ref_difftest_regcpy = dlsym(handle,”difftest_regcpy”)将ref_difftest_regcpy指向difftest_regcpy

​       ref_difftest_regcpy(&cpu,difftest_to_ref);



### 运行时环境

- 内联汇编语句

```c
//halt(code)-->nemu_trap(code)-->
asm volatile("mv a0, %0; ebreak" : :"r"(code));//把结束码移动到通用寄存器中
//功能上和nemu/src/isa/riscv32/inst.c中nemu_trap的行为对应
//NEMU_TRAP(thispc:s->pc,code：R(10)) —> set_nemu_state(NEMU_END, thispc:s->pc,code:R(10)),monitor根据结束码报告程序结束的原因R(10)==a0
```

- 批处理模式运行NEMU
	- hint: NEMU中实现了批处理模式，启动NEMU之后可以直接运行客户程序，riscv.mk  nemu.mk  riscv32-nemu.mk –> AM Makefile –> NEMU Makefile
	- to do: 修改AM/NEMU的Makefile可以默认启动批处理模式的NEMU. 
	- sol:  一条线索是从main函数进去，engine_start()—>sdb_mainloop()—>sdb_set_batch_mode()，只要enable此函数，就可以实现批处理模式的nemu. init_monitor()–>parse_args()—>-b选项，打开sdb_set_batch_mode()—>从AM的makefile传到NEMU的makefile，查看AM的makefile，NEMUFLAGS添加-b(类比-l nemu-log.txt)。
	
- abstract-machine是如何生成native的可执行文件?
	- sol: 参照native.mk AM-Makefile, gcc将string.c编译成.o文件;  将am&klib源文件编译成.o, 通过ar打包成.a文件   (archive); linker将.o文件和.a文件链接成native的可执行文件(image)

- 为什么要有AM？AM和操作系统提供的运行时环境有什么不同？

  -  n个程序运行在m个架构上，m个架构各自维护halt()，程序调用halt()可以结束运行，只需维护n+m份代码

  - 运行时环境可以分为：架构相关AM and 架构无关klib/string.c klib/stdio.c
  - ISA软硬件接口，约定俗成的东西，编译器需要按照手册约定来生成代码，硬件需要按照手册约定来执行指令

- stdarg是如何实现的? 

  -  int **sprintf**(char *out, const char *fmt, ...)  参数数目是可变的，为了获得数目可变的参数，stdarg.h 包含一些函数调用参数的宏

- TRM运行时环境  链接脚本

	- ​		

- 我们在`am-kernels/tests/cpu-tests/tests/add.c`中定义了宏`NR_DATA`, 同时也在`add()`函数中定义了局部变量`c`和形参`a`, `b`, 但你会发现在符号表中找不到和它们对应的表项, 为什么会这样? 思考一下, 什么才算是一个符号(symbol)?

	-   local variable分配在栈帧

- string.c编译到native时默认链接glibc，在abstract-machine/klib/include/klib.h中定义__NATIVE_USE_KLIB__把native的库函数链接到klib，以此在真机上测试klib实现是否正确。想一下为什么? (link script)

   - 

- 在Linux下编写一个Hello World程序, 然后使用`strip`命令丢弃可执行文件中的符号表

	丢弃hello.o中的符号表时，对hello.o链接出现以下报错：

	/usr/bin/ld: error in hello.o(.eh_frame); no .eh_frame_hdr table will be created
	/usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/Scrt1.o: in function `_start':
	(.text+0x1b): undefined reference to main'`collect2: error: ld returned 1 exit status
	
- string.c，在riscv32-nemu上跑string，测试不通过说明 klib（软件）或 nemu（硬件）的实现有问题

   - klib: native上用gdb调试klib (AM-Makefile中CFLAGS添加-g)
   - nemu: string.c测试riscv32-nemu实现是否正确
   - klib-tests(写入、只读、输出)*
   	- native上glibc库函数测试编写的测试代码
   	- 用测试代码测试klib实现
   	- klib测试NEMU实现是否正确 



![pa-concept](https://ysyx.oscc.cc/docs/assets/pa-concept.06c3e97b.png)



# TRM

## R

## I

imm[11:0] can only hold values in range [-2^11^,2^11^-1]=[-2048,2047]

immediates is always sign-extended to 32bits before use in an arithmatic/logic operation.

LW LH LB LHU LBU

## S

SW SH SB

## B

 The 12-bit B-immediate encodes signed offsets in multiples of 2 bytes

 +-4KB = -2^12^ Byte = -2^10^instructions * 4Byte

- The immediate is left shifted by 1 before adding to the PC
- The LSB is 0

so , RISC-V conditonal branches can only reach 4KB each side of PC

## U

imm[31:12] upper 20 bits of 32-bit instrction word

LUI –> load upper immediate –> rd = imm<<12

AUIPC -> add upper immediate to pc –> rd = pc + imm<<12

## J

jal ra,Func_name

- rd = ra = s->pc + 4
- s->dnpc = s->pc + offset (+- 1MiB = +-2^20^ Byte = -2^18^ instrucitons *4Byte)
- specifically: j Label = jal x0,Label (Discard return address)

jalr rd,rs,immediate

- rd = ra = s->pc + 4
- s->dnpc = rs + immediate
- specifically: ret = jr ra = jalr x0,ra,0   s->dnpc=ra=s->pc+4

call function at any 32-bit absolute address –> lui x1,<hi20bits>  jalr ra,x1,<lo12bits>

Jump PC-relatice with 32-bit offset –> auipc x1,<hi20bits>  jalr x0,x1,<lo12bits>

## RV32IM

- cpu_exec() 

- execute() 模拟了cpu的工作方式，不断执行指令，执行n次for循环—>

	- exec_once()
	- trace_and_difftest()
	- 检查NEMU的状态是否为NEMU_RUNNING
	- device_update()

- exec_once() cpu执行当前pc指向的一条指令，并更新pc.  

	- s->pc=pc当前指令的pc, s->snpc=pc下一条静态指令的pc.

	- isa_exec_once() 随着取指的过程改变s->snpc()的值,s->snpc()为下一条指令的pc

- 取指(内存访问): 

	- 进入isa_exec_once()

	- inst_fetch(&s->snpc,len), 根据len更新s->snpc, 指向下一条指令 

	- ```text
		x   x+1  x+2  x+3
		+----+----+----+----+
		| 34 | 12 | 00 | 00 |
		+----+----+----+----+  0x1234  little-endian
		```

	- vaddr_ifetch()–>paddr_read()—>pmem_read()访问物理内存—>host_read()—>guest_to_host()

	- 回到isa_exec_once() 

- 译码: 

	- 进入decode_exec()   s->dnpc=s->snpc
	- INSTPAT_START()
	- INSTPAT(模式字符串01? , 指令名称, 指令类型RISBUJ, 指令执行操作)
	- INSTPA_END()

- 执行: 

	- C模拟指令执行的真正行为. auipc    R(rd) = s->pc + imm  s->dnpc修改
	- decode_exec()执行结束, 将返回0 —> isa_exec_once() —> exec_once()

- 更新pc:

	- 顺序执行: snpc==dnpc | 跳转指令: snpc=/=dnpc (snpc静态指令, dnpc动态指令)
	- cpu.pc = s->dnpc，dnpc维护cpu.pc





# IOE

- MMIO vs PIO

	paddr_read/write()判断addr落在物理空间还是设备空间

	前者：pmem_read/write()访问物理地址 | 后者：map_read()/map_write()映射到目标空间访问设备

	由于端口映射io不能满足cpu需要访问一段很大的连续存储空间（显存）的需求，内存映射io应运而生。对于64位机器，有48根物理地址线，可访问的存储空间有256TB那么大，从中划分出3MB给设备显存不痛不痒。将这段物理地址区间映射到设备，cpu读写物理地址区间就相当于读写设备中的数据。内存和设备在cpu看来都是一个字节编址的对象。

	- 普通指令，状态机按照TRM的模型进行状态转移

	- 输出操作，状态机正常更新pc

	- 输入操作，状态机的转移会分叉，转移到何种状态取决于执行这条指令时设备的状态

- make程序是如何得到错误码1的?

- 理解volatile关键字

	-   

- 理解mainargs

	-   

- serial.c —> printf—>alu-tests


- timer.c—>real-time clock tests
- keyboard.c—>


- vga.c—>







# CTE

- 冯诺伊曼机：计算机执行完一条指令，自动执行下一条指令
- 批处理系统（操作系统）：准备好后台程序，前台程序执行结束时会自动执行下一个前台程序
- 用户程序执行结束之后, 可以跳转到操作系统的代码继续执行

- 操作系统可以加载一个新的用户程序来执行，操作系统不能崩溃，需要保护起来，用户程序的执行流不能切入操作系统的函数，call/jal过于随意

RISC-V32硬件保护机制： M–>S(OS)–> U(应用程序)，硬件中加入一些与特权级检查相关的门电路，特权级低的模式尝试去执行一些系统级的操作，CPU会抛出异常信号

自陷制令ecall 操作系统预先设置好的跳转目标(mtvec–>异常入口地址)











# VME

- Multi-Programming

	-  内存中可以同时存在多个进程

	-  执行流在进程之间相互切换





# OS 

-  [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/)
-  [jyy](https://jyywiki.cn/OS/2023/index.html)
-  [6.s081](https://pdos.csail.mit.edu/6.S081/2021/schedule.html)
-  [CS162](https://cs162.org/)

