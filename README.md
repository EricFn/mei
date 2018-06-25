1. 什么是webpack?
可以看做一个模块化打包机，分析项目结构，处理模块化依赖，转换成为浏览器可运行的代码。

代码转换: TypeScript 编译成 JavaScript、SCSS,LESS 编译成 CSS.
文件优化：压缩 JavaScript、CSS、HTML 代码，压缩合并图片。
代码分割：提取多个页面的公共代码、提取首屏不需要执行部分的代码让其异步加载。
模块合并：在采用模块化的项目里会有很多个模块和文件，需要构建功能把模块分类合并成一个文件。
自动刷新：监听本地源代码的变化，自动重新构建、刷新浏览器。

构建把一系列前端代码自动化去处理复杂的流程，解放生产力。
2. 进入webpack世界
初始化项目
	npm install webpack webpack-cli -D
webpack4抽离出了webpack-cli,所以我们需要下载2个依赖。

Webpack 启动后会从Entry里配置的Module开始递归解析 Entry 依赖的所有 Module。 每找到一个 Module， 就会根据配置的Loader去找出对应的转换规则，对 Module 进行转换后，再解析出当前 Module 依赖的 Module。 这些模块会以 Entry 为单位进行分组，一个 Entry 和其所有依赖的 Module 被分到一个组也就是一个 Chunk。最后 Webpack 会把所有 Chunk 转换成文件输出。 在整个流程中 Webpack 会在恰当的时机执行 Plugin 里定义的逻辑。

webpack需要在项目根目录下创建一个webpack.config.js来导出webpack的配置,配置多样化,可以自行定制,下面讲讲最基础的配置。
module.exports = {
	entry: './src/index.js',
	output: {
		path: path.join(__dirname, './dist'),
		filename: 'main.js',
	}
}

entry代表入口，webpack会找到该文件进行解析
output代表输入文件配置
path把最终输出的文件放在哪里
filename输出文件的名字

有时候我们的项目并不是spa，需要生成多个js html，那么我们就需要配置多入口。
module.exports = {
	entry: {
		pageA: './src/pageA.js',
		pageB: './src/pageB.js'
	},
	output: {
		path: path.join(__dirname, './dist'),
		filename: '[name].[hash:8].js',
	},
}
entry配置一个对象，key值就是chunk： 代码块，一个 Chunk 由多个模块组合而成，用于代码合并与分割。看看filename[name]: 这个name指的就是chunk的名字，我们配置的key值pageA pageB，这样打包出来的文件名是不同的，再来看看[hash]，这个是给输出文件一个hash值，避免缓存，那么:8是取前8位。
这里有人会有疑问了，项目是多页面的，应该有pageA.html``pageA.js``pageA.css, 那么我应该生成多个html，这个只是做了JS的入口区分，我不想每一个页面都去复制粘贴一个html,并且html是大部分重复的，可能不同页面只需要修改title,下面来看看这个问题怎么解决:
需要引入一个webpack的plugin:
npm install html-webpack-plugin -D
该插件可以给每一个chunk生成html,指定一个template,可以接收参数，在模板里面使用，下面来看看如何使用:
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
	entry: {
		pageA: './src/pageA.js',
		pageB: './src/pageB.js'
	},
	output: {
		path: path.join(__dirname, './dist'),
		filename: '[name].[hash:8].js',
	},
	plugins: [
		 new HtmlWebpackPlugin({
            template: './src/templet.html',
            filename: 'pageA.html',
            title: 'pageA',
            chunks: ['pageA'],
            hash: true,
            minify: {
                removeAttributeQuotes: true
            }
        }),
        new HtmlWebpackPlugin({
            template: './src/templet.html',
            filename: 'pageB.html',
            title: 'pageB',
            chunks: ['pageB'],
            hash: true,
            minify: {
                removeAttributeQuotes: true
            }
        }),
	]
}
在webpack中，插件的引入顺序没有规定，这个在后面在继续详说。

template: html模板的路径地址
filename: 生成的文件名
title: 传入的参数
chunks: 需要引入的chunk
hash: 在引入JS里面加入hash值 比如: <script src='index.js?2f373be992fc073e2ef5'></script>
removeAttributeQuotes: 去掉引号，减少文件大小<script src=index.js></script>
具体文档

这样在dist目录下就生成了pageA.html和pageB.html并且通过配置chunks，让pageA.html里加上了script标签去引入pageA.js。那么现在还剩下css没有导入，css需要借助loader去做，所以现在要下载几个依赖，以scss为例，less同理
npm install css-loader style-loader sass-loader node-sass -D

css-loader: 支持css中的import
style-loader: 把css写入style内嵌标签
sass-loader: scss转换为css
node-sass: scss转换依赖

来看看如何配置loader
module.exports = {
	module: {
        rules: [
        		{
        			test: /\.scss$/,
        			use: ['style-loader', 'css-loader', 'sass-loader'],
        			exclude: /node_modules/
        		}
        ]
    }
}

test: 一个正则表达式，匹配文件名
use: 一个数组，里面放需要执行的loader，倒序执行，从右至左。
exclude: 取消匹配node_modules里面的文件

如果想把css作为一个单独的文件，需要用到一个插件来做(webpack4.0.0以上版本需要next版本):
 npm i extract-text-webpack-plugin@next -D
const ExtractTextPlugin = require('extract-text-webpack-plugin');
module.exports = {
	entry: './src/index.js',
	output: {
		path: path.join(__dirname, './dist'),
		filename: 'main.js',
    },
    module: {
        rules: [
            {
                test: /\.scss$/,
                use: ExtractTextPlugin.extract({
                    // style-loader 把css直接写入html中style标签
                    fallback: 'style-loader',
                    // css-loader css中import支持
                    // loader执行顺序 从右往左执行
                    use: ['css-loader', 'sass-loader']
                }),
                exclude: /node_modules/
            }
        ]
    },
    plugins: [
        new ExtractTextPlugin('[name].[contenthash:8].css'),
    ]
}

需要在plugins里加入插件name: chunk名字 contenthash:8: 根据内容生成hash值取前8位
修改loader配置下的use: fallback: 兼容方案

这样就实现了js,html,css的打包，那么再来看看一些常用的loader:

babel-loader: 用babel转换代码
url-loader: 依赖于file-loader,把图片转换成base64嵌入html,如果超出一定阈值则交给file-loader

	rules: [
		 // 处理js
		 {
            test: /\.js?$/,
            exclude: /node_modules/,
            use: ['babel-loader']
        },
        // 处理图片
        {
            test: /\.(png|jpg|gif|ttf|eot|woff(2)?)(\?[=a-z0-9]+)?$/,
            use: [{
                loader: 'url-loader',
                options: {
                    query: {
                        // 阈值 单位byte
                        limit: '8192',
                        name: 'images/[name]_[hash:7].[ext]',
                    }
                }
            }]
        },
	]
babel的配置建议在根目录下新建一个.babelrc文件
{
    "presets": [
        "env",
        "stage-0", 
        "react"
    ],
    "plugins": [
        "transform-runtime",
        "transform-decorators-legacy",
        "add-module-exports"
    ]
}

presets: 预设, 一个预设包含多个插件 起到方便作用 不用引用多个插件
env:  只转换新的句法，例如const let => ..等 不转换 Iterator、Generator、Set、Maps、Proxy、Reflect、Symbol、Promise、Object.assign。
stage-0: es7提案转码规则  有 0 1 2 3 阶段  0包含 1 2 3里面的所有
react: 转换react jsx语法
plugins: 插件 可以自己开发插件 转换代码(依赖于ast抽象语法数)
transform-runtime: 转换新语法，自动引入polyfill插件，另外可以避免污染全局变量
transform-decorators-legacy: 支持装饰器
add-module-exports: 转译export default {};  添加上module.exports = exports.default 支持commonjs

因为我们在文件名中加入hash值，打包多次后dist目录变得非常多文件，没有删除或覆盖，这里可以引入一个插件，在打包前自动删除dist目录，保证dist目录下是当前打包后的文件:
plugins: [
	new CleanWebpackPlugin(
	    // 需要删除的文件夹或文件
	    [path.join(__dirname, './dist/*.*')],
	    {
	        // root目录
	        root: path.join(__dirname, './')
	    }
	),
]
指定extension之后可以不用在require或是import的时候加文件扩展名,会依次尝试添加扩展名进行匹配:
resolve: {
    extensions: ['.js', '.jsx', '.scss', '.json'],
},
3. 优化实战 高级装备
天下武功唯快不破，优化方案千千万万，各取所需吧。
提出公共的JS文件
webpack4中废弃了webpack.optimize.CommonsChunkPlugin插件，用新的配置项替代
module.exports = {
	entry: './src/index.js',
	output: {
		path: path.join(__dirname, './dist'),
		filename: 'main.js',
	},
    optimization: {
        splitChunks: {
            cacheGroups: {
                commons: {
                    chunks: 'initial',
                    minChunks: 2,
                    maxInitialRequests: 5,
                    minSize: 2,
                    name: 'common'
                }
            }
        }
    },
}
把多次import的文件打包成一个单独的common.js
HappyPack
在webpack运行在node中打包的时候是单线程去一件一件事情的做，HappyPack可以开启多个子进程去并发执行，子进程处理完后把结果交给主进程
 npm i happypack -D
需要改造一下loader配置，此loader用子进程去处理
const HappyPack = require('happypack');
module.exports = {
	entry: './src/index.js',
	output: {
		path: path.join(__dirname, './dist'),
		filename: 'main.js',
    },
    module: {
        rules: [
            {
                test: /\.jsx?$/,
                exclude: /node_modules/,
                use: 'happypack/loader?id=babel',
            },
        ]
    },
    plugins: [
        new HappyPack({
            id: 'babel',
            threads: 4,
            loaders: ['babel-loader']
        }),
    ]
}

id: id值，与loader配置项对应
threads: 配置多少个子进程
loaders: 用什么loader处理
具体文档

作用域提升
如果你的项目是用ES2015的模块语法，并且webpack3+，那么建议启用这一插件，把所有的模块放到一个函数里，减少了函数声明，文件体积变小，函数作用域变少。
module.exports = {
	entry: './src/index.js',
	output: {
		path: path.join(__dirname, './dist'),
		filename: 'main.js',
    },
    plugins: [
        new webpack.optimize.ModuleConcatenationPlugin(),
    ]
}
提取第三方库
方便长期缓存第三方的库,新建一个入口，把第三方库作为一个chunk，生成vendor.js
module.exports = {
    entry: {
        main: './src/index.js',
        vendor: ['react', 'react-dom'],
    },
}
DLL动态链接
第三库不是经常更新，打包的时候希望分开打包，来提升打包速度。打包dll需要新建一个webpack配置文件，在打包dll的时候，webpack做一个索引，写在manifest文件中。然后打包项目文件时只需要读取manifest文件。
webpack.vendor.js
const webpack = require('webpack');
const path = require('path');

module.exports = {
    entry: {
        vendor: ['react', 'react-dom'],
    },
    output: {
        path: path.join(__dirname, './dist'),
        filename: 'dll/[name]_dll.js',
        library: '_dll_[name]',
    },
    plugins: [
        new webpack.DllPlugin({
            path: path.join(__dirname, './dist/dll', 'manifest.json'),
            name: '_dll_[name]',
        }),
    ]
};
path: manifest文件的输出路径
name: dll暴露的对象名，要跟output.library保持一致
context: 解析包路径的上下文，这个要跟接下来配置的dll user一致
webpack.config.js
module.exports = {
    entry: {
        main: './src/index.js',
        vendor: ['react', 'react-dom'],
    },
    plugins: [
        new webpack.DllReferencePlugin({
            manifest: path.join(__dirname, './dist/dll', 'manifest.json')
        })
    ]
}
html
<script src="vendor_dll.js"></script>
4. 线上和线下
在生产环境和开发环境其实我们的配置是存在相同点，和不同点的，为了处理这个问题，会创建3个文件:

webpack.base.js: 共同的配置
webpack.dev.js: 在开发环境下的配置
webpack.prod.js: 在生产环境的配置

通过webpack-merge去做配置的合并，比如:
开发环境
const path = require('path');
const webpack = require('webpack');
const merge = require('webpack-merge');
const base = require('./webpack.base');

const dev = {
    devServer: {
        contentBase: path.join(__dirname, '../dist'),
        port: 8080,
        host: 'localhost',
        overlay: true,
        compress: true,
        open:true,
        hot: true,
        inline: true,
        progress: true,
    },
    devtool: 'inline-source-map',
    plugins: [
        new webpack.HotModuleReplacementPlugin(),
        new webpack.NamedModulesPlugin(),
    ]
}
module.exports = merge(base, dev);
开发环境中我们可以启动一个devServer静态文件服务器，预览我们的项目，引入base配置文件，用merge去合并配置。

contentBase: 静态文件地址
port: 端口号
host: 主机
overlay: 如果出错，则在浏览器中显示出错误
compress: 服务器返回浏览器的时候是否启动gzip压缩
open: 打包完成自动打开浏览器
hot: 模块热替换 需要webpack.HotModuleReplacementPlugin插件
inline: 实时构建
progress: 显示打包进度
devtool: 生成代码映射，查看编译前代码，利于找bug
webpack.NamedModulesPlugin: 显示模块的相对路径

生产环境
再来看看生产环境最重要的代码压缩，混淆:
const path = require('path');
const merge = require('webpack-merge');
const WebpackParallelUglifyPlugin = require('webpack-parallel-uglify-plugin');
const base = require('./webpack.base');

const prod = {
    plugins: [
        // 文档: https://github.com/gdborton/webpack-parallel-uglify-plugin
        new WebpackParallelUglifyPlugin(
            {
                uglifyJS: {
                    mangle: false,
                    output: {
                        beautify: false,
                        comments: false
                    },
                    compress: {
                        warnings: false,
                        drop_console: true,
                        collapse_vars: true,
                        reduce_vars: true
                    }
                }
            }
        ),
    ]
}
module.exports = merge(base, prod);
webpack-parallel-uglify-plugin可以并行压缩代码，提升打包效率
uglifyJS配置:

mangle: 是否混淆代码
output.beautify: 代码压缩成一行 true为不压缩 false压缩
output.comments: 去掉注释
compress.warnings: 在删除没用到代码时 不输出警告
compress.drop_console: 删除console
compress.collapse_vars: 把定义一次的变量，直接使用，取消定义变量
compress.reduce_vars: 合并多次用到的值，定义成变量
具体文档

5. 成为头号玩家
想要成为头号玩家，玩转配置可不行，当然还要做一些loader和plugin的开发，去为项目做一些优化，解决痛点。
loader
loader是一个模块导出函数，在正则匹配成功的时候调用，webpack把文件数组传入进来，在this上下文可以访问loader API

this.context: 当前处理文件的所在目录，假如当前 Loader 处理的文件是 /src/main.js，则 this.context 就等于 /src。
this.resource: 当前处理文件的完整请求路径，包括 querystring，例如 /src/main.js?name=1。
this.resourcePath: 当前处理文件的路径，例如 /src/main.js。
this.resourceQuery: 当前处理文件的 querystring。
this.target: 等于 Webpack 配置中的 Target
this.loadModule: 但 Loader 在处理一个文件时，如果依赖其它文件的处理结果才能得出当前文件的结果时， 就可以通过 - - -  this.loadModule(request: string, callback: function(err, source, sourceMap, module)) 去获得 request 对应文件的处理结果。
this.resolve: 像 require 语句一样获得指定文件的完整路径，使用方法为 resolve(context: string, request: string, callback: function(err, result: string))。
this.addDependency: 给当前处理文件添加其依赖的文件，以便再其依赖的文件发生变化时，会重新调用 Loader 处理该文件。使用方法为 addDependency(file: string)。
this.addContextDependency: 和 addDependency 类似，但 addContextDependency 是把整个目录加入到当前正在处理文件的依赖中。使用方法为 addContextDependency(directory: string)。
this.clearDependencies: 清除当前正在处理文件的所有依赖，使用方法为 clearDependencies()。
this.emitFile: 输出一个文件，使用方法为 emitFile(name: string, content: Buffer|string, sourceMap: {...})。
this.async: 返回一个回调函数，用于异步执行。

下面来看看less-loader和style-loader如何实现:
let less = require('less');
module.exports = function (source) {
    const callback = this.async();
    less.render(source, (err, result) => {
        callback(err, result.css);
    });
}
module.exports = function (source) {
    let script = (`
      let style = document.createElement("style");
      style.innerText = ${JSON.stringify(source)};
      document.head.appendChild(style);
   `);
    return script;
}
plugin
webpack整个构建流程有许多钩子，开发者可以在指定的阶段加入自己的行为到webpack构建流程中。插件由以下构成:

一个 JavaScript 命名函数。
在插件函数的 prototype 上定义一个 apply 方法。
指定一个绑定到 webpack 自身的事件钩子。
处理 webpack 内部实例的特定数据。
功能完成后调用 webpack 提供的回调。

整个webpack流程由compiler和compilation构成,compiler只会创建一次，compilation如果开起了watch文件变化，那么会多次生成compilation. 那么这2个类下面生成了需要事件钩子
compiler hooks 文档
compilation hooks 文档
写一个小插件，生成所有打包的文件列表(webpack4不推荐使用compiler.plugin来注册插件，webpack5将不支持):
class FileListPlugin{
    constructor(options) {
        this.options = options;
    }
    apply(compiler) {
        compiler.hooks.emit.tap('FileListPlugin',function (compilation) {
            let fileList = 'filelist:\n\n';
            for (let filename in compilation.assets) {
                fileList += ('- '+filename+'\n');
            }
            compilation.assets['filelist.md']={
                source() {
                    return fileList;
                },
                size() {
                    return fileList.length
                }
            }
        });
    }
}
module.exports = FileListPlugin;

作者：lihuanji
链接：https://juejin.im/post/5ac9dc9af265da23884d5543
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
