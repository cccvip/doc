## 第一章节

书中代码golang1.4版本,目前使用golang1.20.3版本,导致很多类有所区别。推荐使用sourcegraph,能精确定位到需要找到函数

调试源码以及编译原理部分

编译原理可分为以下几个部分
- 词法与语法分析
- 类型检查和 AST 转换
- 通用 SSA 生成
- 机器代码生成

书中举了一个例子：Golang常见内置函数make,声明变量slice,map,channel的时候,会使用类型检查转换为具体的实现
```shell
   case ir.OMAKESLICE:
		n := n.(*ir.MakeExpr)
		e.spill(k, n)
		e.discard(n.Len)
		e.discard(n.Cap)
	case ir.OMAKECHAN:
		n := n.(*ir.MakeExpr)
		e.discard(n.Len)
	case ir.OMAKEMAP:
		n := n.(*ir.MakeExpr)
		e.spill(k, n)
		e.discard(n.Len)
```
整体感受：对golang的编译过程有初步认识,以及对抽象语法树有点兴趣,感觉可以基于这个实现很多有趣的功能





