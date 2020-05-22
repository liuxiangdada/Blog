## 什么是webpack

官方文档上明确定义了webpack，**本质上，webpack是一个现代化JavaScript应用程序的静态模块打包器**

webpack处理项目时，会递归的构建一个依赖关系图，该图记录了所有模块之间的依赖关系，然后根据需求将这些模块打包成一个或多个bundle

webpack就像一条生产线，经过一系列的处理将源文件转换成输出结果，在这条流水线上，每个流程的职责单一，且只有上一个处理流程完成后才转入下一个流程，
在整个生产线的处理流程中，存在着一些关键节点，执行到这些节点时，会对外广播事件，插件就像是一个被插入到生产线上的额外功能，在监听到自己关系的事件时，
就能加入生产线，完成自己的功能，从而改变生产线的运作

webpack本身的整个处理流程形成闭环，但它又允许插件以事件的形式在可控的范围内改变生产线，各个不同的广播事件，保证每个插件执行功能的有序性，使得webpack的扩展性很好

## 核心概念

### Entry

入口起点指示webpack应该使用哪个模块来作为构建其内部依赖图的开始，webpack依据配置找到入口，找出其引用的模块，然后层层处理，最后输出到bundle文件中

### Output

output告知webpack在哪里输出bundle文件以及如何命名这些文件，一般整个应用程序都会被编译到指定的输出文件夹中

### Module

在webpack中，一切皆可被视为模块，一个模块对应一个文件，webpack从Entry开始递归找出所有依赖的模块

### Chunk

代码块，一个chunk由多个模块打包而成，用于代码的分割合并

### Loader

webpack本身只能识别JS文件，我们引入各种各样的loader让webpack能够处理那些非JS文件

loader所做的事情就是将其他类型的文件转化为webpack能够识别的有效模块，webpack依据这些模块构建依赖图

### Plugin

loader被用于转化模块，而plugin则用于一些更宽泛更复杂的任务，可以说，除开loader的工作，其他都能依靠plugin实现，比如代码压缩、优化

## webpack的打包流程是怎样的

前面我们说，webpack的工作流程类似于流水线，是串行的，从启动到结束会依次执行以下流程：

1. 初始化参数，从配置文件和shell命令参数导出最终配置
2. 开始编译，webpack内部维护了一个类似于Compiler的类，根据配置初始化Compile对象，执行其run方法开始打包
3. 根据配置确定入口，找到入口文件
4. 从入口文件出发，根据文件后缀和loader数组依次调用，对模块进行翻译，同时找出模块的依赖，递归自身，直到处理完所有模块
        
        处理同一后缀的文件loader数组从后往前依次执行
5. 完成模块的编译和确定模块的依赖关系
6. 根据入口和模块间的依赖关系，组装成一个个包含多个模块的Chunk，再把每个chunk转化成单独的文件加入到输出列表，这里仍然可以修改最终输出
7. 确定输出内容后，根据配置将文件写入目标文件夹以及命名文件

在上面的各个流程，webpack会在特定时机广播事件，插件监听到自身相关的事件后，执行特定的逻辑，调用webpack提供的API改变最终输出

## 自己实现一个简易的webpack

首先我们确定webpack的配置，在根目录下创建配置文件

    // webpack.config.js
    const path = require('path')

    module.exports = {
      entry: './src/index.js',
      output: {
        filename: 'bundle.js',
        path: path.resolve(__dirname, './dist')
      }
    }

接着增加script命令：

    scripts: {
      'webpack': node ./core/index.js,
      'print': node ./dist/bundle.js
    }

根据上面的webpack打包流程，我们需要实现：

- 转译代码，将代码转译成浏览器可识别的代码（浏览器默认不支持ES Module）
- 获取代码中的import信息，生成依赖图
- 根据依赖图，加载各个模块的信息并导入到bundle文件中

转译代码直接使用babel，安装相应包

    npm i -D @babel/core @babel/preset-env @babel/parse @babel/traverse

我们封装一个parse模块，用于处理代码

    module.exports = {
      // 将字符串代码转化成ast
      getAst () {},
      // 获取模块的依赖
      getDependecies () {},
      // 根据依赖得到处理后的字符串代码
      getCode () {}
    }

接着定义一个Complier类，实现webpack的核心功能

    class Compile {
      constructor (options) {
        this.entry = options.entry
        this.output = options.output

        this.modules = []
      }

      // 处理模块，得到ast信息、依赖信息
      build (filename) {
        const ast = getAst(filename)
        const dependecies = getDependecies(ast, filename)
        const code = getCode(ast)

        return {
          filename,
          dependecies,
          code
        }
      }

      // webpack主程序，递归寻找依赖，构建依赖图
      run () {
        this.modules.push(this.build(this.entry))

        for (let i = 0; i < this.modules.length; i++) {
          const dependecies = this.modules[i]

          for(dependecy in dependecies) {
            this.modules.push(this.build(dependecies[dependecy]))
          }
        }

        const graph = this.modules.reduce((graph, item) => {
          return {
            ...graph,
            dependecies: item.dependecies,
            code: item.code
          }
        }, {})

        this.generator(graph)
      }

      // 编译成可直接执行的bundle
      generator (code) {
        // 如果没有dist文件夹
        if (!fs.existsSync(this.output.path)) {
          fs.mkdirSync(this.output.path)
        }

        const filePath = path.join(this.output.path, this.output.filename)

        const bundle `(function (graph) {
          function require (module) {
            function localRequire (relativePath) {
              return require(graph[module].dependecies[relativePath])
            }

            var exports = {}
            ;(function (require, exports, code) {
              eval(code)
            })(localRequire, exports, graph[module].code)

            return exports
          }

          require('${this.entry}')
        })(${JSON.stringify(code)})`

        // 如果存在文件，先删除
        if (fs.existsSync(filePath)) {
          fs.unlinkSync(filePath)
        }

        fs.writeFileSync(filePath, bundle, 'utf-8')
      }
    }

    module.exports = Compile

然后调用一下就行了

    const compile = require('./compile')
    const config = require('../webpack.config')

    new compile(config).run()

具体的细节请查看我的esay-webpack项目 [传送门](https://github.com/loofk/easy-webpack)