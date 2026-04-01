# Angular常用命令
## 创建项目
创建新项目
```bash
ng new [app-name]
```
指定包管理器
```bash
--package-manager=pnpm
```
如果是v22前的Angular版本，需在创建时选择`zoneless`以提高性能。

## 创建组件
```bash
ng generate component [component-name]
```
或将`generate`简写为`g`,将`component`简写为`c`，即
```bash
ng g c [component-name]
```
创建后的`selector`名称会在输入的`component-name`前加上`app-`前缀。

*written in 4/1/2026 By Ruize li*

## 创建服务
```bash
ng g service [service-name]
```