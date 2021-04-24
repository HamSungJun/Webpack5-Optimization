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
    new BundleAnalyzerPlugin(),
    new MiniCssExtractPlugin()
  ]
}
```
`webpack.dev.js`
```javascript
const { merge } = require('webpack-merge')
const common = require('./webpack.common')

module.exports = merge(common, {
  mode: 'development',
  devtool: 'inline-source-map',
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
  devtool: 'source-map',
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

<hr />

`하나의 entry 에서 번들링 하는 경우, 여러 entry에서 번들링 하는 경우`

> 웹팩 설정에서 하나의 엔트리 파일만을 진입점으로 설정하고 있다면 `Lodash`, `Moment`와 같은 외부 벤더 모듈들이 번들 파일에 같이 포함된다.하지만 진입점이 여러개인 경우, 별도의 최적화 설정을 정의해주지 않으면 모든 번들 결과 파일에 같은 벤더 모듈이 포함되는 결과가 생기고 곧 어플리케이션 접근시 같은 리소스를 두번 요청해서 다운받는 결과로 이어질 수 있다.

먼저 진입점을 하나만 설정하고 그 파일에 외부 벤더 모듈을 임포트하여 번들링해본 결과이다.

```bash
$ npm run build:dev
...
asset entryA.bundle.js 1.08 MiB [emitted] (name: entryA)
```

```bash
$ npm run build:prod
...
asset entryA.1f90470991f2fc5c7286.bundle.js 359 KiB [emitted] [immutable] [minimized] [big] (name: entryA) 1 related asset
```

| Development | Production |
|:---:|:----:|
|![](https://images.velog.io/images/tjdwns5123/post/333025e5-0d2b-4a3a-ad63-0049e9369373/image.png)|![](https://images.velog.io/images/tjdwns5123/post/e1ad492a-e030-4f7e-8a37-73da113e4140/image.png)|

기본적으로 웹팩 설정의 mode 설정을 `development` 에서 `production` 을 부여하는 것만으로도 `minimize`, `uglify`를 달성할 수 있다. webpack 5버전에 들어서 mode가 `production` 인 경우 내장된 [Terser-Webpack-Plugin](https://webpack.js.org/plugins/terser-webpack-plugin/)을 통해 리소스 최적화를 진행하기 때문이다.

최적화는 진행되었지만 진입점 파일이 여러곳일 경우 추가적인 처리가 좀더 필요하다. 이번엔 진입점을 여러개로 두고 각 진입점 파일에 동일한 벤더 모듈을 `import` 하여 사용하였다.

```bash
$ npm run build:dev
...
Entrypoint entryA 1.09 MiB = entryA.css 1 bytes entryA.bundle.js 1.09 MiB
Entrypoint entryB 1.09 MiB = entryB.css 1 bytes entryB.bundle.js 1.09 MiB
```
```bash
$ npm run build:prod
...
Entrypoint entryA [big] 359 KiB (1.57 MiB) = entryA.css 1 bytes entryA.21605d326788f03707cc.bundle.js 359 KiB 1 auxiliary asset
Entrypoint entryB [big] 359 KiB (1.57 MiB) = entryB.css 1 bytes entryB.ddd2e7affd6eb768d265.bundle.js 359 KiB 1 auxiliary asset

```

| Development | Production |
|:---:|:----:|
|![](https://images.velog.io/images/tjdwns5123/post/5af31391-d9ab-4ae1-ac08-219d2259b4a8/image.png)|![](https://images.velog.io/images/tjdwns5123/post/d7dabc9b-1da9-4638-8c7e-754587fede06/image.png)|

mode의 차이로 인해서 번들링 결과 파일의 사이즈는 이전처럼 줄어들었지만 같은 어플리케이션에서 두 번들링 결과 파일을 이용하는 경우 외부 벤더 모듈까지 두번 번들링 되어버린것을 확인할 수 있다. 배포상황에서는 리소스 낭비로 이어질 수 있는 상황이다.

이런 상황을 방지하기 위해서 웹팩은 진입점 파일로부터 디펜던시 그래프를 확인하여 동일하게 임포트 되는 모듈에 대해 별도의 번들링 파일을 구성하여 중복을 없앨 수 있는 기능을 지원하고 있다. 웹팩 설정에 다음 옵션을 추가한다.

```javascript
// webpack.common.js
module.exports = {
  ...
  optimization: {
    splitChunks: {
      chunks: 'all',
      name: 'chunk-vendors'
    }
  },
}
```

그 다음 다시 빌드하여 분석 이미지를 확인해보자.

| Development | Production |
|:---:|:----:|
|![](https://images.velog.io/images/tjdwns5123/post/e46f3363-635c-40db-b9a3-6db6dd910ff4/image.png)|![](https://images.velog.io/images/tjdwns5123/post/2af831ff-4398-4a16-ba43-fcc86b021a18/image.png)|

결과적으로 두 진입점에서 공통적으로 사용하는 벤더 모듈은 별도의 번들 파일에 생성되어 위치하게 되었으며 기존의 진입점 파일에서는 상호 배타적으로 사용하는 모듈들만 남게 된다.

<hr />

`Tree Shaking 동작 여부 판단`

임포트한 모듈의 내부 소스코드중에 사용하는 코드만 번들링에 포함하고 사용하지 않는 코드는 제외하는지 확인하고자 시나리오를 하나 마련했다.

```
$ tree ./animal

animal
├── cat
│   └── index.js
├── dog
│   └── index.js
└── index.js
```

```javascript
// entryA.js
import animal from './lib/animal'

animal.Cat.A()
animal.Cat.C()

animal.Dog.B()
animal.Dog.D()
```
```javascript
// animal/index.js
import Cat from './cat'
import Dog from './dog'

export default {
  Cat,
  Dog
}
```
```javascript
// animal/cat/index.js
function A () {
  console.log('Cat.A')
}
function B () {
  console.log('Cat.B')
}
function C () {
  console.log('Cat.C')
}
function D () {
  console.log('Cat.D')
}

export default {
  A,
  B,
  C,
  D
}
```
```javascript
// animal/dog/index.js
function A () {
  console.log('Dog.A')
}
function B () {
  console.log('Dog.B')
}
function C () {
  console.log('Dog.C')
}
function D () {
  console.log('Dog.D')
}

export default {
  A,
  B,
  C,
  D
}
```

트리 쉐이킹이 `ESM` 형태의 문법으로 모듈을 가져오고 내보내야 한다고 해서 이런 형태로 작성했다. `entryA` 진입점 파일의 내용을 보면 `animal` 의 내부모듈 `cat` 의 A, C 와 `dog` 의 B, D 함수를 호출했다. 웹팩이 정말 똑똑하다면 내가 호출한 4개의 함수 로직만 번들 결과 파일에 포함시키고 나머지는 제외했을 거라 기대해보았다.

번들 결과파일에서 호출하지 않은 함수도 번들링이 되었는지 확인해보았다.
안타깝게도, 내가 호출하지 않은 `Cat.B()` 함수가 포함되어 있었다.

![](https://images.velog.io/images/tjdwns5123/post/fd8eedf3-db28-4f6e-9319-e402d00a6c3e/image.png)

혹시 `development` 모드로 빌드해서 그랬을지 몰라서 `production` 모드로도 빌드해 보았다.

![](https://images.velog.io/images/tjdwns5123/post/8484df8c-15d9-4b8f-a274-50de83d43fc6/image.png)

실패.

왜 그럴까?

다시 한번 도큐먼트를 읽어보았다.

![](https://images.velog.io/images/tjdwns5123/post/014314e2-6828-4d44-a961-e87d33c411ac/image.png)

`usedExports: true` 를 추가해주고 라이브러리에서 딱 사용할 코드만 임포트하여 사용하는 식으로 변경해 보아야 겠다.

```javascript
// entryA.js
import {Cat, Dog} from './lib/animal'

Cat.A()
Cat.C()

Dog.B()
Dog.D()
```
```javascript
// animal/index.js
export * as Cat from './cat'
export * as Dog from './dog'
```
```javascript
// animal/cat/index.js
export function A () {
  console.log('Cat.A')
}
export function B () {
  console.log('Cat.B')
}
export function C () {
  console.log('Cat.C')
}
export function D () {
  console.log('Cat.D')
}

export default {
  A,
  B,
  C,
  D
}
```
```javascript
// animal/dog/index.js
export function A () {
  console.log('Dog.A')
}
export function B () {
  console.log('Dog.B')
}
export function C () {
  console.log('Dog.C')
}
export function D () {
  console.log('Dog.D')
}

export default {
  A,
  B,
  C,
  D
}
```

결과는?

![](https://images.velog.io/images/tjdwns5123/post/91d76eb4-07f4-426e-ae2a-6792177275bd/image.png)

오옷! 번들 결과파일에서 모듈 전체 소스코드중 사용하는 코드만 번들에 포함되었다!

참고로 `Tree-Shaking` 은 `Production` 모드에서만 동작한다고 한다.

#### `4. 결론`

배포 단계에서 특히 Single Page Application의 형태로 웹 어플리케이션을 로드하는 경우 빌드한 자바스크립트 리소스의 크기가 클 수록 실질적으로 앱을 구동할 수 있게 되는 Term이 길어질 수 밖에 없다. 웹팩이 리소스를 번들링하는 방식과 이러한 그것을 최소화 할 수 있는 방법들을 계속 확인해 나간다면 최종 번들 파일의 사이즈를 최적화하여 유저들에게 더 좋은 경험을 선사해 줄 수 있을 것이라 기대한다. 

#### `5. 참고 문헌`

- [Webpack 5 release (2020-10-10)](https://webpack.js.org/blog/2020-10-10-webpack-5-release/)

- [Major-Changes-Optimization](https://webpack.js.org/blog/2020-10-10-webpack-5-release/#major-changes-optimization)
- [Tree-Shaking](https://webpack.js.org/guides/tree-shaking/)

- [Babel](https://babeljs.io)

- [@babel/preset-env](https://babeljs.io/docs/en/babel-preset-env)
    
#### 6. [Github Repo](https://github.com/HamSungJun/Webpack5-Optimization)