Input和Output分别是父组件向子组件传递信息和子组件向父组件传递信息的方法，分别类似于Vue中的props和emit。

# Input
在子组件中定义input变量（input变量也是一个signal变量）

```TypeScript
@Component
export class CustomSlider {
    value = input<number>(0);
}
```

在父组件的模板部分中使用时将值进行绑定

```html
<custom-slider [value]="50"/>
```

# Output

在子组件中定义output变量，output变量同vue中的signal有类似之处

```TypeScript
@Component
export class CustomSlider {
    countOp = output<boolean>(false);

    onChange() {
        this.count.emit();
    }
}
```
父组件中监听事件
```html
<custom-slider (countOp)="countChange($event)"/>
```
```TypeScript
@Component
export class root {
    count = signal(0);

    countChange(newCnt: number) {
        count.update((currentCnt)=>{
            currentCnt = newCnt;
        });
    }
}
```
对于较老的项目也存在对其的支持
```TypeScript
export class CustomSlider {
  @Output('valueChanged') changed = new EventEmitter<number>();
}
```