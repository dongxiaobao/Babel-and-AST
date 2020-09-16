### Babel是如何读懂JS代码的

Babel是一个编译器，能将ES2015+的代码转换正ES2015。

#### Babel工作的三个阶段

Babel的功能非常纯粹，以字符串的形式将源代码传给它，它就会返回一段新的代码字符串（以及sourcemap）。他既不会运行你的代码，也不会将多个代码打包到一起，它就是个编译器，输入语言是ES6+，编译目标语言是ES5。



Babel的编译过程跟绝大多数其他语言的编译器大致同理，分为三个阶段：

1. **解析**：将代码字符串解析成抽象语法树(**AST**)
2. **变换**：对抽象语法树进行变换操作
3. **再建**：根据变换后的抽象语法树再生成代码字符串

像我们在.babelrc里配置的presets和plugins都是在第2步工作的。

举个例子，首先你输入的代码如下：

```js
if (1 > 0) {
    alert('hi');
}
```

经过第一步解析成抽象语法树,是长这个样子的

这里只展示比较关键的部分

```json
{
  "type": "Program",                          // 程序根节点
  "body": [                                   // 一个数组包含所有程序的顶层语句
    {
      "type": "IfStatement",                  // 一个if语句节点
      "test": {                               // if语句的判断条件
        "type": "BinaryExpression",           // 一个双元运算表达式节点
        "operator": ">",                      // 运算表达式的运算符
        "left": {                             // 运算符左侧值
          "type": "Literal",                  // 一个常量表达式
          "value": 1                          // 常量表达式的常量值
        },
        "right": {                            // 运算符右侧值
          "type": "Literal",
          "value": 0
        }
      },
      "consequent": {                         // if语句条件满足时的执行内容
        "type": "BlockStatement",             // 用{}包围的代码块
        "body": [                             // 代码块内的语句数组
          {
            "type": "ExpressionStatement",    // 一个表达式语句节点
            "expression": {
              "type": "CallExpression",       // 一个函数调用表达式节点
              "callee": {                     // 被调用者
                "type": "Identifier",         // 一个标识符表达式节点
                "name": "alert"
              },
              "arguments": [                  // 调用参数
                {
                  "type": "Literal",
                  "value": "hi"
                }
              ]
            }
          }
        ]
      },
      "alternative": null                     // if语句条件未满足时的执行内容
    }
  ]
}
```

用图像更简单地表达上面的结构：

<img src="https://github.com/dongxiaobao/Babel-and-AST/blob/master/640.png" alt="640" style="zoom:50%;" />

第1步转换的过程中可以验证语法的正确性，同时由字符串变为对象结构后更有利于精准地分析以及进行代码结构调整。



第2步原理就很简单了，就是遍历这个对象所描述的抽象语法树，遇到哪里需要做一下改变，就直接在对象上进行操作，比如我把IfStatement给改成WhileStatement就达到了把条件判断改成循环的效果。



第3步也简单，递归遍历这颗语法树，然后生成相应的代码，大概的实现逻辑如下：

```json
const types = {
  Program (node) {
    return node.body.map(child => generate(child));
  },
  IfStatement (node) {
    let code = `if (${generate(node.test)}) ${generate(node.consequent)}`;
    if (node.alternative) {
      code += `else ${generate(node.alternative)}`;
    }
    return code;
  },
  BlockStatement (node) {
    let code = node.body.map(child => generate(child));
    code = `{ ${code} }`;
    return code;
  },
  ......
};
function generate(node) {
  return types[node.type](node);
}
const ast = Babel.parse(...);            // 将代码解析成语法树
const generatedCode = generate(ast);     // 将语法树重新组合成代码
```

#### babel 工作原理

需要用到两个工具包 `@babel/core`、`@babel/preset-env`

当我们配置 babel 的时候，不管是在 `.babelrc` 或者 `babel.config.js` 文件里面配置的都有 `presets` 和 `plugins` 两个配置项（还有其他配置项，这里不做介绍）

##### 插件和预设的区别

一个预设里面包含了许多的插件

```json
// .babelrc
{
  "presets": ["@babel/preset-env"], // 预设
  "plugins": [] // 插件
}
```

当我们配置了 `presets` 中有 `@babel/preset-env`，那么 `@babel/core` 就会去找 `preset-env` 预设的插件包，它是一套

babel 核心包并不会去转换代码，核心包只提供一些核心 API，真正的代码转换工作由插件或者预设来完成，

比如要转换箭头函数，会用到这个 plugin，`@babel/plugin-transform-arrow-functions`，

当需要转换的要求增加时，我们不可能去一一配置相应的 plugin，这个时候就可以用到预设了，也就是 presets。presets 是 plugins 的集合，一个 presets 内部包含了很多 plugin。

##### babel 插件的使用

现在我们有一个箭头函数，要想把它转成普通函数，我们就可以直接这么写：

```js
const babel = require('@babel/core')
const code = `const fn = (a, b) => a + b`

// babel 有 transform 方法会帮我们自动遍历，使用相应的预设或者插件转换相应的代码
const r = babel.transform(code, {
  presets: ['@babel/preset-env'],
})
console.log(r.code)
// 打印结果如下
// "use strict";
// var fn = function fn() { return a + b; };
```



此时我们可以看到最终代码会被转成普通函数，但是我们，只需要箭头函数转通函数的功能，不需要用这么大一套包，只需要一个箭头函数转普通函数的包，我们其实是可以在 `node_modules` 下面找到有个叫做 `plugin-transform-arrow-functions` 的插件，这个插件是专门用来处理 箭头函数的，我们就可以这么写：

```js
const r = babel.transform(code, {
  plugins: ['@babel/plugin-transform-arrow-functions'],
})
console.log(r.code)
// 打印结果如下
// const fn = function () { return a + b; };
```



我们可以从打印结果发现此时并没有转换我们变量的声明方式还是 const 声明，只是转换了箭头函数



##### 编写自己的插件

现在我们来个实战把 `const fn = (a, b) => a + b` 转换为 `const fn = function(a, b) { return a + b }`



```js
const babel = require('@babel/core')
const code = `const fn = (a, b) => a + b` // 转换后 const fn = function(a, b) { return a + b }
const arrowFnPlugin = {
  // 访问者模式
  visitor: {
    // 当访问到某个路径的时候进行匹配
    ArrowFunctionExpression(path) {
      // 拿到节点
      const node = path.node
      console.log('ArrowFunctionExpression -> node', node)
    },
  },
}

const r = babel.transform(code, {
  plugins: [arrowFnPlugin],
})

console.log(r)
```

此时我们拿到的结果是这样的节点结果是 这样的，其实就是 `ArrowFunctionExpression` 的 AST，此时我们要做的是把 `ArrowFunctionExpression` 的结构替换成 `FunctionExpression`的结构，但是需要我们组装类似的结构，这么直接写很麻烦，但是 babel 为我们提供了一个工具叫做 `@babel/types`



`@babel/types` 有两个作用：

1. 判断这个节点是不是这个节点（ArrowFunctionExpression 下面的 path.node 是不是一个 ArrowFunctionExpression）
2. 生成对应的表达式



那么接下来我们就开始生成一个 `FunctionExpression`，然后把之前的 `ArrowFunctionExpression` 替换掉，我们可以看 `types` 文档，找到 `functionExpression`，该方法接受相应的参数我们传递过去即可生成一个 `FunctionExpression`



```
t.functionExpression(id, params, body, generator, async)
```

- id: Identifier (default: null) id 可传递 null
- params: Array<LVal> (required) 函数参数，可以把之前的参数拿过来
- body: BlockStatement (required) 函数体，接受一个 `BlockStatement` 我们需要生成一个
- generator: boolean (default: false) 是否为 generator 函数，当然不是了
- async: boolean (default: false) 是否为 async 函数，肯定不是了

还需要生成一个 `BlockStatement`，我们接着看文档找到 `BlockStatement` 接受的参数

```
t.blockStatement(body, directives)
```

看文档说明，blockStatement 接受一个 body，那我们把之前的 body 拿过来就可以直接用，不过这里 body 接受一个数组

我们细看 AST 结构，函数表达式中的 `BlockStatement` 中的 `body` 是一个 `ReturnStatement`，所以我们还需要生成一个 `ReturnStatement`

现在我们就可以改写 AST 了



```js
const babel = require('@babel/core')
const t = require('@babel/types')
const code = `const fn = (a, b) => a + b` // const fn = function(a, b) { return a + b }
const arrowFnPlugin = {
  // 访问者模式
  visitor: {
    // 当访问到某个路径的时候进行匹配
    ArrowFunctionExpression(path) {
      // 拿到节点然后替换节点
      const node = path.node
      console.log('ArrowFunctionExpression -> node', node)
      // 拿到函数的参数
      const params = node.params
      const body = node.body
      const functionExpression = t.functionExpression(null, params, t.blockStatement([body]))
      // 替换原来的函数
      path.replaceWith(functionExpression)
    },
  },
}
const r = babel.transform(code, {
  plugins: [arrowFnPlugin],
})
console.log(r.code) // const fn = function (a, b) { return a + b; };
```



#### 抽象语法树是如何产生的

抽象语法树（`Abstract Syntax Tree`）简称 `AST`，是源代码的抽象语法结构的树状表现形式。`webpack`、`eslint` 等很多工具库的核心都是通过抽象语法树这个概念来实现对代码的检查、分析等操作。

我们常用的浏览器就是通过将 js 代码转化为抽象语法树来进行下一步的分析等其他操作。所以将 js 转化为抽象语法树更利于程序的分析。

<img src="https://github.com/dongxiaobao/Babel-and-AST/blob/master/641.png" alt="641" style="zoom:50%;" />

如上图中变量声明语句，转换为 AST 之后就是右图中显示的样式

左图中对应的：

- `var` 是一个关键字
- `AST` 是一个定义者
- `=` 是 Equal 等号的叫法有很多形式，在后面我们还会看到
- `is tree` 是一个字符串
- `;` 就是 Semicoion

首先一段代码转换成的抽象语法树是一个对象，该对象会有一个顶级的 type 属性 `Program`；第二个属性是 `body` 是一个数组。

`body` 数组中存放的每一项都是一个对象，里面包含了所有的对于该语句的描述信息

```json
type:         描述该语句的类型  --> 变量声明的语句
kind:         变量声明的关键字  --> var
declaration:  声明内容的数组，里面每一项也是一个对象
            type: 描述该语句的类型
            id:   描述变量名称的对象
                type: 定义
                name: 变量的名字
            init: 初始化变量值的对象
                type:   类型
                value:  值 "is tree" 不带引号
                row:    "\"is tree"\" 带引号
```

##### 词法分析和语法分析

`JavaScript` 是解释型语言，一般通过 词法分析 -> 语法分析 -> 语法树，就可以开始解释执行了

**词法分析**：也叫`扫描`，是将字符流转换为记号流(`tokens`)，它会读取我们的代码然后按照一定的规则合成一个个的标识

比如说：`var a = 2` ，这段代码通常会被分解成 `var、a、=、2`

```json
;[
  { type: 'Keyword', value: 'var' },
  { type: 'Identifier', value: 'a' },
  { type: 'Punctuator', value: '=' },
  { type: 'Numeric', value: '2' },
]
```

当词法分析源代码的时候，它会一个一个字符的读取代码，所以很形象地称之为扫描 - `scans`。当它遇到空格、操作符，或者特殊符号的时候，它会认为一个话已经完成了。

**语法分析**：也称`解析器`，将词法分析出来的数组转换成树的形式，同时验证语法。语法如果有错的话，抛出语法错误。

```json

{
  ...
  "type": "VariableDeclarator",
  "id": {
    "type": "Identifier",
    "name": "a"
  },
  ...
}
```

语法分析成 AST    [官方解答文档](https://esprima.org/)

##### AST 能做什么

- 语法检查、代码风格检查、格式化代码、语法高亮、错误提示、自动补全等
- 代码混淆压缩
- 优化变更代码，改变代码结构等

比如说，有个函数 `function a() {}` 我想把它变成 `function b() {}`

比如说，在 `webpack` 中代码编译完成后 `require('a') --> __webapck__require__("*/**/a.js")`

##### AST 解析流程

准备工具：

- esprima：code => ast 代码转 ast
- estraverse: traverse ast 转换树
- escodegen: ast => code

在推荐一个常用的 AST 在线转换网站：https://astexplorer.net/

比如说一段代码 `function getUser() {}`，我们把函数名字更改为 `hello`，看代码流程

```js
const esprima = require('esprima')
const estraverse = require('estraverse')
const code = `function getUser() {}`
// 生成 AST
const ast = esprima.parseScript(code)
// 转换 AST，只会遍历 type 属性
// traverse 方法中有进入和离开两个钩子函数
estraverse.traverse(ast, {
  enter(node) {
    console.log('enter -> node.type', node.type)
  },
  leave(node) {
    console.log('leave -> node.type', node.type)
  },
})
```

输出结果如下：

```txt
enter -> node.type.Program
enter -> node.type.FunctionDeclaration
enter -> node.type.Identifier
leave -> node.type.Identifier
enter -> node.type.BlockStatement
leave -> node.type.BlockStatement
leave -> node.type.FunctionDeclaration
leave -> node.type.Program
```

由此可以得到 AST 遍历的流程是深度优先，遍历过程如下：
<img src="https://github.com/dongxiaobao/Babel-and-AST/blob/master/ast遍历的流程.png" alt="ast遍历的流程" style="zoom:50%;" />

##### 修改函数名字

此时我们发现函数的名字在 `type` 为 `Identifier` 的时候就是该函数的名字，我们就可以直接修改它便可实现一个更改函数名字的 `AST` 工具

```js
// 转换树
estraverse.traverse(ast, {
  // 进入离开修改都是可以的
  enter(node) {
    console.log('enter -> node.type', node.type)
    if (node.type === 'Identifier') {
      node.name = 'hello'
    }
  },
  leave(node) {
    console.log('leave -> node.type', node.type)
  },
})
// 生成新的代码
const result = escodegen.generate(ast)
console.log(result)
// function hello() {}
```

