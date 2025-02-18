---
title: Alhagi实现记录
date: 2024-11-17 21:35:19
updated: 2024-11-17 21:35:19
tags:
  - 编辑器
  - markdown
categories:
  - notes
---

# 参考资料

https://spec.commonmark.org/0.31.2/

# CommonMark解析策略

1. 先将文档分解为块结构，但不解析文本，链接引用定义可以先收集起来
2. 将标题和段落中的纯文本解析为行内元素，此时可使用之前收集好的链接引用定义

## 强调和链接的解析策略

使用分隔符栈来解析，每个栈元素需要包含如下信息：

- 分隔符类型`[`, `![`, `*`, `_`
- 分隔符数量？
- 分隔符是否激活状态
- 分隔符是否为潜在的开启符号，或者潜在的关闭符号，或者都是

当碰到符号`]`的时候，调用`look for link or image`

当碰到输入结束时，调用强调语法处理流程，此时直到碰到栈底元素为`NULL`

#### look for link or image 

 从分隔符栈顶倒找`[`或者`![`

- 如果没找到，则返回纯文本的`]`
- 如果找到一个，但没激活，则从栈里移除那个没激活的分隔符，接着返回纯文本的`]`
- 如果找到一个并且是激活的，则查看是否为行内/引用/压缩/short cut的链接或者图像
  - 如果不是，则从栈中移除开始标签，并且返回纯文本的`]`
  - 如果是，则
    - 返回链接或者图像节点，添加到开启符号的文本节点后
    - 在当前行内元素中执行`process emphasis`，将`[`作为栈底（碰到`[`则结束）
    - 移除栈中的`[`
    - 如果当前行内元素是链接而不是图片，则将前面所有的`[`符号都设为非激活状态，防止后面解析到嵌套的链接

### process emphasis

该流程需要一个参数`stack_bottom`，来确认可以抵达的元素，NULL代表可以到栈底，否则，在碰到stack_bottom中的元素之前停止查找。

将当前指针current_position指向stack_bottom上面的一个元素

为每种分隔符类型（`_*`）都设置一个openers_bottom，初始化为stack_bottom,根据关闭分隔符运行的长度（模3）和关闭分隔符是否同时为开启分隔符来索引，用于记录每种分隔符类型搜索开启分隔符的下限。

接着循环如下步骤直到没有潜在的关闭结构

- 将当前指针向前移动（栈底向栈顶），直到找到第一个潜在的关闭结构（有可能当前符号就是）
- 向栈底移动（这里应该不是动当前指针），注意要在stack_bottom和当前分隔符类型的openers_bottom之上，找到第一个匹配的（相同的分隔符以及数量）开启结构
  - 如果找到
    - 判断是强调还是强烈强调，如果开启结构和结束结构的长度都大于等于2，则为强烈强调，否则为强调（强调优先级低）
    - 在开启节点的文本节点后插入对应的强调节点
    - 移除开启结构和结束结构中间的所有分隔符
    - 从开启结构和结束结构中移除一个或者两个分隔符（根据当前判断为强调还是强烈强调），如果该结构变空，则从栈中移除，同时设置当前指针为下一个元素
  - 如果没有找到
    - 设置当前类型分隔符的openers_bottom到当前指针之前，因为当前指针之前没有当前类型的开启结构了
    - 如果说当前指针所指的关闭结构并不同时是一个开启结构，则从栈中移除当前结构，因为它也已经当不成关闭结构了
    - 设置当前指针为下一个元素

# 源码阅读记录

```js
// 反斜杠
var C_BACKSLASH = 92;
// 依次为十六进制的|十进制的|实体名格式的HTML实体匹配
var ENTITY = "&(?:#x[a-f0-9]{1,6}|#[0-9]{1,7}|[a-z][a-z0-9]{1,31});";
// HTML标签名
var TAGNAME = "[A-Za-z][A-Za-z0-9-]*";
// HTML属性名
var ATTRIBUTENAME = "[a-zA-Z_:][a-zA-Z0-9:._-]*";
// 非引号字符
var UNQUOTEDVALUE = "[^\"'=<>`\\x00-\\x20]+";
// 被单引号括住的值
var SINGLEQUOTEDVALUE = "'[^']*'";
// 被双引号括住的值
var DOUBLEQUOTEDVALUE = '"[^"]*"';
// 属性值的组成
var ATTRIBUTEVALUE =
    "(?:" +
    UNQUOTEDVALUE +
    "|" +
    SINGLEQUOTEDVALUE +
    "|" +
    DOUBLEQUOTEDVALUE +
    ")";
// 属性值定义
var ATTRIBUTEVALUESPEC = "(?:" + "\\s*=" + "\\s*" + ATTRIBUTEVALUE + ")";
// 属性定义
var ATTRIBUTE = "(?:" + "\\s+" + ATTRIBUTENAME + ATTRIBUTEVALUESPEC + "?)";
// 开始标签
var OPENTAG = "<" + TAGNAME + ATTRIBUTE + "*" + "\\s*/?>";
// 结束标签
var CLOSETAG = "</" + TAGNAME + "\\s*[>]";
// HTML注释
var HTMLCOMMENT = "<!-->|<!--->|<!--[\\s\\S]*?-->"
// PI
var PROCESSINGINSTRUCTION = "[<][?][\\s\\S]*?[?][>]";
// 声明
var DECLARATION = "<![A-Za-z]+" + "[^>]*>";
// CDATA
var CDATA = "<!\\[CDATA\\[[\\s\\S]*?\\]\\]>";
// HTML标签
var HTMLTAG =
    "(?:" +
    OPENTAG +
    "|" +
    CLOSETAG +
    "|" +
    HTMLCOMMENT +
    "|" +
    PROCESSINGINSTRUCTION +
    "|" +
    DECLARATION +
    "|" +
    CDATA +
    ")";
// HTML开头的行
var reHtmlTag = new RegExp("^" + HTMLTAG);
// 反斜杠或者正则表达式
var reBackslashOrAmp = /[\\&]/;
// 允许被转义的字符
var ESCAPABLE = "[!\"#$%&'()*+,./:;<=>?@[\\\\\\]^_`{|}~-]";
// 转义字符或者HTML实体，gi为flag，代表全局+不区分大小写
var reEntityOrEscapedChar = new RegExp("\\\\" + ESCAPABLE + "|" + ENTITY, "gi");
// xml中的特殊字符（需要转义）
var XMLSPECIAL = '[&<>"]';
// 搜索全局的XML特殊字符
var reXmlSpecial = new RegExp(XMLSPECIAL, "g");
```



