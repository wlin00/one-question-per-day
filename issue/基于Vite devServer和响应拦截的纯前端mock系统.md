## 一、 关于mock
```
  mock假数据是为了让开发变得更方便，市面上常见的工具如：apifox、postman等，都能快速完成mock能力；
  本文主要介绍一下基于vite devServer和响应拦截的纯前端简易mock。
```

## 二、关于vite devServer
```
  我们知道，当vite devServer遇到一个不存在的地址，它并不会报404，而是永远渲染首页；即我们可以认为我们在vite devServer请求任何api都会有一个响应；那么基于这个思路我们可以结合axios来拦截响应，返回我们自定义的假数据来完成mock功能。
```

## 三、核心代码实现
(1) 首先，我们来到`发起请求的页面`，该页面业务逻辑主要是拉取了`标签列表`数据，其数据结构是一个`对象数组`;
我们可以在请求的params参数中添加一个标记`tagIndex`，表示当前会在`开发环境`对该接口进行`假数据mock`，而mock得到的数据则是`tagIndex`对应的数据去获取;
本次项目使用的技术栈主要是`vue3+tsx`
```ts
  // 发起请求的页面
  import { http } from '../utils/Http';
  import { ref, onMounted } from 'vue';

  export const ItemCreate = defineComponent({
    setup: (props, context) => {
      const refExpensesTags = ref<Tag[]>([])
      onMounted(async() => {
        const response = await http.get<{ resource: Tag[] }>('/tags', {
          kind: 'expense',
          _mock: 'tagIndex', // 加入_mock参数来让响应拦截器进行mock处理
        })
      })
      return () => (...)
    }
  })
```

(2) 我们来到封装好的 `http方法`， 该方法对 `axios`进行了上层封装，向外导出一个`Http`的类和一个`http`的方法方便进行http请求，也是为了更好的`控制代码`；
在这个工具方法中，我们针对mock，主要需要进行`响应拦截`, 即修改响应的状态码status和数据data，并且需要做好`隔离`，即只针对`本地开发环境`+`请求params中添加了标记`的请求进行mock
```ts
  // Http.tsx 响应拦截
  http.instance.interceptors.response.use((response: AxiosResponse) => { 
    mock(response) // 若当前请求成功且非mock，则本次响应拦截return false即无效化
    return response
  }, (error) => {
    if (mock(error.response)) { // 若mock数据有返回值，则使用mock
      return error.response
    } else {
      throw error
    }
  })
```
(3) 我们可以看到，在该拦截器中，对于所有响应，都会走mock方法；那么我们可以在mock方法中区分出不进行mock的方法，然后`return false`来使当前拦截器`无效化`
```ts
  // Http.tsx mock方法
  import { mockTagIndex } from './mock';
  const mock = (response: AxiosResponse) => {
    if (!['localhost', '127.0.0.1'].includes(location.hostname)) {
      return false
    }
    switch (response.config?.params?._mock) {
      case 'tagIndex': // 对标签列表接口的mock处理
        [response.status, response.data] = mockTagIndex(response.config)
        return true
      case '...more api' // 更多api的mock处理  
    }
    return false
  }
```
(4) 接下来在 `mock.tsx` 文件中完成对应接口的`假数据mock`处理就可以了；在造假数据方面，本次使用了`faker-js`来创建mock数据
```ts
  // mock.tsx
  import { faker } from '@faker-js/faker'
  import { AxiosRequestConfig } from 'axios';
  faker.setLocale('zh_CN')
  type Mock = (config: AxiosRequestConfig) => [number, any]

  export const mockTagIndex: Mock = (config) => { // 标签列表查询接口mock，返回faker-js制造的标签列表数据
    let id = 0
    const createId = () => ++id
    const createTag = (n = 1, attrs?: any) => new Array(n).fill(undefined).map(() => ({
      id: createId(),
      name: faker.lorem.word(),
      sign: faker.internet.emoji(),
      kind: config.params.kind,
      ...attrs
    }))
    if (config.params.kind === 'expenses')  {
      return [200, { resource: createTag(7) }]
    } else {
      return [200, { resource: createTag(20) }]
    }
  }
```

## 四、Http.tsx完整代码
```ts
  import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from "axios";
  import { Toast } from 'vant'
  import { debounce } from './index';
  import { mockSession, mockTagIndex } from './mock';

  // message(data) 弹出错误提示
  const message = debounce((msg: string) => {
    Toast.fail(msg)
  }, 800)

  type JSONValue = string | boolean | number | null | JSONValue[] | { [key: string]: JSONValue }[] // 定义post/patch请求的参数中的value格式，可能为 string、boolean、number、null、数组、对象数组等

  export class Http {
    instance: AxiosInstance
    constructor(baseURL: string){
      this.instance = axios.create({
        baseURL
      })
    }
    // 封装axios - 暴露出一个Http类和一个http请求实例 - get/post/patch/destroy, 
    get<T extends any>(url: string, query?: Record<string, string>, config?: Omit<AxiosRequestConfig, 'url' | 'params' | 'method'>) { // Omit用于在AxiosRequestConfig类型中去除指定字段
      return this.instance.request<T>({ // 传入范型T可定义本次请求的返回值类型
        ...config,
        url,
        params: query,
        method: 'get'
      })
    }
    delete<T extends any>(url: string, query?: Record<string, string>, config?: Omit<AxiosRequestConfig, 'url' | 'params' | 'method'>) {
      return this.instance.request<T>({
        ...config,
        url,
        params: query,
        method: 'delete'
      })
    }
    post<T extends any>(url: string, data?: Record<string, JSONValue>, config?: Omit<AxiosRequestConfig, 'url' | 'data' | 'method'>) {
      return this.instance.request<T>({
        ...config,
        url,
        data,
        method: 'post'
      })
    }
    patch<T extends any>(url: string, data?: Record<string, JSONValue>, config?: Omit<AxiosRequestConfig, 'url' | 'data' | 'method'>) {
      return this.instance.request<T>({
        ...config,
        url,
        data,
        method: 'patch'
      })
    }
  }

  export const http = new Http('/api/v1')

  const mock = (response: AxiosResponse) => {
    if (location.hostname !== 'localhost'
      && location.hostname !== '127.0.0.1'
      ) { return false }
    switch (response.config?.params?._mock) {
      case 'tagIndex':
        [response.status, response.data] = mockTagIndex(response.config)
        return true
      case 'session':
        [response.status, response.data] = mockSession(response.config)
        return true
    }
    return false
  }


  // 请求拦截，登陆后响应头添加token
  http.instance.interceptors.request.use((config: AxiosRequestConfig) => {
    const jwt = localStorage.getItem('jwt')
    if (jwt) {
      config.headers!.Authorization = `Bearer ${jwt}`
    }
    return config
  })

  // 响应拦截，mock处理
  http.instance.interceptors.response.use((response: AxiosResponse) => { 
    mock(response) // 若当前请求成功且非mock，则本次响应拦截return false即无效化
    return response
  }, (error) => {
    if (mock(error.response)) { // 若mock数据有返回值，则使用mock
      return error.response
    } else {
      throw error
    }
  })

  // 响应拦截，非200状态码提示
  http.instance.interceptors.response.use((response: AxiosResponse) => {
    return response
  }, (error) => {
    if (error.response) {
      // 根据api响应头的状态码进行提示
      message(error.response.data?.message)
    }
    throw error
  })
```


