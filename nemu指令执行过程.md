# NEMU指令执行过程
一条指令被输入到nemu时，一般会经过以下三个阶段:
1. 匹配相应指令
2. 提取rd、rs、imm等
3. 执行对应操作，并更新pc



## 匹配阶段  
一条指令可被格式化为以下形式:
```c
??????? ????? ????? ??? ????? 00101 11
```  
问号部分为待填部分，0和1都可填入，这里我们需要关注的则是非问号部分，一般来讲也就是操作码所在的部分，而指令匹配也是依靠这部分进行实现的。  
其中在匹配阶段会生生成三个数字分别为*key, mask, shift*。key为非问号部分、mask为用于表达非问号部分的掩码被用来匹配输入指令的操作码部分，shift
的作用则是将输入指令(包括生成的key和mask)去掉最低位的任意匹配部分
，目前对于这样操作的推测是用于减少计算量。  
生成完*key, mask, shift*后会将输入指令通过与运算与掩码进行匹配取出操作码部分，再与key进行比较判断是否为同一个操作码，如果是则进行相应操作，否则进行下一个操作码的匹配，
在进行所有操作码的匹配后如果还未匹配到则为无效指令。

## 提取阶段
在匹配到操作码后，通过对应的指令类型(在*INSTPAT*宏函数中进行指定)
，代码中给出的则是I、U、S。不同的指令类型根据ISA手册的定义去取出rd、rs、imm，并将imm通过编译器进行有符号扩展再转为uint形式。

## 执行阶段
在提取完对应数字后根据操作码进行相应操作，相应操作的执行则是通过
*INSTPAT_MATCH*宏函数进行，如下列代码所示:
```c

#define INSTPAT_MATCH(s, name, type, ... /* execute body */ ) { \
  int rd = 0; \
  word_t src1 = 0, src2 = 0, imm = 0; \
  decode_operand(s, &rd, &src1, &src2, &imm, concat(TYPE_, type)); \
  __VA_ARGS__ ; \
}

// --- pattern matching wrappers for decode ---
#define INSTPAT(pattern, ...) do { \
  uint64_t key, mask, shift; \
  pattern_decode(pattern, STRLEN(pattern), &key, &mask, &shift); \
  if ((((uint64_t)INSTPAT_INST(s) >> shift) & mask) == key) { \
    INSTPAT_MATCH(s, ##__VA_ARGS__); \
    goto *(__instpat_end); \
  } \
} while (0)

INSTPAT_START();
INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc  , U, R(rd) = s->pc + imm);
...
...
INSTPAT_END();
```
如果查看源代码中完整的宏函数定义则会发现*INSTPAT_START*会生成一个__instpat_end的指向动态标签的指针变量，而*INSTPAT_END*会放置此动态标签;  
其中指令执行的部分会通过*INSTPAT_MATCH*中的__VA_ARGS__进行展开，从而执行相应操作。
如果匹配到相应操作码并执行成功，则直接跳到末尾*INSTPAT_END*放置的动态标签，并重置R(0)寄存器。

另外，更新pc则不在此过程中进行，而是通过*inst_fetch*函数更新。

