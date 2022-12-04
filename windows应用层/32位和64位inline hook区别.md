## 1
对于系统API的hook，windows 系统为了达成hotpatch的目的，每个API函数的最前5个字节均为：
```
8bff   move edi,edi
55     push ebp
8bec   mov ebp,esp
```
其中move edi,edi这条指令是为了专门用于hotpatch而插入的，微软通过将这条指令跳转到一个short jmp，然后一个long jmp可以跳转到任意4G范围内的代码

假设我们要求把0x12345678这个地址的函数hook，使其跳转到0x12345690，我们可以将这5个字节替换为：0xe9 (0x12345690-0x12345678-5) ，以达到跳转到0x12345690这个地址的目的（此处，注意大小端系统的区别），这条指令是相对跳转。

64位系统没有了上面这样的方便之处

## 2
32位需要jmp的偏移为32位，64位为64位

## 3
32位下有的人会使用
```
68 78 56 34 12 push 0x12345678
c3 ret
```
的方式，但是此方式在64位下并不适用，因为push立即数不论是在32位还是64位都是push DWORD大小的数据
```
//64位模式下 68 78 56 34 12 12 34 56 78 c3会变成
push 0x12345678 //仍只push 4字节
```
64位inline hook一般如下：
```
0xff25 [0x00000000]
0xef cd ab 89 67 45 23 01  
```
这里用第二种方法，将一个old_func_address的前x个字节修改为跳转到我们的new_func_address，步骤：

1，反汇编old_func_address处的指令，累加其长度，依次反汇编下去，直到长度大于12，例如为15；

2，复制这15个字节的指令，将old_func_address的前12个字节修改为:0xff25 0x00000000 new_func_address（8个字节）；

3，跳转完成之后将其前15个字节还原。

这个方法要求函数长度大于15个字节，所以有一个方法用于适用于小于15字节长度的函数：

通过一个0xe9 tmp_address跳转到我们申请的空间（该空间地址与old_func_address的间隔在4G范围内，通过VirsualAlloc函数达成），在tmp_address处再long jmp（0xff25 …）。

```
ff25 + 4字节地址0x00000000 + 8字节绝对地址
4字节地址是相对下一条指令的偏移，然后8字节地址就是在前面4字节地址上的数据

比如说ff25 00000000 0x1234567887654321

就是跳转到0x1234567887654321
```

>https://xz.aliyun.com/t/9166
>https://www.ired.team/offensive-security/code-injection-process-injection/how-to-hook-windows-api-using-c++
