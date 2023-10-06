# node 手写编译器-minicompiler

使用语言：node, C++
本文涉及：编译器的词法分析，抽象语义树生成，语法分析，代码生成
本文重点内容：

1. 实现正则表达式分析器
2. 实现简易版 Flex
3. 实现 LR0 语法，并简单介绍其他语法(LL, LR1, SLR, LSPR)
4. 实现简易版 Bison
5. 实现生成汇编代码
6. 实现简易编译器功能，提供编译时期类型检查和推断，支持加减乘除支持函数的递归调用，以及

会包含的：

1. 实现 nfa，以及联合 nfa => dfa，进行词法分析
2. 实现 dfa，进行语法分析，并基于此构建抽象语法树(AST)
3. 基于 ast 进行语言义分析(类型检查以及是否符合语言规范)
4. 基于 ast 生成汇编代码，虽然本文没有显示的终中间代码生成过程，但是也有类似的思想

不会包含的：
本语言比较简单，不包含复杂数据结构的支持，如数组对象等其他功能；不会涉及复杂的编译器后端知识：如垃圾回收，寄存器染色，数据流分析等等

minic 语法：

1. 运算符：支持+-\*/运算，不支持优先级(, )
2. 类型：支持自然数(int 类型)，字符串，布尔值以及 void
3. 语句：支持函数调用，嵌套的 if-else 语句（if-else必须成对出现，如下的语法是不允许的）
4. 和 C 一样，必须要有 main 函数
5. 变量必须声明的时候同时赋值，如下的声明是不允许的
6. 允许块级作用域
7. 允许返回值是条件表达式，运算表达式
8. 运算符左右两侧仅可以是变量或是数字

## 一、minic 语法：

```c
int sum(int x, int y) {
  return x + y;
}

int feb(x: int) {
  if (x == 0) {
    return x;
  }
  return x + feb(x-1);
}
```

下面我会从以下四个方面依次入手：

<ol>
<li>parser</li>
    <ul>
    <li>正则</li>
    <li>todo</li>
    </ul>
<li>sematic</li>
  <ul>
    <li>slr</li>
    <li>语义分析</li>
    <li>todo</li>
  </ul>
<li>代码生成</li>
<li>可视化</li>

## 二、parser

在词法分析阶段，输入是字符串，输出是 token 流，这里 token 流有一个坑，一开始我的输出是枚举值的数组，类似这样: `[TYPE, ID, BRACE, ...]`，但是这样会有问题，因为词素值没办法保留下来，所以后面改成了如下输出: `[(line_num, TYPE, int), (line_num, ID, feb)]`，(行号，类型, 词素值)三元组
学过编译原理都知道，需要有自动机把输入流，转成一个个 token，比如当词法分析器读完 int 的时候，这时候就会返回一个 TYPE token，但是如果是 int1，就应该返回一个 ID token，所以这里涉及到贪婪读取，另外对于像 if 这种关键字，如果同时满足多种终结状态，应该涉及到优先级

那么如何构建自动机？
首先我们的目标是对每种特定的 token 设置对应的正则表达式，并生成对应的 dfa，用过 flex 的同学应该都知道，比如：

然后我们把这些 dfa 构建为一个大的 nfa，再把 nfa 转为 dfa

我们的词法分析器支持以下几个单元：
+: a+,
?: a?,
_: a_,
连接: ab，
逻辑或: a|b，
字符集: [a-z]
支持少部分字符转义：\s,\ t, \n

如何把正则表达式构建为 dfa：对于每一个单元正则表达式，可以直接生成对应的节点，但是有些问题我们需要注意：
我们的输入是一个个正则表达式，正则表达式本身可以理解为是一个个单元，而这些单元又可能是字符或者其他单元和成的，如：
a|b 就是一个单元，但是其组成就是 2 个字符
[a-z]|a 就是一个单元+一个普通字符
另外对于中括号这种还需要特殊处理，思路如下：即使是字符也会抽象成节点的概念，另外在生成自动机的过程中，由于存在[]，+，\*等等这样的修饰符号，考虑使用 stack 进行存储，一般的，形式化如下：对于正则表达式 xxxxx
其中 X 为可能的任意字符，`stack = []`

```c
i = 0
pattern = 'XXXX'
while (i < len(pattern)) {
  auto node = getNode(pattern[i]);
  stack.push(node);
}

node getNode(string s) {
  if (s === ']')
  if (s === '+')
  ...
}
```

其中 getNode 函数做的事请就是识别特定的符号，然后用对应的正则表达式函数生成即可，比如对于特殊的符号: [],需要收集其包含的字符集，然后构建一个如图的基本正则单元；再比如 a+，我们需要从 stack 内 pop 出一个元素，生成对应的基本正则单元。

最后，所有的特殊符号处理完成，也就意味着栈内只有正则表达式单元这种数据结构，这些节点之间没有任何修饰符(因为修饰符之前都被处理完了，除了连接运算)，然后我们依次连接这些节点即可，另外对于每个正则的叶子节点确定，即当前栈顶元素的最后一个节点(因为栈顶元素对应正则的末尾)。

我们可以提供相应的函数，我们的词法分析器包含以下几种基本正则单元：

1. 连接运算符

```node
export function connect(from: VertexNode, to: VertexNode): VertexNode {
  // from的尾和to的头相互连接,注意circle
  let cur = graph.getVertex(from.index); // 获取邻接表
  const memo: number[] = [];
  while (cur.firstEdge && !memo.includes(cur.index)) {
    memo.push(cur.index);
    cur = graph.getVertex(cur.firstEdge.index);
  }

  graph.getVertex(cur.index).firstEdge = new Node(
    to.index,
    graph.getVertex(cur.index).firstEdge
  );
  return from;
}
```

2. 逻辑或

```node
export function or(a: VertexNode, b: VertexNode): VertexNode {
  const nodeStart = new VertexNode(Graph.node_id, null);
  graph.addVertexNode(nodeStart, nodeStart.index);
  nodeStart.firstEdge = new Node(a.index, null, a.edgeVal || null);
  nodeStart.firstEdge.next = new Node(b.index, null, b.edgeVal || null);
  const nodeEnd = new VertexNode(Graph.node_id, null);
  graph.addVertexNode(nodeEnd, nodeEnd.index);
  connect(a, nodeEnd);
  connect(b, nodeEnd);
  return nodeStart;
}
```

3. 字符集

```node
export function characters(chars: string[]) {
  const nodeStart = new VertexNode(Graph.node_id, null);
  graph.addVertexNode(nodeStart, nodeStart.index);
  const nodeEnd = new Node(Graph.node_id, null, chars);
  const tmp = new VertexNode(nodeEnd.index, chars);
  graph.addVertexNode(tmp, tmp.index);

  const pre = nodeStart.firstEdge;
  nodeStart.firstEdge = nodeEnd;
  nodeEnd.next = pre;

  return nodeStart;
}
```

4. \*修饰符

```node
export function mutipliy(wrapped: VertexNode): VertexNode {
  const nodeStart = new VertexNode(Graph.node_id, null);
  graph.addVertexNode(nodeStart, nodeStart.index);
  const tmp = new Node(wrapped.index, null, null);
  nodeStart.firstEdge = tmp;
  let cur = graph.getVertex(wrapped.index); // 获取邻接表
  while (cur.firstEdge) {
    cur = graph.getVertex(cur.firstEdge.index);
  }
  connect(cur, nodeStart);
  return nodeStart;
}
```

5. +修饰符

```node
export function plus(base: VertexNode) {
  // 基于old新建节点
  let nodeStart = new VertexNode(Graph.node_id, base.edgeVal);
  nodeStart.firstEdge = base.firstEdge;
  const res = nodeStart;
  graph.addVertexNode(nodeStart, nodeStart.index);
  let cur = base?.firstEdge;
  while (cur) {
    const vertexNode = graph.getVertex(cur?.index);
    const tmp = new VertexNode(Graph.node_id, vertexNode.edgeVal);
    nodeStart.firstEdge = new Node(tmp.index, null, vertexNode.edgeVal);
    nodeStart = tmp;
    tmp.firstEdge = base.firstEdge;
    graph.addVertexNode(tmp, tmp.index);
    cur = vertexNode.firstEdge;
  }
  return mutipliy(res);
}
```

不过比较困扰的是这些节点的数据结构如何存储是一件要考虑周到的事：
需要节点 id，由于自动机是有向图，并且可能带环，并且节点和节点之间可能存在不止一条边，考虑了下，还是用 linjie 表存储(主要是第一版的代码是这样的，再加上如果感觉节点之间的连接可能在某些情况下比较少，临界矩阵比较浪费内存)，firstEdge 指向其所有的临界边，edgeVal 是边上的值，对于该图的搜索，使用 bfs+dfs+检测环。
通过以上流程，我们得到了一个 nfa，由该 nfa 可以得到各个节点的边，以及叶子节点(并且叶子节点上放置了相应的动作)

nfa => dfa:
这一步使用的是虎书上的算法，这里不过多描述，简单来说就是把很多个大的 nfa 联合构建成一个 dfa 从而得到一张状态转移表

## 三、构建抽象语法树

这里我用的是简单的 LR(0)上下无关文法，关于什么是上下无关文什么是有关文，请戳->,理论上上下无关文法 LR(1) > LSPR > SLR > LR(0)
LR(0): 没有提前预测的符号，容易出现 shift-reduce 冲突以及 reduce-reduce 冲突，所以需要设计适合的文法；
SLR: 有简单的预测，可以解决 shift-reduce 冲突，但是无法解决 reduce-reduce 冲突
LR(1): 可以解决 shift-reduce 冲突，也解决 reduce-reduce 冲突
LSPR: 由于 LR(1)的表特别大，在此基础上做了优化

根据相应文法生成 ast，本质上是利用栈和 nfa，看如下文法的 LR(0)生成过程：
E-> Program $
Program -> Assign == Assign
Assign -> Assign + Token
Assign -> Token
Token -> id

LR(0)dfa

![](../飞书20230430-125517.jpg)

LR(0)对应的状态转移表：
![](../飞书20230430-124728.jpg)

可以看到在状态 9 是存在移位-规约冲突的，这是因为 LR(0)默认在所有的终结符号处做规约，slr 语法是比 LR(0),更为广泛的一种语法，它只在特定的地方放置规约动作。具体来说，在上面的例子里，在状态 9 处，只有规约式 1 的移位指针到达里末尾，所以可以看下规约式 1 后面可能会紧接着什么符号，也就是 followSet(Program) = {\$}，所以只在$处放置规约动作。
下面是 slr 分析表：
![](../飞书20230430-163301.jpg)

有了这张表就可以基于输入(一般是 token 流)去做语法分析了，刷过算法题的都应该猜到了会用栈来作为数据结果，是的，每一次自底向上规约都可以看作是栈的 pop 动作，并且在这次动作中可以做很多事情，比如生成我们想要的树节点，如：

在设计 AST 结构的同时需要同时兼顾到 AST 的结构方便语义分析和代码生成，该语言的 ast 结构大致如下：

```node
Program_Class {
  expr: Function_Class {
    formal_list: [ 'x' ],
    name: 'total',
    expressions: Branch_Class {
      ifCond: Cond_Class {
        lExpr: Indentifier_Class { token: 'x' },
        rExpr: Int_Contant_Class { token: '0' },
        op: '=='
      },
      statementTrue: Return_Class { expr: Indentifier_Class { token: 'x' } },
      statementFalse: Assign_Class {
        name: 'm',
        ltype: 'int',
        r: Caller_Class {
          params_list: [ undefined ],
          id: 'total',
          params: Sub_Class {
            lvalue: Indentifier_Class { token: 'x' },
            rvalue: Int_Contant_Class { token: '1' }
          },
          next: undefined
        },
        next: Return_Class {
          expr: Add_Class {
            lvalue: Indentifier_Class { token: 'x' },
            rvalue: Indentifier_Class { token: 'm' }
          }
        }
      }
    },
    formals: Formal_Class { name: 'x', type: 'int', next: undefined },
    next: Function_Class {
      formal_list: [ 'x', 'y' ],
      name: 'sum',
      expressions: Return_Class {
        expr: Add_Class {
          lvalue: Indentifier_Class { token: 'x' },
          rvalue: Indentifier_Class { token: 'y' }
        }
      },
      formals: Formal_Class {
        name: 'y',
        type: 'int',
        next: Formal_Class { name: 'x', type: 'int', next: undefined }
      },
      next: Function_Class {
        formal_list: [],
        name: 'main',
        expressions: Assign_Class {
          name: 'x',
          ltype: 'int',
          r: Caller_Class {
            params_list: [ '10' ],
            id: 'total',
            params: Int_Contant_Class { token: '10' },
            next: undefined
          },
          next: Caller_Class {
            params_list: [ 'x' ],
            id: 'print',
            params: Indentifier_Class { token: 'x' },
            next: undefined
          }
        },
        formals: undefined,
        next: undefined,
        return_type: 'int'
      },
      return_type: 'int'
    },
    return_type: 'int'
  }
}
```

其树节点是各种表达式类，源码在 tree.ts 文件里，


follow集和first集生成：
follow集：
``` node
  _processFollow(_nts: string) {
    // 3 若存在一个表达式 X -> ABCD 则 Follow(A) 需要加上 First(B) - ε，若First(B) 包含 ε，则Follow(A) 需要加上 First(C) - ε，向右迭代... 迭代至表达式结束。
    for (const nts of this.nonTerminalSymbol) {
      for (let grammarArr of this.lfh2rfh.get(nts)) {
      
        const len = grammarArr.value.length;
        let index = -1;
        // 找到 B
        for (let i = 0; i < len; i += 1) {
          const ch = grammarArr.value[i];
          if (ch === _nts) {
            index = i;
            break;
          }
        }
        if (index === -1) continue;
        while (index + 1 < len) {
          const ch = grammarArr.value[index + 1];
          if (this.terminalSymbol.has(ch)) {
            // 终结符直接加入
            this._followSet[_nts].add(ch);
            break;
          } else {
            const set = this._firstSet[ch];
            for (const v of Array.from(set || []))
              if (v !== EMPTY) this._followSet[_nts].add(v);
            if (!set?.has(EMPTY)) break;
            index += 1;
          }
        }
      }
    }
  }

  startBuildFollowSet() {
    for (let nts of this.nonTerminalSymbol) {
      this._followSet[nts] = new Set();
    }
    this._followSet[START].add(END);
    while (true) {
      let flag = false;
      for (let nts of this.nonTerminalSymbol) {
        this._processFollow(nts);
        if (isEqual(prevFollowSet, this._followSet)) flag = true;
        else flag = false;
        prevFollowSet = cloneDeep(this._followSet);
      }
      if (flag) break;
    }
  }
```

```node
  // 求firset集这里做了点改动，一般需要手动消除左递归文法，这里在代码里如果遇到了左递归直接结束
  _processFirst(nts: string, memo: Set<string>) {
    memo.add(nts);
    for (const grammarArr of this.lfh2rfh.get(nts)) {
      const len = grammarArr.value.length;
      for (let i = 0; i < len; i += 1) {
        const ch = grammarArr.value[i];
        if (this.terminalSymbol.has(ch)) {
          this._firstSet[nts].add(ch);
          break;
        } else {
          if (i === 0 && ch === nts) {
            // 避免左递归
            break;
          }
          if (memo.has(ch)) {
            // 避免循环递归
            break;
          }
          
          this._processFirst(ch, memo);
          for (const v of this._firstSet[ch]) this._firstSet[nts].add(v);
          if (!this._firstSet[ch].has(EMPTY)) break;
        }
      }
    }
  }
  startuildFirstSet() {
    for (let nts of this.nonTerminalSymbol) this._firstSet[nts] = new Set();
    for (let nts of this.nonTerminalSymbol) {
      const memo = new Set([]);
      this._processFirst(nts, memo);
    }
  }
 ```


## 五、AST生成
如下所示，在提前定义好的文法产生式做对应的规约动作，如下case 0就是<code>Formals -> int, TOKEN.ID</code>，在这里利用栈内的元素生成Formal_Class；就这样每次在对应的文法产生式上对对应的做规约动作，从而完成自底向上的ast的构建

```node
[Formals -> int, TOKEN.ID]: 0,
[Formals -> Formals, ',', int, TOKEN.ID]: 1

case 0:
  res = new Formal_Class(yyvalsp[0][2], yyvalsp[1][2]);
  break;
case 1:
  res = new Formal_Class(yyvalsp[0][2], yyvalsp[1][2], yyvalsp[3]);
  break;

```

## 六、语义分析
在语义分析阶段可以做类型检查和基本的校验，这里放置了一些基本的类型检查动作：
1. 必须要有main函数，main函数的返回值必须是整形
2. 其他函数的返回值和实际返回值类型对应
3. 赋值语句左右两侧类型一致
4. 同一个作用域不得出现同名变量
5. 变量必须先声明并初始化才能使用




## 七、汇编代码生成
思路：自上向下进行利用堆栈机代码生成，仅使用了3个寄存器，a0，v0，t0，其中v0是辅助寄存器帮助函数返回值存储以及系统调用的退出和打印；
总结了下，最重要的点是对声明变量的内存分配以及取变量的时候，要知道对应的作用域链，该从哪个作用取，只要我们对基本的一些单元表达式做好了代码生成的工作，后面就是搭积木的工作了；下面是该语言的函数栈示意图：
![](../飞书20230502-104729.jpg)

这里如何取参数？由于函数栈在扩增的时候，不太方便通过sp指针获取参数和变量的存储位置，所以这里使用fp指针去作为基地址，寻找参数和局部变量

关于作用域问题是采用的树结构存储(双向链表)，每次从当前所在作用域内寻找变量，再继续依次向上寻找，直到找到函数级作用域；

以下是具体的几个单元表达式生成的例子：

a + b
lw $a0 局部 or 参数 / li $a0, imm
sw $a0, 0($29)
addiu $29, $29, -4
lw $a0 局部 or 参数 / li $a0, imm
add $a0, $t0, $a0


对于int x = 2，直接用li 命令存储，并在函数的局部变量处向函数栈内放置元素;
对于int x = a + b, 先利用a+b元表达式生成对应的代码，再用lw指令存储单元表达式的值，


## 八、编写代码高亮语法插件：

我这里是速成版，比较简单，只涉及简单的语法部分
需要安装：
```
$ npm install -g vsce
$ npm install -g yo
```

如果是需要编写全新的插件，则允许：
yo code 选择 new Language Support，提示一些问题，按需填写即可，插件名称尽可能唯一，不然在插件市场里不好搜，运行完命令之后会有一个生成目录，编写高亮语法的文件在 xxx.tmLanguage.json 文件里，如果你只是配置一个 VS Code 中已有语言的语法，记得删掉生成的 package.json 中的 languages 配置。

https://www.bookstack.cn/read/VS-Code-Extension-Doc-ZH/docs-language-extensions-syntax-highlight-guide.md#fu57uu(编写插件vscode)

这里看下我的规则：


以及vscode里的settings.json文件：
``` json
  "editor.tokenColorCustomizations":{
    "[Default Dark+]": { // 这里是自己所选择的主题颜色，我的是vscode默认的颜色
      "textMateRules": [
        {
          "scope": "identifier.name", // 自定义或者符合标准规范的命名，对应插件里的xxx.tmLanguage.json文件里的name选项
          "settings": {
              "foreground": "#33ba8f"
          }
      },
      {
        "scope": "id.name.mc",
        "settings": {
            "foreground": "#eb8328"
        }
      }
      ]
    }
  }
```
xxx.tmLanguage.json里的配置：
``` json
{
	"$schema": "https://raw.githubusercontent.com/martinring/tmlanguage/master/tmlanguage.json",
	"name": "minic",
	"patterns": [
		{
			"include": "#keywords"
		},
    {
      "include": "#type"
    },
    {
      "include": "#number"
    },
    {
      "include": "#id"
    },
    {
      "include": "#comment"
    }
	],
	"repository": {
    "type": {
			"patterns": [{
				"name": "support.type.primitive.mc",
				"match": "\\b(void|int|bool)\\b" // 类型
			}]
		},
    "keywords": {
			"patterns": [{
				"name": "keyword.control.mc",
				"match": "\\b(if|while|for|return|else)\\b" // 关键字
			}]
		},
    "number": {
			"patterns": [{
				"name": "constant.numeric.mc",
				"match": "\\b[0-9]+\\b" 
			}]
		},
    "id": {
			"patterns": [{
				"name": "id.name.mc",
				"match": "\\b[a-z][a-z0-9]*\\b"
			}]
		},
    "comment": {
			"patterns": [{
				"name": "comment.line.double-dash",
				"match": "^//.*" //注释
			}]
		}
	},
	"scopeName": "source.mc"
}
```


踩坑点：

1. 因为是根据分词 token 那一套来的，其实也就是对你的语言的文件后缀名，比如我这里是 mc 后缀，会做一个词法分析，词法分析的正则是自己编写的，然后每个正则也有对应的 name，vscode 在配置(setting.json 文件)里可以根据这些 name 做颜色的映射。所以一定要记得简单的正则要在两边加\\b\\b，表示单词分界，我一开始没加这个总是不生效

2. 关于 name 的命名，其实有一套规范的标准，当然也可以随心自定义，我这里的？？？就是自定义的。

```
$ cd myExtension
$ vsce package
# 生成 myExtension.vsix,这时候本地可以看到一些二进制插件文件
$ vsce publish

```

如果以前没有 vscode 插件市场的账户需要注册
https://code.visualstudio.com/api/working-with-extensions/publishing-extension
注册完之后，记得本地保存下密钥，因为会经常用的

取消发布的插件：
vsce unpublish -p 密钥 包 id
包 id 可以在 vscode 的对应插件的信息里可以看到，(一般点击插件的设置图标就可以看到包 id 选项)

TextMate 语法最好对正则比较熟悉
具体的详细资料：
https://code.visualstudio.com/api/language-extensions/semantic-highlight-guide#semantic-token-scope-map

https://macromates.com/manual/en/language_grammars#naming-conventions（TextMate语法规则）
https://macromates.com/manual/en/regular_expressions（相关正则）

## 九、前端可视化
canvas


项目踩坑点：

1. 我本地全局和项目下都是安装了 ts-nodede1，但是通过 npm run ts-node，就是起不起来：Cannot find module 'typescript'，后来在https://github.com/TypeStrong/ts-node/issues/707，好像是版本的问题，找到了最好的解决方案：npm link typescript，再运行即可

项目难点：
其实整个项目是在学习斯坦福编译原理的时候有顺便一起写的，写下来也挺累的，因为已经在工作了，在编写本项目前，也看了下斯坦福的 cool 源码，然后了解了下 bison 和 flex，顺便对照了了下虎书的伪代码，下面总结下一些比较重要或者困难的点：

1. 正则表达式部分，这一部分纯粹自己手写，代码设计可能看着有点邋遢，主要思想是借鉴虎书上的，虎书只画了几个图，也没有给伪代码，主要是怎么设计 nfa 的节点相关数据结构存储，一个是是用简单的链表，后面发现可能存在出度为多个的节点，就换用邻接表存储了，每个节点的信息包含了所有入度边的信息；其次正则表达式分析这里用的也是 stack 对正则表达式进行分析，栈内的元素可能是节点也可能是字符串，在每次 pop 的时候需要注意对当前出栈的节点进行遍历拿到头尾节点，方便节点之间的连接。然后是将各个 nfa 联合成一个大 nfa 的过程，采用 dfa + bfa + 备忘录 + 回溯；最好是将 nfa => dfa 的过程，这一部分参照伪代码的，应该没啥大问题；

2. ast 生成部分：这一步文法分析表生成是参照伪代码的，后面的 ast 生成过程是纯手写，其实在实现的过程中，也发现整个过程和 bison 的 yy 文件如出一辙，不知道 bison 是怎么实现的 😯，本来是想弄成简单的 bison 文件编写文法规则的，但是时间不允许，后面再抽空看下。ast 生成这一部分主要是在规约动作上放置一些用户定义动作，这里就是写死的生成对应文法表达式的类，自底向上构建 ast

3. 这比一部分理论其实挺多的，但是在本项目里只要考虑周到即可，防止在编译的最好阶段报错，也是纯手写的

4. 汇编生成部分：这一部分个人觉得最恶心了，因为汇编比较难调试，每次有 bug 都需要汇编语句一句一句看，其实回过头看，这样把元表达式（像+-_/赋值语句）的寄存器分配设计写好了，就可以拿它们组合成更大的原件。这里也是纯手写的，有一个比较困扰我的点，像 if-else 里面声明的变量如何在运行时候分配内存？我们都知道局部变量在不考虑用寄存器优化的情况下，一般都是存储在栈内的，像参数都是基于 fp(栈指针)进行寻址的，比如形式参数 x, y，在生成汇编的时候就要记录，这里我拿数组的[x, y]，这样 x 的偏移量就是 fp + 1 _ 4, y 就是 fp + 2 \* 4，但是像 if else 里面的声明的变量只有运行到对应的分支才能分配内存，也就是说另一个没有执行的分支是不需要分配内存的。

5. 前端部分：用的是原生 canvas，主要是平时工作很少用 canvas，之前学过，不用就忘，顺便学习下，主要是整个 canvas 类的设计吧。
