## 解决引入node_module文件出现未编译（webpack加强理解）

* 题记：前面为了解决项目中遇到的引入elementUI里面的js依赖文件，无法转化为es5 的问题进行了探索。
* 先了解问题可能出现的问题切入点
1. 由于出现babel打包过程中出现语法错误导致文件打包过程中出现跳过错误的问题，导致该过程中该部分的文件无法转化为es5。
2. 由于出现使用import 导致编译的时候确实会转化为require来引入文件，但是引入的文件存在es6是无法打包的。
* 首先通过使用直接引入原文件的方式（可以保证是elementUI的源码）排除了第一种的推断
* 二在同事的提示下发现nodemodule的模块都会编译打包生成新的文件该文件确实是按照es5实现的

### 总结

当要依赖nodemoule文件的时候需要使用引入文件的时候需要做的是
1. 查看文件的语法是否浏览器支持（es6，typescript，coffiscript）
2. 如果不支持：

第一： 查看nodemodule文件里面是否存在你需要的打包文件一般的文件名为（lib，util等）

第二： 如果不存在最好使用babel工具[网址](http://babeljs.io/repl/)进行文件的打包，实现文件直接引入的方式进行操作（推荐）

第三： 使用.babel文件结合webpack自定义打包（费时间，需要理解第三方代码结构）

为了对问题更加了解需要对webpack进行理解

前端模块化解决问题可以分为amd[规范](https://link.zhihu.com/?target=https%3A//github.com/amdjs/amdjs-api/wiki/AMD)和cmd[规范](https://link.zhihu.com/?target=https%3A//github.com/seajs/seajs/issues/242)
amd 提前加载 cmd 延迟加载 他们都是基于commond.js在浏览器的规范。。扯远了。后面会阐述为什么不能直接引入。

一下文档参考[webpack官网](https://webpack.js.org/concepts/)

首先理解webpack是一切皆模块（废话不然怎么实现前端组件化）

webpack 

1. entry （webpack进入文件的入口） You can specify an entry point (or multiple entry points) by configuring the entry property in the webpack configuration. It defaults to ./src.
2. output （输出打包后的文件路径和名字）You can configure this part of the process by specifying an output field in your configuration:
3. loaders （加载器主要是过滤文件编译不同的语言形成浏览器可以识别的文件js css html image）webpack loaders transform all types of files into modules that can be included in your application's dependency graph 
4. plugins （插件 这个就丰富了有开发插件watch driver 等等）The plugin interface is extremely powerful and can be used to tackle a wide variety of tasks
5. mode （模式选择可以在不同的模式运行不同的处理方式进行优化打包）you can enable webpack's built-in optimizations that correspond with the selected mode


从上面简单的介绍就可以看出为什么直接引入node_module 会出现问题了。简单的讲就是比如打开element/build/config.js 文件就知道里面也是使用了webpack 所以也是一个项目文件当它构建好后依赖的文件也是打包进入它自定义的文件然后暴露接口。如果按照单个文件方式打包（就是在打包路径中名字一一对应内容一一对应）就可以直接在外部引用不会有错误。但是如果把多个文件打包成一个就需要按照依赖一个一个引入。造成这个问题的原因就是entry（你的项目入口只在你的项目没有在node_module 里面）

来讲讲webpack的配置吧！

    const path = require('path');
    const HtmlWebpackPlugin = require('html-webpack-plugin');
    const CleanWebpackPlugin = require('clean-webpack-plugin');


    module.exports = {
        entry: {
            app: './src/index.js',
            print: './src/print.js'
        },
        devtool:'inline-source-map',
        devServer:{
            contentBase: './dist'
        },
        plugins: [
            new CleanWebpackPlugin(['dist']),
            new HtmlWebpackPlugin({
                title: 'Output Management'
            })
        ],
        output: {
            filename: '[name].bundle.js',
            path: path.resolve(__dirname, 'dist'),
            publicPath:'/'
        },
        module: {
            rules: [
                {
                    test:/\.css$/,
                    use : [
                        'style-loader',
                        'css-loader'
                    ]
                },
                {
                    test:/\.(png|svg|jpg|gif)$/,
                    use:[
                        'file-loader'
                    ]
                }
            ]
        }
    };

上面是我按照官方文档配置的webpack已经进行到devServer章节

主要是实现了dev环境的mapper 和打包环境的index.html的创建以及文件的监听server服务已经filewatch等等。

就按照目前的配置介绍

    const path = require('path');
    const HtmlWebpackPlugin = require('html-webpack-plugin');
    const CleanWebpackPlugin = require('clean-webpack-plugin');

1. 这里主要引入依赖的文件模块path路径node原生模块应该类似 linux 的ll ls 这一类的操作。
2. html-webpack-plugin 就是完成生成index。html文件并且自动生成依赖引入
3. clean-webpack-plugin 清除打包的残留物，每次执行 npm run build 的时候可以触发dist目录的删除来实现文件清除

        entry: {
                    app: './src/index.js',
                    print: './src/print.js'
                },

入口文件的位置和名字也可以使用正则匹配

        devtool:'inline-source-map',
                devServer:{
                    contentBase: './dist'
                },

上面的是mapper可以实现出错的文件的查找，主要方便调试下面的server环境需要导入express模块实现服务端口的配置

        plugins: [
            new CleanWebpackPlugin(['dist']),
            new HtmlWebpackPlugin({
                title: 'Output Management'
            })
        ],
上面已经介绍

        output: {
            filename: '[name].bundle.js',
            path: path.resolve(__dirname, 'dist'),
            publicPath:'/'
        },

输出文件的名字和地址该name 是入口文件配置的

        module: {
            rules: [
                {
                    test:/\.css$/,
                    use : [
                        'style-loader',
                        'css-loader'
                    ]
                },
                {
                    test:/\.(png|svg|jpg|gif)$/,
                    use:[
                        'file-loader'
                    ]
                }
            ]
        }

这个需要实现需要引入webpack的css和html的插件实现文件的加载和输出test匹配正则规则 后缀 use就是对文件的转译 所以再次说明文件是从硬盘读写不是简单的引入就可以实现文件的转码。



