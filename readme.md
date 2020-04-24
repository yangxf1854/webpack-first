npx 是npm从5.2版开始增加的命令，想要解决的主要问题，就是调用项目内部安装的模块
## 初始化项目

npm init y (也可以使用yarn)

### 安装webpack webpack-cli
npm install webpack webpack-cli -D
备注（前端变更迅速，所以这边记录下webpack的版本号）、
|————webpack@4.43.0
|____webpack-cli@3.3.10
从webpackV4.0.0开始，webpack是开箱即用的，在不引入任何配置文件的情况下就可以使用

### 新建 src/index.js，随便在文件中写点什么
/* index.js */
class Animal {
  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }
}

const dog = new Animal('dog');

### 使用npx webpack --mode=development 进行构建，默认是 production 模式，我们为了更清楚得查看打包后的代码，使用 development 模式

### 执行完之后此时可以看到项目下多了 dist 目录，里面有一个打包出来的文件 main.js
webpack 有默认的配置，如默认的入口文件是 ./src , 默认打包到 dist/main.js 。更多的默认配置可以查看 node_modules/webpack/lib/WebpackOptionsDefaulter.js。

### 查看 dist/main.js 文件，可以看到， src/index.js 并没有被转义为低版本的代码

```
(function(module, exports) {

eval("class Animal {\n  constructor(name) {\n    this.name = name;\n  }\n\n  getName() {\n    return this.name;\n  }\n}\n\nconst dog = new Animal('dog');\n\n//# sourceURL=webpack:///./src/index.js?");
 })
});
```

## 将js转义为低版本
webpack的四大核心其中之一就是loader，用于对源代码进行转换，这正是现在所需要的

将js代码向低版本转换，我们需要 babel-loader

### babel-loader
首先安装一下 babel-loader
npm install babel-loader -D
此外，还需要配置babel， 为此要安装一下以下依赖：
npm install @babel/core @babel/preset-env @babel/plugin-transform-runtime -D

npm install @babel/runtime @babel/runtime-corejs3

### 新建 webpack.config.js,如下：
```
module.exports = {
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        use: ['babel-loader'],
        exclude: /node_modules/, // 排除node_modules目录
      }
    ]
  }
}
```
建议loader指定include 或是exclude，指定其中一个即可，因为node_modules目录通常不需要我们去编译，排除后，有效提升编译效率

这里，我们可以在.babelrc中编写babel配置，也可以在weback.config.js中进行配置

### 创建一个.babelrc,配置如下：
```
{
  "presets": ["@babel/preset-env"],
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "corejs": 3
      }
    ]
  ]
}
```
重新执行npx webpack --mode=develoment,查看dist/main.js,

### 在webpack中配置babel
```
module.exports = {
  module: {
    rules: [
      {
        test: /\.jsx?$/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ["@babel/preset-env"],
            plugins: [
              [
                "@babel/plugin-transform-runtime",
                {
                  "corejs": 3
                }
              ]
            ]
          }
        },
        exclude: /node_modules/, // 排除node_modules目录
      }
    ]
  }
}
```
注意：
> loader需要配置在module.rules中，rules是一个数组
> loader的格式为：
```
{
    test: /\.jsx?$/,//匹配规则
    use: 'babel-loader'
}
```
或者
#### 适用于只有一个 loader 的情况
```
{
    test: /\.jsx?$/,
    loader: 'babel-loader',
    options: {
        //...
    }
}
```
test 字段是匹配规则，针对符合规则的文件进行处理
use 字段有几种写法：
> 可以是一个字符串串
> 可以是一个数组，例如处理css文件是，use['style-loader', 'css-loader']
> 数组的每一项既可以是字符串也可以是一个对象，当我们需要在webpack配置文件中对loader进行配置，就需要将其编写为一个对象，并且在次对象的option字段中进行配置如：
```
rules: [
    {
        test: /\.jsx?$/,
        use: {
            loader: 'babel-loader',
            options: {
                presets: ["@babel/preset-env"]
            }
        },
        exclude: /node_modules/
    }
]
```

## mode
我们在使用webpack进行打包的时候，一直运行的都是 npx webpack --mode=development ,是否可以将mode配置在webpack.config.js中呢，答案是可以的。

### 将mode增加到webpack.config.js中
```
module.exports = {
    //....
    mode: "development",
    module: {
        //...
    }
}

```
mode配置项，告知webpack使用相应模式的内置优化

mode配置项，支持以下两个字符串值：
> 会将 process.env.NODE_ENV 的值设为 development。启用 NamedChunksPlugin 和 NamedModulesPlugin。
> 会将 process.env.NODE_ENV 的值设为 production。启用 FlagDependencyUsagePlugin, FlagIncludedChunksPlugin, ModuleConcatenationPlugin, NoEmitOnErrorsPlugin, OccurrenceOrderPlugin, SideEffectsFlagPlugin 和 UglifyJsPlugin
现在只要运行 npx webpack 进行编译即可

## 在浏览器中查看页面
有时候会指定打包文件中带有hash，那么没次生成的js文件名会有所不同，那不能让我们每次都人工去修改html

可以使用html-webpack-plugin插件来帮助我们完成这些事情
首先安装下插件
npm install html-webpack-plugin -D

新建一个public目录，并在其中新建一个 index.html 文件，
修改 webpack.config.js 文件
```
//首先引入插件
const HtmlWebpackPlugin = require('html-webpack-plugin');
module.exports = {
    //...
    plugins: [
        //数组 放着所有的webpack插件
        new HtmlWebpackPlugin({
            template: './public/index.html',
            filename: 'index.html', //打包后的文件名
            minify: {
                removeAttributeQuotes: false, //是否删除属性的双引号
                collapseWhitespace: false, //是否折叠空白
            },
            // hash: true //是否加上hash，默认是 false
        })
    ]
}

```
此时执行npx webpack,可以看到dist目录下新增了index.html文件，其中自动插入了script脚本，引入的是我们打包之后的js文件

### html-webpack-plugin的config妙用小技巧：
如果脚手架不只是给自己用，还给其他业务用，一个功能可能对应多个js或者css文件，每次都是业务自行修改public/index.html,也挺麻烦，首先要搞清楚每个功能要引入的文件，然后才能对index.html进行修改

此时我们可以新增一个配置文件，业务通过设置true或false来选出自己需要的功能，我们再根据配置文件的内容，为每个业务生成的html文件

现在可以试下，首先，在public目录下新增一个config.js(文件名随意)，将其内容设置为：
```
//public/config.js 除了以下的配置之外，这里面还可以有许多其他配置，例如,pulicPath 的路径等等
module.exports = {
  dev: {
    template: {
      title: '杨',
      header: false,
      footer: false,
    }
  },
  build: {
    template: {
      title: '锅巴',
      header: true,
      footer: false,
    }
  }
}
```
现在，修改下webpack.config.js
```
const HtmlWebpackPlugin = require('html-webpack-plugin');
const isDev = process.env.NODE_ENV === 'development';
const config = require('./public/config')[isDev ? 'dev' : 'build'];

module.exports = {
  mode: isDev ? 'development' : 'production',
  // ...
  plugins: [
    // 数组 存放所有的webpack插件
    new HtmlWebpackPlugin({
      template: './public/index.html',
      filename: 'index.html', // 打包后的文件名
      config: config.template,
      minify: {
        removeAttributeQuotes: false, // 是否删除属性的双引号
        collapseWhitespace: false, // 是否折叠空白
      },
      // hash: true // 是否加上hash，默认是false
    })
  ]
}
```

相应的，我们需要修改下我们的 public/index.html 文件(嵌入的js和css并不存在，仅作为示意)

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <% if(htmlWebpackPlugin.options.config.header) { %>
    <link rel="stylesheet" type="text/css" href="//common/css/header.css">
    <% } %>
    <title><%= (htmlWebpackPlugin.options.config.title) %></title>
</head>

<body>
</body> 
<% if(htmlWebpackPlugin.options.config.header) { %>
<script src="//common/header.min.js" type="text/javascript"></script> 
<% } %>
</html>
```

process.env中默认并没有NODE_ENV,这里配置下package.json的scripts
为了兼容window和mac跨平台设置环境变量，我们安装下cross-env:
npm install cross-env -D

然后配置下package.json
```
"script": {
  "dev": "cross-env NODE_ENV=development webpack",
  "build": "cross-env NODE_ENV=production webpack"
}
```

然后运行 npm run dev 和 npm run build, 对比下dist/index.html, 可以看到 npm run build生成的 index.html文件中引入了对应的css和js，并且对应的title也不一样

[更多](https://github.com/jantimon/html-webpack-plugin#configuration)配置

## 在浏览器中展示效果
先安装依赖:
npm install webpack-dev-server -D

修改下package.json 文件中的 scripts:
```
"script": {
  "dev": "cross-env NODE_ENV=development webpack-dev-server",
  "build": "cross-env NODE_ENV=production webpack"
}
```
更多<https://www.webpackjs.com/configuration/dev-server/>配置


## devtool
devtool 中的一些设置，可以帮助我们将编译后的的代码映射回原始源代码，不同的值会明显影响到到构建和重新构建的速度
添加配置:
// webpack.config.js
`
  module.exports = {
    devtool: isDev ? 'cheap-module-source-map' : 'source-map',
  }
  `


这边开发环境选择 'cheap-module-source-map',生产环境用'source-map';
更多<https://www.webpackjs.com/configuration/devtool/>


## 处理样式文件
webpack 不能处理css ，需要借助loader
> .css style-loader、css-loader、postcss-loader(处理兼容性问题)
> sass 和 less less-loader、sass-loader
这边配置下less和css文件
先安装下依赖：
npm install style-loader css-loader postcss-loader less-loader autoprefixer less -D

修改webpack.config.js
```
module.exports = {
    //...
    module: {
        rules: [
            {
                test: /\.(le|c)ss$/,
                use: ['style-loader', 'css-loader', {
                    loader: 'postcss-loader',
                    options: {
                        plugins: function () {
                            return [
                                require('autoprefixer')({
                                    "overrideBrowserslist": [
                                        ">0.25%",
                                        "not dead"
                                    ]
                                })
                            ]
                        }
                    }
                }, 'less-loader'],
                exclude: /node_modules/
            }
        ]
    }
}
```

测试下，在src下新建一个less文件,在 src/index.js 下引入
npm run dev启动页面，发现页面变成红色起效了

上面代码的配置：
(*) style-loader 动态创建style标签， 将 css 引入到 head 中
(*) css-loader 负责处理 @import 等语句
(*) post-loader 和 autoprefixer，自动生成浏览器兼容性前缀
(*) less-loader 负责处理编译 .less 将其转换成 css

注意：
loader 的执行顺序是从右向左执行的，也就是后面的 loader 先执行，上面 loader 的执行顺序为: less-loader ---> postcss-loader ---> css-loader ---> style-loader
当然，loader 其实还有一个参数，可以修改优先级，enforce 参数，其值可以为: pre(优先执行) 或 post (滞后执行)。
现在，我们已经可以处理 .less 文件啦，.css 文件只需要修改匹配规则，删除 less-loader 即可。


## 图片字体文件处理
处理本地资源文件：
(+) url-loader 
(+) file-loader
url-loader 功能类似于 file-loader，但是 url-loader 在文件大小（单位 byte）低于指定的限制时，可以返回一个 DataURL。
安装依赖:
npm install url-loader -D
控制台提示安装url-loader必须安装file-loader，继续安装file-loader：
npm install file-loader -D

修改webpack.config.js中的配置
```
module.exports = {
    //...
    modules: {
        rules: [
            {
                test: /\.(png|jpg|gif|jpeg|webp|svg|eot|ttf|woff|woff2)$/,
                use: [
                    {
                        loader: 'url-loader',
                        options: {
                            limit: 10240, //10K
                            esModule: false 
                        }
                    }
                ],
                exclude: /node_modules/
            }
        ]
    }
}

```

默认情况下，生成的文件的文件名就是文件内容的 MD5 哈希值并会保留所引用资源的原始扩展名,通过option参数进行修改
```
use: [
    {
        loader: 'url-loader',
        options: {
            limit: 10240, //10K
            esModule: false,
            name: '[name]_[hash:6].[ext]'
        }
    }
]

```
当本地资源较多时，有时候我们希望将他能打包在一个文件夹下，我们就要在url-loader 的 option 中指定 outpath ，如outputPath: 'asset'


## 处理html中的本地图片
安装html-withimg-loader
npm install html-withimg-loader -D
修改webpack.config.js
```
module.exports = {
    //...
    module: {
        rules: [
            {
                test: /.html$/,
                use: 'html-withimg-loader'
            }
        ]
    }
}

```

## 入口配置entry
```
//webpack.config.js
module.exports = {
    entry: './src/index.js' //webpack的默认配置
}
```
entry的值：
(*) 字符时，就是以对应的文件为入口
(*) 数组时，表示有多个主入口

## 出口配置
配置 output 选项可以控制 webpack 如何输出编译文件
```
//webpack.config.js
module.exports = {
    output: {
        path: path.resolve(__dirname, 'dist'), //必须是绝对路径
        filename: 'bundle.[hash].js',
        publicPath: '/' //通常是CDN地址
    }
}

```

## 没次打包前清空dist目录
安装插件：
npm install clean-webpack-plugin -D
webpack.config.js
```
const { CleanWebpackPlugin } = require('clean-webpack-plugin');

module.exports = {
    //...
    plugins: [
        //不需要传参数喔，它可以找到 outputPath
        new CleanWebpackPlugin() 
    ]
}
```

## 希望dist目录下某个文件夹不被清空
```
//webpack.config.js
module.exports = {
    //...
    plugins: [
        new CleanWebpackPlugin({
            cleanOnceBeforeBuildPatterns:['**/*', '!dll', '!dll/**'] //不删除dll目录下的文件
        })
    ]
}

```




原作者：刘小夕
参考原文链接：<https://juejin.im/post/5e5c65fc6fb9a07cd00d8838>
来源：掘金
