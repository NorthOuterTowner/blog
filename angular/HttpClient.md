HTTP Client属于Service，因此需要通过依赖注入的方式进行使用。
## 配置configure
首先怕配置provider, 在`app.config.ts`中添加配置
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
    http = inject(HttpClient);
    /*Other Logic content*/

    this.http.get<Response>('url').subscribe((res)=>{
        console.log(res);
    })
} 
```
此处的get请求不同于fetch返回的Promise对象，其返回的Observable对象，可以使用rxjs进行复杂操作。
**需要注意**：HttpClient返回的是冷`Observable`，必须调用`subscribe()`方法，才可以真正的发出请求。

## Service
一般来说，需要将请求的发送封装在`Service`中，通过依赖注入将`Service`注入到使用的`ts`文件中。因此，可以首先根据需要创建`service.ts`。
```TypeScript
// advice.service.ts
import { HttpClient } from '@angular/common/http'

export class AdviceService {
    private http = inject(HttpClient);
    searchAdvice() {
        return this.http.get<Response>('url', timeout:3000)
            .pipe(
                map((response) => {
                    return response.slip.advice;
                }),
                catchError((error) => {
                    alert(error.message);
                    return of(`Error: ${error.message}`);
                }),
                finalize(() => {
                    return this.isLoading.set(false);
                })
            );
    }
    fetchAdvice() {
        this.searchAdvice().subscribe((advice) => {
            this.currentAdvice.set(advice)
        })
    }
}
```

*written in 4/5/2026 By Ruize li*