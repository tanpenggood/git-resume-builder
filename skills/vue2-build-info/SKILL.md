---
name: vue2-build-info
description: Vue2项目打包构建信息注入（Webpack）。在index.html中注入打包环境与构建时间戳，用于版本追溯。Use when user says "注入版本号"、"注入环境信息"、"构建信息"、"build info"或需要在打包时记录环境和时间信息。
---

# Vue2 构建信息注入（Webpack）

## 功能

在 Vue 项目打包构建时，自动向 `index.html` 注入以下信息：
- **构建环境** (buildEnv): development / staging / production
- **构建时间戳** (buildTimestamp): yyyy-MM-dd HH:mm:ss 格式的构建时间

注入方式：通过 `<meta>` 标签嵌入页面，便于前端读取和运维排查。

## 前置条件

- Vue2 + Webpack 项目（使用 Vue CLI 创建，vue.config.js 配置）
- 项目中存在 `public/index.html` 文件

## 使用步骤

### 1. 修改 vue.config.js

在 `chainWebpack` 函数中添加以下配置：

```javascript
chainWebpack(config) {
    // ... 其他配置

    // 生成 yyyy-MM-dd HH:mm:ss 格式的时间戳
    const now = new Date();
    const buildTimestamp = `${now.getFullYear()}-${(now.getMonth() + 1).toString().padStart(2, '0')}-${now.getDate().toString().padStart(2, '0')} ${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;

    // 向 HtmlWebpackPlugin 传递自定义选项
    config.plugin('html').tap(args => {
        args[0].buildEnv = process.env.VUE_APP_ENV; // 传递构建环境
        args[0].buildTimestamp = buildTimestamp;     // 传递构建时间戳
        return args;
    });

    // ... 其他配置
}
```

### 2. 修改 public/index.html

在 `<head>` 标签内添加以下 meta 标签：

```html
<head>
    <!-- ... 其他内容 -->

    <!-- 构建信息注入 -->
    <meta name="buildEnv" content="<%= htmlWebpackPlugin.options.buildEnv %>">
    <meta name="buildTimestamp" content="<%= htmlWebpackPlugin.options.buildTimestamp %>">
</head>
```

### 3. 前端读取构建信息（可选）

在 Vue 组件中读取注入的构建信息：

```javascript
// utils/buildInfo.js
export function getBuildInfo() {
    return {
        env: document.querySelector('meta[name="buildEnv"]')?.content || 'unknown',
        timestamp: document.querySelector('meta[name="buildTimestamp"]')?.content || 'unknown'
    };
}
```

## 完整示例

### vue.config.js 关键配置

```javascript
'use strict';
const path = require('path');

function resolve(dir) {
    return path.join(__dirname, dir);
}

module.exports = {
    publicPath: process.env.VUE_APP_BASE_URL || '/',
    outputDir: 'dist',
    assetsDir: 'static',
    lintOnSave: false,
    productionSourceMap: false,

    chainWebpack(config) {
        config.plugins.delete('preload');
        config.plugins.delete('prefetch');

        // 生成 yyyy-MM-dd HH:mm:ss 格式的时间戳
        const now = new Date();
        const buildTimestamp = `${now.getFullYear()}-${(now.getMonth() + 1).toString().padStart(2, '0')}-${now.getDate().toString().padStart(2, '0')} ${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`;

        // 向 HtmlWebpackPlugin 传递自定义选项
        config.plugin('html').tap(args => {
            args[0].buildEnv = process.env.VUE_APP_ENV; // 传递构建环境
            args[0].buildTimestamp = buildTimestamp;     // 传递构建时间戳
            return args;
        });

        // ... 其他配置
    },
};
```

### index.html 关键配置

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
    <meta name="renderer" content="webkit">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">

    <!-- 构建信息注入 -->
    <meta name="buildEnv" content="<%= htmlWebpackPlugin.options.buildEnv %>">
    <meta name="buildTimestamp" content="<%= htmlWebpackPlugin.options.buildTimestamp %>">

    <link rel="icon" href="<%= BASE_URL %>favicon.ico">
    <title><%= webpackConfig.name %></title>
</head>
<body>
    <div id="app"></div>
</body>
</html>
```

## 环境变量说明

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| VUE_APP_ENV | 构建环境 | development / staging / production |
| buildEnv | 注入到页面的环境标识 | development / staging / production |
| buildTimestamp | 注入到页面的构建时间 | 2025-11-18 15:05:23 |

## 常见问题

### Q: 为什么不用 filenameHashing 添加版本号？

A: 本方案使用时间戳作为版本标识，更直观且便于追溯。如需禁用 hash，可设置：
```javascript
module.exports = {
    filenameHashing: false, // 打包时不使用hash值
};
```

### Q: 如何在不同环境使用不同配置？

A: 可通过 `.env` 文件设置 `VUE_APP_ENV`：
```bash
# .env.development
VUE_APP_ENV=development

# .env.production
VUE_APP_ENV=production
```

## 参考

- [Vue CLI 配置参考](https://cli.vuejs.org/zh/config/)
- [Webpack HtmlWebpackPlugin](https://github.com/jantimon/html-webpack-plugin)
