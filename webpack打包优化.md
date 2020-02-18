## webpack打包优化

- 首先是学会去如何分析`webpack`打包的各个模块，如果减小模块的体积。目前比较主流的分析是通过`webpack`插件`webpack-bundle-analyzer`实现的。它是以treemap的形式展现出来，很形象直观，还有一些具体的交互形式。既可以查看你项目中用到的所有依赖，也可以直观看到各个模块体积在整个项目中的占比。
  * 使用方法:
    1. 首先安装对应的npm包，`npm install webpack-bundle-analyzer --save-dev`。
    2. 在webpack配置中的`plugins`中添加插件，`plugins: [new BundleAnalyzerPlugin()]`。
- 移除不必要或过大的文件。针对于像`moment`、`lodash`这样的第三方工具库，如果不做特殊优化，打包后体积会很大。可以采用`cdn`的方式在`html`中引入，或者使用`webpack`配置对其进行优化。
  * 以`moment`为例，使用`webpack`优化，主要用到了`IgnorePlugin`和`ContextReplacementPlugin`这两个库。
    ```js
    // 插件配置
    plugins: [  // 忽略moment.js中所有的locale文件  
    new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/),],
    // 使用方式
    const moment = require('moment');
    // 引入zh-cn locale文件
    require('moment/locale/zh-cn');
    moment.locale('zh-cn');
    ...
    ...
    // 插件配置
    plugins: [  // 只加载locale zh-cn文件  
    new webpack.ContextReplacementPlugin(/moment[\/\\]locale$/, /zh-cn/),],
    // 使用方式
    const moment = require('moment');
    moment.locale('zh-cn');
    ```
- 使用`DLL`动态链接库，把不常更新的 `module` 进行编译打包，然后每次开发和上线就只针对开发过程中相关的文件进行打包。主要用到了`DllPlugin`和`DLLReferencePlugin`两个插件，具体使用方法可以参照博客[实践`DllPlugin`来优化`webpack`打包速度](https://juejin.im/entry/57a6dee4a633bd00604d0e73)。
- 按需加载模块。以蚂蚁金服的`antd`作为示例。以下是官网提供的方法。
  ```js
  // .babelrc or babel-loader option
  {  
    "plugins": [    
      ["import", 
        {      
          "libraryName": "antd",      
          "libraryDirectory": "es",      
          "style": "css" 
          // `style: true` 会加载 less 文件    
        }
      ]  
    ]
  }
  // 基本使用
  import { DatePicker } from 'antd';
  ```
- 异步加载模块。对于页面中不会立即使用到的模块，可以采用异步加载的方式，提高渲染速度。
  ```js
  // 通常
  import foo from "foo";
  // 异步
  const foo = () => import("foo");
  ```
- 针对于生产环境，压缩混淆代码并移除console。主要是通过插件`UglifyJSPlugin`实现。
  ```js
  new webpack.optimize.UglifyJsPlugin(
  {
    compress: {    
      warnings: false,
      drop_console: true,    
      pure_funcs: ['console.log']  
    },  
    sourceMap: false
  }
  )
  ```
- 使用`HappyPack`提升打包速度。`HappyPack`是让`webpack`对`loader`的执行过程，从单一进程形式扩展为多进程模式，也就是将任务分解给多个子进程去并发的执行，子进程处理完后再把结果发送给主进程。从而加速代码构建 与 `DLL`动态链接库结合来使用更佳。
  ```js
  const HappyPack = require('happypack');
  const os = require('os'); 
  // node 提供的系统操作模块 
  // 根据我的系统的内核数量 指定线程池个数 也可以其他数量
  const happyThreadPool = HappyPack.ThreadPool({
    size: os.cpus().length
  })
  module: {    
    rules: [       
      {           
        test: /\.js$/,           
        use: 'happypack/loader?id=babel',           
        exclude: /node_modules/,           
        include: 
        path.resolve(__dirname, 'src')       
      }    
    ]
  },
  plugins: [    
    new HappyPack({ // 基础参数设置        
    id: 'babel', // 上面loader?后面指定的id        
    loaders: ['babel-loader?cacheDirectory'], // 实际匹配处理的loader        
    threadPool: happyThreadPool,        // cache: true // 已被弃用        
    verbose: true    
    });
  ]
  ```
- 使用`Tree Shaking`移除`JavaScript`中用不上的代码，它依赖静态的ES6模块化语法，例如通过impot和export导入导出。在`mode`为`production`的情况下，会默认开启`Tree Shaking`。
- 优化`webpack`相关属性的配置。
  1. 优化`resolve.modules`配置。减少`webpack`对于文件检索的范围，提高打包效率。
    ```js
    const path = require('path');
    function resolve(dir) { 
      // 转换为绝对路径   
      return path.join(__dirname, dir);
    }
    resolve: {    
      modules: [ 
      // 优化模块查找路径        
      path.resolve('src'),        
      path.resolve('node_modules') 
      // 指定node_modules所在位置 当你import 第三方模块时 直接从这个路径下搜索寻找    
      ]
    }
    ```
  * 优化`loader`配置。主要是缩小匹配范围和缓存`loader`的执行结果，同样是为了提高`webpack`匹配文件的速度，从而提高文件打包效率。
    ```js
    module: {
      rules: [        
        {            
          test: /\.js$/,            
          use: 'babel-loader?cacheDirectory', // 缓存loader执行结果 发现打包速度已经明显提升了            
          exclude: /node_modules/, // 排除node_modules文件夹           
          include: path.resolve(__dirname, 'src')        
        }    
      ]
    }
    ```
  * 合理使用`module.noParse`。用了`noParse`的模块将不会被`loaders`解析，所以当我们使用的库如果太大，并且其中不包含`import` `require`、`define`的调用，我们就可以使用这项配置来提升性能, 让`Webpack`忽略对部分没采用模块化的文件的递归解析处理。
    ```js
    // 忽略对jquery lodash的进行递归解析
    module: {    
    // noParse: /jquery|lodash/    
    // 从 webpack 3.0.0 开始    
      noParse: function(content) {        
        return 
    /jquery|lodash/.test(content)    
      }
    }
    ```
### 参考博客

* 可以直接看博客，讲的更加详细，我只是简单汇总了一下。
* [https://juejin.im/entry/5abbc5596fb9a028d5672b13](https://juejin.im/entry/5abbc5596fb9a028d5672b13)
* [https://www.jeffjade.com/2017/08/12/125-webpack-package-optimization-for-speed/](https://www.jeffjade.com/2017/08/12/125-webpack-package-optimization-for-speed/)
* [https://www.jeffjade.com/2017/08/06/124-webpack-packge-optimization-for-volume/](https://www.jeffjade.com/2017/08/06/124-webpack-packge-optimization-for-volume/)
* [https://juejin.im/entry/5b1e3098e51d4506d5367fc6](https://juejin.im/entry/5b1e3098e51d4506d5367fc6)
