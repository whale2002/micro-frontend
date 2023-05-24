我们将子应用打包成类库，在主应用中加载这个库（systemjs）
system 模块规范 umd amd esModule commonjs
生产环境打包成 system 模块规范

将我们通过 react 编写的应用通过 webpack 打包成 systemjs 规范，后面就可以直接使用 system 加载这个子应用
