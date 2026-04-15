# 在Angular项目中进行页面跳转

## 进行route配置

要使用路由进行跳转，需要保证app目录下存在`app.routes.ts`，对其进行配置，在配置一个路径时，要么选择`component`，要么选择`redirectTo`，不可同时选择。
```typescript
export const routes: Routes = [
  { path: 'login', component: Login },
  { path: 'home', component: Home },
  { path: '**', redirectTo: 'login' }
];
```
需要注意以下几点：
1. `routes`选项必须可导出，否则在配置中无法生效。
2. 对于最新版的路由配置，通过provider已经自动进行配置，如果是老版本，仍然需要其他繁琐的配置。
3. 在`Angular`中，默认路由的配置使用`**`而不是`*`,否则不会生效。

## 在应用中引入

在配置完路由后，需要查看`app.ts`的配置，确保`app.ts`引入了`RouterOutlet`，这样才能在`app.html`中使用`RouterOutlet`进行对应页面的展示。之后在`app.html`中进行使用
```html
<router-outlet />
```

## 在页面中进行路由的切换
使用routerLink配置要切换的路由即可，
```html
  <a routerLink="/home">首页</a>
```

## 高级用法

### lazy load
在访问对应路由时才需要进行加载，使用`loadComponent`参数，值为一个匿名函数，在执行`import`内容后需要使用`then`，因为`import`的内容是一个异步的对象，需要在他完成后将内容返回。
```ts
// app.routes.ts
export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => 
      import('./features/dashboard/dashboard.component')
      .then(m => m.DashboardComponent)
  }
];
```

### 嵌套路由
单独配置`childer`参数即可。
```ts
// app.routes.ts
export const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () => 
      import('./pages/admin/admin.component')
      .then(m => m.AdminComponent),
    children: [
      {
        path: 'users',
        loadComponent: () => 
          import('./pages/admin/users/users.component')
          .then(m => m.UsersComponent)
      },
      {
        path: 'stats',
        loadComponent: () => 
          import('./pages/admin/stats/stats.component')
          .then(m => m.StatsComponent)
      }
    ]
  }
];
```

*written in 4/6/2026 By Ruize li*