// vue.config.js
module.exports = {
  chainWebpack: (config) => {
    // 修改图片规则
    config.module
      .rule('images')
      .test(/\.(png|jpe?g|gif|webp|avif)(\?.*)?$/)
      .use('url-loader')
      .loader('url-loader')
      .tap(options => {
        // 修改 limit 参数为 20KB
        options.limit = 20 * 1024; // 20KB
        return options;
      });
  }
};