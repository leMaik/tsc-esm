***tsc-esm*** is a small wrapper library can be used to replace Typescript's `tsc` compilation command when generating modern Javascript with ES6 modules.

### The problem

In order to use ES6 modules either in a browser, a Node (with `"type": "module"`) or a Deno environment, a `".js"` extension is mandatory at the end of each import statement.

For example, you can do:

```ts
import foo from './foo.js'
```

but you cannot do:


```ts
import foo from './foo'
```

However Typescript compiler *do understand* the latter case and it's even recommended to use it. Some linters will complain if you add the unnecessary ".js" or ".ts" extension.

In the same way, Typescript can understant when you a `/foo/index.ts` directory structure, you can do

```ts
import foo from '/foo'
```

and Typescript will understand you need to import the `index.ts` file.

That's actually great... Until you want to compile your Typescript code to modern Javascript with ES6 modules. As it is explained well enough in this long issue: [provide a way to add the '.js' file extension to the end of module specifiers](https://github.com/microsoft/TypeScript/issues/16577), the Typescript team considers that since users can manually add themselves a ".js" extension to all their imports, this is not an issue.

In my opinion it is still an issue because:

1. valid Typescript code can be compiled with no errors and still generate *invalid Javascript* (and all platforms disagree it is invalid: Browser, Node and Deno),
2. even if it works, it is semantically incorrect to write `import foo from "/foo/index.js"` when your directory structure is `/foo/index.ts`, because you are explicitly importing to a file that does not exist yet (it will exist after compilation).

### The solution

Use `tsc-esm` instead of `tsc`.

`tsc-esm` works in two simple steps:

1. it calls `tsc`,
2. it uses the [grubber](https://www.npmjs.com/package/@digitak/grubber) library to safely parse the generated javascript files and patch the import expressions.

***It is highly recommended*** that you have a tsconfig.json configuration file in your root project with either `compilerOptions.outDir` or `include` option set ; otherwise all `.js` files in your project will be scanned and transformed.

#### CLI

##### Global installation

```
npm i -g @digitak/tsc-esm
tsc-esm
```

##### Local installation

```
npm i -g @digitak/tsc-esm
```

Then add a script in your package.json:

```json
{
   "scripts": {
      "build": "tsc-esm"
   }
}
```

Then you can run:

```
npm run build
```

#### API

```ts
import { build } from '@digitak/tsc-esm'

build()
```

Or:

```ts
import { compile, patch } from '@digitak/tsc-esm'

compile()
patch()
```

##### Aliases

You can pass aliases to the `build` or the `patch` functions to control how some paths should be transformed or not:

```ts
function build(aliases?: Array<AliasResolver>): void
function patch(aliases?: Array<AliasResolver>): void

type AliasResolver = {
   find: RegExp; // the path to match
   replacement: string | null; // the replacement value
};
```

The function [String::replace](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace) is used internally so you can replace your path using special patterns like `$1`or `$&`.

If the replacement value is null, the path will be left untransformed.

It can be useful when working when libraries that don't have clean type definitions.

If you have a `chokidar` dependency for example you might need to tell `tsc-esc` to not patch this specific import:

```ts
build([
   { find: /^chokidar$/, replacement: null },
])
```

Then all `import chokidar from 'chokidar'` statements will be left unchanged, otherwise it would have been transformed into `import chokidar from 'chokidar/index.js'` which is not typed.
