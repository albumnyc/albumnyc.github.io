---
layout: post
title: webpack4&babel7
subtitle: webpack4&babel7
date: 2018-12-10
author: NYC
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
  - Git/webpack
---

## 项目升级 bable7

> bable7 已经出现 2 个月，各项性能已经稳定，应导师要求升级 babel7

    babel7更新的优点：
    1. 引入peerDependency上 @babel / core某些面向用户的封装（ 例如babel - loader， @babel / cli等）
    2.（ babel - polyfill 是通过向全局对象和内置对象的prototype上添加方法实现的， 会造成全局变量污染）。 Babel 提供了另外一种方案 transform - runtime，
    它在编译过程中只是将需要polyfill的代码引入了一个指向 core - js 中对应模块的链接(alias)

    安装方式
    npm i babel - upgrade - g
    babel - upgrade--write // 会帮助我们自动写出配置

    新版babel都是这样的结构
        "@babel/core": "^7.0.0",
        "@babel/preset-env": "^7.0.0",
        "@babel/preset-react": "^7.0.0",

        通过配置 polyfill为 false 或 useBuiltIns 为true(二选一即可)， 从而不引入 core - js 的 polyfill
    防止编译出来之后的文件过大
    其中useBuiltIns如果设为 "usage"，

    Babel 会根据实际代码中使用的ES6 / ES7代码， 以及与你指定的targets， 按需引入对应的 polyfill， 而无需在代码中直接引入
    import '@babel/polyfill'，
    避免输出的包过大， 同时又可以放心使用各种新语法特性

    @babel / plugin - proposal - decorators 必须在 @babel / plugin - proposal - class - properties 之前配置
        // 因为decorator是在class生命之前



    在升级过程中， webpack - cli注意版本号要高于3， 否则会报错
    由于失误不小心升级了less。 在less配置的时候， 手动加上javascriptEnabled: true;
    允许内敛js， 在2.X版本是默认支持的， 因为js为了安全防止内敛js导致报错

# webpack

    new webpack.DllReferencePlugin({
        context: __dirname, // 用来指导Webpack匹配manifest中库的路径
        manifest: require(path.join(__dirname, '../build/dll', 'dll-manifest.json')),
        //用来引入刚才输出的manifest文件
    }),
    new webpack.HotModuleReplacementPlugin(), // HMR
    new HtmlWebpackIncludeAssetsPlugin({
        assets: ['/vendors.dll.js'],
        append: false, //
        publicPath: '',
        hash: true,
    }),
    //动态顺序插入dll.js, vendor.js， manifest.js和页面功能js由htmlWebpackPlugin插入到最后
    new FriendlyErrorsPlugin(),
    // new HappyPack({
    //     id: 'happy-babel-js',
    //     loaders: ['babel-loader?cacheDirectory=true'],
    //     threadPool: happyThreadPool
    // }),
    // new HappyPack({
    //     id: 'happy-babel-ts',
    //     threadPool: happyThreadPool,
    //     loaders: [
    //         {
    //             path: 'ts-loader',
    //             query: { happyPackMode: true }
    //         }
    //     ]
    // }),
    new ProgressBarPlugin({
        // build的进度条
        format: '  build [:bar] ' + chalk.green.bold(':percent') + ' (:elapsed seconds)',
    }),

## webapck.config.dll.js

    module.exports = {
        mode: 'development',
        entry: {
            vendors: [
                'react',
                'react-router-dom',
                'antd',
                'prop-types',
                'immer',
                'fbemitter',
                'crypto-js',
                'axios',
                'classnames',
                'babel-polyfill',
                'mobx',
                'mobx-react',
                'react-dom',
                'echarts',
                'moment',
                'fetch-polyfill',
            ],
        },
        output: {
            path: path.resolve(__dirname, '../build/dll'),
            filename: '[name].dll.js',
            library: '[name]',
        },
        plugins: [
            new webpack.DllPlugin({
                context: __dirname,
                name: '[name]',
                path: path.join(__dirname, '../build/dll', 'dll-manifest.json'),
            }),
        ],
    };
