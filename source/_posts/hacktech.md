---
title: hacktech
date: 2023-05-06 15:39:38
tags: 渗透，爆破
---

内网渗透+爆破：https://blog.csdn.net/m0_46684679/article/details/117854834


安全集锦：https://www.zhihu.com/column/c_1334810805263515648


网络防火墙：https://zhuanlan.zhihu.com/p/159088465


格式: echo -e "\033[字背景颜色;字体颜色m字符串\033[0m" 

-e对特殊字符做转义

eg:
echo -e "\033[41;36m something here \033[0m" 

其中41的位置代表底色, 36的位置是代表字的颜色 

正则通配符和linux通配符是不一样的，
*（星号）是linux中的通配符，代表一个或一个以上的所有字符。linux的隐藏文件和隐藏文件夹都是以.（点号）开头，所以.*应该是代表当前目录下的所有隐藏目录和隐藏文件夹。
如果是./*则表示当前目录下的所有文件和所有目录，因为.（点号）还有代表当前目录的意思

https://zhuanlan.zhihu.com/p/96272363，日志应该是最初开发的一部分
其次，结束调试不要删除日志

写日志的地方：
关键方法调用：时间和调用参数
上下游对接处
可能存在异常的地方


其他console方法：
console.count: 用于计算函数被调用的次数
console.log.countReset

console.group: 用于折叠
console.groupCollapsed()
console.groupEnd

consoel.time()
console.timeEnd()

console.table()


console.dir(obj, { depth:  })


## Chrome Devtools
Network:
查看，过滤网络请求列表，查看请求详情
模拟弱网环境
搜索headers以及response内容
使用requeset block


Source
查看加载的资源文件
编辑css和js，修改css和js
snippets管理
断点调试
通过workspace关联到本地

1. 如何让source里的文件树看起来更清晰
——查看author-deployed，但是项目必须有source-map

2. 如何让call stack中仅出现关心的文件
——debug ignore list
在实际操作里，在step in的过程中，如果遇到了一个不想调试的文件直接右击文件的代码区域，add ignore list，下一次step in的时候就不会出现这个文件了，

如何恢复？
——顶级settings，找到ignore list

3. 如何对js的修改，reload之后还可以生效
——overrides，选择本地目录添加到overrides中，

4. 如何找出哪一行代码影响了我的元素
——dom change breakponits
比如找到哪一行删除了某个dom节点：右击某个dom节点——》break on-》node removal

4. 如何找出哪一行代码发起了请求
——source面板找到xhr/fetch breakpoints，输入拦截的url即可


5. 异常断点：source面板找到break points，勾选pause on caught/uncaught points
捕获的or未捕获的

6. 事件断点
全局事件：source面板找到event listener break points，选择相应的事件即可


7. 如何打断点，不会中断？
source 面板，某一行，右击选择edit break points =》选择log points，输入想打印的语句即可











字颜色:30-----------37
30:黑 
31:红 
32:绿 
33:黄 
34:蓝色 
35:紫色 
36:深绿 
37:白色 

字背景颜色范围:40----47
40:黑 
41:深红 
42:绿 
43:黄色 
44:蓝色 
45:紫色 
46:深绿 
47:白色

字体加亮颜色:90------------97
90:黑 
91:红 
92:绿 
93:黄 
94:蓝色 
95:紫色 
96:深绿 
97:白色

背景加亮颜色范围:100--------------------107
40:黑 
41:深红 
42:绿 
43:黄色 
44:蓝色 
45:紫色 
46:深绿 
47:白色


===============================================ANSI控制码的说明 
\33[0m 关闭所有属性 
\33[1m 设置高亮度 
\33[4m 下划线 
\33[5m 闪烁 
\33[7m 反显 
\33[8m 消隐 
\33[30m -- \33[37m 设置前景色 
\33[40m -- \33[47m 设置背景色 
\33[nA 光标上移n行 
\33[nB 光标下移n行 
\33[nC 光标右移n行 
\33[nD 光标左移n行 
\33[y;xH设置光标位置 
\33[2J 清屏 
\33[K 清除从光标到行尾的内容 
\33[s 保存光标位置 
\33[u 恢复光标位置 
\33[?25l 隐藏光标 
\33[?25h 显示光标

 

\x1b[2J\x1b[$;1H    $表示行位