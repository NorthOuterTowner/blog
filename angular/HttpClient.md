HTTP Client属于Service，因此需要通过依赖注入的方式进行使用。
## 配置configure
在`app.config.ts`中添加配置
```typescript
export const appConfig: ApplicationConfig = {
    providers: [
        provideHttpClient(withFetch());
    ]
}
```
## 在程序中注入
```typescript
import { HttpClient } from '@angular/common/http'
export class App {
    http = inject(HttpClient);
} 
```
*个人认为：* `Angular`中的`DI`使用与`Springboot`有类似之处。在`Springboot`中注入依赖时使用需使用`@Component`将内容注入`IOC`容器，随后通过`@Autowired`获取所需的变量。
```Java
@Autowired
private Service InjectService;
```
## 使用HttpClient
```typescript
export class App {
    HttpClient = inject(HttpClient);
    /*Other Logic content*/

    this.http.get<Response>('url').subscribe((res)=>{
        console.log(res);
    })
} 
```
此处的get请求不同于fetch返回的Promise对象，其返回的Observable对象，可以使用rxjs进行复杂操作。