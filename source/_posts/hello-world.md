---
title: 旧项目接入qiankun微前端方案实践——hash模式
---

### 1、前言

这里先放一个官方文档链接：[qiankun](https://qiankun.umijs.org/zh/guide/getting-started)
基本的操作，包括各种接入调用，基础配置啥的，感兴趣的同学，可以自行点击上面的官网链接查看。
这里，我主要阐述一下，作为主应用如何与业务子应用对接开发。

### 2、在主应用中注册微应用

每增加一个子应用都要配置一个对象

```
registerMicroApps([
  {
    name: 'react app', // 子应用名称
    entry: '//localhost:7100',  // 子应用访问域名
    container: '#yourContainer', // 子应用放置节点id
    activeRule: '/yourActiveRule', // 路由规则
  }
]);

```

这里面 container 是给装子应用的容器，一般是 div 给个 ID 就行，具体放在哪个页面，由你自己的业务决定。
activeRule 是路由规则，根据路由模式决定，下面会着重讲。

### 3、关于 activeRule（路由规则），主应用和子应用分别怎么设置

由于主应用的路由一直以来选用的是 hash 模式，所以目前不管是子应用还是主应用，都只推荐是用 hash 模式，才能更平滑且低成本低的接入 qiankun 微前端方案。

```
registerMicroApps([
  {
    name: "vueApp",
    entry: "//192.168.20.49:8080",
    container: "#appContainer",
    activeRule: "/#/micro/app-vue"
  }
]);
```

这是我本地的一个 demo 例子，点击菜单打开子应用，那就在本身的路由视图那里加一个 id 为 appContainer 的 div，双方都用 hash 模式，activeRule 要加上'/#/'，micro 是我单独给子应用加的一个路由目录。

### 4、子应用设置

main.js 参照官网，在顶部加入`import "./public-path";`文件内容如下：

```javascript
if (window.__POWERED_BY_QIANKUN__) {
  // eslint-disable-next-line no-undef
  __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
}
```

其次，在 main.js 文件中加入代码：

```javascript
let router = null;
let instance = null;
function render(props = {}) {
  const { container } = props;
  router = new VueRouter({
    // base: window.__POWERED_BY_QIANKUN__ ? "/app-vue/" : "/",
    routes,
  });

  // 判断 qiankun 环境则进行路由拦截，判断跳转路由是否有 /micro 开头前缀，没有则加上
  if (window.__POWERED_BY_QIANKUN__) {
    router.beforeEach((to, from, next) => {
      if (!to.path.includes("/micro")) {
        next({
          path: "/micro/app-vue" + to.path,
        });
      } else {
        next();
      }
    });
  }

  instance = new Vue({
    router,
    store,
    render: (h) => h(App),
  }).$mount(
    container ? container.querySelector("#operatMicroApp") : "#operatMicroApp"
  );
}
// 独立运行时
if (!window.__POWERED_BY_QIANKUN__) {
  render();
}
export async function bootstrap() {
  console.log("[vue] vue app bootstraped");
}
export async function mount(props) {
  console.log("[vue] props from main framework", props);
  render(props);
}
export async function unmount() {
  instance.$destroy();
  instance.$el.innerHTML = "";
  instance = null;
  router = null;
}
```

最后，需要在路由文件中加入跟主应用一样路由匹配规则，上述中规则是`activeRule: "/#/micro/app-vue"`，所以如下：

```javascript
import Home from "../views/Home.vue";

let prefix = "";

// 判断是 qiankun 环境则增加路由前缀
if (window.__POWERED_BY_QIANKUN__) {
  prefix = "/micro/app-vue";
}

const routes = [
  {
    path: prefix + "/",
    name: "home",
    component: Home,
  },
  {
    path: prefix + "/about",
    name: "about",
    // route level code-splitting
    // this generates a separate chunk (about.[hash].js) for this route
    // which is lazy-loaded when the route is visited.
    component: () =>
      import(/* webpackChunkName: "about" */ "../views/About.vue"),
  },
];

export default routes;
```

看到这里，你应该明白了，路由规则得保持跟主应用中一致，才能保证命中。
