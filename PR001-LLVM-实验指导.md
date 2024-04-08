# 编译原理研讨课LLVM实验PR001实验指导

## 熟悉Clang的安装和使用

### 编译安装LLVM和Clang：

**第一步**：登录到本组服务器的帐号，参见实验环境说明。

每个组有两个账号密码组合：

1. 服务器登录：`llvm01:llvm01!llvm01!`，其中`01`为组号；
2. Gitlab账号：`llvm01:llvm01!llvm01!`，其中`01`为组号；

**第二步**：将源代码从Gitlab服务器clone到本地：

配置自己的git偏好设置:

```shell
git config --global user.name "llvm01 1"
git config --global user.email "llvm01xxx@xxxx.com"
git config --global core.editor vim
# vim or emacs, up to you
```

拷贝代码，拷贝teacher的gitlab仓库：

```shell
git clone http://124.16.71.62:8000/teacher/llvm.git
```

输入Gitlab账号和密码，形如：`llvm01:llvm01!llvm01!`

**第三步**：编译和安装LLVM和Clang

1. 建立install文件夹：`mkdir ~/llvm-install`

2. 建立构建目录(Out of Source)：`mkdir -p build && cd build`

3. 通过CMake生成GNU标准的Makefile，并使用make进行编译和安装:

​	3.1 设定编译过程使用的*gcc*和*g++*: `export CC=/usr/bin/gcc && export CXX=/usr/bin/g++`

​	3.2 生成Makefile：

​	`cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=~/llvm-install ../llvm`，其中	*CMAKE_INSTALL_PREFIX*代表了编译完成以后的安装目录，*../llvm*代表了源代码目录

​	3.3 使用*make*进行编译：`make -j15`，其中*j15*代表使用15个线程并行编译。该过程约需要数分钟。如果仅仅是更改了*Clang*的源代码 （也就是大家做作业的过程当中），则无需执行第二步。

​	3.4 安装到`CMAKE_INSTALL_PREFIX`当中：`make install`

### 生成和查看C程序对应的AST

**第一步**，准备一个简单的C程序`test.c`：

```c
int f(int x) {
  int result = (x / 42);
  return result;
}
```

**第二步**，使用`clang`将AST给dump出来：`~/llvm-install/bin/clang -Xclang -ast-dump -fsyntax-only test.c`

### 使用GDB调试Clang

由于我们是使用GNU的*gcc*和*g++*编译生成的*Clang*，这意味着需要使用gdb来对*Clang*进行跟踪调试。当然，如果你使用的是LLVM编译生成*Clang*，对应的调试工具将是*lldb*。

调试*Clang*的典型流程：

1. 打开gdb：`gdb`
2. 打开要调试的*clang*可执行文件，通过*file*命令: `file ~/llvm-install/bin/clang`，将会有`Reading symbols from /home/clang1/llvm-install/bin/clang...done`字样的输出。
3. 设定调试子进程，因为*Clang*会派生一个新进程来执行编译流程，命令：`set follow-fork child`
4. 在处理Pragma的入口函数处打断点：`b clang::PragmaNamespace::HandlePragma`
5. 执行编译：`r example.c`，其中*example.c*是传递给*Clang*的参数
6. 进行正常调试

常见命令介绍：

`r args`，运行已经加载的应用，*args*是传递给应用的参数。

`l`，列出代码

`b` , 在指定位置打断点

`c`，继续执行程序

### 为llvm仓库添加本组的远程gitlab仓库

```shell
git remote rm origin
git remote add origin http://124.16.71.62:8000/llvm01/llvm.git
```

### VSCode连接remote使用

为了方便大家对代码进行修改，可以使用vscode远程连接服务器进行文件的修改。

1. 在大家自己的笔记本上安装VSCode

2. 在VSCode中搜索安装“Remote Explorer”的扩展

3. 点击VSCode左侧的“Remote Explorer”图标，点击“Open SSH Config”打开config文件

4. 在config文件中输入服务器的 Host（compiler-server，自己可以随意命名），HostName（服务器IP地址，需要根据自己的修改），User（用户名，需要根据自己的修改）

   ```shell
   Host compiler-server
     HostName 124.16.71.63
     User llvm22
   ```

5. 点开左侧Remote的“更新”图标，展开“SSH”，会出现”compiler-server“的远程配置，选择”Connect in Current Window“或者”Connect in New Window“。

通过以上方式直接使用VSCode连接远程服务器使用

为了方便VSCode连接，还可以进行设置**免密登录**：将本机的.ssh/id_isa.pub中的文件内容拷贝到服务器的~/.ssh/authorized_keys中。

### VSCode中的gdb使用

1. Run -> Add Configuration -> C/C++: (gdb) launch 将在项目目录下产生一个.vscode/launch.json 文件

2. 将 "program" 改为 clang 的路径（最好使用绝对路径），"args" 可以改为待编译的测试源代码路径

3. 在"setupCommands" 中加入

```
{
"description": "Fork child process",
"text": "-gdb-set follow-fork child"
}
```

4. 然后可以正常调试。

### 添加 TensorSliceExpr的 Clang 前端支持

LLVM-PR001 实验要求为Tensor的Slice语法添加支持AST支持，使得修改后的前端能够parse 形如此类的程序

```shell
Tensor A = [1,2,3,4]
Tensor B = [5,6,7,8]
Tensor C = A[1:3] // C = [2,3]
```

parse结果如下所示：

```
TranslationUnitDecl 0x5a25108 <<invalid sloc>> <invalid sloc>
|-TypedefDecl 0x5a259a0 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x5a256a0 '__int128'
|-TypedefDecl 0x5a25a08 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x5a256c0 'unsigned __int128'
|-TypedefDecl 0x5a25cc8 <<invalid sloc>> <invalid sloc> implicit __NSConstantString 'struct __NSConstantString_tag'
| `-RecordType 0x5a25ae0 'struct __NSConstantString_tag'
|   `-Record 0x5a25a58 '__NSConstantString_tag'
|-TypedefDecl 0x5a25d60 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x5a25d20 'char *'
|   `-BuiltinType 0x5a251a0 'char'
|-TypedefDecl 0x5a5f4b8 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list 'struct __va_list_tag [1]'
| `-ConstantArrayType 0x5a5f460 'struct __va_list_tag [1]' 1 
|   `-RecordType 0x5a5f2e0 'struct __va_list_tag'
|     `-Record 0x5a5f250 '__va_list_tag'
|-RecordDecl 0x5a5f508 <test1.c:1:9, col:16> col:16 struct Tensor
|-TypedefDecl 0x5a5f608 <col:1, col:23> col:23 referenced Tensor 'struct Tensor':'struct Tensor'
| `-ElaboratedType 0x5a5f5b0 'struct Tensor' sugar
|   `-RecordType 0x5a5f590 'struct Tensor'
|     `-Record 0x5a5f670 'Tensor'
|-RecordDecl 0x5a5f670 prev 0x5a5f508 <line:2:1, line:5:1> line:2:8 struct Tensor definition
| `-FieldDecl 0x5a5f710 <line:4:2, col:6> col:6 member 'int'
`-FunctionDecl 0x5a5f998 <line:6:1, line:10:1> line:6:5 main 'int (int, char **)'
  |-ParmVarDecl 0x5a5f770 <col:10, col:14> col:14 argc 'int'
  |-ParmVarDecl 0x5a5f880 <col:20, col:31> col:26 argv 'char **':'char **'
  `-CompoundStmt 0x5a5ff60 <col:34, line:10:1>
    |-DeclStmt 0x5a5fc48 <line:7:1, col:21>
    | `-VarDecl 0x5a5fac0 <col:1, col:20> col:8 used A 'Tensor':'struct Tensor' cinit
    |   `-InitListExpr 0x5a5fc00 <col:12, col:20> 'Tensor':'struct Tensor'
    |     `-IntegerLiteral 0x5a5fb20 <col:13> 'int' 1
    |-DeclStmt 0x5a5fdf8 <line:8:1, col:21>
    | `-VarDecl 0x5a5fc70 <col:1, col:20> col:8 B 'Tensor':'struct Tensor' cinit
    |   `-InitListExpr 0x5a5fdb0 <col:12, col:20> 'Tensor':'struct Tensor'
    |     `-IntegerLiteral 0x5a5fcd0 <col:13> 'int' 5
    `-DeclStmt 0x5a5ff48 <line:9:1, col:18>
      `-VarDecl 0x5a5fe20 <col:1, col:17> col:8 C 'Tensor':'struct Tensor' cinit
        `-ImplicitCastExpr 0x5a5ff30 <col:12, col:17> 'Tensor':'struct Tensor' <LValueToRValue>
          `-TensorSliceExpr 0x5a5fee8 <col:12, col:17> 'Tensor':'struct Tensor' lvalue
            |-DeclRefExpr 0x5a5fe80 <col:12> 'Tensor':'struct Tensor' lvalue Var 0x5a5fac0 'A' 'Tensor':'struct Tensor'
            |-IntegerLiteral 0x5a5fea8 <col:14> 'int' 1
            |-<<<NULL>>>
            `-IntegerLiteral 0x5a5fec8 <col:16> 'int' 3
```



### TensorSliceExpr的语法

```shell
[ index ]
[ lower-bound : ]
[ : ]
[ :: step ]
[ lower-bound :: step ]
[ :: ]
[ lower-bound ::]
[ lower-bound : upper-bound ]
[ : upper-bound ]
[ lower-bound : upper-bound : ]
[ : upper-bound : ]
[ lower-bound : upper-bound : step ]
[ : upper-bound : step ]
```





### 实验设计

####   1. 增加C99Tensor的编译选项

```
tools/clang/include/clang/Frontend/LangStandards.def
```

  在这个文件中，仿照    LANGSTANDARD_ALIAS_DEPR(gnu99, "gnu9x")    增加c99tensorc的支持， 这样在使用我们编译版本的clang时，可以通过 -std=c99-tensorc指定使用TensorC的标准编译。

####   2.增加TensorSliceExpr.h节点

  标准C语言不支持Slice的语法，在这一步我们需要增加对SliceExpr的支持，为降低第一次作业的难度，该AST节点的实现已经给出，但需要学生在课程报告中给出对该节点源码的解释。

​	代码位于：

```
tools/clang/include/clang/AST/ExprTensorC.h
```

####   3.增加LangStandard支持

```
tools/clang/include/clang/Frontend/LangStandards.def
```

  在LangFeatures 枚举类型中仿照OpenCL、ImplicitInt等增加C99TensorC

  实现 isC99TensorC函数，可以仿照isCPlusPlus2a函数实现

####  4.为TensorSlice实现parser

```
tools/clang/lib/Parse/ParseExpr.cpp
```

在ParseExpr.cpp文件，函数 ExprResult Parser::ParsePostfixExpressionSuffix(ExprResult LHS) 中，增加对TensorSliceExpr的parser支持。



完成上述四点后，使用自己编译的clang可以dump出TensorSliceExpr的AST节点：

```
`-TensorSliceExpr 0x5a5fee8 <col:12, col:17> 'Tensor':'struct Tensor' lvalue
```











