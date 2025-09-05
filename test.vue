// vue.config.js
const { defineConfig } = require('@vue/cli-service')

module.exports = defineConfig({
  transpileDependencies: true,
  
  // 方式1: 使用 chainWebpack (推荐)
  chainWebpack: config => {
    // 图片处理规则
    config.module
      .rule('images')
      .test(/\.(png|jpe?g|gif|webp|svg)(\?.*)?$/)
      .use('url-loader')
      .loader('url-loader')
      .tap(options => {
        options.limit = 20 * 1024 // 20KB
        options.fallback = {
          loader: 'file-loader',
          options: {
            name: 'img/[name].[hash:8].[ext]',
            esModule: false
          }
        }
        return options
      })
  },

  // 或者方式2: 使用 configureWebpack
  // configureWebpack: {
  //   module: {
  //     rules: [
  //       {
  //         test: /\.(png|jpe?g|gif|webp|svg)(\?.*)?$/,
  //         use: {
  //           loader: 'url-loader',
  //           options: {
  //             limit: 20 * 1024,
  //             name: 'img/[name].[hash:8].[ext]',
  //             esModule: false
  //           }
  //         }
  //       }
  //     ]
  //   }
  // }
})