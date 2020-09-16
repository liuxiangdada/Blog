# 阅读源码前的准备

## 包引用机制

ES6的模块是静态编译，什么意思呢？就是JS引擎在解析JS文件时，会将所有import提到文件顶部，并执行模块内的代码，建立导出内容和模块中对应部分的指针映射，输出接口动态绑定，看起来就像在运行导入的代码之前就已经执行了模块代码一样，Vue正是利用了ES6的模块机制实现了导入Vue构造函数之前的一些原型方法的注入

## 编译

我们知道Vue是支持多种版本的，兼容性很强，有运行时版本，有带编译器的版本，有CommonJS格式，有ES Module格式，也有UMD格式，更可以在WEEX平台使用，在了解Vue的实现之前，我们先了解一下这些基础概念

### 运行时版本

这个版本的Vue只接受render函数，不能传入template属性，这是因为Vue内部渲染时只接受render函数作为渲染基础，如果要使用该版本，要么用户自己手写render函数，要么借助webpack的vue-loader插件将在Vue文件中定义template标签转化为render函数

### 编译器版本

带编译器版本可以不用手写render函数，支持传入template字符串的形式或者写在HTML里，内部的编译器会自动将template字符串转成render函数

### 不同格式之间的差异

UMD：全名通用模块规范，设计的初衷是用来统一CommonJS和AMD，让人们写一套规范就能在浏览器和服务端使用

- CommonJS：低版本的打包工具（如webpack1）都是使用的CommonJS标准的，如果不使用CommonJS版本的Vue，将无法在低版本打包工具构建的项目中运行
- ES Module：新版打包工具（webpack2或rollup）使用ES Module语法，建议使用ES Module版本
- UMD：在浏览器中使用script引入时，建议使用UMD版本

## flow

flow是JS语言的静态类型检查工具，它保证了在大规模项目开发时代码更加高效、可控，在vue2.x普遍使用flow，而在vue3.x已经全面拥抱更为强大的typescript

## rollup

rollup是基于ES6语法的模块打包器，与webpack或者gulp不同，它专注于JS代码的打包，不会生成额外的代码，所以适合一些JS类库的开发，rollup的另一个优点是支持tree-shaking

## 目录组织结构

```
src
├── compiler        # 编译相关 
├── core            # 核心代码 
├── platforms       # 不同平台的支持
├── server          # 服务端渲染
├── sfc             # .vue 文件解析
├── shared          # 共享代码（工具方法、常量）
```