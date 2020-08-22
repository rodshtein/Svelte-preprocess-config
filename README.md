# Svelte preprocess config RIGHT WAY

## Проблема
Часто бывает, что начинает глючить подсветка кода, или в подсветке всё ок, но сборка падает. 
Всё потому, что гайд [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) учит нас хранить конфиги препроцессора в разных местах: 
- часть, у нас, в **rollup.config.js**,
- часть в **postcss.config.js**,
- а конфиг для линтинга в **svelte.config.js**.

Это может сильно запутать. Поэтому хорошая тактика хранить конфиг в одном месте, при этом чтобы все инструменты имели к нему доступ. 
При такой схеме мы будем править конфиг один раз сводя ошибки к минимуму.

## А как вообще работает вся эта магия? 
**Парсинг**

Начинается всё со [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess), у него есть несколько тактик, самая практичная сделать два файла: 
- в **svelte.config.js** хранить основные настройки препроцессора,
- в **postcss.config.js**, хранить настройки postcss _(если мы его используем)_. 

[Svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) нужно подсунуть файл основного конфига. Если у нас **postcss: true**, или ключ с плагинами postcss или в svelte-файлах есть атрибуты с **type/lang = postcss** и [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) не найдёт подтянутых postcss плагинов в первом файле, то второй он найдёт сам и будет использовать плагины из этого файла, а плагины в первом — **проигнорирует**. 

[Svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) сам [подтягивает](https://github.com/michael-ciniawsky/postcss-load-config) все необходимые плагины из **postcss.config.js**. Это значит, что все зависимости должны быть установлены, если мы что-то забыли, то увидим предупреждение в консоли. 

Дальше [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) по отдельности обрабатывает стили, разметку и скрипты наших файлов. Он определяет синтаксис глядя на атрибуты **type/lang** или использует дефолтные настройки, [которые можно поменять](https://github.com/sveltejs/svelte-preprocess/blob/master/docs/preprocessing.md#auto-preprocessing).

**Линтинг**

Для работы линтера нужно расширение для vscode - [Sveltejs Language Tools](https://github.com/sveltejs/language-tools). Оно определяет синтаксис глядя на те же атрибуты **type/lang**. Так, что если вы используете отличные от дефолтных (js, css, html) синтаксисы, то вам придётся каждый раз использовать эти атрибуты, чтобы линтер понимал, какой перед ним синтаксис. 

Определив синтаксис линтер будет искать его настройки в файле svelte.config.js. Если он встретит ключ **postcss** с объектом настроек, то возьмёт настройки из него. Если встретит ключ/занчение **postcss: true** он пойдёт искать **postcss.config.js** откуда подтянет плагины и настройки для линтинга.


**Что в итоге**

Тут нужно добавить, что мы можем хранить часть конфига прямо в **rollup.config.js**, но нам всё равно нужны два других файла для линтинга. В этих двух инструментах очень много условностей из-за которых можно нехило запутаться и наш конфиг будет работать не так как хотелось. 

Всё это грёбаная

<img src="magic.gif">

## Решение
[Sveltejs Language Tools](https://github.com/sveltejs/language-tools) для проверки синтаксиса использует файл **svelte.config.js**. В нём мы и будем хранить весь конфиг _(см. ниже)_ Единственное отличие в том, что если раньше мы хранили PostCSS конфиг в отдельном файле, в виде простого объекта, а [svelte-preprocess](https://github.com/sveltejs/svelte-preprocess) сам подтягивал зависимости, то тут нам придётся всё подтягивать вручную — никакой магии, зато наглядно. 

Наш конфиг экспортит объект с двумя функциями: 
- первая нужна для работы линтера,
- вторую мы подтянем в конфиг роллапа, где в зависимости от окружения (isDev) она вернёт нужную конфигурацию сборки.

После правки конфига перезапустим линтер: 
- ctrl/cmd-shift-p, 
- введите svelte restart, 
- выберите Svelte: Restart Language Server.

Теперь, когда весь наш конфиг в одном месте, мы контролируем загрузку плагинов и будем уверены, что линтер и сборщик используют единую конфигурацию.

> **Важное замечение** Мы всё ещё можем не подтягивать зависимости postcss вручную, прописав конфиг [отдельным файлом](https://github.com/michael-ciniawsky/postcss-load-config#postcssrcjs-or-postcssconfigjs). Достаточно в **svelte.config.js** прописать `postcss: true`, тогда все инструменты будут искать конфиг postcss в **postcss.config.js** из корня проекта. Но тогда нам нужно отдельно прокинуть переменную окружения для настройки сборки. И мы опять получим «магический» конфиг разбросанный по разным файлам.

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
