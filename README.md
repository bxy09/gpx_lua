# 诸葛量化 Lua 编程指南
诸葛量化是一个金融策略自动生成平台，其核心在于一个快速的策略回测平台、实盘托管系统以及指标公式的编辑与智能搜索算法。我们除了提供自建的策略模板外，还开放了 Lua 语言的编程接口。有编程基础的用户可以通过自己编写程序，利用我们的平台进行自己的策略编写。并且该接口同样可以享受到我们所提供的指标公式的编辑与智能搜索算法所带来的便利。

目前 Lua 语言的编程接口尚处于测试阶段，我们将继续改善策略的回测效率以及开放更多的接口方便用户使用。

由于需要考虑安全等因素，我们通过白名单限制了 Lua 语言所能调用的库。如果您有自己的需求，可以与我们联系。目前我们所使用的 Lua 版本为最广受使用的 5.1 版。

开放库与函数名单：
```
库：
math

函数：
pairs
tostring
print (修改版，日志将输出到用户策略执行的任务中)
```
