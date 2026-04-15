# 在Vue中进行页面跳转

## 配置路由和对应Vue界面
在`src/router/index.ts`中定义一个`routes`，其类型为`Array<RouterRecordRaw>`，需要定义`path`和`component`，即可将路径和组件对应。

之后通过`createRouter`创建`router`内容，并设定`history`属性，最后导出`router`。
```typescript
import { createRouter, createWebHistory } from "vue-router";
import type { RouteRecordRaw } from "vue-router";
import Home from "../views/Home.vue";
import User from "../views/userViews/User.vue";

const routes: Array<RouteRecordRaw> = [
  {
    path: "/",
    name: "Home",
    component: Home,
  },
  {
    path: "/user",
    name: "User",
    component: User
  }
];

const router = createRouter({
  history: createWebHistory(),
  routes,
});

export default router;
```

在项目根目录的`main.ts`中导入`router`，然后使用即可。
```typescript
app.use(router)
```

## 使用通配符

为在用户访问不存在路由的时候可以有较好的用户体验，可以使用通配符进行判断，在配置路由时加入下面内容即可。其中`:`表示一个动态参数，即该`path`的内容接受一个任意名称的动态参数，此处`pathMatch`名称是基于习惯，实际操作中可以进行修改，`.*`匹配任意字符串零次或多次，末尾再加`*`可以将路由按`/`划分为数组，即可通过`route.params.pathMatch`获得`route`数组。
```typescript
{
    path: '/:pathMatch(.*)*',
    name: '404',
    component: NotFound404,
}
```
上文所述的动态参数是可以在前端通过访问的路由动态获取的参数，如定义一个名为
```
http://localhost:5173/user/:id
```
的路由，可以在访问对应路由时，在`vue`文件中使用
```js
const userId = route.params.id
```
获取对应的参数。

## 更改启动的端口号

在使用`vite`的情况下，修改`vite.config.ts`即可。
```ts
export default defineConfig({
  plugins: [vue()],
  server: {
    port: 6173
  }
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src')
    }
  }
})

```

## 子路径与子组件
需要设置子路径和子组件的情况下，通过`children`参数进行设置即可，内部的语法同外部一致。
```ts
  {
    path: "/user",
    name: "User",
    component: User,
    children: [
      {
        path: "profile",        // 子路径，实际访问：/user/profile
        component: UserProfile,
      },
      {
        path: "settings",       // 实际访问：/user/settings
        component: UserSettings,
      },
    ],
  },
```

## 路径跳转
路径跳转可通过声明式的方法或则编程式的方法实现，声明式的方法直接写在模板中，
```html
<router-link to="/about">关于我们</router-link>
```

*written By Ruize.li on 11/4/2026*