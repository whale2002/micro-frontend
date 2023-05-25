我们将子应用打包成类库，在主应用中加载这个库（systemjs）

system 模块规范 umd amd esModule commonjs

生产环境打包成 system 模块规范

将我们通过 react 编写的应用通过 webpack 打包成 systemjs 规范，后面就可以直接使用 system 加载这个子应用

## 实现 systemjs

- 本质就是先加载依赖列表，再去加载真正的逻辑
- 内部通过 script 标本加载资源，给 window 快照保存先后状态

```js
<script type="systemjs-importmap">
  {
    "imports": {
      "react-dom": "https://cdn.bootcdn.net/ajax/libs/react-dom/18.2.0/umd/react-dom.development.js",
      "react": "https://cdn.bootcdn.net/ajax/libs/react/18.2.0/umd/react.development.js"
    }
  }
</script>

<script>
  const newMapUrl = {};

  // 解析依赖对象
  function processScript() {
    Array.from(document.querySelectorAll("script")).forEach((script) => {
      if (script.type === "systemjs-importmap") {
        const imports = JSON.parse(script.innerHTML).imports;
        Object.entries(imports).forEach(([key, value]) => {
          newMapUrl[key] = value;
        });
      }
    });
  }

  function load(id) {
    return new Promise((resolve, reject) => {
      const script = document.createElement("script");
      script.async = true;
      script.src = newMapUrl[id] || id;
      document.head.appendChild(script);

      // 此时会执行加载的代码
      script.addEventListener("load", () => {
        let _lastRefister = lastRegister;
        lastRegister = undefined;

        resolve(_lastRefister);
      });
    });
  }
  // 快照
  let set = new Set(); // 1）先保存window上的属性

  function saveGlobalProperty() {
    for (let k in window) {
      set.add(k);
    }
  }

  saveGlobalProperty();

  function getLastGlobalProperty() {
    // 看下window上新增的属性
    for (let k in window) {
      if (set.has(k)) continue;

      set.add(k);
      return window[k]; // 我通过script新增的变量
    }
  }

  let lastRegister;

  class SystemJS {
    import(id) {
      return Promise.resolve(processScript())
        .then(() => {
          const lastSepIndex = location.href.lastIndexOf("/");
          const baseURL = location.href.slice(0, lastSepIndex + 1); // 包含/

          if (id.startsWith("./")) {
            return baseURL + id.slice(2);
          }
        })
        .then((id) => {
          let execute;
          // 获取到具体的资源路径
          return load(id)
            .then((register) => {
              // execute是真正的渲染逻辑
              // setters 是用来保存加载后的资源，加载资源调用setters
              const { setters, execute: exe } = register[1](() => {});
              execute = exe;

              return [register[0], setters];
            })
            .then(([registeration, setters]) => {
              return Promise.all(
                registeration.map((dep, i) => {
                  return load(dep).then(() => {
                    const property = getLastGlobalProperty();
                    // 比如这里的property可能是React或ReactDOm
                    // 加载完毕后，会在window上增添属性 window.React window.ReactDOM
                    // 加载完毕依赖以后，设置资源
                    setters[i](property);
                  });
                  // 拿到的是函数，加载资源 将加载后的模块传递给这个setter
                })
              );
            })
            .then(() => {
              execute();
            });
        });
    }

    register(deps, declare) {
      // 将回调的结果保存起来
      lastRegister = [deps, declare];
    }
  }

  const System = new SystemJS();

  System.import("./index.js").then(() => {
    console.log("模块加载完毕");
  });
</script>
```
