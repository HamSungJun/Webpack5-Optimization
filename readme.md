이 포스팅은 `Webpack 5` 버전으로 `ESM` 자바스크립트 모듈 디펜던시를 번들링하였을 때 사이즈를 최소화 하는 과정에 대한 내용입니다.

최근에 사내에 `Vue.js` 로 진행한 프로젝트 작업물에 대해서 웹팩 빌드를 진행하여 배포할 일이 있었다. 별 생각없이 Vue-Cli의 기본 스크립트인 Build 명령어를 통해서 번들링을 하였는데 생각보다 용량이 너무 커서(1MB 이상) 이를 최소화 할 수 있는 방법들이 무엇이 있는지 자료로 남겨두면 좋겠다고 생각하였다.

서술은 실험실에서 실험하는 것처럼 하겠다.

#### `1. 구동 환경`
```bash
# 빈 디렉토리를 하나 만들어서 NPM 패키지를 관리할 수 있도록 설정하자.
$ npm init -y
```
```bash
# 웹팩 모듈은 어플리케이션을 구동하는 런타임에 사용하는 모듈이 아니므로 devDependencies에 설치한다.
$ npm install webpack webpack-cli webpack-merge webpack-bundle-analyzer -D
```
```bash
# 자바스크립트를 작성할때 ES6 이상의 문법 표현을 사용하는 경우 크로스 브라우징을 위해 트랜스파일러를 사용할 필요가 있다.
# 빌드 단계에서 트랜스파일링 되므로 devDependencies에 설치한다.
$ npm install @babel/core @babel/preset-env @babel/cli core-js babel-loader -D
```
```bash
# 자바스크립트 모듈을 작성하면서 별도의 스타일을 작성하는 경우 필요하다.
# 이 모듈들 또한 런타임에 구동되는 것이 아니므로 devDependencies에 설치한다.
$ npm install mini-css-extract-plugin css-loader sass-loader sass -D
```
```bash
# 어플리케이션이 이 두개의 vendor를 이용한다고 가정하기 위해 설치했다.
# 두 모듈은 런타임에 사용하므로 dependencies에 추가한다.
$ npm install lodash moment -S
```

#### `2. 프로젝트 설정 내용`

`파일 시스템 구조`
```
$ tree -I node_modules ./

./
├── dist
├── package.json
├── src
│   ├── entryA.js
│   ├── entryB.js
│   ├── index.js
│   └── lib
│       ├── animal
│       │   ├── cat
│       │   │   └── index.js
│       │   ├── dog
│       │   │   └── index.js
│       │   └── index.js
│       └── computer
│           ├── index.js
│           ├── keyboard
│           │   └── index.js
│           ├── monitor
│           │   └── index.js
│           └── mouse
│               └── index.js
├── webpack.common.js
├── webpack.dev.js
└── webpack.prod.js
```
`Package.json`
```javascript
{
  "name": "Webpack5-Optimization",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "build:dev": "webpack --config webpack.dev.js",
    "build:prod": "webpack --config webpack.prod.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@babel/cli": "^7.13.16",
    "@babel/core": "^7.13.16",
    "@babel/preset-env": "^7.13.15",
    "babel-loader": "^8.2.2",
    "core-js": "^3.11.0",
    "css-loader": "^5.2.4",
    "eslint": "^7.25.0",
    "eslint-config-standard": "^16.0.2",
    "eslint-plugin-import": "^2.22.1",
    "eslint-plugin-node": "^11.1.0",
    "eslint-plugin-promise": "^4.3.1",
    "mini-css-extract-plugin": "^1.5.0",
    "sass": "^1.32.11",
    "sass-loader": "^11.0.1",
    "webpack": "^5.35.1",
    "webpack-bundle-analyzer": "^4.4.1",
    "webpack-cli": "^4.6.0",
    "webpack-merge": "^5.7.3"
  },
  "dependencies": {
    "lodash": "^4.17.21",
    "moment": "^2.29.1"
  }
}
```
`webpack.common.js`
```javascript
/**
* webpack.dev.js, webpack.prod.js 설정에서 공통적으로 이용하기 위한 설정입니다.
* 이후 webpack-merge 모듈을 통해 설정을 병합하여 웹팩 빌드시 설정으로 사용됩니다.
*/
const path = require('path')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer')
module.exports = {
  entry: {
    entryA: path.resolve(__dirname, 'src/entryA.js'),
    entryB: path.resolve(__dirname, 'src/entryB.js')
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
    clean: true
  },
  module: {
    rules: [
      {
        test: /.js$/i,
        use: [
          'babel-loader'
        ]
      },
      {
        test: /\.(sa|sc|c)ss$/i,
        use: [
          /**
          * MiniCssExtractPlugin을 사용할 경우 가져온 스타일 시트를 별도의 파일로 산출합니다.
          * 스타일 내용까지 번들 파일에 내장되길 원한다면 해당 로더를 제거하거나 주석처리 하면 됩니다.
          */
          MiniCssExtractPlugin.loader,
          'css-loader',
          'sass-loader'
        ]
      }
    ]
  },
  /**
  * entry를 하나만 두는 경우 lodash, moment 같은 모듈의 내용이 번들에 포함되어도 크게 문제가 없지만
  * entry가 여러개일 경우 여러개의 번들이 생성되는 과정에서 vendor 모듈이 중복되어 번들링 되는 문제가
  * 발생할 수 있습니다. 이 optimization 설정을 통해 중복되는 벤더 모듈을 별도의 번들 파일에 포함 시킬
  * 수 있습니다. webpack-bundle-analyzer가 다이렉트하는 페이지에서 확인할 예정입니다.
  */
  optimization: {
    splitChunks: {
      chunks: 'all',
      name: 'chunk-vendors'
    }
  },
  plugins: [
    new BundleAnalyzerPlugin()
  ]
}
```
`webpack.dev.js`
```javascript
const { merge } = require('webpack-merge')
const common = require('./webpack.common')

module.exports = merge(common, {
  mode: 'development',
  output: {
    filename: '[name].bundle.js'
  }
})
```
`webpack.prod.js`
```javascript
const { merge } = require('webpack-merge')
const common = require('./webpack.common')

module.exports = merge(common, {
  mode: 'production',
  output: {
    filename: '[name].[contenthash].bundle.js'
  }
})
```
`.babelrc`
```javascript
/**
* @babel/preset-env 플러그인은 .browserslist나 package.json에 작성한 브라우저 지원항목을 확인해서
* 해당 브라우저들의 지원 스펙에 맞게 트랜스파일링 하는 것을 지원합니다.
*/
{
    "presets": [[
        "@babel/preset-env",
        {
            // Tree Shaking을 위해 ESM 모듈이 CommonJS 형태로 변환되는 것을 방지합니다.
            "modules": false,
            // 실제 소스에서 사용된 ES6 이상의 표현식이나 자료구조에 대해서만 Polyfill을 추가합니다.
            "useBuiltIns": "usage",
            // 더이상 core-js 3미만의 버전은 지원하지 않으므로 해당 버전으로 설치하여 설정했습니다.
            "corejs": 3
        }
    ]]
}
```
#### `3. 실험 목록`

- 하나의 `entry` 에서 번들링 하는 경우, 여러 `entry`에서 번들링 하는 경우

- `Tree Shaking` 동작 여부 판단

#### `4. 참고 문헌`

- [Webpack 5 release (2020-10-10)](https://webpack.js.org/blog/2020-10-10-webpack-5-release/)
	- [#Major-Changes-Optimization](https://webpack.js.org/blog/2020-10-10-webpack-5-release/#major-changes-optimization)

- [Babel](https://babeljs.io)
	- [@babel/preset-env](https://babeljs.io/docs/en/babel-preset-env)
    
#### 5. [Github Repo](https://github.com/HamSungJun/Webpack5-Optimization)