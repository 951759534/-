# webpack3.0迁移指南
1.是什么?
webpack可以看作是模块打包机  分析项目结构 将Javascript转换打包成合适
格式转换给浏览器使用   
   1. 新建项目   
   2. 执行npm init  新建package.json 文件 npm install webpack --save-dev     
   3. 在项目目录下新建webpack.config.js  使用commonJs的写法暴露出一个对象
  entry: 配置文件 入口 可以是单一入口 也可以是多入口   参数对象 key:入口名称 value: 入口位置    
  output  配置打包文件后的格式规范   
  moudule  配置模块 各种loader 
  plugins  配置插件 根据不同需要配置不同功能的插件  
  devServer 配置开发环境下的服务器
  
  output
    
              output 对象 包含 
              filename://输出文件名称   
              path: //输出文件位置     
              publicPath:  //输出文件 引入资源位置  
  
  
  
  module  
  
              通过使用loader  webpack可以对不同的格式文件进行特殊处理 
              可以吧es6或es7预防转换成大多浏览器兼容的代码
              module对象包含rules[]  
              rules下 是对象 
              例
                 module:{
                        rules: [
                            {
                              test: /\.css$/,
                              use: [ 'style-loader', 'css-loader' ]
                            }
                          ]
                    },
               
               
    
  plugin   
              
              插件
              列举插件作用 清除dist目录
              分离js中css  当然 在loader 中也要进行特殊处理  
              模块热更新插件 提升开发效率     
              
              
  devServer
  
            hot:  boolean  是否启用模块热替换 
            host  开启服务名称 希望外部访问 可替换成0.0.0.0
            inline:  是否启用内联模式   
            proxy: 是否启用代理  
            compress 是否开启网络压缩
            
  
  
  webpack优化  
  
          生产环境下  在插件中添加js压缩  html压缩 在loader  添加压缩  
          生成带有hash值文件  便于静态资源版本管理   
          将公用库文件在入口文件提取出来  在插件中使用  new webpack.optimize.CommonsChunkPlugin 提取公共库  
          如果是单页面应用将公共的css提取出来 
          在jsloader中使用cacheh缓存 使rebuild更快  
          在jsloader中 使用exclude将node-module排除   
          在Webpack3.0 版本中，提供了一个新的功能：Scope Hoisting，
          又译作“作用域提升”。在Webpack2中，打包后的文件里每个模块都会被包装在一个单独的闭包中，这些闭包会导致JS执行速度变慢，
          Scope Hoisting则可以将所有模块打包进一个大的闭包中。只需在配置文件中添加一个新的插件，
          就可以让 Webpack 打包出来的代码文件更小、运行的更快：
          
  编写文件 webpack 
     如下
     
            const webpack = require('webpack');
            const path = require('path');
            const HtmlWebpackPlugin = require("html-webpack-plugin");
            const ExtractTextPlugin = require('extract-text-webpack-plugin') // 将你的行内样式提取到单独的css文件里,
            const UglifyJSPlugin = require('uglifyjs-webpack-plugin')
            const glob = require('glob')
            const CleanWebpackPlugin = require('clean-webpack-plugin');
            
            const NODE_ENV = process.env.NODE_ENV
            const CDN_BASE_DIR = ''
            const CDN_HOST = ''
            const isPro = (NODE_ENV === 'pro')
            const webpackConfig = {
              entry : {
                vendor: [
                  'react',
                  'react-dom',
                ],
              },
              output:{
                path: path.join(__dirname, `dist/${CDN_BASE_DIR}`), // 打包后生成的目录
                publicPath: isPro ? `${CDN_HOST}/${CDN_BASE_DIR}/` : '/', // 模板、样式、脚本、图片等资源对应的server上的路径
                filename: `js/${isPro ? '[name]-[chunkhash:8].js' : '[name].js'}`,
                chunkFilename: `js/${isPro ? '[name]-[chunkhash:8].js' : '[name].js'}`
              },
              module: {
                rules: [
                  {
                    test: /\.css$/,
                    use:
                    //["style-loader", "css-loader", "postcss-loader"]
                        ExtractTextPlugin.extract({
                          fallback: "style-loader",
                          use: ["css-loader" , "postcss-loader"]
                        })
                  },
                  {
                    test: /\.js$/,
                    exclude: /node_modules/,
                    use: {
                      loader: 'babel-loader?cacheDirectory'
                    }
                  },
                  {
                    test: /\.(gif|png|jpe?g|svg)$/i,
                    use: [{
                      loader: 'url-loader',
                      options: {
                        limit: 1024,
                        name:isPro?'images/[name]-[hash:8].[ext]':'images/[name].[ext]'
                      }
                    }]
                  }
                ]
              },
              resolve: {
                extensions: [" ", '.js', '.css'],
              },
              plugins:[
                new webpack.optimize.CommonsChunkPlugin({
                  name: 'vendor',
                  filename: 'js/vendor.js'
                }),
                new webpack.HotModuleReplacementPlugin({
                  warnings: false
                }),
                new ExtractTextPlugin(isPro?'css/[name]-[hash:8].css':'css/[name].css'),
                new webpack.optimize.ModuleConcatenationPlugin()
              ],
              devServer:{
                hot: true,
                inline:true,
                port:8080
              }
            };
            if(NODE_ENV === "pro"){
              webpackConfig.plugins.push( new CleanWebpackPlugin( ['dist/*',],　 //匹配删除的文件
                  {
                    root: __dirname,       　　　　　　　　　　//根目录
                    verbose:  true,        　　　　　　　　　　//开启在控制台输出信息
                    dry:      false        　　　　　　　　　　//启用删除文件
                  }))
              webpackConfig.plugins.push(new UglifyJSPlugin())
              webpackConfig.plugins.push(new webpack.DefinePlugin({
                'process.env': {
                  NODE_ENV: JSON.stringify('production'),
                },
              }))
            }else{
              webpackConfig.entry['main'] =  path.resolve(__dirname,'./src/app.js')
              webpackConfig.plugins.push(new HtmlWebpackPlugin({
                template: path.resolve(__dirname, "src/index.html"),
                name: "index",
                filename: "index.html",
                inject: true
              }))
            }
            const entries = glob.sync('./src/**/index.js')
            for (const path of entries) {
              const chunkName = path.slice('./src/'.length, -'/index.js'.length)
              webpackConfig.entry[chunkName] = path
              webpackConfig.plugins.push(new HtmlWebpackPlugin({
                template: path.replace('index.js', 'index.html'),
                filename: chunkName + '.html',
                chunks: ['vendor', chunkName],
                inject: true
              }))
            }
            module.exports = webpackConfig
       
         webpack.config.js分为五个大部分
  
                module.exports={
                                   //入口文件的配置项
                                   entry:{},
                                   //出口文件的配置项
                                   output:{},
                                   //模块：例如解读CSS,图片如何转换，压缩
                                   module:{},
                                   //插件，用于生产模版和各项功能
                                   plugins:[],
                                   //配置webpack开发服务功能
                                   devServer:{}
                       }
          
              
              
    
