---
title: Production
sort: 8
contributors:
  - henriquea
  - rajagopal4890
  - markerikson
  - simon04
  - kisnows
  - chrisVillanueva
  - swapnilmishra
  - bring2dip
  - redian
  - skipjack
  - xgqfrms
  - kelset
  - xgirma
  - mehrdaad
---

In this guide we'll dive into some of the best practices and utilities for building a production site or application.

T> This walkthrough stems from [Tree Shaking](/guides/tree-shaking) and [Development](/guides/development). Please ensure you are familiar with the concepts/setup introduced in those guides before continuing on.


## Setup

The goals of _development_ and _production_ builds differ greatly. In _development_, we want strong source mapping and a localhost server with live reloading or hot module replacement. In _production_, our goals shift to a focus on minified bundles, lighter weight source maps, and optimized assets to improve load time. With this logical separation at hand, we typically recommend writing __separate webpack configurations__ for each environment.

While we will separate the _production_ and _development_ specific bits out, note that we'll still maintain a "common" configuration to keep things DRY. In order to merge these configurations together, we'll use a utility called [`webpack-merge`](https://github.com/survivejs/webpack-merge). With the "common" configuration in place, we won't have to duplicate code within the environment-specific configurations.

Let's start by installing `webpack-merge` and splitting out the bits we've already worked on in previous guides:

``` bash
npm install --save-dev webpack-merge
```

__project__

``` diff
  webpack-demo
  |- package.json
- |- webpack.config.js
+ |- webpack.common.js
+ |- webpack.dev.js
+ |- webpack.prod.js
  |- /dist
  |- /src
    |- index.js
    |- math.js
  |- /node_modules
```

__webpack.common.js__

``` diff
+ const path = require('path');
+ const CleanWebpackPlugin = require('clean-webpack-plugin');
+ const HtmlWebpackPlugin = require('html-webpack-plugin');
+
+ module.exports = {
+   entry: {
+     app: './src/index.js'
+   },
+   plugins: [
+     new CleanWebpackPlugin(['dist']),
+     new HtmlWebpackPlugin({
+       title: 'Production'
+     })
+   ],
+   output: {
+     filename: '[name].bundle.js',
+     path: path.resolve(__dirname, 'dist')
+   }
+ };
```

__webpack.dev.js__

``` diff
+ const merge = require('webpack-merge');
+ const common = require('./webpack.common.js');
+
+ module.exports = merge(common, {
+   mode: 'development',
+   devtool: 'inline-source-map',
+   devServer: {
+     contentBase: './dist'
+   }
+ });
```

__webpack.prod.js__

``` diff
+ const merge = require('webpack-merge');
+ const common = require('./webpack.common.js');
+
+ module.exports = merge(common, {
+   mode: 'production',
+ });
```

In `webpack.common.js`, we now have our `entry` and `output` setup configured and we've included any plugins that are required for both environments. In `webpack.dev.js`, we've set ``mode`` to ``development`` .Also, we've added the recommended `devtool` for that environment (strong source mapping), as well as our simple `devServer` configuration. Finally, in `webpack.prod.js`,``mode`` is set to ``production`` which loads `UglifyJSPlugin` which was first introduced by the [tree shaking](/guides/tree-shaking) guide.

Note the use of `merge()` in the environment-specific configurations to easily include our common configuration in `dev` and `prod`. The `webpack-merge` tool offers a variety of advanced features for merging but for our use case we won't need any of that.


## NPM Scripts

Now let's repoint our `scripts` to the new configurations. We'll use the _development_ one for our `webpack-dev-server`, `npm start`, script and the _production_ one for our `npm run build` script:

__package.json__

``` diff
  {
    "name": "development",
    "version": "1.0.0",
    "description": "",
    "main": "webpack.config.js",
    "scripts": {
-     "start": "webpack-dev-server --open",
+     "start": "webpack-dev-server --open --config webpack.dev.js",
-     "build": "webpack"
+     "build": "webpack --config webpack.prod.js"
    },
    "keywords": [],
    "author": "",
    "license": "ISC",
    "devDependencies": {
      "clean-webpack-plugin": "^0.1.17",
      "css-loader": "^0.28.4",
      "csv-loader": "^2.1.1",
      "express": "^4.15.3",
      "file-loader": "^0.11.2",
      "html-webpack-plugin": "^2.29.0",
      "style-loader": "^0.18.2",
      "webpack": "^3.0.0",
      "webpack-dev-middleware": "^1.12.0",
      "webpack-dev-server": "^2.9.1",
      "webpack-merge": "^4.1.0",
      "xml-loader": "^1.2.1"
    }
  }
```

Feel free to run those scripts and see how the output changes as we continue adding to our _production_ configuration.


## Minification

Note that while the [`UglifyJSPlugin`](/plugins/uglifyjs-webpack-plugin) is a great place to start for minification, there are other options out there. Here are a few more popular ones:

- [`BabelMinifyWebpackPlugin`](https://github.com/webpack-contrib/babel-minify-webpack-plugin)
- [`ClosureCompilerPlugin`](https://github.com/roman01la/webpack-closure-compiler)

If you decide to try another, just make sure your new choice also drops dead code as described in the [tree shaking](/guides/tree-shaking) guide.


## Source Mapping

We encourage you to have source maps enabled in production, as they are useful for debugging as well as running benchmark tests. That said, you should choose one with a fairly quick build speed that's recommended for production use (see [`devtool`](/configuration/devtool)). For this guide, we'll use the `source-map` option in _production_ as opposed to the `inline-source-map` we used in _development_:

__webpack.prod.js__

``` diff
  const merge = require('webpack-merge');
  const UglifyJSPlugin = require('uglifyjs-webpack-plugin');
  const common = require('./webpack.common.js');

  module.exports = merge(common, {
+   devtool: 'source-map',
    plugins: [
-     new UglifyJSPlugin()
+     new UglifyJSPlugin({
+       sourceMap: true
+     })
    ]
  });
```

T> Avoid `inline-***` and `eval-***` use in production as they can increase bundle size and reduce the overall performance.


## Specify the Environment

Many libraries will key off the `process.env.NODE_ENV` variable to determine what should be included in the library. For example, when not in _production_ some libraries may add additional logging and testing to make debugging easier. However, with `process.env.NODE_ENV === 'production'` they might drop or add significant portions of code to optimize how things run for your actual users. We can use webpack's built in [`DefinePlugin`](/plugins/define-plugin) to define this variable for all our dependencies:

__webpack.prod.js__

``` diff
+ const webpack = require('webpack');
  const merge = require('webpack-merge');
  const UglifyJSPlugin = require('uglifyjs-webpack-plugin');
  const common = require('./webpack.common.js');

  module.exports = merge(common, {
    devtool: 'source-map',
    plugins: [
      new UglifyJSPlugin({
        sourceMap: true
-     })
+     }),
+     new webpack.DefinePlugin({
+       'process.env.NODE_ENV': JSON.stringify('production')
+     })
    ]
  });
```

T> Technically, `NODE_ENV` is a system environment variable that Node.js exposes into running scripts. It is used by convention to determine dev-vs-prod behavior by server tools, build scripts, and client-side libraries. Contrary to expectations, `process.env.NODE_ENV` is not set to `"production"` __within__ the build script `webpack.config.js`, see [#2537](https://github.com/webpack/webpack/issues/2537). Thus, conditionals like `process.env.NODE_ENV === 'production' ? '[name].[hash].bundle.js' : '[name].bundle.js'` within webpack configurations do not work as expected.

If you're using a library like [`react`](https://reactjs.org/), you should actually see a significant drop in bundle size after adding this plugin. Also note that any of our local `/src` code can key off of this as well, so the following check would be valid:

__src/index.js__

``` diff
  import { cube } from './math.js';
+
+ if (process.env.NODE_ENV !== 'production') {
+   console.log('Looks like we are in development mode!');
+ }

  function component() {
    var element = document.createElement('pre');

    element.innerHTML = [
      'Hello webpack!',
      '5 cubed is equal to ' + cube(5)
    ].join('\n\n');

    return element;
  }

  document.body.appendChild(component());
```


## Minimize CSS

It is crucial to minimize your CSS on production, please see [Minimizing for Production](/plugins/mini-css-extract-plugin/#minimizing-for-production) section.


## CLI Alternatives

Some of what has been described above is also achievable via the command line. For example, the `--optimize-minimize` flag will include the `UglifyJSPlugin` behind the scenes. The `--define process.env.NODE_ENV="'production'"` will do the same for the `DefinePlugin` instance described above. And, `webpack -p` will automatically invoke both those flags and thus the plugins to be included.

While these short hand methods are nice, we usually recommend just using the configuration as it's better to understand exactly what is being done for you in both cases. The configuration also gives you more control on fine tuning other options within both plugins.
