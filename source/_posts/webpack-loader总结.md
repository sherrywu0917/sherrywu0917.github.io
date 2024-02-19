---
title: webpack-loader总结
date: 2023-11-02 12:12:12
tags: [webpack, loader]
---

### css处理loader
#### module css处理
- style-loader: 处理css插入方式：inline或者extract，SSR模式下使用isomorphic-style-loader
- css-loader: 处理css中资源引用
- resolve-url-loader: 处理sass中import和url()相对路径
- post-loader: 添加浏览器前缀autoprefixer｜生产模式中压缩cssnano | rem转换postcss-px2rem
- sass-loader: 将sass/scss编译为css

#### jsx css处理
- babel-loader: 结合babel的presets和plugins配置处理js
- styled-jsx: 处理jsx格式写的style
``` js
export default () => (
  <div className="root">
    <style jsx>{`
      .root {
        color: green;
      }
    `}</style>
  </div>
)
```
- post-loader: 添加浏览器前缀autoprefixer｜生产模式中压缩cssnano
- sass-loader: 将sass/scss编译为css

#### MiniCssExtractPlugin
和`style-loader`功能相似：This plugin should be used only on production builds without style-loader in the loaders chain, especially if you want to have HMR in development.
``` js
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const devMode = process.env.NODE_ENV !== 'production';

module.exports = {
  plugins: [
    new MiniCssExtractPlugin({
      // Options similar to the same options in webpackOptions.output
      // both options are optional
      filename: devMode ? '[name].css' : '[name].[hash].css',
      chunkFilename: devMode ? '[id].css' : '[id].[hash].css',
    }),
  ],
  module: {
    rules: [
      {
        test: /\.(sa|sc|c)ss$/,
        use: [
          {
            loader: MiniCssExtractPlugin.loader,
            options: {
              hmr: process.env.NODE_ENV === 'development',
            },
          },
          'css-loader',
          'postcss-loader',
          'sass-loader',
        ],
      },
    ],
  },
};
```

### babel配置
- presets预设配置
`@babel/env`和`@babel/preset-env`的写法一致，babel已经知道它是一个preset，会自动加上`preset-`
``` js
presets: [
    ['@babel/env', {
        modules: 'commonjs',
        // todo 取消注释编译报错，参考：https://github.com/babel/babel/issues/8559
        targets: {
            node: true
        }
    }],
    '@babel/react',
    '@babel/preset-typescript',
]
```
