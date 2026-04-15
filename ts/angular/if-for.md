# 模板中使用条件和循环

在`Angular`引入的模板HTML中进行使用，从而在模板中使用if和for，其功能与`Vue`中的v-if和v-for一致。

## Use
### if
```typescript
@if (a > b) {
  {{ a }} is greater than {{ b }}
} @else if (b > a) {
  {{ a }} is less than {{ b }}
} @else {
  {{ a }} is equal to {{ b }}
}
```

### for
在使用for的过程中必须使用`track`，`track`需要是`iterable`对象中可以唯一标识的变量，如果没有这一类型的变量，可以使用`$index`，但建议在`iterable`对象中创建一个唯一的变量，以提高渲染速度。
```typescript
@for (item of items; track item.id) {
  {{ item.name }}
}
```

# Official Guide
[angular.dev/guide/templates/control-flow](https://angular.dev/guide/templates/control-flow)

*written in 4/1/2026 By Ruize li*