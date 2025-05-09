> Undecided yet what bundler to use? We suggest using [SvelteKit](https://kit.svelte.dev), or Vite with [vite-plugin-svelte](https://github.com/sveltejs/vite-plugin-svelte/).

# svelte-loader

[![Build Status](https://travis-ci.org/sveltejs/svelte-loader.svg?branch=master)](https://travis-ci.org/sveltejs/svelte-loader)

A [webpack](https://webpack.js.org) loader for [svelte](https://svelte.technology).


## Install

```
npm install --save svelte svelte-loader
```


## Usage

Configure inside your `webpack.config.js`:

```javascript
  ...
  resolve: {
    // see below for an explanation
    alias: {
      svelte: path.resolve('node_modules', 'svelte/src/runtime') // Svelte 3: path.resolve('node_modules', 'svelte')
    },
    extensions: ['.mjs', '.js', '.svelte'],
    mainFields: ['svelte', 'browser', '...'],
    conditionNames: ['svelte', 'browser', '...'],
  },
  module: {
    rules: [
      ...
      // The following two loader entries are only needed if you use Svelte 5+ with TypeScript.
      // Also make sure your tsconfig.json includes `"useDefineForClassFields": true` or "target" is at least "ES2022"` in order to not downlevel class syntax
      {
        test: /\.svelte\.ts$/,
        use: [ "svelte-loader", { loader: "ts-loader", options: { transpileOnly: true } }],
      },
      // This is the config for other .ts files - the regex makes sure to not process .svelte.ts files twice
      {
        test: /(?<!\.svelte)\.ts$/,
        loader: "ts-loader",
        options: {
          transpileOnly: true, // you should use svelte-check for type checking
        }
      },
      {
        // Svelte 5+:
        test: /\.(svelte|svelte\.js)$/,
        // Svelte 3 or 4:
        // test: /\.svelte$/,
        // In case you write Svelte in HTML (not recommended since Svelte 3):
        // test: /\.(html|svelte)$/,
        use: 'svelte-loader'
      },
      {
        // required to prevent errors from Svelte on Webpack 5+, omit on Webpack 4
        test: /node_modules\/svelte\/.*\.mjs$/,
        resolve: {
          fullySpecified: false
        }
      }
      ...
    ]
  }
  ...
```

Check out the [example project](https://github.com/sveltejs/template-webpack).

### resolve.alias

The [`resolve.alias`](https://webpack.js.org/configuration/resolve/#resolvealias) option is used to make sure that only one copy of the Svelte runtime is bundled in the app, even if you are `npm link`ing in dependencies with their own copy of the `svelte` package. Having multiple copies of the internal scheduler in an app, besides being inefficient, can also cause various problems.

### resolve.mainFields

Webpack's [`resolve.mainFields`](https://webpack.js.org/configuration/resolve/#resolve-mainfields) option determines which fields in `package.json` are used to resolve identifiers. If you're using Svelte components installed from npm, you should specify this option so that your app can use the original component source code, rather than consuming the already-compiled version (which is less efficient).

### resolve.conditionNames

Webpack's [`resolve.conditionNames`](https://webpack.js.org/configuration/resolve/#resolveconditionnames) option determines which fields in the `exports` in `package.json` are used to resolve identifiers. If you're using Svelte components installed from npm, you should specify this option so that your app can use the original component source code, rather than consuming the already-compiled version (which is less efficient).

### Extracting CSS

If your Svelte components contain `<style>` tags, by default the compiler will add JavaScript that injects those styles into the page when the component is rendered. That's not ideal, because it adds weight to your JavaScript, prevents styles from being fetched in parallel with your code, and can even cause CSP violations.

A better option is to extract the CSS into a separate file. Using the `emitCss` option as shown below would cause a virtual CSS file to be emitted for each Svelte component. The resulting file is then imported by the component, thus following the standard Webpack compilation flow. Add [MiniCssExtractPlugin](https://github.com/webpack-contrib/mini-css-extract-plugin) to the mix to output the css to a separate file.

```javascript
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
  ...
  module: {
    rules: [
      ...
      {
        test: /\.(svelte|svelte\.js)$/,
        use: {
          loader: 'svelte-loader',
          options: {
            emitCss: true,
          },
        },
      },
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          {
            loader: 'css-loader',
            options: {
              url: false, // necessary if you use url('/path/to/some/asset.png|jpg|gif')
            }
          }
        ]
      },
      ...
    ]
  },
  ...
  plugins: [
    new MiniCssExtractPlugin('styles.css'),
    ...
  ]
  ...
```

Additionally, if you're using multiple entrypoints, you may wish to change `new MiniCssExtractPlugin('styles.css')` for `new MiniCssExtractPlugin('[name].css')` to generate one CSS file per entrypoint.

Warning: in production, if you have set `sideEffects: false` in your `package.json`, `MiniCssExtractPlugin` has a tendency to drop CSS, regardless of whether it's included in your svelte components.

Alternatively, if you're handling styles in some other way and just want to prevent the CSS being added to your JavaScript bundle, use

```javascript
...
use: {
  loader: 'svelte-loader',
  options: {
    compilerOptions: {
      css: false
    }
  },
},
...
```

### Source maps

JavaScript source maps are enabled by default, you just have to use an appropriate [webpack devtool](https://webpack.js.org/configuration/devtool/).

To enable CSS source maps, you'll need to use `emitCss` and pass the `sourceMap` option to the `css-loader`. The above config should look like this:

```javascript
module.exports = {
    ...
    devtool: "source-map", // any "source-map"-like devtool is possible
    ...
    module: {
      rules: [
        ...
        {
          test: /\.css$/,
          use: [
            MiniCssExtractPlugin.loader,
            {
              loader: 'css-loader',
              options: {
                sourceMap: true
              }
            }
          ]
        },
        ...
      ]
    },
    ...
    plugins: [
      new MiniCssExtractPlugin('styles.css'),
      ...
    ]
    ...
};
```

This should create an additional `styles.css.map` file.

### Svelte Compiler options

You can specify additional arbitrary compilation options with the `compilerOptions` config key, which are passed directly to the underlying Svelte compiler:
```js
...
use: {
  loader: 'svelte-loader',
  options: {
    compilerOptions: {
      // additional compiler options here
      generate: 'ssr', // for example, SSR can be enabled here
    }
  },
},
...
```

### Using preprocessors like TypeScript

Install [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) and add it to the loader options:

```js
const sveltePreprocess = require('svelte-preprocess');
...
use: {
  loader: 'svelte-loader',
  options: {
    preprocess: sveltePreprocess()
  },
},
...
```

Now you can use other languages inside the script and style tags. Make sure to install the respective transpilers and add a `lang` tag indicating the language that should be preprocessed. In the case of TypeScript, install `typescript` and add `lang="ts"` to your script tags.

### Hot Reload

> Hot Module Reloading is currently not supported for Svelte 5+

This loader supports component-level HMR via the community supported [svelte-hmr](https://github.com/rixo/svelte-hmr) package. This package serves as a testbed and early access for Svelte HMR, while we figure out how to best include HMR support in the compiler itself (which is tricky to do without unfairly favoring any particular dev tooling). Feedback, suggestion, or help to move HMR forward is welcomed at [svelte-hmr](https://github.com/rixo/svelte-hmr/issues) (for now).

Configure inside your `webpack.config.js`:

```javascript
// It is recommended to adjust svelte options dynamically, by using
// environment variables
const mode = process.env.NODE_ENV || 'development';
const prod = mode === 'production';

module.exports = {
  ...
  module: {
    rules: [
      ...
      {
        test: /\.(svelte|svelte\.js)$/,
        use: {
          loader: 'svelte-loader',
          options: {
            compilerOptions: {
              // NOTE Svelte's dev mode MUST be enabled for HMR to work
              dev: !prod, // Default: false
            },

            // NOTE emitCss: true is currently not supported with HMR
            // Enable it for production to output separate css file
            emitCss: prod, // Default: false
            // Enable HMR only for dev mode
            hotReload: !prod, // Default: false
            // Extra HMR options, the defaults are completely fine
            // You can safely omit hotOptions altogether
            hotOptions: {
              // Prevent preserving local component state
              preserveLocalState: false,

              // If this string appears anywhere in your component's code, then local
              // state won't be preserved, even when noPreserveState is false
              noPreserveStateKey: '@!hmr',

              // Prevent doing a full reload on next HMR update after fatal error
              noReload: false,

              // Try to recover after runtime errors in component init
              optimistic: false,

              // --- Advanced ---

              // Prevent adding an HMR accept handler to components with
              // accessors option to true, or to components with named exports
              // (from <script context="module">). This have the effect of
              // recreating the consumer of those components, instead of the
              // component themselves, on HMR updates. This might be needed to
              // reflect changes to accessors / named exports in the parents,
              // depending on how you use them.
              acceptAccessors: true,
              acceptNamedExports: true,
            }
          }
        }
      }
      ...
    ]
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    ...
  ]
}
```

You also need to add the [HotModuleReplacementPlugin](https://webpack.js.org/plugins/hot-module-replacement-plugin/). There are multiple ways to achieve this.

If you're using webpack-dev-server, you can just pass it the [`hot` option](https://webpack.js.org/configuration/dev-server/#devserverhot) to add the plugin automatically.

Otherwise, you can add it to your webpack config directly:

```js
const webpack = require('webpack');

module.exports = {
  ...
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    ...
  ]
}
```

### CSS @import in components

It is advised to inline any css `@import` in component's style tag before it hits `css-loader`.

This ensures equal css behavior when using HMR with `emitCss: false` and production.

Install `svelte-preprocess`, `postcss`, `postcss-import`, `postcss-load-config`.

Configure `svelte-preprocess`:

```javascript
const sveltePreprocess = require('svelte-preprocess');
...
module.exports = {
  ...
  module: {
    rules: [
      ...
      {
        test: /\.(svelte|svelte\.js)$/,
        use: {
          loader: 'svelte-loader',
          options: {
            preprocess: sveltePreprocess({
              postcss: true
            })
          }
        }
      }
      ...
    ]
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    ...
  ]
}
...
```

Create `postcss.config.js`:

```javascript
module.exports = {
  plugins: [
    require('postcss-import')
  ]
}
```

If you are using autoprefixer for `.css`, then it is better to exclude emitted css, because it was already processed with `postcss` through `svelte-preprocess` before emitting.

```javascript
  ...
  module: {
    rules: [
      ...
      {
        test: /\.css$/,
        exclude: /svelte\.\d+\.css/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader'
        ]
      },
      {
        test: /\.css$/,
        include: /svelte\.\d+\.css/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader'
        ]
      },
      ...
    ]
  },
  ...
```

This ensures that global css is being processed with `postcss` through webpack rules, and svelte component's css is being processed with `postcss` through `svelte-preprocess`.

## License

MIT
