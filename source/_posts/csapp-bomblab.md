---
title: CSAPP：BombLab
date: 2023/12/4 20:50:54
---
通读main函数，可以轻易发现每个阶段是调用`phase_n`函数进行。 
## Phase 1 
```x86asm
public phase_1
phase_1 proc near             ; CODE XREF: main+9A↑p
; __unwind {
sub     rsp, 8                ; Integer Subtraction
mov     esi, offset aBorderRelation ; "Border relations with Canada have never"...
call    strings_not_equal     ; Call Procedure
test    eax, eax              ; Logical Compare
jz      short loc_400EF7      ; Jump if Zero (ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_400EF7:                   ; CODE XREF: phase_1+10↑j
add     rsp, 8                ; Add
retn                          ; Return Near from Procedure
; } // starts at 400EE0
phase_1 endp
```
`main`函数调用`phase_1`时从`read_line`函数读取了一行输入，将`char *`存入了`rdi`中。`strings_not_equal`的第一个参数也是通过`rdi`传递，所以`phase_1`没有修改`rdi`。  
然后`phase_1`从`rodata`段，也就是系统`mmap`程序到内存的位置，读取字符串，将指针放入`esi`中，作为`strings_not_equal`的第二个参数。  
`strings_not_equal`函数负责比较字符串，具体实现省略，主要覆盖了一下几种情况的比较

1. 首先判断字符串是否等长
2. 然后对比每个字符串，直到字符串结束

`strings_not_equal`在字符串不相等的情况下返回1，相等的情况下返回0。  
`test`指令对其进行判断，如果是零，则跳转到返回块，如果不是，炸弹爆炸。  

## Phase 2

```x86asm
; __int64 __fastcall phase_2(__int64)
public phase_2
phase_2 proc near             ; CODE XREF: main+B6↑p
var_38= dword ptr -38h
var_34= byte ptr -34h
var_20= byte ptr -20h
; __unwind {
push    rbp
push    rbx
sub     rsp, 28h              ; Integer Subtraction
mov     rsi, rsp
call    read_six_numbers      ; Call Procedure
cmp     [rsp+38h+var_38], 1   ; Compare Two Operands
jz      short loc_400F30      ; Jump if Zero (ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
jmp     short loc_400F30      ; Jump
; ---------------------------------------------------------------------------
loc_400F17:                   ; CODE XREF: phase_2+30↓j
                              ; phase_2+3E↓j
mov     eax, [rbx-4]
add     eax, eax              ; Add
cmp     [rbx], eax            ; Compare Two Operands
jz      short loc_400F25      ; Jump if Zero (ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_400F25:                   ; CODE XREF: phase_2+22↑j
add     rbx, 4                ; Add
cmp     rbx, rbp              ; Compare Two Operands
jnz     short loc_400F17      ; Jump if Not Zero (ZF=0)
jmp     short loc_400F3C      ; Jump
; ---------------------------------------------------------------------------
loc_400F30:                   ; CODE XREF: phase_2+12↑j
                              ; phase_2+19↑j
lea     rbx, [rsp+38h+var_34] ; Load Effective Address
lea     rbp, [rsp+38h+var_20] ; Load Effective Address
jmp     short loc_400F17      ; Jump
; ---------------------------------------------------------------------------
loc_400F3C:                   ; CODE XREF: phase_2+32↑j
add     rsp, 28h              ; Add
pop     rbx
pop     rbp
retn                          ; Return Near from Procedure
; } // starts at 400EFC
phase_2 endp
```
首先调用了`read_six_numbers`函数，获得六个数字。如果第一个数字不是1，则引爆炸弹。  
如果是1，跳转到`0x400F30`的代码块。  
实际上单独的那一个jmp指令，永远不是执行，因为`explode_bomb`会使进程退出。但是编译器无法判断，所以会添加上一个兜底的跳转。代码结构如下所示（仅为概念演示）
```c
if (first_number != 1)
    explode_bomb()
loc_400F30();
```
通过`lea`获取了栈上的空间，然后跳转到`0x400F17`的代码块。  
`rbx`指向当前遍历到的数字（初始是第二个），把上一个的结果复制到`eax`中，`eax`加上自身，也就是乘以2，和当前的数字比较，不同则引爆炸弹。  
循环块不深入解释

## Phase 3
```x86asm
jpt_400F75                    ; jump table for switch statement
                              ; DATA XREF: phase_3+32↑r
dq offset loc_400F7C
dq offset loc_400FB9
dq offset loc_400F83
dq offset loc_400F8A
dq offset loc_400F91
dq offset loc_400F98
dq offset loc_400F9F
dq offset loc_400FA6
; __int64 __fastcall phase_3(__int64)
public phase_3
phase_3 proc near             ; CODE XREF: main+D2↑p
var_10= dword ptr -10h
var_C= dword ptr -0Ch
; __unwind {
sub     rsp, 18h              ; Integer Subtraction
lea     rcx, [rsp+18h+var_C]  ; Load Effective Address
lea     rdx, [rsp+18h+var_10] ; Load Effective Address
mov     esi, offset aDD       ; "%d %d"
mov     eax, 0
call    ___isoc99_sscanf      ; Call Procedure
cmp     eax, 1                ; Compare Two Operands
jg      short loc_400F6A      ; Jump if Greater (ZF=0 & SF=OF)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_400F6A:                   ; CODE XREF: phase_3+20↑j
cmp     [rsp+18h+var_10], 7   ; switch 8 cases
ja      short def_400F75      ; jumptable 0000000000400F75 default case
mov     eax, [rsp+18h+var_10]
jmp     ds:jpt_400F75[rax*8]  ; switch jump
; ---------------------------------------------------------------------------
loc_400F7C:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 0CFh             ; jumptable 0000000000400F75 case 0
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
loc_400F83:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 2C3h             ; jumptable 0000000000400F75 case 2
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
loc_400F8A:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 100h             ; jumptable 0000000000400F75 case 3
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
loc_400F91:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 185h             ; jumptable 0000000000400F75 case 4
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
loc_400F98:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 0CEh             ; jumptable 0000000000400F75 case 5
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
loc_400F9F:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 2AAh             ; jumptable 0000000000400F75 case 6
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
loc_400FA6:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 147h             ; jumptable 0000000000400F75 case 7
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
def_400F75:                   ; CODE XREF: phase_3+2C↑j
call    explode_bomb          ; jumptable 0000000000400F75 default case
; ---------------------------------------------------------------------------
mov     eax, 0
jmp     short loc_400FBE      ; Jump
; ---------------------------------------------------------------------------
loc_400FB9:                   ; CODE XREF: phase_3+32↑j
                              ; DATA XREF: .rodata:jpt_400F75↓o
mov     eax, 137h             ; jumptable 0000000000400F75 case 1
loc_400FBE:                   ; CODE XREF: phase_3+3E↑j
                              ; phase_3+45↑j
                              ; phase_3+4C↑j
                              ; phase_3+53↑j
                              ; phase_3+5A↑j
                              ; phase_3+61↑j
                              ; phase_3+68↑j
                              ; phase_3+74↑j
cmp     eax, [rsp+18h+var_C]  ; Compare Two Operands
jz      short loc_400FC9      ; Jump if Zero (ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_400FC9:                   ; CODE XREF: phase_3+7F↑j
add     rsp, 18h              ; Add
retn                          ; Return Near from Procedure
; } // starts at 400F43
phase_3 endp
```
p3是一个经典的`switch`结构，`sscanf`复制读取两个数字，第一个数字作为`jtable`的偏移，第二个数字在`case`中比较，由于有8个`case`，所以有8个答案  
## Phase 4
```x86asm
; __int64 __fastcall func4(__int64, __int64, int)
public func4
func4 proc near               ; CODE XREF: func4+1B↓p
                              ; func4+30↓p
                              ; phase_4+3C↓p
; __unwind {
sub     rsp, 8                ; Integer Subtraction
mov     eax, edx
sub     eax, esi              ; Integer Subtraction
mov     ecx, eax
shr     ecx, 1Fh              ; Shift Logical Right
add     eax, ecx              ; Add
sar     eax, 1                ; Shift Arithmetic Right
lea     ecx, [rax+rsi]        ; Load Effective Address
cmp     ecx, edi              ; Compare Two Operands
jle     short loc_400FF2      ; Jump if Less or Equal (ZF=1 | SF!=OF)
lea     edx, [rcx-1]          ; Load Effective Address
call    func4                 ; Call Procedure
add     eax, eax              ; Add
jmp     short loc_401007      ; Jump
; ---------------------------------------------------------------------------
loc_400FF2:                   ; CODE XREF: func4+16↑j
mov     eax, 0
cmp     ecx, edi              ; Compare Two Operands
jge     short loc_401007      ; Jump if Greater or Equal (SF=OF)
lea     esi, [rcx+1]          ; Load Effective Address
call    func4                 ; Call Procedure
lea     eax, [rax+rax+1]      ; Load Effective Address
loc_401007:                   ; CODE XREF: func4+22↑j
                              ; func4+2B↑j
add     rsp, 8                ; Add
retn                          ; Return Near from Procedure
; } // starts at 400FCE
func4 endp
; =============== S U B R O U T I N E =======================================
; __int64 __fastcall phase_4(__int64)
public phase_4
phase_4 proc near             ; CODE XREF: main+EE↑p
var_10= dword ptr -10h
var_C= dword ptr -0Ch
; __unwind {
sub     rsp, 18h              ; Integer Subtraction
lea     rcx, [rsp+18h+var_C]  ; Load Effective Address
lea     rdx, [rsp+18h+var_10] ; Load Effective Address
mov     esi, offset aDD       ; "%d %d"
mov     eax, 0
call    ___isoc99_sscanf      ; Call Procedure
cmp     eax, 2                ; Compare Two Operands
jnz     short loc_401035      ; Jump if Not Zero (ZF=0)
cmp     [rsp+18h+var_10], 0Eh ; Compare Two Operands
jbe     short loc_40103A      ; Jump if Below or Equal (CF=1 | ZF=1)
loc_401035:                   ; CODE XREF: phase_4+20↑j
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_40103A:                   ; CODE XREF: phase_4+27↑j
mov     edx, 0Eh
mov     esi, 0
mov     edi, [rsp+18h+var_10]
call    func4                 ; Call Procedure
test    eax, eax              ; Logical Compare
jnz     short loc_401058      ; Jump if Not Zero (ZF=0)
cmp     [rsp+18h+var_C], 0    ; Compare Two Operands
jz      short loc_40105D      ; Jump if Zero (ZF=1)
loc_401058:                   ; CODE XREF: phase_4+43↑j
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_40105D:                   ; CODE XREF: phase_4+4A↑j
add     rsp, 18h              ; Add
retn                          ; Return Near from Procedure
; } // starts at 40100C
phase_4 endp
```
p4调用一个递归函数，读取两个数字，第一个作为递归的参数，第二个与递归结果做运算。两者取或结果为0则通过，不为0则爆炸。  
可以得到递归期望返回一个0，第二个参数也应该是0。  
  
还原递归的代码如下  
p4中指定了a2初始为0，a3初始为14  
```c
int func4(int a1, int a2 = 0, int a3 = 14)
{
  int v3; // ecx
  int result; // rax

  v3 = (a3 - a2) / 2 + a2;
  if ( v3 > a1 )
    return 2 * func4(a1, a2);
  result = 0LL;
  if ( v3 < a1 )
    return 2 * func4(a1, v3 + 1) + 1;
  return result;
}
```
分析递归三要素可以得到答案，或者直接编译这个函数进行暴力枚举，也可以得到答案。  
其实还可以通过`dlopen`打开这个二进制文件，然后`dlsym`可以拿到这个`extern C`函数，直接调用，不需要还原代码。
## Phase 5
```x86asm
array_3449                    ; DATA XREF: phase_5+37↑r
db  6Dh ; m
db  61h ; a
db  64h ; d
db  75h ; u
db  69h ; i
db  65h ; e
db  72h ; r
db  73h ; s
db  6Eh ; n
db  66h ; f
db  6Fh ; o
db  74h ; t
db  76h ; v
db  62h ; b
db  79h ; y
db  6Ch ; l
; unsigned __int64 __fastcall phase_5(_BYTE *)
public phase_5
phase_5 proc near             ; CODE XREF: main+10A↑p
var_28= qword ptr -28h
var_18= byte ptr -18h
var_12= byte ptr -12h
var_10= qword ptr -10h
; __unwind {
push    rbx
sub     rsp, 20h              ; Integer Subtraction
mov     rbx, rdi
mov     rax, fs:28h
mov     [rsp+28h+var_10], rax
xor     eax, eax              ; Logical Exclusive OR
call    string_length         ; Call Procedure
cmp     eax, 6                ; Compare Two Operands
jz      short loc_4010D2      ; Jump if Zero (ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
jmp     short loc_4010D2      ; Jump
; ---------------------------------------------------------------------------
loc_40108B:                   ; CODE XREF: phase_5+4A↓j
                              ; phase_5+75↓j
movzx   ecx, byte ptr [rbx+rax] ; Move with Zero-Extend
mov     byte ptr [rsp+28h+var_28], cl
mov     rdx, [rsp+28h+var_28]
and     edx, 0Fh              ; Logical AND
movzx   edx, ds:array_3449[rdx] ; Move with Zero-Extend
mov     [rsp+rax+28h+var_18], dl
add     rax, 1                ; Add
cmp     rax, 6                ; Compare Two Operands
jnz     short loc_40108B      ; Jump if Not Zero (ZF=0)
mov     [rsp+28h+var_12], 0
mov     esi, offset aFlyers   ; "flyers"
lea     rdi, [rsp+28h+var_18] ; Load Effective Address
call    strings_not_equal     ; Call Procedure
test    eax, eax              ; Logical Compare
jz      short loc_4010D9      ; Jump if Zero (ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
align 10h
jmp     short loc_4010D9      ; Jump
; ---------------------------------------------------------------------------
loc_4010D2:                   ; CODE XREF: phase_5+20↑j
                              ; phase_5+27↑j
mov     eax, 0
jmp     short loc_40108B      ; Jump
; ---------------------------------------------------------------------------
loc_4010D9:                   ; CODE XREF: phase_5+62↑j
                              ; phase_5+6E↑j
mov     rax, [rsp+28h+var_10]
xor     rax, fs:28h           ; Logical Exclusive OR
jz      short loc_4010EE      ; Jump if Zero (ZF=1)
call    ___stack_chk_fail     ; Call Procedure
; ---------------------------------------------------------------------------
loc_4010EE:                   ; CODE XREF: phase_5+85↑j
add     rsp, 20h              ; Add
pop     rbx
retn                          ; Return Near from Procedure
; } // starts at 401062
phase_5 endp
```
p5读取6个字符长度的字符串，将这个字符串每一位与`0xF`做与运算，得到的结果在`array_3449`中作为偏移，最后得到一个字符串，并和`flyers`这个字符串进行对比，相同则通过，不同则引爆炸弹。  
主要逻辑在`0x40108B`代码块  
`rax`作为循环变量，每次循环加一，遍历长度为5，加到6终止循环。  
`rdx`作为`array_3449`的偏移，计算结果会保存到这个寄存器。栈加上`0x10`是新字符串储存的位置。
`strings_not_equal`前面已经分析过，不再分析。
## Phase 6
```x86asm
public node1
node1 LinkedListNode <10000014Ch, offset node2>
                              ; DATA XREF: phase_6:loc_401183↑o
                              ; phase_6+B0↑o
public node2
node2 LinkedListNode <2000000A8h, offset node3>
                              ; DATA XREF: .data:node1↑o
public node3
node3 LinkedListNode <30000039Ch, offset node4>
                              ; DATA XREF: .data:node2↑o
public node4
node4 LinkedListNode <4000002B3h, offset node5>
                              ; DATA XREF: .data:node3↑o
public node5
node5 LinkedListNode <5000001DDh, offset node6>
                              ; DATA XREF: .data:node4↑o
public node6
node6 LinkedListNode <6000001BBh, 0>
                              ; DATA XREF: .data:node5↑o
LinkedListNode <0>
; __int64 __fastcall phase_6(__int64)
public phase_6
phase_6 proc near             ; CODE XREF: main+126↑p
var_78= dword ptr -78h
var_60= byte ptr -60h
var_58= qword ptr -58h
var_50= byte ptr -50h
var_28= byte ptr -28h
; __unwind {
push    r14
push    r13
push    r12
push    rbp
push    rbx
sub     rsp, 50h              ; Integer Subtraction
mov     r13, rsp
mov     rsi, rsp
call    read_six_numbers      ; Call Procedure
mov     r14, rsp
mov     r12d, 0
loc_401114:                   ; CODE XREF: phase_6+5D↓j
mov     rbp, r13
mov     eax, [r13+0]
sub     eax, 1                ; Integer Subtraction
cmp     eax, 5                ; Compare Two Operands
jbe     short loc_401128      ; Jump if Below or Equal (CF=1 | ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_401128:                   ; CODE XREF: phase_6+2D↑j
add     r12d, 1               ; Add
cmp     r12d, 6               ; Compare Two Operands
jz      short loc_401153      ; Jump if Zero (ZF=1)
mov     ebx, r12d
loc_401135:                   ; CODE XREF: phase_6+57↓j
movsxd  rax, ebx              ; Move with Sign-Extend Doubleword
mov     eax, [rsp+rax*4+78h+var_78]
cmp     [rbp+0], eax          ; Compare Two Operands
jnz     short loc_401145      ; Jump if Not Zero (ZF=0)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_401145:                   ; CODE XREF: phase_6+4A↑j
add     ebx, 1                ; Add
cmp     ebx, 5                ; Compare Two Operands
jle     short loc_401135      ; Jump if Less or Equal (ZF=1 | SF!=OF)
add     r13, 4                ; Add
jmp     short loc_401114      ; Jump
; ---------------------------------------------------------------------------
loc_401153:                   ; CODE XREF: phase_6+3C↑j
lea     rsi, [rsp+78h+var_60] ; Load Effective Address
mov     rax, r14
mov     ecx, 7
loc_401160:                   ; CODE XREF: phase_6+79↓j
mov     edx, ecx
sub     edx, [rax]            ; Integer Subtraction
mov     [rax], edx
add     rax, 4                ; Add
cmp     rax, rsi              ; Compare Two Operands
jnz     short loc_401160      ; Jump if Not Zero (ZF=0)
mov     esi, 0
jmp     short loc_401197      ; Jump
; ---------------------------------------------------------------------------
loc_401176:                   ; CODE XREF: phase_6+8B↓j
                              ; phase_6+B5↓j
mov     rdx, [rdx+8]
add     eax, 1                ; Add
cmp     eax, ecx              ; Compare Two Operands
jnz     short loc_401176      ; Jump if Not Zero (ZF=0)
jmp     short loc_401188      ; Jump
; ---------------------------------------------------------------------------
loc_401183:                   ; CODE XREF: phase_6+A9↓j
mov     edx, offset node1
loc_401188:                   ; CODE XREF: phase_6+8D↑j
mov     [rsp+rsi*2+78h+var_58], rdx
add     rsi, 4                ; Add
cmp     rsi, 18h              ; Compare Two Operands
jz      short loc_4011AB      ; Jump if Zero (ZF=1)
loc_401197:                   ; CODE XREF: phase_6+80↑j
mov     ecx, [rsp+rsi+78h+var_78]
cmp     ecx, 1                ; Compare Two Operands
jle     short loc_401183      ; Jump if Less or Equal (ZF=1 | SF!=OF)
mov     eax, 1
mov     edx, offset node1
jmp     short loc_401176      ; Jump
; ---------------------------------------------------------------------------
loc_4011AB:                   ; CODE XREF: phase_6+A1↑j
mov     rbx, [rsp+78h+var_58]
lea     rax, [rsp+78h+var_50] ; Load Effective Address
lea     rsi, [rsp+78h+var_28] ; Load Effective Address
mov     rcx, rbx
loc_4011BD:                   ; CODE XREF: phase_6+DC↓j
mov     rdx, [rax]
mov     [rcx+8], rdx
add     rax, 8                ; Add
cmp     rax, rsi              ; Compare Two Operands
jz      short loc_4011D2      ; Jump if Zero (ZF=1)
mov     rcx, rdx
jmp     short loc_4011BD      ; Jump
; ---------------------------------------------------------------------------
loc_4011D2:                   ; CODE XREF: phase_6+D7↑j
mov     qword ptr [rdx+8], 0
mov     ebp, 5
loc_4011DF:                   ; CODE XREF: phase_6+101↓j
mov     rax, [rbx+8]
mov     eax, [rax]
cmp     [rbx], eax            ; Compare Two Operands
jge     short loc_4011EE      ; Jump if Greater or Equal (SF=OF)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_4011EE:                   ; CODE XREF: phase_6+F3↑j
mov     rbx, [rbx+8]
sub     ebp, 1                ; Integer Subtraction
jnz     short loc_4011DF      ; Jump if Not Zero (ZF=0)
add     rsp, 50h              ; Add
pop     rbx
pop     rbp
pop     r12
pop     r13
pop     r14
retn                          ; Return Near from Procedure
; } // starts at 4010F4
phase_6 endp
```
p6也读取了6个数字。
前面看起来很吓人，其实就是判断你输入的数据有没有重复或者超过了6。用了两个嵌套的循环，很常规的算法。  
`loc_401153`才是真正代码开始的地方。  
首先还是一个循环，这个循环遍历了整个输入的数组，然后将7减去这些数字，得到一个与原来的输入在数轴上关于3.5对称的新数组。  
接着又是一个循环，循环变量在`rsi`中，`rsi`从0变到24，步进为4，使用时又除以4，实际上就是`rsi`从0变到6。`rsi`作为新数组的偏移。  
接着将数组中的当前数字移入`ecx`中，如果`ecx`小于等于1，则给edx赋值为node1在内存中的地址。如果大于1，则又是一个循环，给`edx`赋值为`node1`的地址，然后把`node1+8`赋值给`edx`，直到这个循环变量（`eax`）等于`ecx`。  
通过观察上面的操作，可以发现`node{n}`第一个数据为8字节长的数字，第二个为8字节长的偏移，也就是下一个`node`，实际上就是一个链表。  
知道了这是一个链表，下面就好做了。  
每次`rsi`的循环每次都会在栈上保存按照新数组顺序找到的`node{n}`
接下来这个循环对得到的新链表的`next`字段进行了重构，使其指向物理地址的下一个`node{n}`。
接下来的循环则对新链表进行遍历，`ebp`作为循环变量，从5减到0，`node`只有6个，刚好遍历到倒数第二个时循环结束。  
这个循环首先将当前遍历的`node`的下一个读取到`eax`，再和当前的进行比较，如果下一个比当前的大，则引爆炸弹。  
所以最后可以得到答案就是一个数组，将7减去每一位，得到新的索引，通过新的索引得到新的链表，这个链表需要是递增的。  
## Secret Phase
```x86asm
public n1
n1 BinaryTreeNode <24h, offset n21, offset n22, 0>
                              ; DATA XREF: secret_phase+2C↑o
public n21
n21 BinaryTreeNode <8, offset n31, offset n32, 0>
                              ; DATA XREF: .data:n1↑o
public n22
n22 BinaryTreeNode <32h, offset n33, offset n34, 0>
                              ; DATA XREF: .data:n1↑o
public n31
n31 BinaryTreeNode <6, offset n41, offset n42, 0>
                              ; DATA XREF: .data:n21↑o
public n32
n32 BinaryTreeNode <16h, offset n43, offset n44, 0>
                              ; DATA XREF: .data:n21↑o
public n33
n33 BinaryTreeNode <2Dh, offset n45, offset n46, 0>
                              ; DATA XREF: .data:n22↑o
public n34
n34 BinaryTreeNode <6Bh, offset n47, offset n48, 0>
                              ; DATA XREF: .data:n22↑o
public n41
n41 BinaryTreeNode <1, 0, 0, 0>
                              ; DATA XREF: .data:n31↑o
public n42
n42 BinaryTreeNode <7, 0, 0, 0>
                              ; DATA XREF: .data:n31↑o
public n43
n43 BinaryTreeNode <14h, 0, 0, 0>
                              ; DATA XREF: .data:n32↑o
public n44
n44 BinaryTreeNode <23h, 0, 0, 0>
                              ; DATA XREF: .data:n32↑o
public n45
n45 BinaryTreeNode <28h, 0, 0, 0>
                              ; DATA XREF: .data:n33↑o
public n46
n46 BinaryTreeNode <2Fh, 0, 0, 0>
                              ; DATA XREF: .data:n33↑o
public n47
n47 BinaryTreeNode <63h, 0, 0, 0>
                              ; DATA XREF: .data:n34↑o
public n48
n48 BinaryTreeNode <3E9h, 0, 0, 0>
                              ; DATA XREF: .data:n34↑o
; const char *read_line()
public read_line
read_line proc near           ; CODE XREF: main+92↑p
                              ; main+AE↑p
                              ; main+CA↑p
                              ; main+E6↑p
                              ; main+102↑p
                              ; main+11E↑p
                              ; secret_phase+1↑p
; __unwind {
sub     rsp, 8                ; Integer Subtraction
mov     eax, 0
call    skip                  ; Call Procedure
test    rax, rax              ; Logical Compare
jnz     short loc_40151F      ; Jump if Not Zero (ZF=0)
mov     rax, cs:stdin@@GLIBC_2_2_5
cmp     cs:infile, rax        ; Compare Two Operands
jnz     short loc_4014D5      ; Jump if Not Zero (ZF=0)
mov     edi, offset aErrorPremature ; "Error: Premature EOF on stdin"
call    _puts                 ; Call Procedure
mov     edi, 8                ; status
call    _exit                 ; Call Procedure
; ---------------------------------------------------------------------------
loc_4014D5:                   ; CODE XREF: read_line+21↑j
mov     edi, offset name      ; "GRADE_BOMB"
call    _getenv               ; Call Procedure
test    rax, rax              ; Logical Compare
jz      short loc_4014EE      ; Jump if Zero (ZF=1)
mov     edi, 0                ; status
call    _exit                 ; Call Procedure
; ---------------------------------------------------------------------------
loc_4014EE:                   ; CODE XREF: read_line+44↑j
mov     rax, cs:stdin@@GLIBC_2_2_5
mov     cs:infile, rax
mov     eax, 0
call    skip                  ; Call Procedure
test    rax, rax              ; Logical Compare
jnz     short loc_40151F      ; Jump if Not Zero (ZF=0)
mov     edi, offset aErrorPremature ; "Error: Premature EOF on stdin"
call    _puts                 ; Call Procedure
mov     edi, 0                ; status
call    _exit                 ; Call Procedure
; ---------------------------------------------------------------------------
loc_40151F:                   ; CODE XREF: read_line+11↑j
                              ; read_line+6B↑j
mov     edx, cs:num_input_strings
movsxd  rax, edx              ; Move with Sign-Extend Doubleword
lea     rsi, [rax+rax*4]      ; Load Effective Address
shl     rsi, 4                ; Shift Logical Left
add     rsi, 603780h          ; Add
mov     rdi, rsi
mov     eax, 0
mov     rcx, 0FFFFFFFFFFFFFFFFh
repne scasb                   ; Compare String
not     rcx                   ; One's Complement Negation
sub     rcx, 1                ; Integer Subtraction
cmp     ecx, 4Eh ; 'N'        ; Compare Two Operands
jle     short loc_40159A      ; Jump if Less or Equal (ZF=1 | SF!=OF)
mov     edi, offset aErrorInputLine ; "Error: Input line too long"
call    _puts                 ; Call Procedure
mov     eax, cs:num_input_strings
lea     edx, [rax+1]          ; Load Effective Address
mov     cs:num_input_strings, edx
cdqe                          ; EAX -> RAX (with sign)
imul    rax, 50h ; 'P'        ; Signed Multiply
mov     rdi, 636E7572742A2A2Ah
mov     ds:input_strings[rax], rdi
mov     rdi, 2A2A2A64657461h
mov     ds:qword_603788[rax], rdi
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_40159A:                   ; CODE XREF: read_line+B4↑j
sub     ecx, 1                ; Integer Subtraction
movsxd  rcx, ecx              ; Move with Sign-Extend Doubleword
movsxd  rax, edx              ; Move with Sign-Extend Doubleword
lea     rax, [rax+rax*4]      ; Load Effective Address
shl     rax, 4                ; Shift Logical Left
mov     byte ptr ds:input_strings[rcx+rax], 0
add     edx, 1                ; Add
mov     cs:num_input_strings, edx
mov     rax, rsi
add     rsp, 8                ; Add
retn                          ; Return Near from Procedure
; } // starts at 40149E
read_line endp
; unsigned __int64 phase_defused()
public phase_defused
phase_defused proc near       ; CODE XREF: main+9F↑p
                              ; main+BB↑p
                              ; main+D7↑p
                              ; main+F3↑p
                              ; main+10F↑p
                              ; main+12B↑p
                              ; secret_phase+4A↑p
var_70= byte ptr -70h
var_6C= byte ptr -6Ch
var_68= byte ptr -68h
var_10= qword ptr -10h
; __unwind {
sub     rsp, 78h              ; Integer Subtraction
mov     rax, fs:28h
mov     [rsp+78h+var_10], rax
xor     eax, eax              ; Logical Exclusive OR
cmp     cs:num_input_strings, 6 ; Compare Two Operands
jnz     short loc_40163F      ; Jump if Not Zero (ZF=0)
lea     r8, [rsp+78h+var_68]  ; Load Effective Address
lea     rcx, [rsp+78h+var_6C] ; Load Effective Address
lea     rdx, [rsp+78h+var_70] ; Load Effective Address
mov     esi, offset aDDS      ; "%d %d %s"
mov     edi, offset unk_603870
call    ___isoc99_sscanf      ; Call Procedure
cmp     eax, 3                ; Compare Two Operands
jnz     short loc_401635      ; Jump if Not Zero (ZF=0)
mov     esi, offset aDrevil   ; "DrEvil"
lea     rdi, [rsp+78h+var_68] ; Load Effective Address
call    strings_not_equal     ; Call Procedure
test    eax, eax              ; Logical Compare
jnz     short loc_401635      ; Jump if Not Zero (ZF=0)
mov     edi, offset aCursesYouVeFou ; "Curses, you've found the secret phase!"
call    _puts                 ; Call Procedure
mov     edi, offset aButFindingItAn ; "But finding it and solving it are quite"...
call    _puts                 ; Call Procedure
mov     eax, 0
call    secret_phase          ; Call Procedure
loc_401635:                   ; CODE XREF: phase_defused+3E↑j
                              ; phase_defused+51↑j
mov     edi, offset aCongratulation ; "Congratulations! You've defused the bom"...
call    _puts                 ; Call Procedure
loc_40163F:                   ; CODE XREF: phase_defused+1B↑j
mov     rax, [rsp+78h+var_10]
xor     rax, fs:28h           ; Logical Exclusive OR
jz      short loc_401654      ; Jump if Zero (ZF=1)
call    ___stack_chk_fail     ; Call Procedure
; ---------------------------------------------------------------------------
loc_401654:                   ; CODE XREF: phase_defused+89↑j
add     rsp, 78h              ; Add
retn                          ; Return Near from Procedure
; } // starts at 4015C4
phase_defused endp
; unsigned __int64 secret_phase()
public secret_phase
secret_phase proc near        ; CODE XREF: phase_defused+6C↓p
; __unwind {
push    rbx
call    read_line             ; Call Procedure
mov     edx, 0Ah              ; base
mov     esi, 0                ; endptr
mov     rdi, rax              ; nptr
call    _strtol               ; Call Procedure
mov     rbx, rax
lea     eax, [rax-1]          ; Load Effective Address
cmp     eax, 3E8h             ; Compare Two Operands
jbe     short loc_40126C      ; Jump if Below or Equal (CF=1 | ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_40126C:                   ; CODE XREF: secret_phase+23↑j
mov     esi, ebx
mov     edi, offset n1
call    fun7                  ; Call Procedure
cmp     eax, 2                ; Compare Two Operands
jz      short loc_401282      ; Jump if Zero (ZF=1)
call    explode_bomb          ; Call Procedure
; ---------------------------------------------------------------------------
loc_401282:                   ; CODE XREF: secret_phase+39↑j
mov     edi, offset aWowYouVeDefuse ; "Wow! You've defused the secret stage!"
call    _puts                 ; Call Procedure
call    phase_defused         ; Call Procedure
pop     rbx
retn                          ; Return Near from Procedure
; } // starts at 401242
secret_phase endp
; __int64 __fastcall fun7(__int64, __int64)
public fun7
fun7 proc near                ; CODE XREF: fun7+13↓p
                              ; fun7+29↓p
                              ; secret_phase+31↓p
; __unwind {
sub     rsp, 8                ; Integer Subtraction
test    rdi, rdi              ; Logical Compare
jz      short loc_401238      ; Jump if Zero (ZF=1)
mov     edx, [rdi]
cmp     edx, esi              ; Compare Two Operands
jle     short loc_401220      ; Jump if Less or Equal (ZF=1 | SF!=OF)
mov     rdi, [rdi+8]
call    fun7                  ; Call Procedure
add     eax, eax              ; Add
jmp     short loc_40123D      ; Jump
; ---------------------------------------------------------------------------
loc_401220:                   ; CODE XREF: fun7+D↑j
mov     eax, 0
cmp     edx, esi              ; Compare Two Operands
jz      short loc_40123D      ; Jump if Zero (ZF=1)
mov     rdi, [rdi+10h]
call    fun7                  ; Call Procedure
lea     eax, [rax+rax+1]      ; Load Effective Address
jmp     short loc_40123D      ; Jump
; ---------------------------------------------------------------------------
loc_401238:                   ; CODE XREF: fun7+7↑j
mov     eax, 0FFFFFFFFh
loc_40123D:                   ; CODE XREF: fun7+1A↑j
                              ; fun7+23↑j
                              ; fun7+32↑j
add     rsp, 8                ; Add
retn                          ; Return Near from Procedure
; } // starts at 401204
fun7 endp
```
隐藏关我感觉解密比入口简单，入口隐藏比较深。这个阶段我更建议用gdb去debug `phase_defused`。这里进行静态分析，就不走捷径了。  
首先在整个程序的汇编中其实就可以找到secret_phase这个函数，也能看到，只有在`phase_defused`中有调用，那么入口肯定在`phase_defused`中。  
在phase_defuesd中判断了`num_input_strings`是否等于6，这是一个全局变量，在`bss`段，每次调用`read_line`成功都会将这个值加上1。  
在`phase_defused`中读取了`0x603870`的数据，但是查找别的地方，没有出现这个地址。目光再回到`read_line`，有一段关键的代码。
```x86asm
mov     edx, cs:num_input_strings
movsxd  rax, edx              ; Move with Sign-Extend Doubleword
lea     rsi, [rax+rax*4]      ; Load Effective Address
shl     rsi, 4                ; Shift Logical Left
add     rsi, 603780h          ; Add
```
这段代码将`num_input_strings`乘以5（`lea`），再左移4位，也就是乘以`2^4`，得到`80*num_input_strings`，最后加上`0x603780`。  
稍微计算一下可以得到当`num_input_strings`等于3的时候，得到的结果刚好是`0x603870`。
而`read_line`返回的就是上面的结果，这个结果作为调用每一个阶段的参数，所以触发隐藏阶段就要在第三个阶段的答案后面加上对应的触发。  
这个触发是在`phase_defused`中判断的，与p1一样，能看到这个触发应该是一个字符串`DrEvil`。  
分析`pahse_deused`，读取了一个数字，传递给`fun7`，而`fun7`也是一个递归函数，直接还原代码。  
分析递归三要素得到答案，也可以像上面提到的编译函数暴力枚举，或者通过`dlopen`外部调用拿到`n1`的地址和`fun7`的地址进行暴力枚举。  
其实观察一下可以发现n1是一个二叉树的节点，整个二叉树一共三层，不过看不出来二叉树也能解出答案 ; )
```c
int fun7(BinaryTreeNode* a1, int a2)
{
  int result; // rax

  if ( !a1 )
    return 0xFFFFFFFF;
  if ( a1->val > a2 )
    return 2 * (unsigned int)fun7(a1->left, a2);
  result = 0;
  if ( a1->val != a2 )
    return 2 * (unsigned int)fun7(a1->right, a2) + 1;
  return result;
}
```
## 答案
`/`是多个答案的分割符，选择其中一个即可
```
Border relations with Canada have never been better.
1 2 4 8 16 32
0 207/1 311/2 707/3 256/4 389/5 206/6 682/7 327
0 0/1 0/3 0/7 0 DrEvil
9?>567/)/.%&'/IONEFG/Y_^UVW/ionefg
4 3 2 1 6 5
22
```
