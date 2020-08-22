# Svelte preprocess config RIGHT WAY

## Проблема
Часто бывает, что начинает глючить подсветка кода, или в подсветке всё ок, но сборка падает. 
Всё потому, что гайд [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) учит нас хранить конфиги препроцессора в разных местах: 
- часть, у нас, в **rollup.config.js**;
- часть в **postcss.config.js**;
- а конфиг для линтинга в **svelte.config.js**.

Это может сильно запутать. Поэтому хорошая тактика хранить конфиг в одном месте, при этом чтобы все инструменты имели к нему доступ. 
При такой схеме мы будем править конфиг один раз сводя ошибки к минимуму.

## Решение
[Sveltejs Language Tools](https://github.com/sveltejs/language-tools) для проверки синтаксиса использует файл **svelte.config.js**. В нём мы и будем хранить весь конфиг _(см. ниже)_ Единственное отличие в том, что если раньше мы хранили PostCSS конфиг в отдельном файле, в виде простого объекта, а [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) сам подтягивал зависимости, то тут нам придётся всё подтягивать вручную — никакой магии, зато наглядно. 

Наш конфиг экспортит объект с двумя функциями: 
- первая нужна для работы линтера;
- вторую мы подтянем в конфиг роллапа, где в зависимости от окружения (isDev) она вернёт нужную конфигурацию сборки.

После правки конфига перезапустим линтер: 
- ctrl/cmd-shift-p, 
- введите svelte restart, 
- выберите Svelte: Restart Language Server.

Теперь, когда весь наш конфиг в одном месте, мы будем уверены, что линтер и сборщик используют единую конфигурацию.

> **Важное замечение** Мы всё ещё можем не подтягивать зависимости postcss вручную, прописав конфиг [отдельным файлом](https://github.com/michael-ciniawsky/postcss-load-config#postcssrcjs-or-postcssconfigjs). Достаточно в **svelte.config.js** прописать `postcss: true`, тогда все инструменты будут искасть конфиг postcss в **postcss.config.js** из корня проекта. Но тогда нам нужно отдельно прокинуть переменную окружения для настройки сборки. И мы опять получим «магический» конфиг разбросанный по разным файлам.

### svelte.config.js
```
const sveltePreprocess = require('svelte-preprocess');
const easyImport = require('postcss-easy-import');
const mixins = require('postcss-mixins');
const nested = require('postcss-nested');
const presetEnv = require('postcss-preset-env');
const sugarss = require('sugarss');


function getSP(isDev = false) {
  return sveltePreprocess({
    sourceMap: isDev,
    pug: true,
    postcss: {
      map: isDev,
      parser: sugarss,
      plugins: [
        easyImport(),
        mixins(),
        nested(),
        presetEnv({
          browsers: "last 2 versions",
          stage: 0,
          features: {
            "nesting-rules": true,
          },
        }),
      ],
    },
  });
}

module.exports = {
    preprocess: getSP(true),
    getSP,
};
```
### rollup.config.js (sapper example)
```
// sapper def
import resolve from '@rollup/plugin-node-resolve';
import replace from '@rollup/plugin-replace';
import commonjs from '@rollup/plugin-commonjs';
import svelte from 'rollup-plugin-svelte';
import babel from '@rollup/plugin-babel';
import { terser } from 'rollup-plugin-terser';
import config from 'sapper/config/rollup.js';
import pkg from './package.json';

const sveltePreprocess = require('./svelte.config').getSP;

const mode = process.env.NODE_ENV;
const dev = mode === 'development';
const legacy = !!process.env.SAPPER_LEGACY_BUILD;

const onwarn = (warning, onwarn) =>
  (warning.code === 'MISSING_EXPORT' && /'preload'/.test(warning.message)) ||
  (warning.code === 'CIRCULAR_DEPENDENCY' && /[/\\]@sapper[/\\]/.test(warning.message)) ||
  onwarn(warning);

export default {
  client: {
    input: config.client.input(),
    output: config.client.output(),
    plugins: [
      replace({
        'process.browser': true,
        'process.env.NODE_ENV': JSON.stringify(mode)
      }),
      svelte({
        preprocess: sveltePreprocess(dev),
        dev,
        hydratable: true,
        emitCss: true
      }),
      resolve({
        browser: true,
        dedupe: ['svelte']
      }),
      commonjs(),

      legacy && babel({
        extensions: ['.js', '.mjs', '.html', '.svelte'],
        babelHelpers: 'runtime',
        exclude: ['node_modules/@babel/**'],
        presets: [
          ['@babel/preset-env', {
            targets: '> 0.25%, not dead'
          }]
        ],
        plugins: [
          '@babel/plugin-syntax-dynamic-import',
          ['@babel/plugin-transform-runtime', {
            useESModules: true
          }]
        ]
      }),

      !dev && terser({
        module: true
      })
    ],

    preserveEntrySignatures: false,
    onwarn,
  },

  server: {
    input: config.server.input(),
    output: config.server.output(),
    plugins: [
      replace({
        'process.browser': false,
        'process.env.NODE_ENV': JSON.stringify(mode)
      }),
      svelte({
        preprocess: sveltePreprocess(dev),
        generate: 'ssr',
        hydratable: true,
        dev
      }),
      resolve({
        dedupe: ['svelte']
      }),
      commonjs()
    ],
    external: Object.keys(pkg.dependencies).concat(require('module').builtinModules),

    preserveEntrySignatures: 'strict',
    onwarn,
  },

  serviceworker: {
    input: config.serviceworker.input(),
    output: config.serviceworker.output(),
    plugins: [
      resolve(),
      replace({
        'process.browser': true,
        'process.env.NODE_ENV': JSON.stringify(mode)
      }),
      commonjs(),
      !dev && terser()
    ],

    preserveEntrySignatures: false,
    onwarn,
  }
};

```
