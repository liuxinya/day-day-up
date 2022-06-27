### 项目目录

```
> build
> public
> src
  > common
  > views
    > index
      index.tsx
      index.html
      index.less
    > demo
      demo.tsx
      demo.html
      demo.less
craco.config.js
webpack.tools.js

```

## craco.config.js
```js
const CracoLessPlugin = require('craco-less');
const webpack = require('webpack');
const {
    allSitePath,
    removeFromArrayByCondition,
    htmlPlugin,
    resolveApp,
    resolveModule,
} = require('./webpack.tools');


// 是否是线上环境
const isProd = process.env.NODE_ENV === 'production';

// 线上环境又分沙盒和正式线上
const buildEnv = process.env.IS_BUILD_ONLINE === 'true' ? 'online' : 'sandbox';

const publicPathMap = {
    'sandbox': 'aaa',
    'online': 'bbb',
};

// env 配置
// env设置必须放到 webpack 配置合并之前

// jsx
process.env.DISABLE_NEW_JSX_TRANSFORM = 'true';
if (isProd) {

    // 关掉webpack内联运行时的脚本
    process.env.INLINE_RUNTIME_CHUNK = 'false';

    // 关掉souremap
    process.env.GENERATE_SOURCEMAP = 'false';
}


module.exports = {
    webpack: {
        configure: (webpackConfig, {env, paths}) => {
            const isDev = env === 'development';
            // 修改基础paths
            paths.appHtml = resolveApp('src/views/index/index.html');
            paths.appIndexJs = resolveModule(resolveApp, 'src/views/index/index', paths);

            // 修改入口
            webpackConfig.entry = allSitePath(isDev, paths);

            // 修改出口
            webpackConfig.output.filename = isDev ? 'js/[name].bundle.js' : 'js/[name].[contenthash:8].js';
            webpackConfig.output.publicPath = isDev ? '/' : (publicPathMap[buildEnv] || '');

            // 删除插件 HtmlWebpackPlugin 和 ManifestPlugin
            let plugins = webpackConfig.plugins;
            removeFromArrayByCondition(plugins, item => {
                return item.constructor.name === 'HtmlWebpackPlugin' || item.constructor.name === 'ManifestPlugin';
            });
            // 增加插件
            plugins = [
                // html
                ...htmlPlugin(isDev, paths),
                // 环境变量
                new webpack.DefinePlugin({
                    'IS_BUILD_ONLINE': JSON.stringify(process.env.IS_BUILD_ONLINE),
                }),
                // 编译进度
                new webpack.ProgressPlugin(),
                ...plugins,
            ];
            webpackConfig.plugins = plugins;
            return webpackConfig;
        },
    },
    babel: {
        plugins: [
            'babel-plugin-transform-typescript-metadata',
            ['@babel/plugin-proposal-decorators', {'legacy': true}],
            ['@babel/plugin-proposal-class-properties', {'loose': true}],
            'react-hot-loader/babel',
        ],
        presets: [
            '@babel/preset-typescript',
        ],
    },
    plugins: [
        {
            plugin: CracoLessPlugin,
            options: {
                lessLoaderOptions: {
                    lessOptions: {
                        modifyVars: {'@primary-color': '#1DA57A'},
                        javascriptEnabled: true,
                    },
                },
            },
        },
    ],
};

```

## webpack.tools.js

```js

const path = require('path');
const fs = require('fs');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const glob = require('glob');


const appDirectory = fs.realpathSync(process.cwd());

const resolveApp = relativePath => path.resolve(appDirectory, relativePath);

// 可以通过命令控制只打包哪些模块
function getEntryFilesFormCommandLine(paths) {
    const argvs = process.argv;
    const flagIndex = argvs.indexOf('--page');
    let entryFiles = null;
    if (flagIndex !== -1) {
        entryFiles = argvs.slice(flagIndex + 1).map(item => {
            return `${paths.appSrc}/views/${item}`;
        });
    }
    return entryFiles;
}

// 查看文件是否存在
// 支持文件的后缀类型为 paths.moduleFileExtensions
function resolveModule(resolveFn, filePath, paths) {
    const extension = paths.moduleFileExtensions.find(extension =>
        fs.existsSync(resolveFn(`${filePath}.${extension}`))
    );
    if (extension) {
        return resolveFn(`${filePath}.${extension}`);
    }
    // 默认js
    return resolveFn(`${filePath}.js`);
}


// 根据指定条件从数组中删除某一项
// 用于删除webpack plugins中原有插件
function removeFromArrayByCondition(arr, condition, setValue = (arr3, ii, vv) => {
    arr3[ii] = vv;
}) {
    condition = (!condition || typeof condition !== 'function') ? () => false : condition;
    let count = 0;
    for (let i = 0; i < arr.length; i++) {
        if (condition(arr[i], i)) {
            count++;
        } else {
            count > 0 && (setValue(arr, i - count, arr[i]));
        }
    }
    for (let i = 0; i < count; i++) {
        arr.pop();
    }
    return arr;
}

// 多页面工具函数, 拿到所有的入口文件
function allSitePath(isEnvDevelopment, paths) {
    let entryFiles = getEntryFilesFormCommandLine(paths);
    !entryFiles && (entryFiles = glob.sync(paths.appSrc + '/views/*'));

    // eslint-disable-next-line prefer-const
    let map = {};
    entryFiles.forEach(item => {
        const filename = item.substring(item.lastIndexOf('/') + 1);
        // const filePath = `${item}/${filename}.tsx`;
        const filePath = resolveModule(resolveApp, `${item}/${filename}`, paths);
        map[filename] = [
            'react-hot-loader/patch',
            isEnvDevelopment && require.resolve('react-dev-utils/webpackHotDevClient'),
            filePath,
        ].filter(Boolean);
    });
    return map;
}

// 重写htmlPlugin，返回所有模块的hmlt plugin
function htmlPlugin(isEnvDevelopment, paths) {
    const fileNameLists = Object.keys(
        allSitePath(isEnvDevelopment, paths)
    );

    // eslint-disable-next-line prefer-const
    let arr = [];
    fileNameLists.forEach(item => {
        const filename = item.substring(item.lastIndexOf('/') + 1);

        arr.push(
            new HtmlWebpackPlugin(
                Object.assign(
                    {},
                    {
                        inject: true,
                        filename: item + '.html',
                        chunks: [item],
                        template: path.resolve(paths.appSrc, `views/${filename}/${filename}.html`),
                    },
                    isEnvDevelopment
                        ? undefined
                        : {
                            minify: {
                                removeComments: true,
                                collapseWhitespace: true,
                                removeRedundantAttributes: true,
                                useShortDoctype: true,
                                removeEmptyAttributes: true,
                                removeStyleLinkTypeAttributes: true,
                                keepClosingSlash: true,
                                minifyJS: true,
                                minifyCSS: true,
                                minifyURLs: true,
                            },
                        }
                )
            )
        );
    });
    return arr;
}


module.exports = {
    allSitePath,
    removeFromArrayByCondition,
    htmlPlugin,
    resolveApp,
    resolveModule,
};

```

----------

* 可通过`--page [模块名]`单独打包某一些模块，提升开发效率

* 如果想单独编译 `index`

```
yarn start --page index
```

* 如果想单独编译 `index`、`demo`、`other`

```
yarn start --page index demo other
```