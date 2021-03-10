# Proj 2 Parser

> Date: 2021/03/06 - 2021/03/10

## 1 基础概念

主要基础概念均出自于《编译原理》，略过。

阅读 [TiDB 源码阅读系列之 TiDB SQL Parser 的实现](https://pingcap.com/blog-cn/tidb-source-code-reading-5/) 笔记如下：

### 1.1 词法/语法分析器

1. **Lex**
   Lexical Analyzer Generator的简称，主要功能是根据用户编写的Lex文件，生成一个词法分析器，[flex](https://github.com/westes/flex) 源码分析见笔者博客。[Link](http://blog.haochengxia.com/jekyll/update/2021/03/15/flex-Source-Code-Analysis.html)
2. **Yacc**
   Yet Another Compiler Compiler的简称，[goyacc](https://gitlab.com/cznic/goyacc) 源码分析见笔者博客。[Link](http://blog.haochengxia.com/jekyll/update/2021/03/16/goyacc-Source-Code-Analysis.html)
3. **ANTLR**
   Another Tool for Language Recognition的简称，强大的语法分析器，对于Java程序员更加常用，笔者无使用场景，略。

### 1.2 数据库语法基本结构

参见MySQL手册。[Link](https://dev.mysql.com/doc/refman/5.7/en/)

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

随后定义优先级和结合性，此部分规则见链接 [Link](https://www.gnu.org/software/bison/manual/html_node/Precedence.html#Precedence)。或笔者[翻译版本](./5.3%20操作符优先级.md)。


```yacc
%start	Start
```

表示整个解析过程从Start开始。

随后即为 SQL 语法的产生式和每个规则对应的 aciton。

以 StatementList 对应的 action 为例子:
```yacc
StatementList:
	Statement
	{
		if $1 != nil {
			s := $1
			if lexer, ok := yylex.(stmtTexter); ok {
				s.SetText(lexer.stmtText())
			}
			parser.result = append(parser.result, s)
		}
	}
|	StatementList ';' Statement
	{
		if $3 != nil {
			s := $3
			if lexer, ok := yylex.(stmtTexter); ok {
				s.SetText(lexer.stmtText())
			}
			parser.result = append(parser.result, s)
		}
	}
```

对应如下的产生式：
![eq2.1.1](http://latex.codecogs.com/svg.latex?StatementList\rightarrow\ Statement)

![eq2.1.2](http://latex.codecogs.com/svg.latex?StatementList\rightarrow\ StatementList\ ';'\ Statement)


其中\$1表示产生式右边的第一个参数，第一个block的含义即为：
如果statement非空，从词法分析器拿到lexer和ok，如果词法分析无错误，执行`s.SetText(lexer.stmtText())`，添加解析结果。

---

然后来分析本次proj需要做的部分：
```
JoinTable:
	/* Use %prec to evaluate production TableRef before cross join */
	TableRef CrossOpt TableRef %prec tableRefPriority
	{
		$$ = &ast.Join{Left: $1.(ast.ResultSetNode), Right: $3.(ast.ResultSetNode), Tp: ast.CrossJoin}
	}
	/* My code start. */

	/* My code end. */
```

可以看到产生式为：
![eq2.1.3](http://latex.codecogs.com/svg.latex?JoinTable\rightarrow\ TableRef\ CrossOpt\ TableRef)

其后部分用于显示指定此规则的优先级：
![eq2.1.4](http://latex.codecogs.com/svg.latex?\\%prec\ tableRefPriority)

因此可以看出，需要完成的工作为：补完JoinTable的产生式和相应动作。

通过parser test判断需要添加的语法：
Fail: `select * from t1 join t2 left join t3 on t2.id = t3.id`
这里的`left join`也即`left outer join`。对应产生式如下：
```
TableRef JoinType OuterOpt "JOIN" TableRef "ON" Expression
```

当然此处也可以作弊直接参阅MySQL [JOIN](https://dev.mysql.com/doc/refman/5.7/en/join.html)语法


## 2.2 实现
参考[ast](https://github.com/pingcap/tidb/tree/source-code/ast)包的实现，可以看到结构中On的条件被放
到了这一成员中`On *OnCondition`。

```go
// Join represents table join.
type Join struct {
	node
	resultSetNode

	// Left table can be TableSource or JoinNode.
	Left ResultSetNode
	// Right table can be TableSource or JoinNode or nil.
	Right ResultSetNode
	// Tp represents join type.
	Tp JoinType
	// On represents join on condition.
	On *OnCondition
	// Using represents join using clause.
	Using []*ColumnName
	// NaturalJoin represents join is natural join
	NaturalJoin bool
}
```

因此添加如下代码即可：

```
|	TableRef JoinType OuterOpt "JOIN" TableRef "ON" Expression
	{
		on := &ast.OnCondition{Expr: $7.(ast.ExprNode)}
		$$ = &ast.Join{Left: $1.(ast.ResultSetNode), Right: $5.(ast.ResultSetNode), Tp: $2.(ast.JoinType), On: on}
	}
```
## 2.3 结果
```bash
$ go test
PASS: consistent_test.go:32: testConsistentSuite.TestKeywordConsistent  0.001s
PASS: lexer_test.go:63: testLexerSuite.TestAtLeadingIdentifier  0.000s
PASS: lexer_test.go:160: testLexerSuite.TestComment     0.000s
PASS: lexer_test.go:226: testLexerSuite.TestIdentifier  0.000s
PASS: lexer_test.go:356: testLexerSuite.TestIllegal     0.000s
PASS: lexer_test.go:295: testLexerSuite.TestInt 0.000s
PASS: lexer_test.go:103: testLexerSuite.TestLiteral     0.000s
PASS: lexer_test.go:270: testLexerSuite.TestOptimizerHint       0.000s
PASS: lexer_test.go:324: testLexerSuite.TestSQLModeANSIQuotes   0.000s
PASS: lexer_test.go:38: testLexerSuite.TestSingleChar   0.000s
PASS: lexer_test.go:53: testLexerSuite.TestSingleCharOther      0.000s
PASS: lexer_test.go:257: testLexerSuite.TestSpecialComment      0.000s
PASS: lexer_test.go:29: testLexerSuite.TestTokenID      0.000s
PASS: lexer_test.go:88: testLexerSuite.TestUnderscoreCS 0.000s
PASS: lexer_test.go:182: testLexerSuite.TestscanQuotedIdent     0.000s
PASS: lexer_test.go:191: testLexerSuite.TestscanString  0.000s
PASS: parser_test.go:1384: testParserSuite.TestAnalyze  0.000s
PASS: parser_test.go:517: testParserSuite.TestBuiltin   0.002s
PASS: parser_test.go:1296: testParserSuite.TestCommentErrMsg    0.000s
PASS: parser_test.go:923: testParserSuite.TestDDL       0.002s
PASS: parser_test.go:300: testParserSuite.TestDMLStmt   0.001s
PASS: parser_test.go:1333: testParserSuite.TestEscape   0.000s
PASS: parser_test.go:1356: testParserSuite.TestExplain  0.000s
PASS: parser_test.go:470: testParserSuite.TestExpression        0.000s
PASS: parser_test.go:870: testParserSuite.TestIdentifier        0.000s
PASS: parser_test.go:1346: testParserSuite.TestInsertStatementMemoryAllocation  0.001s
PASS: parser_test.go:1405: testParserSuite.TestQuotedSystemVariables    0.000s
PASS: parser_test.go:1466: testParserSuite.TestQuotedVariableColumnName 0.000s
PASS: parser_test.go:1371: testParserSuite.TestSQLModeANSIQuotes        0.000s
PASS: parser_test.go:1316: testParserSuite.TestSQLNoCache       0.000s
PASS: parser_test.go:1306: testParserSuite.TestSQLResult        0.000s
PASS: parser_test.go:429: testParserSuite.TestSetVariable       0.000s
PASS: parser_test.go:1393: testParserSuite.TestSideEffect       0.000s
PASS: parser_test.go:42: testParserSuite.TestSimple     0.003s
PASS: parser_test.go:230: testParserSuite.TestSpecialComments   0.000s
PASS: parser_test.go:1257: testParserSuite.TestType     0.001s
OK: 36 passed
PASS
ok      github.com/pingcap/tidb/parser  0.864s
```

## Reference
[1] [SQL解析系列(golang)--goyacc实战](https://zhuanlan.zhihu.com/p/264367718)

[2] [[原创]Flex和Bison中巧用单双引号提升语法文件的可读性](https://blog.csdn.net/u014038143/article/details/78202271)