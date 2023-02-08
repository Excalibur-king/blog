# 前言

我在公司实习的时候参与的一个 react + typescript 的项目，这个项目在自动化部署测试环境时很慢，每次都要 10 分钟左右。当时我抱着研究研究的心态，安装了 speed-measure-webpack-plugin 插件。测试发现 TerserPlugin、babel-loader 这哥俩耗时贼长，项目打包耗时6分钟左右。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbf7f072a8cc467883ffb588a2792e85~tplv-k3u1fbpfcp-watermark.image?)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ae52e1d44014f55978a839c48b6bfaf~tplv-k3u1fbpfcp-watermark.image?)

# esbuild

在调查研究优化方案的时候，发现了 [esbuild](https://esbuild.github.io/) , an extremely fast bundler for the web

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/889c368195d44c8b9808e96f10a366d7~tplv-k3u1fbpfcp-watermark.image?)

在了解到有 [esbuild-loader](https://github.com/privatenumber/esbuild-loader)后，便萌生想用它替代 babel-loader 的念头。看看文档。

# esbuild-loader

⚠️ esbuild is not stable yet and can have dramatic differences across releases. Using a different version of esbuild is not guaranteed to work.

翻译一下：esbuild 还不稳定，并且在不同版本之间可能存在巨大差异。不能保证可以正常工作。

嗯~，看来我们最好使用稳定版本的 esbuild。

[*esbuild-loader*](https://github.com/privatenumber/esbuild-loader) lets you harness the speed of esbuild in your Webpack build by offering faster alternatives for transpilation (eg. babel-loader/ts-loader) and minification (eg. Terser)!

翻译一下：esbuild-loader 通过提供更快的转译（例如 babel-loader/ts-loader）和缩小（例如 Terser）替代方案，让您在 Webpack 构建中利用 esbuild 的速度！

看来官方确实推荐使用 esbuild-loader 呢，除了替换 babel-loader，也提供了 ESBuildMinifyPlugin 替换 terser-plugin。

当然官方也有压缩 css 的方案。

# craco + esbuild-loader 改写 rca

接下来就是实践环节，我们使用 [craco](https://github.com/dilanx/craco) 来改写 react-create-app 的 webpack 配置

craco 提供了 loaderByname、addAfterLoader、removeLoaders api 查找、增加、删除 Webpack Config 中的 loader，试一下

```
const {
  loaderByName,
  removeLoaders,
  addAfterLoader,
  addPlugins,
  removePlugins,
  pluginByName,
} = require('@craco/craco');
const { ESBuildMinifyPlugin } = require('esbuild-loader');
module.exports = {
  webpack: {
    configure: (webpackConfig, { env, paths }) => {
      addAfterLoader(webpackConfig, loaderByName('babel-loader'), {
        test: /\.(js|ts|tsx|jsx)$/,
        include: [paths.appSrc],
        loader: require.resolve('esbuild-loader'),
        options: {
          loader: 'tsx',
          target: 'es2016',
        },
      });
      removeLoaders(webpackConfig, loaderByName('babel-loader'));
      addPlugins(
        webpackConfig,
        new ESBuildMinifyPlugin({
          target: 'es2016',
        })
      );
      removePlugins(webpackConfig, pluginByName('TerserPlugin'));
      return smp.wrap(webpackConfig);
    },
  }
};
```
 build 一下，`yarn craco build`

啊这，报错了？？？
```js
(node:41832) UnhandledPromiseRejectionWarning: TypeError: matcher is not a function
```
查了一下 craco 的 [issue](https://github.com/dilanx/craco/issues/159) ，发现官方推荐字面量的写法来增加、删除插件。


```js
plugins: {
  add: [
    new ESBuildMinifyPlugin({
      target: 'es2016',
    }),
  ],
  remove: ['TerserPlugin'],
},
```

看看效果，相当不错：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f1282ed0f9049e4950db660f52ddb28~tplv-k3u1fbpfcp-watermark.image?)

(PS: 好奇怪，为什么没能把 TerserPlugin 删掉)
# craco-esbuilt

查找相关资料的时候发现了有插件 [craco-esbuild](https://www.npmjs.com/package/craco-esbuild) 实现了 rca 接入 esbuild-loader 这一过程。

craco-esbuild 利用了 craco 提供了 [overrideWebpackConfig](https://craco.js.org/docs/plugin-api/hooks/#overridewebpackconfig) hook，将替换 babel-loader，TerserPlugin 的逻辑封装到插件中，同时增加了 options 的判断与获取，重写了增加、删除压缩的方法。放源码。


```js
const fs = require('fs');
const { loaderByName, removeLoaders, addAfterLoader } = require('@craco/craco');
const { ESBuildMinifyPlugin } = require('esbuild-loader');

const removeMinimizer = (webpackConfig, name) => {
  const idx = webpackConfig.optimization.minimizer.findIndex(
    (m) => m.constructor.name === name
  );
  webpackConfig.optimization.minimizer.splice(idx, 1);
};

const replaceMinimizer = (webpackConfig, name, minimizer) => {
  const idx = webpackConfig.optimization.minimizer.findIndex(
    (m) => m.constructor.name === name
  );
  idx > -1 && webpackConfig.optimization.minimizer.splice(idx, 1, minimizer);
};

module.exports = {
  /**
   * To process the js/ts files we replace the babel-loader with the esbuild-loader
   */
  overrideWebpackConfig: ({
    webpackConfig,
    pluginOptions,
    context: { paths },
  }) => {
    const useTypeScript = fs.existsSync(paths.appTsConfig);
    const esbuildLoaderOptions =
      pluginOptions && pluginOptions.esbuildLoaderOptions;

    // add includePaths custom option, for including files/components in other folders than src
    // Used as in addition to paths.appSrc, optional parameter.
    const optionalIncludes =
      (pluginOptions && pluginOptions.includePaths) || [];

    // add esbuild-loader
    addAfterLoader(webpackConfig, loaderByName('babel-loader'), {
      test: /\.(js|mjs|jsx|ts|tsx)$/,
      include: [paths.appSrc, ...optionalIncludes],
      loader: require.resolve('esbuild-loader'),
      options: esbuildLoaderOptions
        ? esbuildLoaderOptions
        : {
            loader: useTypeScript ? 'tsx' : 'jsx',
            target: 'es2015',
          },
    });

    // remove the babel loaders
    removeLoaders(webpackConfig, loaderByName('babel-loader'));

    // Replace terser with esbuild
    const minimizerOptions = (pluginOptions || {}).esbuildMinimizerOptions || {
      target: 'es2015',
      css: true,
    };
    replaceMinimizer(
      webpackConfig,
      'TerserPlugin',
      new ESBuildMinifyPlugin(minimizerOptions)
    );
    // remove the css OptimizeCssAssetsWebpackPlugin
    if (minimizerOptions.css) {
      removeMinimizer(webpackConfig, 'OptimizeCssAssetsWebpackPlugin');
    }
    return webpackConfig;
  },

  /**
   * To process the js/ts files we replace the babel-loader with the esbuild jest loader
   */
  overrideJestConfig: ({ jestConfig, pluginOptions }) => {
    if (pluginOptions && pluginOptions.skipEsbuildJest) return jestConfig;

    const defaultEsbuildJestOptions = {
      loaders: {
        '.js': 'jsx',
        '.test.js': 'jsx',
        '.ts': 'tsx',
        '.test.ts': 'tsx',
      },
    };

    const esbuildJestOptions =
      (pluginOptions && pluginOptions.esbuildJestOptions) ||
      defaultEsbuildJestOptions;

    // Replace babel transform with esbuild
    // babelTransform is first transformer key
    /*
    transform:
      {
        '^.+\\.(js|jsx|mjs|cjs|ts|tsx)$': 'node_modules\\react-scripts\\config\\jest\\babelTransform.js',
        '^.+\\.css$': 'node_modules\\react-scripts\\config\\jest\\cssTransform.js',
        '^(?!.*\\.(js|jsx|mjs|cjs|ts|tsx|css|json)$)': 'node_modules\\react-scripts\\config\\jest\\fileTransform.js'
      }
    */
    const babelKey = Object.keys(jestConfig.transform)[0];

    // We replace babelTransform and add loaders to esbuild-jest
    jestConfig.transform[babelKey] = [
      require.resolve('esbuild-jest'),
      esbuildJestOptions,
    ];

    // Adds loader to all other transform options (2 in this case: cssTransform and fileTransform)
    // Reason for this is esbuild-jest plugin. It considers only loaders or other options from the last transformer
    // You can see it for yourself in: /node_modules/esbuild-jest/esbuid-jest.js:21 getOptions method
    // also in process method line 32 gives empty loaders, because options is already empty object
    // Issue reported here: https://github.com/aelbore/esbuild-jest/issues/18
    Object.keys(jestConfig.transform).forEach((key) => {
      if (babelKey === key) return; // ebuild-jest transform, already has loader

      // Checks if value is array, usually it's not
      // Our example is above on 70-72 lines. Usually default is: {"\\.[jt]sx?$": "babel-jest"}
      // (https://jestjs.io/docs/en/configuration#transform-objectstring-pathtotransformer--pathtotransformer-object)
      // But we have to cover all the cases
      if (
        Array.isArray(jestConfig.transform[key]) &&
        jestConfig.transform[key].length === 1
      ) {
        jestConfig.transform[key].push(esbuildJestOptions);
      } else {
        jestConfig.transform[key] = [
          jestConfig.transform[key],
          esbuildJestOptions,
        ];
      }
    });

    return jestConfig;
  },
};

```



