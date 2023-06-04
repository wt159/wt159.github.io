---
title: 代码格式化工具clang-format的安装和使用
tags: tools
---

## 简介

clang-format是一个由LLVM项目提供的代码格式化工具，可以自动化地格式化C、C++、Objective-C、Java、JavaScript和Protobuf等语言的代码。它可以根据自定义的风格规则，将代码自动格式化成统一的风格，提高代码的可读性和可维护性。

## 安装

### Linux

在Linux系统中，我们可以使用包管理工具来安装clang-format。

在Debian/Ubuntu系统中，可以使用以下命令安装：

```bash
sudo apt-get update
sudo apt-get install clang-format
```

在Fedora系统中，可以使用以下命令安装：

```bash
sudo dnf install clang-tools-extra
```

### macOS

在macOS系统中，我们可以使用Homebrew来安装clang-format。

首先，需要安装Homebrew，可以使用以下命令安装：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

然后，可以使用以下命令安装clang-format：

```bash
brew install clang-format
```

### Windows

在Windows系统中，可以从LLVM官网下载预编译的clang-format二进制文件，然后将其添加到系统环境变量中。

下载地址：[https://releases.llvm.org/download.html](https://releases.llvm.org/download.html)

## 使用

使用clang-format非常简单，只需要在终端中运行以下命令：

```bash
clang-format -style=<style> <filename>
```

其中，`<style>`表示代码格式化的风格，可以选择预定义的风格，也可以自定义风格；`<filename>`表示需要格式化的文件名。

例如，我们可以使用以下命令来格式化一个C++源文件：

```bash
clang-format -style=Google test.cpp
```

这里使用的是Google风格，可以根据需要选择其他风格。

如果需要格式化一个目录下的所有文件，可以使用以下命令：

```bash
find . -iname "*.cpp" -o -iname "*.h" | xargs clang-format -style=Google -i
```

这里使用的是Google风格，`-i`选项表示直接修改文件，不生成新的文件。

## 常用风格

```bash
clang-format --help
```

```bash
--style=\<string\>               - Coding style, currently supports:
                                     LLVM, GNU, Google, Chromium, Microsoft, Mozilla, WebKit.
```

## 自定义风格

如果需要自定义代码格式化风格，可以使用clang-format提供的配置文件。配置文件可以指定各种格式化选项，如缩进、换行、空格等。

配置文件的格式为JSON或YAML，可以根据需要选择一种格式。

例如，以下是一个简单的配置文件，将缩进设置为4个空格：

```json
{
  "IndentWidth": 4
}
```

配置文件的名称为`.clang-format`，可以放置在代码仓库的根目录或者子目录中。当运行clang-format时，它会在当前目录和父目录中查找配置文件，如果找到则使用该文件中的配置项。

### 生成Webkit风格的.clang-format文件

```bash
clang-format -style=Webkit -dump-config > .clang-format
```

## 结论

使用clang-format可以大大提高代码的可读性和可维护性，同时也可以减少代码审查的时间和工作量。在实际开发中，建议使用clang-format来规范代码格式。

## 自用模版

```json
---
Language:        Cpp
# BasedOnStyle:  WebKit
AccessModifierOffset: -4
AlignAfterOpenBracket: DontAlign
AlignArrayOfStructures: None
AlignConsecutiveAssignments:
  Enabled:         false
  AcrossEmptyLines: false
  AcrossComments:  false
  AlignCompound:   false
  PadOperators:    true
AlignConsecutiveBitFields:
  Enabled:         false
  AcrossEmptyLines: false
  AcrossComments:  false
  AlignCompound:   false
  PadOperators:    false
AlignConsecutiveDeclarations:
  Enabled:         false
  AcrossEmptyLines: false
  AcrossComments:  false
  AlignCompound:   false
  PadOperators:    false
AlignConsecutiveMacros:
  Enabled:         true
  AcrossEmptyLines: false
  AcrossComments:  false
  AlignCompound:   false
  PadOperators:    false
AlignEscapedNewlines: Right
AlignOperands:   DontAlign
AlignTrailingComments: false
AllowAllArgumentsOnNextLine: true
AllowAllParametersOfDeclarationOnNextLine: true
AllowShortEnumsOnASingleLine: true
AllowShortBlocksOnASingleLine: Empty
AllowShortCaseLabelsOnASingleLine: false
AllowShortFunctionsOnASingleLine: Inline
AllowShortLambdasOnASingleLine: All
AllowShortIfStatementsOnASingleLine: Never
AllowShortLoopsOnASingleLine: false
AlwaysBreakAfterDefinitionReturnType: None
AlwaysBreakAfterReturnType: None
AlwaysBreakBeforeMultilineStrings: false
AlwaysBreakTemplateDeclarations: MultiLine
AttributeMacros:
  - __capability
BinPackArguments: true
BinPackParameters: true
BraceWrapping:
  AfterCaseLabel:  false
  AfterClass:      false
  AfterControlStatement: Never
  AfterEnum:       false
  AfterFunction:   true
  AfterNamespace:  false
  AfterObjCDeclaration: false
  AfterStruct:     false
  AfterUnion:      false
  AfterExternBlock: false
  BeforeCatch:     false
  BeforeElse:      false
  BeforeLambdaBody: false
  BeforeWhile:     false
  IndentBraces:    false
  SplitEmptyFunction: true
  SplitEmptyRecord: true
  SplitEmptyNamespace: true
BreakBeforeBinaryOperators: All
BreakBeforeConceptDeclarations: Always
BreakBeforeBraces: WebKit
BreakBeforeInheritanceComma: false
BreakInheritanceList: BeforeColon
BreakBeforeTernaryOperators: true
BreakConstructorInitializersBeforeComma: false
BreakConstructorInitializers: BeforeComma
BreakAfterJavaFieldAnnotations: false
BreakStringLiterals: true
ColumnLimit:     100
CommentPragmas:  '^ IWYU pragma:'
QualifierAlignment: Leave
CompactNamespaces: false
ConstructorInitializerIndentWidth: 4
ContinuationIndentWidth: 4
Cpp11BracedListStyle: false
DeriveLineEnding: true
DerivePointerAlignment: false
DisableFormat:   false
EmptyLineAfterAccessModifier: Never
EmptyLineBeforeAccessModifier: LogicalBlock
ExperimentalAutoDetectBinPacking: false
PackConstructorInitializers: BinPack
BasedOnStyle:    ''
ConstructorInitializerAllOnOneLineOrOnePerLine: false
AllowAllConstructorInitializersOnNextLine: true
FixNamespaceComments: false
ForEachMacros:
  - foreach
  - Q_FOREACH
  - BOOST_FOREACH
IfMacros:
  - KJ_IF_MAYBE
IncludeBlocks:   Preserve
IncludeCategories:
  - Regex:           '^"(llvm|llvm-c|clang|clang-c)/'
    Priority:        2
    SortPriority:    0
    CaseSensitive:   false
  - Regex:           '^(<|"(gtest|gmock|isl|json)/)'
    Priority:        3
    SortPriority:    0
    CaseSensitive:   false
  - Regex:           '.*'
    Priority:        1
    SortPriority:    0
    CaseSensitive:   false
IncludeIsMainRegex: '(Test)?$'
IncludeIsMainSourceRegex: ''
IndentAccessModifiers: false
IndentCaseLabels: false
IndentCaseBlocks: false
IndentGotoLabels: true
IndentPPDirectives: None
IndentExternBlock: AfterExternBlock
IndentRequiresClause: true
IndentWidth:     4
IndentWrappedFunctionNames: false
InsertBraces:    false
InsertTrailingCommas: None
JavaScriptQuotes: Leave
JavaScriptWrapImports: true
KeepEmptyLinesAtTheStartOfBlocks: true
LambdaBodyIndentation: Signature
MacroBlockBegin: ''
MacroBlockEnd:   ''
MaxEmptyLinesToKeep: 1
NamespaceIndentation: Inner
ObjCBinPackProtocolList: Auto
ObjCBlockIndentWidth: 4
ObjCBreakBeforeNestedBlockParam: true
ObjCSpaceAfterProperty: true
ObjCSpaceBeforeProtocolList: true
PenaltyBreakAssignment: 2
PenaltyBreakBeforeFirstCallParameter: 19
PenaltyBreakComment: 300
PenaltyBreakFirstLessLess: 120
PenaltyBreakOpenParenthesis: 0
PenaltyBreakString: 1000
PenaltyBreakTemplateDeclaration: 10
PenaltyExcessCharacter: 1000000
PenaltyReturnTypeOnItsOwnLine: 60
PenaltyIndentedWhitespace: 0
PointerAlignment: Right
PPIndentWidth:   -1
ReferenceAlignment: Pointer
ReflowComments:  true
RemoveBracesLLVM: false
RequiresClausePosition: OwnLine
SeparateDefinitionBlocks: Leave
ShortNamespaceLines: 1
SortIncludes:    CaseSensitive
SortJavaStaticImport: Before
SortUsingDeclarations: true
SpaceAfterCStyleCast: false
SpaceAfterLogicalNot: false
SpaceAfterTemplateKeyword: true
SpaceBeforeAssignmentOperators: true
SpaceBeforeCaseColon: false
SpaceBeforeCpp11BracedList: true
SpaceBeforeCtorInitializerColon: true
SpaceBeforeInheritanceColon: true
SpaceBeforeParens: ControlStatements
SpaceBeforeParensOptions:
  AfterControlStatements: true
  AfterForeachMacros: true
  AfterFunctionDefinitionName: false
  AfterFunctionDeclarationName: false
  AfterIfMacros:   true
  AfterOverloadedOperator: false
  AfterRequiresInClause: false
  AfterRequiresInExpression: false
  BeforeNonEmptyParentheses: false
SpaceAroundPointerQualifiers: Default
SpaceBeforeRangeBasedForLoopColon: true
SpaceInEmptyBlock: true
SpaceInEmptyParentheses: false
SpacesBeforeTrailingComments: 1
SpacesInAngles:  Never
SpacesInConditionalStatement: false
SpacesInContainerLiterals: true
SpacesInCStyleCastParentheses: false
SpacesInLineCommentPrefix:
  Minimum:         1
  Maximum:         -1
SpacesInParentheses: false
SpacesInSquareBrackets: false
SpaceBeforeSquareBrackets: false
BitFieldColonSpacing: Both
Standard:        Latest
StatementAttributeLikeMacros:
  - Q_EMIT
StatementMacros:
  - Q_UNUSED
  - QT_REQUIRE_VERSION
TabWidth:        8
UseCRLF:         false
UseTab:          Never
WhitespaceSensitiveMacros:
  - STRINGIZE
  - PP_STRINGIZE
  - BOOST_PP_STRINGIZE
  - NS_SWIFT_NAME
  - CF_SWIFT_NAME
...
```
