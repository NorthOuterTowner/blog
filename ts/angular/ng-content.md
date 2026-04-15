# 插槽

## ng-content

ng-content用于对拆分后的组件进行进一步的抽象，以加强组件的复用性，具体来说，在有些情况下，需要从父组件向子组件进行部分内容的传输，该情况类似于vue中的`slot`语法。

### Use
在子组件中使用`<ng-content/>`，此处即为插槽位置，父组件需要插入的`HTML`标签将会显示在此处。
```typescript
@Component({
  selector: 'custom-card',
  template: '<div class="card-shadow"> <ng-content/> </div>',
})
```
在父组件调用子组件的模块中传入需要的内容即可。
```HTML
<custom-card>
  <p>This is the projected content</p>
</custom-card>
```

### Official Guide
[angular.dev/guide/components/content-projection](https://angular.dev/guide/components/content-projection)

*written in 4/1/2026 By Ruize li*