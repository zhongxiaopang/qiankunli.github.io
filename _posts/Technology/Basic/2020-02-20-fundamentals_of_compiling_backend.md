---

layout: post
title: 《编译原理之美》笔记——后端部分
category: 技术
tags: Basic
keywords:  Fundamentals Compiling

---

## 简介（持续更新）

* TOC
{:toc}

![](/public/upload/basic/language_compile.jpg)

我们使用的语言抽象程度越来越高，每一次抽象对下一层的复杂性做了屏蔽，因此使用起来越来越友好。而编译技术，则帮你一层层地还原这个抽象过程，重新转换成复杂的底层实现。

## 程序运行机制

### 程序和操作系统的关系

程序视角的堆栈结构 

![](/public/upload/basic/program_memory_view.jpg)

操作系统加载可执行文件到内存的，并且定位到代码区里程序的入口开始执行。操作系统看到的是一个个指令 和 一个个内存地址

![](/public/upload/basic/os_memory_view.jpg)

为什么会有一块动态内存区域？os 启动的时候， 一波三折， 不停的往内存加载更大的程序， 第一波占用的内存第二波就干别的用了。c 语言手动malloc 和 free，其实微观上，**汇编语言也是在字节层面上不停地malloc和free**。

os 提供systemcall malloc 内存，但没有提供 systemcall  malloc 一个heap 或 stack 出来。


### 汇编语言基础

计算机的处理器有很多不同的架构，比如 x86-64、ARM、Power 等，每种处理器的指令集都不相同，那也就意味着汇编语言不同。本文以x86-64 架构为例


    #include <stdio.h>
    int main(int argc, char* argv[]){
        printf("Hello %s!\n", "Richard");
        return 0;
    }

对应的汇编代码

    .section    __TEXT,__text,regular,pure_instructions
        .build_version macos, 10, 14    sdk_version 10, 14
        .globl  _main                   ## -- Begin function main
        .p2align    4, 0x90
    _main:                                  ## @main
        .cfi_startproc
    ## %bb.0:
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register %rbp
        leaq    L_.str(%rip), %rdi
        leaq    L_.str.1(%rip), %rsi
        xorl    %eax, %eax
        callq   _printf
        xorl    %eax, %eax
        popq    %rbp
        retq
        .cfi_endproc
                                            ## -- End function 
        .section    __TEXT,__cstring,cstring_literals
    L_.str:                                 ## @.str
        .asciz  "Hello %s!\n"

    L_.str.1:                               ## @.str.1
        .asciz  "Richard"

    .subsections_via_symbols

这段代码里有指令、伪指令、标签和注释四种元素，每个元素单独占一行。
1. 伪指令以“.”开头，末尾没有冒号“：”。伪指令是是辅助性的，不是真正的 CPU 指令，就是写给汇编器的，汇编器在生成目标文件时会用到这些信息。
2. 标签以冒号“:”结尾，用于对伪指令生成的数据或指令做标记。可以代表一段代码或者常量的地址，其他代码可以访问标签。可一开始，我们没法知道这个地址的具体值，**必须生成目标文件后，才能算出来**。所以，标签会简化汇编代码的编写。
3. 注释，以“#”号开头，这跟 C 语言中以 // 表示注释语句是一样的。

在代码中，助记符“movq”“xorl”中的“mov”和“xor”是指令，而“q”和“l”叫做后缀，表示操作数的位数。后缀一共有 b, w, l, q 四种，分别代表 8 位、16 位、32 位和 64 位。在指令中使用操作数，可以使用四种格式，它们分别是：立即数、寄存器、直接内存访问和间接内存访问。

|操作数格式|示例|含义|**指令除立即数外都是基于地址的**|
|---|---|---|---|
|立即数以$开头|`movl $40, %eax`|把 40 这个数字拷贝到 %eax 寄存器|
|直接内存访问 |`callq _printf`|调用printf函数|操作数是内存地址<br>_printf是一个函数入口的地址，<br>汇编器帮我们计算出程序装载在内存时，每个字面量和过程的地址|
|寄存器|`subq $8, %rsp`|把%rsp的值减8|**寄存器本身既是存储，也是存储地址**|
|间接内存访问带有括号|`movl $10, (%rsp)`|把`%rsp` 寄存器的值所指向的地址 对应的内存设为10|寄存器的值是内存地址|

### 过程调用和栈帧

    /*function-call1.c */
    #include <stdio.h>
    int fun1(int a, int b){
        int c = 10;
        return a+b+c;
    }
    int main(int argc, char *argv[]){
        printf("fun1: %d\n", fun1(1,2));
        return 0;
    } 

等价的汇编代码如下

    # function-call1-craft.s 函数调用和参数传递
        # 文本段,纯代码
        .section    __TEXT,__text,regular,pure_instructions

    _fun1:
        # 函数调用的序曲,设置栈指针
        pushq   %rbp           # 把调用者的栈帧底部地址保存起来   
        movq    %rsp, %rbp     # 把调用者的栈帧顶部地址,设置为本栈帧的底部

        subq    $4, %rsp       # 扩展栈

        movl    $10, -4(%rbp)  # 变量c赋值为10，也可以写成 movl $10, (%rsp)

        # 做加法
        movl    %edi, %eax     # 第一个参数放进%eax
        addl    %esi, %eax     # 把第二个参数加到%eax,%eax同时也是存放返回值的寄存器
        addl    -4(%rbp), %eax # 加上c的值

        addq    $4, %rsp       # 缩小栈

        # 函数调用的尾声,恢复栈指针为原来的值
        popq    %rbp           # 恢复调用者栈帧的底部数值
        retq                   # 返回

        .globl  _main          # .global伪指令让_main函数外部可见
    _main:                                  ## @main
        
        # 函数调用的序曲,设置栈指针
        pushq   %rbp           # 把调用者的栈帧底部地址保存起来  
        movq    %rsp, %rbp     # 把调用者的栈帧顶部地址,设置为本栈帧的底部
        
        # 设置第一个和第二个参数,分别为1和2
        movl    $1, %edi
        movl    $2, %esi

        callq   _fun1                # 调用函数

        # 为pritf设置参数
        leaq    L_.str(%rip), %rdi   # 第一个参数是字符串的地址
        movl    %eax, %esi           # 第二个参数是前一个参数的返回值

        callq   _printf              # 调用函数

        # 设置返回值。这句也常用 xorl %esi, %esi 这样的指令,都是置为零
        movl    $0, %eax
        
        # 函数调用的尾声,恢复栈指针为原来的值
        popq    %rbp         # 恢复调用者栈帧的底部数值
        retq                 # 返回

        # 文本段,保存字符串字面量                                  
        .section    __TEXT,__cstring,cstring_literals
    L_.str:                                 ## @.str
        .asciz  "Hello World! :%d \n"


C函数翻译成汇编代码，有这样的固定结构。就好像java的 synchronized 关键字 自动有entermonitor 和 exitmonitor 一样。 

    # 函数调用的序曲,设置栈指针
    pushq  %rbp        # 把调用者的栈帧底部地址保存起来  
    movq  %rsp, %rbp   # 把调用者的栈帧顶部地址，设置为本栈帧的底部

    ...

    # 函数调用的尾声,恢复栈指针为原来的值
    popq  %rbp         # 恢复调用者栈帧的底部数值

|函数调用涉及指令|等价指令|备注|
|---|---|---|
|`callq _fun1`|`pushq %rip`<br>`jmp _fun1`|保存下一条指令的地址，用于函数返回继续执行<br>跳转到函数_fun1|
|`pushq %rbp`|`subq $8, %rsp`<br>`movq %rbp, (%rsp)`|把%rsp的值减8，也就是栈增长8个字节，**从高地址向低地址增长**<br>把%rbp的值写到当前栈顶指示的内存位置|
|`popq %rbp`|`movq (%rsp), %rbp`<br>`addq $8, %rsp`|把栈顶位置的值恢复回%rbp，这是之前保存在栈里的值<br>把%rsp的值加8，也就是栈减少8个字节|
|`retq`|`popq %rip`<br>`jmp %rip`|恢复指令指针寄存器|

`pushq %rbp` 执行后情况

![](/public/upload/basic/pushq_rbp.jpg)

1. pushq 和 popq 虽然是单“参数”指令，但一个隐藏的“参数”就是 %rsp。
2. 通过移动 %rsp 指针来改变帧的大小。%rbp 和 %rsp 之间的空间就是当前栈帧。
3. 栈帧先进后出 （一个函数的相关 信息占用一帧）。或者栈帧二字 重点在帧上。%rbp 在函数调用时一次移动 一个栈帧的大小，**%rbp在整个函数执行期间是不变的**。使用 %rbp 前，要先保护起来，在栈帧内存一个备份。
4. 函数内部访问 栈帧 可以使用 `-4(%rbp)`表示地址，表示%rbp 寄存器存储的地址减去4的地址。说白了，**栈帧内可以基于 (%rbp) 随机访问**，`+4(%rsp)`效果类似。
5. **%rsp并不总是指向真实的栈顶**：在 X86-64 架构下，新的规范让程序可以访问栈顶之外 128 字节的内存，所以，我们甚至不需要通过改变 %rsp 来分配栈空间，而是直接用栈顶之外的空间。比如栈帧大小是16，即·`(%rbp)-(%rsp) = 16`，可以在代码中直接使用 内存地址`-32(%rbp)`。但如果函数内 还会调用 其它函数，为了pushq/popq 指令的正确性，编译器会为%rsp 设置正确的值使其 指向栈顶。
6. 除了callq/pushq/popq/retq  指令操作%rsp外，函数执行期间，可以mov (%rsp)使其指向栈顶一步到位，(%rsp)也可以和(%rbp)挨着一步不动，也可以随着变量的分配慢慢移动。
7. 函数调用 使用栈空间的大小 是编译时就可以确定的，但不是说编译时就分配好栈空间了，栈空间运行时动态分配与回收。这与全局变量、常量 编译时就确定好内存分配是不同的。

循环语句和 if 语句的秘密在于比较指令和**有条件跳转指令**（jmp是无条件跳转指令），它们都用到了 EFLAGS 寄存器。

## 生成代码

翻译 AST 自动生成这些汇编代码

### 遍历 AST 手工生成汇编代码

AST 的每个节点有各种属性，遍历节点时，针对节点属性类型做相应处理。**生成汇编代码的过程，基本上就是基于 AST 拼接字符串**

    case PlayScriptParser.ADD:
        // 为加法运算申请一个临时的存储位置，可以是寄存器和栈
        address = allocForExpression(ctx);
        bodyAsm.append("\tmovl\t").append(left).append(", ").append(address).append("\n");  //把左边节点拷贝到存储空间
        bodyAsm.append("\taddl\t").append(right).append(", ").append(address).append("\n");  //把右边节点加上去
        break;

翻译程序的入口generate

1. 生成一个.section 伪指令，表明这是一个放文本的代码段。
2. 遍历 AST 中的所有函数，调用 generateProcedure() 方法为每个函数生成一段汇编代码，再接着生成一个主程序的入口。
3. 在一个新的 section 中，声明一些全局的常量（字面量）

    public String generate() {
        StringBuffer sb = new StringBuffer();

        // 1.代码段的头
        sb.append("\t.section  __TEXT,__text,regular,pure_instructions\n");

        // 2.生成函数的代码
        for (Type type : at.types) {
            if (type instanceof Function) {
                Function function = (Function) type;
                FunctionDeclarationContext fdc = (FunctionDeclarationContext) function.ctx;
                visitFunctionDeclaration(fdc); // 遍历，代码生成到bodyAsm中了
                generateProcedure(function.name, sb);
            }
        }

        // 3.对主程序生成_main函数
        visitProg((ProgContext) at.ast);
        generateProcedure("main", sb);

        // 4.文本字面量
        sb.append("\n# 字符串字面量\n");
        sb.append("\t.section  __TEXT,__cstring,cstring_literals\n");
        for(int i = 0; i< stringLiterals.size(); i++){
            sb.append("L.str." + i + ":\n");
            sb.append("\t.asciz\t\"").append(stringLiterals.get(i)).append("\"\n");
        }

        // 5.重置全局的一些临时变量
        stringLiterals.clear();
        
        return sb.toString();
    }

generateProcedure() 方法把函数转换成汇编代码

1. 生成函数标签、序曲部分的代码、设置栈顶指针、保护寄存器原有的值等。
2. 接着是函数体，比如本地变量初始化、做加法运算等。
3. 最后是一系列收尾工作，包括恢复被保护的寄存器的值、恢复栈顶指针，以及尾声部分的代码。

    private void generateProcedure(String name, StringBuffer sb) {
        // 1.函数标签
        sb.append("\n## 过程:").append(name).append("\n");
        sb.append("\t.globl _").append(name).append("\n");
        sb.append("_").append(name).append(":\n");

        // 2.序曲
        sb.append("\n\t# 序曲\n");
        sb.append("\tpushq\t%rbp\n");
        sb.append("\tmovq\t%rsp, %rbp\n");

        // 3.设置栈顶
        // 16字节对齐
        if ((rspOffset % 16) != 0) {
            rspOffset = (rspOffset / 16 + 1) * 16;
        }
        sb.append("\n\t# 设置栈顶\n");
        sb.append("\tsubq\t$").append(rspOffset).append(", %rsp\n");

        // 4.保存用到的寄存器的值
        saveRegisters();

        // 5.函数体
        sb.append("\n\t# 过程体\n");
        sb.append(bodyAsm);

        // 6.恢复受保护的寄存器的值
        restoreRegisters();

        // 7.恢复栈顶
        sb.append("\n\t# 恢复栈顶\n");
        sb.append("\taddq\t$").append(rspOffset).append(", %rsp\n");

        // 8.如果是main函数，设置返回值为0
        if (name.equals("main")) {
            sb.append("\n\t# 返回值\n");
            sb.append("\txorl\t%eax, %eax\n");
        }

        // 9.尾声
        sb.append("\n\t# 尾声\n");
        sb.append("\tpopq\t%rbp\n");
        sb.append("\tretq\n");

        // 10.重置临时变量
        rspOffset = 0;
        localVars.clear();
        tempVars.clear();
        bodyAsm = new StringBuffer();
    }

### 二进制文件格式和链接

汇编器可以把每一个汇编文件都编译生成一个二进制的目标文件，或者叫做一个模块。在一个文件中调用另一个文件的函数时，并不知道函数的地址。所以，汇编器把这个任务推迟，交给链接器去解决。就好比你去饭店排队吃饭，首先要拿个号（函数的标签），但不知道具体坐哪桌。等叫到你的号的时候（链接过程），服务员才会给你安排一个确定的桌子（函数的地址）。

在 Linux 下，目标文件、共享对象文件、二进制文件，都是采用 ELF 格式。这些二进制文件的格式跟加载到内存中的程序的格式是很相似的。这样有什么好处呢？它可以迅速被操作系统读取，并加载到内存中去，加载速度越快，也就相当于程序的启动速度越快。

在 ELF 格式中，代码和数据也是分开的。这样做的好处是，程序的代码部分，可以在多个进程中共享，不需要在内存里放多份。放一份，然后映射到每个进程的代码区就行了。而数据部分，则是每个进程都不一样的，所以要为每个进程加载一份。





### IR中间表达式
前文通过 从 AST 生成汇编代码 的过程是比较机械的。


IR 在高级语言和汇编语言的中间，与高级语言相比，IR 丢弃了大部分高级语言的语法特征和语义特征，比如循环语句、if 语句、作用域、面向对象等等，它更像高层次的汇编语言；而相比真正的汇编语言，它又不会有那么多琐碎的、与具体硬件相关的细节。


IR看做是一种高层次的汇编语言，主要体现在：
1. 它可以使用寄存器，但**寄存器的数量没有限制**；
2. 控制结构也跟汇编语言比较像，比如有跳转语句，分成多个程序块，用标签来标识程序块等；
3. 使用相当于汇编指令的操作码。这些操作码可以一对一地翻译成汇编代码，但有时一个操作码会对应多个汇编指令。

IR 格式

1. 三地址代码，学习用途
2. LLVM 的 IR，工业级

    1. 静态单赋值（SSA），也就是每个变量（地址）最多被赋值一次，它这种特性有利于运行代码优化算法；
    2. 有更多的细节信息。比如整型变量的字长、内存对齐方式等等
    3. LLVM 汇编则带有一个类型系统。它能避免不安全的数据操作，并且有助于优化算法。
    4. 在 LLVM 汇编中可以声明全局变量、常量

![](/public/upload/basic/llvm_overview.png)

    int fun1(int a, int b){
        int c = 10;
        return a + b + c;
    }

前端工具 Clang生成 LLVM 的汇编码`clang -emit-llvm -S fun1.c -o fun1.ll`

LLVM优化后的编码`clang -emit-llvm -S -O2 fun1.c -o fun1.ll`

    define i32 @fun1(i32, i32) local_unnamed_addr #0 {
        %3 = add i32 %0, 10
        %4 = add i32 %3, %1
        ret i32 %4
    }

### LLVM IR对象模型

LLVM提供了代表 LLVM IR 的一组对象模型，我们只要生成这些对象，就相当于生成了 IR，比通过字符串拼接来生成 LLVM 的 IR方便。

    int fun1(int a, int b){
        return a+b;
    }

直接从 fun1() 函数生成 IR

    Module *mod = new Module("fun1.ll", TheModule);
    //函数原型
    vector<Type *> argTypes(2, Type::getInt32Ty(TheContext));
    FunctionType *fun1Type = FunctionType::get(Type::getInt32Ty(TheContext), //返回值是整数
        argTypes, //两个整型参数
        false);   //不是变长参数
    //函数对象
    Function *fun = Function::Create(fun1Type, 
        Function::ExternalLinkage,   //链接类型
        "fun1",                      //函数名称
        TheModule.get());            //所在模块
    //设置参数名称
    string argNames[2] = {"a", "b"};
    unsigned i = 0;
    for (auto &arg : fun->args()){
        arg.setName(argNames[i++]);
    }
    //创建一个基本块
    BasicBlock *BB = BasicBlock::Create(TheContext,//上下文
                "",     //基本块名称
                fun);  //所在函数
    Builder.SetInsertPoint(BB);   //设置指令的插入点
    //把参数变量存到NamedValues里面备用
    NamedValues.clear();
    for (auto &Arg : fun->args())
        NamedValues[Arg.getName()] = &Arg;
    //做加法
    Value *L = NamedValues["a"];
    Value *R = NamedValues["b"];
    Value *addtmp = Builder.CreateAdd(L, R);
    //返回值
    Builder.CreateRet(addtmp);
    //验证函数的正确性
    verifyFunction(*fun);
    Function *fun1 = codegen_fun1();     //在模块中生成Function对象
    TheModule->print(errs(), nullptr);   //在终端输出IR

## 优化代码

1. 代码优化的目标，是优化程序对计算机资源的使用。我们平常最关心的就是 CPU 资源，有时候还会有其他目标，比如代码大小、内存占用大小、磁盘访问次数、网络通讯次数等等。
2. 从代码优化的对象看，**大多数的代码优化都是在 IR 上做的**，而不是在前一阶段的 AST 和后一阶段汇编代码上进行的

    1. 在 AST 上也能做一些优化，但它抽象层次太高，含有的硬件架构信息太少，难以执行很多优化算法。 
    2. 在汇编代码上进行优化会让算法跟机器相关，当换一个目标机器的时候，还要重新编写优化代码。

3. 优化的范围

    1. 本地优化，针对代码基本块的优化
    2. 全局优化，超越基本块的范围进行分析，我们需要用到控制流图CFG ，是一种有向图，它体现了基本块之前的指令流转关系。一个函数（或过程）里如果包含多个基本块，可以表达为一个 CFG。这种针对一个函数、基于 CFG 的优化，叫做全局优化
    3. 过程间优化，跨越函数的边界，对多个函数之间的关系进行优化

### 独立于机器的优化

一些优化的场景

1. 代数优化,   `x:=x*8` ==> `x:=x<<3`
2. 常数折叠,   `x:=20*3` ==> `x:=60`
3. 删除不可达的基本块,   比如 `if 2<0`
4. 删除公共子表达式,  `x:=a+b;y:=a+b` ==> `x:=a+b;y:=x` 减少一次`a+b`计算
5. 拷贝传播和常数传播
6. 死代码删除

在做全局优化时，情况就要复杂一些：代码不是在一个基本块里简单地顺序执行，而可能经过控制流图（CFG）中的多条路径。如果没有循环，属于有向无环图。否则，就是一个有向有环图。我们对其进行数据流分析，便有机会去除其中的无效变量，甚至直接删掉某个分支的代码。PS：这便是让数学中的图、“半格”理论有了用武之地。

![](/public/upload/basic/directed_cyclic_graph.jpg)

### 依赖于机器的优化

代码生成部分 主要考虑三点：

1. 指令的选择。同样一个功能，可以用不同的指令或指令序列来完成，而我们需要选择比较优化的方案：生成更简短的代码；从多种可能的指令中选择最优的。==> 树覆盖算法 
2. 寄存器分配。每款 CPU 的寄存器都是有限的，我们要有效地利用它。==> 寄存器共享原则 ==> 图染色算法
3. 指令重排序。计算执行的次序会影响所生成的代码的效率。在不影响运行结果的情况下，我们要通过代码重排序获得更高的效率。 算法排序的关键点，是要找出代码之间的数据依赖关系 ==> 数据的依赖图，给图中的每个节点再加上两个属性：一是操作类型，因为这涉及它所需要的功能单元；二是时延属性，也就是每条指令所需的时钟周期。==> List Scheduling算法

我们通常会把 CPU 看做一个整体，把 CPU 执行指令的过程想象成，依此检票进站的过程，改变不同乘客的次序，并不会加快检票的速度。所以，我们会自然而然地认为改变顺序并不会改变总时间。但当我们进入 CPU 内部，会看到 CPU 是由多个功能部件构成的。**一条指令执行时要依次用到多个功能部件，分成多个阶段**，虽然每条指令是顺序执行的，但每个部件的工作完成以后，就可以服务于下一条指令。**在执行指令的阶段，不同的指令也会由不同的单元负责**。所以在同一时刻，不同的功能单元其实可以服务于不同的指令。

当然了，不是所有的指令都可以并行，导致无法并行的原因有几个：

1. 数据依赖约束，如果后一条指令要用到前一条指令的结果，那必须排在它后面。如果有因为使用同一个存储位置，而导致不能并行的，可以用重命名变量的方式消除，这类约束被叫做伪约束，比如

    1. 先写再写：如果指令 A 写一个寄存器或内存位置，B 也写同一个位置，就不能改变 A 和 B 的执行顺序，不过我们可以修改程序，让 A 和 B 写不同的位置。
    2. 先读后写：如果 A 必须在 B 写某个位置之前读某个位置，那么不能改变 A 和 B 的执行顺序。除非能够通过重命名让它们使用不同的位置。
2. 资源约束
    1. 功能部件约束，如果只有一个乘法计算器，那么一次只能执行一条乘法运算。
    2. 指令流出约束，指令流出部件一次流出 n 条指令。
    3. 寄存器约束，寄存器数量有限，指令并行时使用的寄存器不可以超标。

在我国大力促进芯片研发的背景下，如果现在有一个新的 CPU 架构，新芯片需要编译器的支持才可以。你要实现各种指令选择的算法、寄存器分配的算法、指令排序的算法来反映这款 CPU 的特点。对于这个难度颇高的任务，LLVM 的 TableGen 模块会给你提供很大的帮助。这个模块能够帮助你为某个新的 CPU 架构快速生成后端。你可以用一系列配置文件定义你的 CPU 架构的特征，比如寄存器的数量、指令集等等。一旦你定义了这些信息，TableGen 模块就能根据这些配置信息，生成各种算法，如指令选择器、指令排序器、一些优化算法等等。这就像编译器前段工具可以帮你生成词法分析器，和语法分析器一样，能够大大降低开发一个新后端的工作量，所以说，把 LLVM 研究透彻，会有助于你在这样的重大项目中发挥重要作用。

把每次只处理一个数据的指令，叫做 SISD（Single Instruction Single Data），一次只处理一个数据的计算，叫做标量计算。对应的有一类叫做 SIMD（Single Instruction Multiple Data）的指令，一次可以同时处理多个数据的计算，叫做矢量计算。它在一个寄存器里可以并排摆下 4 个、8 个甚至更多标量，构成一个矢量。比如寄存器是 256 位的，可以支持同时做 4 个 64 位数的计算。平常写的程序，编译器也会优化成，尽量使用 SIMD 指令来提高性能。

在代码里，我们会用到寄存器，并且还会用专门的寄存器分配的算法来优化寄存器。可是对于高速缓存，我们没有办法直接控制。那我们有什么办法呢？答案是提高程序的局部性（locality），这个局部性又分为两个：

1. 时间局部性。一个数据一旦被加载到高速缓存甚至寄存器，我们后序的代码都能集中访问这个数据，别等着这个数据失效了再访问，那就又需要从低级别的存储中加载一次。
2. 空间局部性。当我们访问了一条数据之后，很可能马上访问跟这个数据挨着的其他数据。CPU 在一次读入数据的时候，会把相邻的数据都加载到高速缓存，这样会增加后面代码在高速缓存中命中的概率。

**提高局部性这件事情，更多的是程序员的责任**，编译器能做的事情不多。不过，有一种编译优化技术，叫做循环互换优化（loop interchange optimization）可以让程序更好地利用高速缓存和寄存器。

## 未来发展

从某种意义上看，从计算机诞生到现在，我们编写程序的方式一直没有太大的进步。最早，是通过在纸带或卡片上打孔，来写程序；后来产生了汇编语言和高级语言。但是，写程序的本质没有变化，我们只是在用更高级的方式打孔。

### 云计算改变编程模式

当一个软件服务 1 万个用户的时候，增加一个功能可能需要 100 人天的话；针对服务于 1 百万用户的系统，增加同样的功能，可能需要几千到上万人天。同样的，如果功能不变，只是用户规模增加，你同样要花费很多人天来修改系统。那么你可以看出，整体的复杂性是多个因素相乘的结果，而不是简单相加。**云计算最早承诺，当我们需要更多计算资源的时候，简单增加一下就行了。然而，现有软件的架构，其实离这个目标还很远**。那有没有可能把这些复杂性解耦，使得复杂性的增长变成线性或多项式级别（这里是借助算法复杂性的理论）的呢？

1. 基础设施的复杂性，队列、分库分表、负载均衡、服务注册发现、监控等
2. 部署复杂性，代码、分支、编译、测试、配置、灰度发布、回滚等
3. API 的复杂性，能否让 API 调用跟普通语言的函数调用一样简单

编程环境会逐渐跟云平台结合起来，让 API 调用跟普通语言的函数调用一样简单，让开发平台来处理上述复杂性。根据计算机语言的发展规律，我们总是会想办法建立更高的抽象层次，把复杂性隐藏在下层。**就像高级语言隐藏了寄存器和内存管理的复杂性一样**。要求新的编程语言从更高的一个抽象层次上，做编译、转换和优化。我们只需要编写业务逻辑就可以了，当应用规模扩大时，真的只需要增加计算资源就行了；当应用需求变化时，也只需要修改业务逻辑，而不会引起技术细节上的很多工作量。**能解决这些问题的软件，就是云原生的编程语言及其基础设施**。


还可能实现编程环境本身的云化，电脑只负责代码编辑的工作，代码本身放在云上，编译过程以及所需的类库也放在云上。




## 其它

||线程上下文|函数上下文|
|---|---|
|组成|虚拟地址空间<br>寄存器<br>cpu上下文|函数栈帧|
|指令寄存器|%pc|%rip|
|栈寄存器|用户栈在用户切换内存空间时切换<br>内核栈current_task 指向当前的 task_struct<br>用户栈顶指针在内核栈顶部的 pt_regs 结构<br>内核栈顶指针%rsp|%rbp指向栈帧的底部<br>%rsp指向栈帧的顶部|
|调度方/切换方|操作系统|程序自己/编译器|

编译原理要把计算机组成、数据结构、算法、操作系统，以及离散数学中的某些知识都用上。

1. 指令选择、寄存器分配和指令排序，都是NP Complete的。这个时候，如果提前已经知道什么是NP Complete，那么马上就对这类算法有概念，也马上想到可以用什么样的方式来对待这类问题。
2. 编译原理中很多算法都是基于树和图的，所以对这两个数据结构的理解也会有帮助。
3. 至于计算机组成原理，跟后端技术的关联很密切。
4. 程序运行时的环境，内存管理等内容，则跟操作系统有关。