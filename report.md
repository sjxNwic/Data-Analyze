## 设计思路

### 基础部分

#### Lexer.h

###### 问题重述

1. 能够识别“return”、“def”和“var”三个关键字
2. 按照如下要求识别变量名：
    • 变量名以字母开头
    • 变量名由字母、数字和下划线组成
    • 变量名中有数字时，数字应该位于变量名末尾

###### 具体实现

```C
		auto loc = getLastLocation();

    if(isalpha(lastChar)) {
      std::string str;
      bool num_appeared = 0;
      do {
        str += lastChar;
        lastChar = Token(getNextChar());

        if (num_appeared) { 
				if(!isdigit(lastChar)){
						llvm::errs() << "Parse error (" << loc.line << ", " << loc.col << "): number should be at the end of the name\n";
						assert(isdigit(lastChar));
				}
		}
        else if (isdigit(lastChar)) num_appeared = 1;
      } while (isalnum(lastChar) || lastChar == '_');

      if (str == "return") return tok_return;
      if (str == "def") return tok_def;
      if (str == "var") return tok_var;

      identifierStr = str;
      return tok_identifier;
    }
```

* 通过`isalpha`判断首个字符为字母，确定其属于`var`,`def`,`return`或`identifier`
* 设置布尔变量`num_appeared`，判断数字是否出现过
* 将`last char`添加到`str`中，用函数`getNextChar`获取下一个字符
* 若数字出现过则该字符必须为数字。如果非数字则通过`llvm::errs`报错，通过`assert`宏退出执行
* 当字符为字母，数字或下划线时则继续循环
* 判断是否为三个关键字，是则返回对应`Token`，否则返回`tok_identifier`

#### Parser.h

##### TODO1

###### 问题重述

检查变量声明是否有关键字`var`，否则报错

###### 具体实现

```C
		auto loc = lexer.getLastLocation();
    
    if (lexer.getCurToken() != tok_var) {
      return parseError<VarDeclExprAST>("tok_var", "in parseVarDeclaration");
    }
    lexer.getNextToken(); // eat var
```

* 首先通过`getLastLocation`获取当前位置

* 获取当前`Token`，若非`tok_var`，则通过`parseError`报错。否则将指针前移

  

##### TODO2

###### 问题重述

检查接下来是否有变量名，否则报错

###### 具体实现

```C
if (lexer.getCurToken() != tok_identifier)
      return parseError<VarDeclExprAST>("identified", "after 'var' declaration");
```

* 检查`curToken`是否为`tok_identifier`，否则通过`parseError`报错
* 由于词法分析器中已经检查变量名是否合法，所以若得到`tok_identifier`，则一定合法

##### TODO3

###### 问题重述

修改代码使得变量声明支持第三种形式

###### 具体实现

```C
		if (lexer.getCurToken() == '<' || lexer.getCurToken() == '[') {
      type = parseType();
      if (!type)
        return nullptr;
    }
```

修改判断条件，增加对`[`的判断。若出现`[`则进入`parseType`函数

##### TODO4

###### 问题重述

增加对形如`var a[2][3] = [1,2,3,4,5,6]`的变量声明的支持

###### 具体实现

```C
	// TODO: make an extension to support the new type like var a[2][3] = [1, 2, 3, 4, 5, 6];
  std::unique_ptr<VarType> parseType() {
    /*
    if (lexer.getCurToken() != '<')
      return parseError<VarType>("<", "to begin type");
    lexer.getNextToken(); // eat <
    */

    auto type = std::make_unique<VarType>();
    
    if(lexer.getCurToken() == '['){
      do{
        lexer.consume(Token('['));
        if (lexer.getCurToken() != tok_number) return parseError<VarType>("number", "to declare type");
        type->shape.push_back(lexer.getValue());
        if (lexer.getNextToken() != ']') return parseError<VarType>("]", "to end declaration");
        lexer.getNextToken();
      } while (lexer.getCurToken() == '[');

      return type;
    }

    if (lexer.getCurToken() != '<')
      return parseError<VarType>("< or [", "to begin the declaration of the type");
    lexer.getNextToken(); // eat <

    while (lexer.getCurToken() == tok_number) {
      type->shape.push_back(lexer.getValue());
      lexer.getNextToken();
      if (lexer.getCurToken() == ',')
        lexer.getNextToken();
    }

    if (lexer.getCurToken() != '>')
      return parseError<VarType>(">", "to end type");
    lexer.getNextToken(); // eat >
    return type;
  }
```

* 首先判断是否为`[`，若是则进入循环
* 指针前移到`[`之前，判断接下来是否为数字。若为数字则加入到`type`的`shape`field中记录变量类型，否则报错
* 判断接下来是否为`]`，否则报错
* 判断条件为接下来的`Token`是否为`[`，是则继续循环
* 若开头既非`<`也非`[`，则报错

## 实验验证

##### test1

变量名非字母开头

![test_1](https://github.com/Winter-Is-Coming-Stark/Data-Analyze/blob/main/test_1.png)

##### test2-test4

变量名有数字但不在末位连续出现

![test_2](https://github.com/Winter-Is-Coming-Stark/Data-Analyze/blob/main/test_2.png)

![test_3](https://github.com/Winter-Is-Coming-Stark/Data-Analyze/blob/main/test_3.png)

![test_4](https://github.com/Winter-Is-Coming-Stark/Data-Analyze/blob/main/test_4.png)

##### test5

`d[2][3] = [1,2,3,4,5,6]`成功识别

![test_5](/Users/julia27/Documents/编译原理/tiny_project/figs/test_5.png)

##### test6

抽象语法树成功生成

![test_6 ast](/Users/julia27/Documents/编译原理/tiny_project/figs/test_6 ast.png)

成功计算三个矩阵数乘

![test_6 jit](/Users/julia27/Documents/编译原理/tiny_project/figs/test_6 jit.png)

## 心得

* basic部分的实验中，我在阅读源码的过程中对lexer和parser的架构有了直观的理解，在实现与调试的过程中对具体实现有了更深入的了解。
* 阅读代码的过程中，我对代码从下到上逐级封装，层层调用的特性有了更深入的理解，对大型项目的架构有了直观的认识。
* 在阅读代码时，对于之前不太熟悉的c++11标准中的智能指针的使用有了更多的了解。