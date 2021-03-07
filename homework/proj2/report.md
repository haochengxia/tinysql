# Proj 2 Parser

> Date: 2021/03/06 - 2021/03/XX

## 1 基础概念

主要基础概念均出自于《编译原理》，略过。

阅读 [TiDB 源码阅读系列之 TiDB SQL Parser 的实现](https://pingcap.com/blog-cn/tidb-source-code-reading-5/) 笔记如下：

### 1.1 词法/语法分析器

1. **Lex**
   Lexical Analyzer Generator的简称，主要功能是根据用户编写的Lex文件，生成一个词法分析器，[flex](https://github.com/westes/flex) 源码分析见笔者博客 [Link](http://blog.haochengxia.com/jekyll/update/2021/03/09/flex-Source-Code-Analysis.html).
2. **Yacc**
   Yet Another Compiler Compiler的简称，[goyacc](https://gitlab.com/cznic/goyacc) 源码分析见笔者博客 [Link](http://blog.haochengxia.com/jekyll/update/2021/03/10/goyacc-Source-Code-Analysis.html).
3. **ANTLR**
   Another Tool for Language Recognition的简称，强大的语法分析器，对于Java程序员更加常用，笔者无使用场景，略。

### 1.2 数据库语法基本结构



## 2. Homework

## 2.1 分析

这里提供对`TinySQL`中 [parser.y]() 文件的详细解析。略长可直接跳转到下一小节。 [Jump](#22-实现)

开头将go的包管理与依赖用`%{ %}`包裹，随后`%union`声明符号值对应变量类型。

下方为go中对应的结构体。

```go
type yySymType struct {
	yys       int
	offset    int // offset
	item      interface{}
	ident     string
	expr      ast.ExprNode
	statement ast.StmtNode
}
```

下面定义token，绑定至ident域，同时定义token时声明等价字符串，如`identifier`等价于字符串`"identifier"`。

```yacc
%token	<ident>
	identifier      "identifier"
    ...
```

随后定义优先级和结合性，此部分规则见链接 [Link](https://www.gnu.org/software/bison/manual/html_node/Precedence.html#Precedence)。


```yacc
%start	Start
```

表示整个解析过程从Start开始。

随后即为 SQL 语法的产生式和每个规则对应的 aciton。

## 2.2 实现

## 2.3 结果

## Reference
[1] [SQL解析系列(golang)--goyacc实战](https://zhuanlan.zhihu.com/p/264367718)
[2] [[原创]Flex和Bison中巧用单双引号提升语法文件的可读性](https://blog.csdn.net/u014038143/article/details/78202271)