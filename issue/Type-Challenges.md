**介绍：**
```
本文记录Typescript的一些高级用法和实战
```

**题目1，keyof**：
使用keyof改写函数

前言：keyof 是 Typescript 中的一个操作符，可以将一个类型映射为包括它所有成员的联合类型

```typescript
  class People {
    name: string;
    age: number;
    gender: string;
  }
  // 用keyof来获取目标类型的联合类型映射
  type p = keyof People // p: 'string' | 'number' | 'number'
```
 
我们再来声明一个Student的类
```typescript
class Student {
  private info: People;
  constructor (info: People) {
    this.info = info
  }
  public getInfo(key: string) {
    if (key === 'name' || key === 'age' || key === 'gender') {
      return this.info[key]
    }
  }
}
const student = new Student({
  name: 'aa',
  age: 14,
  gender: 'male',
})
console.log(student.getInfo('name')) // 'aa'
```
可以看到getInfo里添加了if语句 用来保证获取的key在People类里，如果输入一个不存在的key则返回undefined
我们也可以通过TS的keyof和extends来改造getInfo方法
下面重写Student类

**代码：**
```typescript
class Student {
  private info: People;
  constructor (info: People) {
    this.info = info
  }
  // 需要对输入的key做限制，保证输入key一定是People类里的key；
  // 可以使用T extends keyof People来让T继承自People的联合类型映射
  public getInfo<T extends keyof People>(T: string): People[T] {
    // 现在输入key必须保证是People类的key，否则会ts错误
    return this.info[key]
  }
}

```


**题目2，实现Pick**：
实现 TS 内置的 Pick<T, K>，但不可以使用它。
从类型 T 中选择出属性 K，构造成一个新的类型。
 
例如：
```typescript
interface Todo {
  title: string
  description: string
  completed: boolean
}

type TodoPreview = MyPick<Todo, 'title' | 'completed'>

const todo: TodoPreview = {
    title: 'Clean room',
    completed: false,
}

// 请写出 MyPick 的实现
type MyPick = any
```

**代码：**
```typescript
// 可知道 MyPick的入参1是Todo类，入参2是可供输入的联合类型
// 出参是一个新的筛选后的类型对象
// 思路：MyPick将限制用户输入的联合类型K，让K的参数收到约束，将其范围约束在Todo类的联合类型映射里
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P]
}
```

