# JS代码规范

## 前言

本文档主要参考 [ESLint 中文文档](https://eslint.bootcss.com/docs/rules/)，结合自身代码风格整合而成

团队项目可遵循下面的细则确保风格统一规范，便于阅读和协同合作

## 细则

1. 缩进：使用两个空格缩进（eslint：indent）
   
   ```
   function hello () {
     console.log('hello')
   }
   ```

2. 引号：除需要转义的情况，字符串统一使用单引号（eslint：quotes）
   
   ```
   console.log('hello')

   $("<div class = 'box'></div>")
   ```

3. 分号：除开可能导致上下行解析问题的5个`token`：括号、方括号、正则开头的斜杆、加号、减号，语句末尾不带分号，一般来说在开头是括号或方括号时在行首加上分号（eslint：semi）
   
   ```
   let a = name

   ;(function (arg) {
     console.log('hello world')
    })('John')
   ```

4. 冒号：键值对中，冒号和值之间留间隔（eslint：key-spacing）
   
   ```
   let obj = { key: val, key2: val2 }
   ```
   
5. 未使用变量：如果一个变量在作用域内未被使用，则不需要定义（eslint：no-unused-vars）
   
   ```
   function myFunction () {
     let unusedVar = something() // × unused
   }
   ```

6. 关键字空格：语言囊括的关键字后面要添加空格（eslint：keyword-spacing）
   
   ```
   if (condition) { ... }   // √

   if(condition) { ... }    // ×
   ```

7. 函数声明：函数名字和括号之间加空格；括号与后面的花括号之间加空格（eslint：space-before-function-paren）
   
   ```
   function name (arguments) { ... }   // √

   function name(arguments){ ... }     // ×
   ```

8. 函数调用：调用时函数名和括号之间不留间隔（eslint：func-call-spacing）
   
   ```
   console.log('hello')    // √

   console.log ('hello')   // ×
   ```

9.  构造函数：构造函数首字母大写（eslint：new-cap）
    
    ```
    function Animal () {}

    let dog = new Animal()
    ```

10. 等号：始终使用 `===` 而不是 `==` ，特殊情况如检查变量是否为 `null` 或 `undefined`（eslint：eqeqeq）
    
    ```
    if (name === 'John')

    if (name !== 'Tom')
    ```

11. 字符串拼接：字符串拼接时通常使用 `+` 符号，其前后应留空格（eslint：space-infix-ops）
    
    ```
    let message = 'hello, ' + name + '!'
    ```

12. 逗号：对象或数组不允许有行末逗号；逗号后面要留空格（eslint：comma-spacing、comma-dangle）
    
    ```
    let arr = [1, 2, 3, 4]

    let obj = { name: 'John', age: 12 }
    ```

13. 始终将逗号置于行末（eslint：comma-style）
    
      ```
      let obj = {
        foo: 'foo',
        bar: 'bar'   // √
      }

      let obj = {
        foo: 'foo'
        ,bar: 'bar'  // ×
      }
      ```

14. 点号：点号操作符必须始终与属性保持在同一行（eslint：dot-location）
    
      ```
      console
      .log('hello')   // √

      console.
      log('hello')    // ×
      ```

15. 条件：`else` 关键字要和花括号保持在同一行（eslint：brace-style）
    
      ```
      if (condition) {

      } else {

      }
      ```

16. 多行 `if` 语句的花括号不能省（eslint：curly）
    
      ```
      if (condition) console.log('hello')

      if (condition) {
        console.log('hello')
        console.log('world')
      }
      ```

17. 条件语句中的赋值语句使用括号包裹，避免误认为将全等号漏写（eslint：no-cond-assign）
    
      ```
      if ((val = text.match(expr))) {

      }

      while ((m = text.match(expr))) {

      }
      ```

18. 多行空行：允许单行空行，但不允许多行空行（eslint：no-multiple-empty-lines）
    
      ```
      // √
      let value = 'hello world'

      console.log(value)

      // ×
      let value = 'hello world'


      console.log(value)
      ```

19. 三目运算符：简短的一行，长的保持 `?` 和 `:` 与他们所负责的代码同行（eslint：operator-linebreak）
    
      ```
      let location = env.development ? 'localhost' : 'www.api.com'

      let location = env.development
        ? 'localhost' 
        : 'www.api.com'
      ```

20. 代码块：单行代码块两边加空格（eslint：block-spacing）
    
      ```
      function foo () { return true }   √

      function foo () {return true}    ×
      ```

21. 命名：变量和函数命名统一使用驼峰命名法（eslint：camelcase）
    
    ```
    function my_function () {}   √

    function myFunction () {}    ×
    ```

22. 取值器：对象中定义了存值器，一定要定义与之对应的取值器（eslint：accessor-pairs）
    
    ```
    // √
    let person = {
      get name () {
        return this._name
      },
      set name () {
        this._name = value
      }
    }

    // ×
    let person = {
      set name () {
        this._name = value
      }
    }
    ```

23. 子类的构造器中一定要使用 `super`（eslint：constructor-super）
    
    ```
    class Dog extends Animal {
      constructor () {
        super()
      }
    }
    ```

24. 不对变量使用 `delete` 操作（eslint：no-delete-var）
    
    ```
    let name

    delete name   // ×
    ```

25. 同一模块多个导入时一次性写完（eslint：no-duplicate-imports）
    
    ```
    import { myFunc1, myFunc2 } from 'module'     // √

    import { myFunc1 } from 'module'

    import { myFunc2 } from 'module'              // ×
    ```

26. `switch` 使用 `break` 来将条件分支正常中断（eslint：no-fallthrough）
    
    ```    
    switch (filter) {
      case 1:
        doSomething()
        break
      case 2:
        doSomethingElse()
        break
    }                            // √

    switch (filter) {
      case 1:
        doSomething()
      case 2:
        doSomethingElse()
    }                            // ×
    ```

27. 不省略小数点前面的0（eslint：no-floating-decimal）
    
    ```
    const discount = 0.5    // √

    const discount = .5     // ×
    ```

28. `eval`：避免使用`eval`；注意隐式的`eval`（eslint：no-eval、no-implied-eval）
    
    ```
    const res = eval('a + b')              // ×

    setTimeout("alert('hello world')")     // ×，带有隐式eval
    ```

29. 嵌套的代码块禁止再定义函数（eslint：no-inner-declarations）
    
    ```
    if (condition) {
      function setAuther () {}   // ×
    }
    ```

30. `return` 语句中的赋值必须有括号包裹（eslint：no-return-assign）
    
    ```
    function sum (a, b) {
      return (result = a + b)
    }                           // √

    function sum (a, b) {
      return result = a + b
    }                           // ×
    ```

31. 代码块首尾留空格（eslint：space-before-blocks）
    
    ```
    if (condition) { ... }      // √

    if (condition){ ... }       // ×
    ```

32. 注释首尾留空格（eslint：spaced-comment）
    
    ```
    // commit     // √
    //commit      // ×

    /*comment*/         // ×
    /* comment */       // √
    ```

33. 检查 `NaN` 的正确姿势是使用 `isNaN()`（eslint：use-isnan）
    
    ```
    if (isNaN(price)) { }   // √

    if (price === NaN) { }  // ×
    ```


## ESLint安装以及使用

### .eslintrc.js

如果要使用 `eslint`，则需要使用 `npm` 安装
```
npm i eslint -D (--save-dev)
```

接着是初始化配置文件
```
./node_modules/.bin/eslint --init
```

运行后会在项目根目录创建一个 `.eslintrc.js` 文件

注意，如果我们要定制化 `lint` 功能，需要配置`extends`、`plugins`、`rules`三个选项

#### extends选项

主要是扩展一些已经完备的规则，常用的有`eslint:recommended`、`standard`、`airbnb`三种风格，我这里选择的是默认的

#### plugins选项

该选项主要是安装插件，使 `eslint` 可以校验不同类型的文件，而不是局限于 `js` 文件，比如校验 `html` 和 `vue`

首先安装插件
```
npm i eslint-plugin-html -D
npm i eslint-plugin-vue -D
```

配置到`.eslintrc.js` 文件中
```
plugins: ["html", "vue"],
```

#### rules选项

自定义规则，详情参见 [ESLint 中文文档](https://eslint.bootcss.com/docs/rules/)

#### 个人常用配置如下：
```
module.exports = {
  root: true,

  parserOptions: {
    parser: "babel-eslint",
    sourceType: "module",
    //ignore eslint error: 'import' and 'export' may only appear at the top level
    allowImportExportEverywhere: true
  },

  env: {
    browser: true,
    node: true,
    es6: true
  },

  globals: {
    Atomics: "readonly",
    SharedArrayBuffer: "readonly",
  },

  extends: [
    "eslint:recommended",
    "plugin:vue/essential"
  ],

  // required to lint *.vue files
  plugins: ["html", "vue"],

  // add your custom rules here
  // off: 0, warn: 1, error: 2
  rules: {
    "space-before-blocks": 2, // blocks need space before and after
    "keyword-spacing": 2,
    "space-before-function-paren": 2, // before and after functionName remain space
    "quotes": [2, "single"], // quotes should be single
    "semi": [2, "never"], // semi never at the last
    "key-spacing": 2,
    "no-multiple-empty-lines": 2, // forbidden mutilple empty lines
    "no-empty": 0,
    "new-cap": 2, // constructor function's first letter should be capital
    "eqeqeq": 2, // use '===' instead '=='
    "no-cond-assign": 2, // forbidden assign in condition
    'no-useless-escape': 0,
    "generator-star-spacing": 0, // allow async-await
    "no-debugger": process.env.NODE_ENV === "production" ? 2 : 0, // allow debugger during development
    "no-console": process.env.NODE_ENV === "production" ? 2 : 0 // allow console only during development
  }
};
```

### .eslintignore

该文件用于指定 `eslint`忽略的文件

### .editorconfig

该文件用于编写代码时的规范，即在编写时告诉编辑器以什么规范辅助编码，主要可以实现 `tab` 或 `space` 缩进，换行缩进的空格数等一些基础的规范，通常配合 `eslint`一起确保项目在团队合作时风格统一

下面是我的配置：

```
root = true

[*]
indent_style = space
indent_size = 2

end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.{js,html,blade.php,css,scss,vue,json}]
indent_size = 2

[*.md]
trim_trailing_whitespace = false

```

### VS Code Settings

我们这里以 `VS Code` 为例，首先去插件市场安装 `eslint` 插件，要想 `eslint` 生效，还必须修改编辑器的 `settings.json` 文件：
```
// 配置eslint的校验文件类型
"eslint.validate": [
  "javascript",
  "javascriptreact",
  "typescript",
  "html",
  "vue"
],

// 保存时让编辑器按eslint的规则格式化文件，可不要
"editor.codeActionsOnSave": {
  "source.fixAll.eslint": true
}
```

### package.json

不通过编辑器校验的话就要使用 `node` 命令执行 `eslint`，我们在 `package.json` 文件下添加 `script` 命令：
```
  "scripts": {
    ...

    "lint": "eslint src tests",
    "format": "eslint src tests --fix"
  },
```

其中 `eslint` 后面紧跟的是需要校验的文件范围，`--fix` 参数表示主动修正一些不符合规范的语法

## GitHook提交检查钩子配置

对于团队合作而言，代码规范可能不会被每个人遵守，为确保提交到服务器的代码不存在坏代码，我们需要在 `git commit` 之前对文件进行校验

`git` 提供了各个阶段的钩子供开发者使用，我们这里使用名为 `pre-commit` 的钩子

首先需要安装 `yorkie`

```
npm i yorkie -D
```

它是尤大根据 `husky` 改写的 `pre-commit` 钩子包，能够从 `package.json` 中读取 `gitHooks` 属性

接着配置 `package.json` 文件
```
"gitHooks": {
  "pre-commit": "npm run format",
  "commit-msg": "node scripts/verify-commit-msg.js"
},
```

这样在提交之前就会自动校验文件是否符合 `eslint` 规范并自动修复一些可选项，这里有个问题，就是我们的文件已经被提交到暂存区了，如果又被修复，新的修改不会被添加到暂存区

我们还需要引入 `lint-staged` 来处理暂存区的文件，比如帮我们把上面的修复文件提交到暂存区，校验其他类型文件等

安装 `lint-staged` 
```
npm install lint-staged -D
```

重写配置文件
```
"gitHooks": {
  "pre-commit": "lint-staged",
  "commit-msg": "node scripts/verify-commit-msg.js"
},
"lint-staged": {
  "*.{js,vue}": [
    "eslint --fix",
    "git add"
  ],
  "*.scss": [
    "stylelint --fix --syntax scss",
    "prettier --write",
    "git add"
  ]
}
```

上面有注意到，我们在 `gitHooks` 中还增加了一个 `commit-msg` 钩子，用于规范提交注释，格式类似于：
```
feat(compiler): add 'comments' option
```

其他具体细则参考 [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog/tree/master/packages/conventional-changelog-angular)

如果需要进一步规范的话，可以使用 `git-cz` 代替 `git commit`，首先安装包：
```
npm install commitizen cz-conventional-changelog -D
```

接着修改 `package.json` 文件
```
  "scripts": {
    ...

    "lint": "eslint src tests",
    "format": "eslint src tests --fix"
    "commit": "git-cz"
  },
  "config": {
    "commitizen": {
      "path": "./node_modules/cz-conventional-changelog"
    }
  }
```

之后，在 `git add` 后运行 `npm run commit` 就可以根据步骤依次填写提交注释了

