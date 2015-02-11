# PostCSS API

## `postcss` function

It is a main enter point of PostCSS:

```js
var postcss = require('postcss');
```

#### `postcss(plugins)`

Returns new `PostCSS` instance and sets plugins from `plugins` array
as CSS processors.

```js
postcss([autoprefixer, cssnext, cssgrace]).process(css).css;
```

Also you can create empty `PostCSS` and set plugins by `PostCSS#use` method.
See `PostCSS#use` docs below for plugin formats.

### `postcss.parse(css, opts)`

Parses CSS and returns `Root` with CSS nodes inside.

```js
// Simple CSS concatenation with source map support
var root1 = postcss.parse(css1, { from: file1 });
var root2 = postcss.parse(css2, { from: file2 });
root1.append(root2).toResult().css;
```

Options:

* `from` is a path to CSS file. You should always set `from`, because it is used
  in map generation and in syntax error messages.
* `safe` enables [Safe Mode] when PostCSS will try to fix CSS syntax errors.
* `map` is a object with [source map options]. Only `map.prev` is used in
  `parse` function.

[source map options]: https://github.com/postcss/postcss#source-map-1
[Safe Mode]:          https://github.com/postcss/postcss#safe-mode

#### `postcss.root(props)`

Creates new `Root` instance:

```js
postcss.root({ after: '\n' }).toString() //=> "\n"
```

#### `postcss.atRule(props)`

Creates new `AtRule` node:

```js
postcss.atRule({ name: 'charset' }).toString() //=> "@charset"
```

#### `postcss.rule(props)`

Creates new `Rule` node:

```js
postcss.rule({ selector: 'a' }).toString() //=> "a {\n}"
```

#### `postcss.decl(props)`

Creates new `Declaration` node:

```js
postcss.decl({ prop: 'color', value: 'black' }).toString() //=> "color: black"
```

#### `postcss.comment(props)`

Creates new `Comment` node:

```js
postcss.comment({ text: 'test' }).toString() //=> "/* test */"
```

## `PostCSS` class

`PostCSS` instance contains plugin to process CSS. You can create one `PostCSS`
instance and use it on many CSS files to initialize plugins only once.

```js
var processor = postcss([autoprefixer, cssnext, cssgrace]);
processor.process(css1).css;
processor.process(css2).css;
```

#### `use(plugin)`

Add plugin to CSS processors.

```js
var processor = postcss();
processor.use(autoprefixer).use(cssnext).use(cssgrace);
```

There is three format of plugin:

1. Function. PostCSS will send a `Root` instance as first argument.
2. An object with `postcss` method. PostCSS will use this method
   as function format below.
3. Other `PostCSS` instance. PostCSS will copy plugins from there.

Plugin function should change `Root` from first argument or return new `Root`.

```js
processor.use(function (css) {
    css.prepend({ name: 'charset', params: '"UTF-8"' });
});
processor.use(function (css) {
    return postcss.root();
});
```

#### `process(css, opts)`

This is a main method of PostCSS. It will parse CSS to `Root`, send this root
to each plugin and then return `Result` object from transformed root.

```js
var result = processor.process(css, { from: 'a.css', to: 'a.out.css' });
```

Input CSS formats are:

* String with CSS.
* `Result` instance from other PostCSS processor. PostCSS will take already
  parsed root from it.
* Any object with `toString()` method. For example, file stream.

Options:

* `from` is a path to input CSS file. You should always set `from`,
  because it is used in map generation and in syntax error messages.
* `to` is a path to output CSS file. You should always set `to` to generates
  correct source map.
* `safe` enables [Safe Mode] when PostCSS will try to fix CSS syntax errors.
* `map` is a object with [source map options].

[source map options]: https://github.com/postcss/postcss#source-map-1

## `Result` class

There is a two way to get `Result` instance. `PostCSS#process(css, opts)`
returns it or you can call `Root#toResult(opts)`.

```js
var result1 = postcss().process(css);
var result2 = postcss.parse(css).toResult();
```

#### `root`

This property contains source `Root` instance.

```js
root.toResult().root == root;
```

#### `opts`

Options from `PostCSS#process(css, opts)` or `Root#toResult(opts)` call.

```js
postcss().process(css, opts).opts == opts;
```

#### `css`

Lazy method, that will stringify `Root` instance to CSS string on first call
and generates source map to `map` property if it is necessary.

```js
postcss().process('a{}').css //=> "a{}"
```

#### `map`

Lazy method, that will generates source map for `Root` changes and stringify
CSS to `css` property.

It contains instance of `SourceMapGenerator` class from [source-map] library.

```js
result.map.toJSON() //=> { version: 3, file: 'a.css', sources: ['a.css'], … }
```

`Result` instance will contain `map` property only if PostCSS decide, that
source map should be in separated file. If source map will be inlined to CSS,
`Result` instance will not have this property.

```js
if ( result.map ) {
    fs.writeFileSync(to + '.map', result.map.toString());
}
```

By default, PostCSS will inline map to CSS, so in most cases this property will
be empty. PostCSS will create it only if user set `map.inline = false`
option or if input source map was in separated file.

[source-map]: https://github.com/mozilla/source-map

## List module

Contains helper to split CSS values.

```js
var list = require('postcss/lib/list');
```

#### `list.space`

Splits values by space with brackets and quotes:

```js
list.split('1px calc(10% + 1px) "border radius"')
//=> ['1px', 'calc(10% + 1px)', '"border radius"']
```

#### `list.comma`

Splits values by comma with brackets and quotes:

```js
list.split('1px 2px, rgba(255, 0, 0, 0.9), "border, top"')
//=> ['1px 2px', 'rgba(255, 0, 0, 0.9)', '"border, top"']
```

## `Input` class

Represents input source of CSS.

```js
var root  = postcss.parse(css, { from: file });
var input = root.source.input;
```

#### `file`

Absolute path to CSS source file from `from` option.

```js
var root  = postcss.parse(css, { from: 'a.css' });
root.source.input.file //=> '/home/ai/a.css'
```

#### `id`

Unique ID of CSS source if user missed `from` and we didn’t know file path.

```js
var root  = postcss.parse(css);
root.source.input.file //=> undefined
root.source.input.id   //=> <input css 1>
```

#### `from`

Contains `file` if user set `from` option or `id` if he didn’t.

```js
var root  = postcss.parse(css, { from: 'a.css' });
root.source.input.file //=> '/home/ai/a.css'

var root  = postcss.parse(css);
root.source.input.from //=> <input css 1>
```

#### `map`

Represents input source map from compilation step before PostCSS, which
generated input CSS (from example, from Sass compiler).

It is used in PostCSS map generator, but `consumer()` method will be helpful
for plugin developer, because it returns instance of `SourceMapConsumer` class
from [source-map] library.

```js
root.source.input.map.consumer().sources //=> ['a.sass']
```

[source-map]: https://github.com/mozilla/source-map

#### `origin(line, column)`

Use input source map to get symbol position in first source
(for example, in Sass):

```js
root.source.input.origin(1, 1) //=> { source: 'a.css', line: 3, column: 1 }
```