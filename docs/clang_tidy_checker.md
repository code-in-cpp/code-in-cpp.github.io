---
title: "使用Clang Tidy开发自定义的检查器"
permalink: /clang_tidy_checker/
layout: default
---


# 准备

clone代码

```shell
git clone https://github.com/llvm/llvm-project.git
```

安装cmake和ninja（make也可以）

```shell
> $ cmake --version                                                      
cmake version 3.16.2

CMake suite maintained and supported by Kitware (kitware.com/cmake).

> $ ninja --version
1.10.1
```

# 开始编译

```shell
cd [llvm-project的目录]
mkdir build
cd build
```

使用Ninja编译可以运行命令

```shell
cmake ../llvm -DLLVM_ENABLE_PROJECTS='clang;clang-tools-extra' -GNinja
```

# 运行下面的命令可以运行clang-tidy的单元测试检查所有的checker是否生效

```
ninja check-clang-tools
```

Clang Tidy（clang12版本）已经支持很多条检查规则，参考[clang 12 已经支持的规则](https://clang.llvm.org/docs/analyzer/checkers.html)。但在实际工程中，我们还是需要添加自己项目特定的检查规则。

# 添加第一条自定义的checker

我们先来添加一条简单的规则，要求所有的catch语句，只能catch常量引用异常。即如下面代码所示：

```cpp
try {
    check();
} catch (const logic_error& e) {
}
```

如果是如下的代码，则不符合代码规范

```cpp
try {
    check();
} catch (logic_error& e) {

}
```

clang-tidy使用clang AST来进行检查，可以理解所有的checker都是clang AST的visitor

如下图所示

```shell
$ cat test.cc
int f(int x) {
  int result = (x / 42);
  return result;
}

# Clang by default is a frontend for many tools; -Xclang is used to pass
# options directly to the C++ frontend.
$ clang -Xclang -ast-dump -fsyntax-only test.cc
TranslationUnitDecl 0x5aea0d0 <<invalid sloc>>
... cutting out internal declarations of clang ...
`-FunctionDecl 0x5aeab50 <test.cc:1:1, line:4:1> f 'int (int)'
  |-ParmVarDecl 0x5aeaa90 <line:1:7, col:11> x 'int'
  `-CompoundStmt 0x5aead88 <col:14, line:4:1>
    |-DeclStmt 0x5aead10 <line:2:3, col:24>
    | `-VarDecl 0x5aeac10 <col:3, col:23> result 'int'
    |   `-ParenExpr 0x5aeacf0 <col:16, col:23> 'int'
    |     `-BinaryOperator 0x5aeacc8 <col:17, col:21> 'int' '/'
    |       |-ImplicitCastExpr 0x5aeacb0 <col:17> 'int' <LValueToRValue>
    |       | `-DeclRefExpr 0x5aeac68 <col:17> 'int' lvalue ParmVar 0x5aeaa90 'x' 'int'
    |       `-IntegerLiteral 0x5aeac90 <col:21> 'int' 42
    `-ReturnStmt 0x5aead68 <line:3:3, col:10>
      `-ImplicitCastExpr 0x5aead50 <col:10> 'int' <LValueToRValue>
        `-DeclRefExpr 0x5aead28 <col:10> 'int' lvalue Var 0x5aeac10 'result' 'int'
```

看看我们


```shell
$ cat test.cc
int f();
struct logic_error {
};

int main() {
try {
    f();
} catch (const logic_error& e) {
}

}
$ clang -Xclang -ast-dump -fsyntax-only test.cc
TranslationUnitDecl 0x7fc2cc024008 <<invalid sloc>> <invalid sloc>
...
`-FunctionDecl 0x7fc2cc05d828 <line:5:1, line:11:1> line:5:5 main 'int ()'
  `-CompoundStmt 0x7fc2cc05dad0 <col:12, line:11:1>
    `-CXXTryStmt 0x7fc2cc05dab0 <line:6:1, line:9:1>
      |-CompoundStmt 0x7fc2cc05d9c0 <line:6:5, line:8:1>
      | `-CallExpr 0x7fc2cc05d9a0 <line:7:5, col:7> 'int'
      |   `-ImplicitCastExpr 0x7fc2cc05d988 <col:5> 'int (*)()' <FunctionToPointerDecay>
      |     `-DeclRefExpr 0x7fc2cc05d940 <col:5> 'int ()' lvalue Function 0x7fc2cc05d550 'f' 'int ()'
      `-CXXCatchStmt 0x7fc2cc05da90 <line:8:3, line:9:1>
        |-VarDecl 0x7fc2cc05da18 <line:8:10, col:29> col:29 e 'const logic_error &'
        `-CompoundStmt 0x7fc2cc05da80 <col:32, line:9:1>
```

首先添加自定义checker

```shell
$ cd [llvm project path]/clang-tools-extra/clang-tidy
$ ./add_new_check.py misc catch-by-const-ref
```

可以看到在llvm的工程的变化

```shell
	modified:   clang-tools-extra/clang-tidy/misc/CMakeLists.txt
	modified:   clang-tools-extra/clang-tidy/misc/MiscTidyModule.cpp
	modified:   clang-tools-extra/docs/clang-tidy/checks/list.rst

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	clang-tools-extra/clang-tidy/misc/CatchByConstRefCheck.cpp
	clang-tools-extra/clang-tidy/misc/CatchByConstRefCheck.h
	clang-tools-extra/docs/clang-tidy/checks/misc-catch-by-const-ref.rst	clang-tools-extra/test/clang-tidy/checkers/misc-catch-by-const-ref.cpp
```

其中 `CatchByConstRefCheck.cpp`和`CatchByConstRefCheck.h`是新添加checker的主要文件

修改`CatchByConstRefCheck.cpp`的内容

matcher是去匹配clang AST

```cpp
void CatchByConstRefCheck::registerMatchers(MatchFinder *Finder) {
  // This is a C++ only check thus we register the matchers only for C++

  if(!getLangOpts().CPlusPlus) {
    return;
  }

  Finder->addMatcher(varDecl(isExceptionVariable(),      // 属于catch变量
                             hasType(references(         // 参数是引用类型
                                 qualType(
                                     unless(             // 没有
                                         isConstQualified()  // const修饰
                                         )
                                     )
                                 )
                               )
                             ).bind("catch"),
                     this);
}
```

check给出诊断和修复
```cpp
void CatchByConstRefCheck::check(const MatchFinder::MatchResult &Result) {
  const auto* varCatch = Result.Nodes.getNodeAs<VarDecl>("catch");
  static const char* diagMsgCatchReference =
      "catch handler catches by non const reference; "
      "catching by const-reference may be more efficient";

  // Emit error message if the type is not const (ref)s

  diag(varCatch->getBeginLoc(), diagMsgCatchReference)
      << FixItHint::CreateInsertion(varCatch->getBeginLoc(), "const ");  //在前面添加const修复代码
}
```

# 修改test文件

把`clang-tools-extra/test/clang-tidy/checkers/misc-catch-by-const-ref.cpp`下的文件修改为如下内容

```cpp

// RUN: %check_clang_tidy %s misc-catch-by-const-ref %t


class logic_error {
public:
  logic_error(const char *message) {}
};

void check()
{}

void catchbyRef() {
  try {
    check();
  } catch (logic_error& e) {
    // CHECK-MESSAGES: :[[@LINE-1]]:12: warning: catch handler catches by non const reference; catching by const-reference may be more efficient [misc-catch-by-const-ref]
    // CHECK-FIXES: {{^}}  } catch (const logic_error& e) {{{$}}
  }
}

void catchbyConstRef() {
  try {
    check();
  } catch (const logic_error& e) {
  }
}
```



