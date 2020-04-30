# vite ⚡

> No-bundle Dev Server for Vue 3 Single-File Components.

## Getting Started

```bash
$ npx create-vite-app <project-name>
$ cd <project-name>
$ npm install
$ npm run dev
```

If using Yarn:

``` bash
$ yarn create vite-app <project-name>
$ cd <project-name>
$ yarn
$ yarn dev
```

## How is This Different from a Bundler-based Setup?

The primary difference is that for `vite` there is no bundling during development. The ES Import syntax in your source code is served directly to the browser, and the browser parses them via native `<script module>` support, making HTTP requests for each import. The dev server intercepts the requests and performs code transforms if necessary. For example, an import to a `*.vue` file is compiled on the fly right before it's sent back to the browser.

There are a few advantages of this approach:

- Since there is no bundling work to be done, the server cold start is extremely fast.

- Code is compiled on demand, so only code actually imported on the current screen is compiled. You don't have to wait until your entire app to be bundled to start developing. This can be a huge difference in apps with dozens of screens.

- Hot module replacement (HMR) performance is decoupled from the total number of modules. This makes HMR consistently fast no matter how big your app is.

Full page reload could be slightly slower than a bundler-based setup, since native ES imports result in network waterfalls with deep import chains. However since this is local development, the difference should be trivial compared to actual compilation time. (There is no compile cost on page reload since already compiled files are cached in memory.)

Finally, because compilation is still done in Node, it can technically support any code transforms a bundler can, and nothing prevents you from eventually bundling the code for production. In fact, `vite` provides a `vite build` command to do exactly that so the app doesn't suffer from network waterfall in production.

`vite` is highly experimental at this stage and is not suitable for production use, but we hope to one day make it so.

## How is This Different from [es-dev-server](https://open-wc.org/developing/es-dev-server.html)?

`es-dev-server` is a great project and we did take some inspiration from it when refactoring `vite` in the early stages. That said, here is why `vite` is different from `es-dev-server` and why we didn't just implement `vite` as a middleware for `es-dev-server`:

- `vite` supports Hot Module Replacement, which surgically updates the updated module without reloading the page. This is a fundamental difference in terms of development experience. `es-dev-server` internals is a bit too opaque to get this working nicely via a middleware.

- `vite` aims to be a single tool that integrates both the dev and the build process. You can use `vite` to both serve and bundle the same source code, with zero configuration.

- `vite` requires native ES module imports. It does not intend to burden itself with support for legacy browsers.

## Features

### Bare Module Resolving

Native ES imports doesn't support bare module imports like

```js
import { createApp } from 'vue'
```

The above will throw an error by default. `vite` detects such bare module imports in all served `.js` files and rewrite them with special paths like `/@modules/vue`. Under these special paths, `vite` performs module resolution to locate the correct files on disk:

- `vue` has special handling: you don't need to install it since `vite` will serve it by default. But if you want to use a specific version of `vue` (only supports Vue 3.x), you can install `vue` locally into `node_modules` and it will be preferred (`@vue/compiler-sfc` of the same version will also need to be installed).

- If a `web_modules` directory (generated by [Snowpack](https://www.snowpack.dev/)) is present, we will try to locate it.

- Finally we will try resolving the module from `node_modules`, using the package's `module` entry if available.

### Hot Module Replacement

- `*.vue` files come with HMR out of the box.

- For `*.js` files, a simple HMR API is provided:

  ```js
  import { foo } from './foo.js'
  import { hot } from '@hmr'

  foo()

  hot.accept('./foo.js', (newFoo) => {
    // the callback receives the updated './foo.js' module
    newFoo.foo()
  })

  // Can also accept an array of dep modules:
  hot.accept(['./foo.js', './bar.js'], ([newFooModule, newBarModule]) => {
    // the callback receives the updated mdoules in an Array
  })
  ```

  Modules can also be self-accepting:

  ```js
  import { hot } from '@hmr'

  export const count = 1

  hot.accept(newModule => {
    console.log('updated: count is now ', newModule.count)
  })
  ```

  Note that `vite`'s HMR does not actually swap the originally imported module: if an accepting module re-exports imports from a dep, then it is responsible for updating those re-exports (and these exports must be using `let`). In addition, importers up the chain from the accepting module will not be notified of the change.

  This simplified HMR implementation is sufficient for most dev use cases, while allowing us to skip the expensive work of generating proxy modules.

### CSS Pre-Processors

Install the corresponding pre-processor and just use it!

``` bash
yarn add -D sass
```
``` vue
<style lang="scss">
/* use scss */
</style>
```

Note importing CSS / preprocessor files from `.js` files, and HMR from imported pre-proccessor files are currently not supported, but can be in the future.

### Building for Production

Starting with version `^0.5.0`, you can run `vite build` to bundle the app and deploy it for production.

- `vite build --root dir`: build files in the target directory instead of current working directory.

- `vite build --cdn`: import `vue` from a CDN link in the built js. This will make the build faster, but overall the page payload will be larger because therer will be no tree-shaking for Vue APIs.

Internally, we use a highly opinionated Rollup config to generate the build. There's not much you can configure from the command line, but if you use the API, then the build is configurable by passing on most options to Rollup (see below).

### API

#### Dev Server

You can customize the server using the API. The server can accept plugins which have access to the internal Koa app instance. You can then add custom Koa middlewares to add pre-processor support:

``` js
const { createServer } = require('vite')

const myPlugin = ({
  root, // project root directory, absolute path
  app, // Koa app instance
  server, // raw http server instance
  watcher // chokidar file watcher instance
}) => {
  app.use(async (ctx, next) => {
    // You can do pre-processing here - this will be the raw incoming requests
    // before vite touches it.
    if (ctx.path.endsWith('.scss')) {
      // Note vue <style lang="xxx"> are supported by
      // default as long as the corresponding pre-processor is installed, so this
      // only applies to <link ref="stylesheet" href="*.scss"> or js imports like
      // `import '*.scss'`.
      console.log('pre processing: ', ctx.url)
      ctx.type = 'css'
      ctx.body = 'body { border: 1px solid red }'
    }

    // ...wait for vite to do built-in transforms
    await next()

    // Post processing before the content is served. Note this includes parts
    // compiled from `*.vue` files, where <template> and <script> are served as
    // `application/javascript` and <style> are served as `text/css`.
    if (ctx.response.is('js')) {
      console.log('post processing: ', ctx.url)
      console.log(ctx.body) // can be string or Readable stream
    }
  })
}

createServer({
  plugins: [
    myPlugin
  ]
}).listen(3000)
```

#### Build

``` js
const { build } = require('vite')

;(async () => {
  // All options are optional.
  // check out `src/node/build.ts` for full options interface.
  const result = await build({
    rollupInputOptions: {
      // https://rollupjs.org/guide/en/#big-list-of-options
    },
    rollupOutputOptions: {
      // https://rollupjs.org/guide/en/#big-list-of-options
    },
    rollupPluginVueOptions: {
      // https://github.com/vuejs/rollup-plugin-vue/tree/next#options
    },
    root: process.cwd(),
    cdn: false,
    write: true,
    minify: true,
    silent: false
  })
})()
```

## TODOs

- Vue file source maps
- Auto loading postcss config

## Trivia

[vite](https://en.wiktionary.org/wiki/vite) is the french word for "fast" and is pronounced `/vit/`.

## License

MIT
