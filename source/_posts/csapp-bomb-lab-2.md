---
title: CSAPP - Bomb Lab 笔记番外篇
date: 2021-01-29 17:34:20
categories: 技术笔记
tags: CSAPP
---

🎵 Dr.Evil: 叮咚，我有一个秘密，悄悄告诉你～

虽然在 Bomb Lab 的提示里只提到了 6 个关卡，然而稍微留心一点便会发现，在反编译的代码中，出现了一个单词 "secret_phase"。
显然，除了明面上的 6 个关卡，还有一个隐藏关等着我们。那么，这个 secret_phase 要如何才能触发呢？
<!--more-->

在反编译代码中搜索 "secret_phase" 关键字，我们发现隐藏关卡的调用发生在 `phase_defused` 这个函数里，触发条件就隐含在下面这段代码中：
```
00000000004015c4 <phase_defused>:
  ...
  4015d8:	83 3d 81 21 20 00 06 	cmpl   $0x6,0x202181(%rip)        # 603760 <num_input_strings>
  4015df:	75 5e                	jne    40163f <phase_defused+0x7b>
  4015e1:	4c 8d 44 24 10       	lea    0x10(%rsp),%r8             # phase_6 defuse 之后进入
  4015e6:	48 8d 4c 24 0c       	lea    0xc(%rsp),%rcx
  4015eb:	48 8d 54 24 08       	lea    0x8(%rsp),%rdx
  4015f0:	be 19 26 40 00       	mov    $0x402619,%esi             # "%d %d %s"
  4015f5:	bf 70 38 60 00       	mov    $0x603870,%edi             # phase 4 的输入
  4015fa:	e8 f1 f5 ff ff       	callq  400bf0 <__isoc99_sscanf@plt>
  4015ff:	83 f8 03             	cmp    $0x3,%eax
  401602:	75 31                	jne    401635 <phase_defused+0x71>
  401604:	be 22 26 40 00       	mov    $0x402622,%esi             # "DrEvil"
  401609:	48 8d 7c 24 10       	lea    0x10(%rsp),%rdi
  40160e:	e8 25 fd ff ff       	callq  401338 <strings_not_equal>
  401613:	85 c0                	test   %eax,%eax
  401615:	75 1e                	jne    401635 <phase_defused+0x71>
  401617:	bf f8 24 40 00       	mov    $0x4024f8,%edi
  40161c:	e8 ef f4 ff ff       	callq  400b10 <puts@plt>
  401621:	bf 20 25 40 00       	mov    $0x402520,%edi
  401626:	e8 e5 f4 ff ff       	callq  400b10 <puts@plt>
  40162b:	b8 00 00 00 00       	mov    $0x0,%eax
  401630:	e8 0d fc ff ff       	callq  401242 <secret_phase>
  ...
```
在 4015d8 行给出了一个注释 "# 603760 <num_input_strings>"，这里实际上是对用户输入个数做了个判断，只有 6 个关卡都 defused 了才会走到 4015e1。
在上一篇笔记中，我们提到过系统调用默认输入参数依次存放于 %rdi, %rsi, %rdx, %rcx, %r8 和 %r9。4015e6 - 4015f5 分别将 5 个地址依次存入了这几个寄存器，然后调用 sscanf。我们用 `x/s` 打印出 0x402619 和 0x603870 分别得到 "%d %d %s" 和 phase 4 的输入。由此可知，这一步期望的 phase 4 的输入为两个数字**外加一个字符串**，而且这三个部分将被分别存放在距离栈 0x8, 0xc 和 0x10 的地址中。在解析第 4 关时我们知道这一关的密钥是几组特殊的数对，显然，这个额外多出来的字符串就是触发隐藏关卡的关键了。
接下来，在 40160e 将这个额外多出的字符串与存于 0x402622 的字符串进行比较，打印 0x402622 可知这个秘密字符串为 **"DrEvil"**。如果二者相等，则会触发 secret phase。

综上，我们得到了触发隐藏关卡的方法：**在 phase 4的输入后面加上一个特殊字符串 "DrEvil"**。

那么，这个秘密关卡又要如何解呢？

---
## Secret Phase - 二叉搜索

先看 secret_phase 代码：
```
0000000000401242 <secret_phase>:
  401242:	53                   	push   %rbx
  401243:	e8 56 02 00 00       	callq  40149e <read_line>           # read the last(7th) line
  401248:	ba 0a 00 00 00       	mov    $0xa,%edx                    # %ead = 10
  40124d:	be 00 00 00 00       	mov    $0x0,%esi                    # %esi = 0
  401252:	48 89 c7             	mov    %rax,%rdi                    # %rdi = user input
  401255:	e8 76 f9 ff ff       	callq  400bd0 <strtol@plt>          # strtol
  40125a:	48 89 c3             	mov    %rax,%rbx
  40125d:	8d 40 ff             	lea    -0x1(%rax),%eax              # x = x - 1
  401260:	3d e8 03 00 00       	cmp    $0x3e8,%eax                  
  401265:	76 05                	jbe    40126c <secret_phase+0x2a>   # if x > 1000, explode bomb
  401267:	e8 ce 01 00 00       	callq  40143a <explode_bomb>
  40126c:	89 de                	mov    %ebx,%esi
  40126e:	bf f0 30 60 00       	mov    $0x6030f0,%edi             # "$"
  401273:	e8 8c ff ff ff       	callq  401204 <fun7>
  401278:	83 f8 02             	cmp    $0x2,%eax
  40127b:	74 05                	je     401282 <secret_phase+0x40>
  40127d:	e8 b8 01 00 00       	callq  40143a <explode_bomb>
  401282:	bf 38 24 40 00       	mov    $0x402438,%edi             # "Wow! You've defused...."
  401287:	e8 84 f8 ff ff       	callq  400b10 <puts@plt>
  40128c:	e8 33 03 00 00       	callq  4015c4 <phase_defused>
  ...
```

401243 - 401255 这一段先调用 readline 读入输入，然后调用了库函数 strtol，将用户输入的字符串转为十进制 long 型整数；
40125a - 401267 判断输入的数值是否小于 1001，如果大于则引爆炸弹，否则继续；
40126c - 401273 将输入的数值和 "0x6030f0" 作为参数，调用了 "<fun7>" 这个函数；
401278 - 40128c 判断 fun7 返回值是否为 2，如果是，则成功破解隐藏关卡，否则引爆炸弹。

接下来，破解这一关的关键就转移到了 <fun7> 这个函数，那就让我们来看看这又是何方妖孽吧～
代码倒是不复杂，且不难看出这又是一个递归函数：
```
0000000000401204 <fun7>:
# %esi 中存入的为输入的值，设为 x 
  401204:	48 83 ec 08          	sub    $0x8,%rsp
  401208:	48 85 ff             	test   %rdi,%rdi                # 初始值为 0x6030f0
  40120b:	74 2b                	je     401238 <fun7+0x34>
  40120d:	8b 17                	mov    (%rdi),%edx              # 取出存于 %rdi 中的值 y
  40120f:	39 f2                	cmp    %esi,%edx 
  401211:	7e 0d                	jle    401220 <fun7+0x1c>       # if (x <= y) goto <A>
  401213:	48 8b 7f 08          	mov    0x8(%rdi),%rdi           # %rdi = (%rdi + 8)
  401217:	e8 e8 ff ff ff       	callq  401204 <fun7>
  40121c:	01 c0                	add    %eax,%eax                # %eax = %eax * 2
  40121e:	eb 1d                	jmp    40123d <fun7+0x39>       # goto <R>
# <A>
  401220:	b8 00 00 00 00       	mov    $0x0,%eax                # %eax = 0
  401225:	39 f2                	cmp    %esi,%edx
  401227:	74 14                	je     40123d <fun7+0x39>       # if %esi == %edx, return
  401229:	48 8b 7f 10          	mov    0x10(%rdi),%rdi          # %rdi = (%rdi + 16)
  40122d:	e8 d2 ff ff ff       	callq  401204 <fun7>
  401232:	8d 44 00 01          	lea    0x1(%rax,%rax,1),%eax    # %eax = %rax + %rax * 1 + 1
  401236:	eb 05                	jmp    40123d <fun7+0x39>       # goto <R>
  401238:	b8 ff ff ff ff       	mov    $0xffffffff,%eax         # %eax = -1
# <R>
  40123d:	48 83 c4 08          	add    $0x8,%rsp
  401241:	c3                   	retq  
```
首先，`<fun7>` 判断了一下 %rdi 这个地址是否为 0，如果为 0，返回 -1，否则接着取出 %rdi 中地址里存放的值与输入进行比较：
* 如果输入值小于这个值，%rdi 变为 %rdi + 8 的地址里的内容，递归调用 `fun7`，将这次调用的返回的值 * 2，然后返回；
* 如果输入值等于这个值，返回，此时 %eax = 0，即返回值为 0；
* 如果输入值小于这个值，%rdi 变为 %rdi + 16 的地址里的内容，递归调用 `fun7`，将这次调用的返回值 * 2 + 1，然后返回。
* 如果

这个逻辑分析下来，是不是有点像二叉搜索呀：%rdi 的初始值 0x6030f0 存放的是二叉树的根节点，然后比较输入值与节点值的大小，移动 %rdi，使其指向左子节点或右子节点。
是不是越想越合理？赶紧打印出 0x6030f0 开始的内存内容看看是不是这样：
![phase secret tree][1]
这下有理有据了，这个树画出来长下面这样（最后一个节点是 1001，偷懒用了 leetcode 的 tree visualizer，结果 4 位数显示不全...）
![phase secret tree visualization][2]

最后一个需要解决的问题，就是要找出 secret phase 到底想搜索的是哪个值。根据上面的分析，我们已知 secret_phase 期望的返回值为 2，通过回溯递归的过程，最终找到目标值就是 22 或 20。

---

🎉 那么，关于 Bomb Lab 的笔记到这里就全部结束啦～我们有缘再见～

---

[1]: /blog/uploads/images/bomb_phasesec_tree.png
[2]: /blog/uploads/images/bomb_phasesec_tree_viz.png