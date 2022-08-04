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
interface People {
  name: string;
  age: number;
  gender: string;
}
class Student {
  constructor (private info: People) {
    this.info = info
  }
  // 需要对输入的key做限制，保证输入key一定是People类里的key；
  // 可以使用T extends keyof People来让T继承自People的联合类型映射
  public getInfo<T extends keyof People>(key: T): People[T] { // T extends keyof 接口，会约束T为接口中的某个合法联合类型
    // 现在输入key必须保证是People类的key，否则会ts错误
    return this.info[key]
  }
}

const stu = new Student({
  name: '123',
  age: 13,
  gender: 'male'
})

console.log(stu.getInfo('name'))
```


**题目2，实现Pick**：
实现 TS 内置的 Pick<T, K>，但不可以使用它。
从类型 T 中选择出属性 K，构造成一个新的类型。
 
示例：
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
type MyPick<T, K extends keyof T> = { // K extends keyof 接口，会约束K为接口T中的某个合法联合类型， 可用in来遍历
  [P in K]: T[P]
}
```


**题目3，实现Readonly**：
不使用内置的 Readonly<T>, 而是自己去实现一个，该 Readonly 会接受一个泛型参数，并返回一个完全一样的类型，但所有类型会被readonly修饰
 
示例：
```typescript
interface Todo {
  title: string,
  desc: string
}
const todo: MyReadonly<Todo> = {
  title: 'Hey',
  desc: 'foo'
}
todo.title = 'Hi'
todo.desc = 'animal' // Error: cannot reassign a readonly property
// 请写出 MyReadonly 的实现
type MyReadonly = any
```

**代码：**
```typescript
// 可知道 MyReadonly的入参是Todo类
// 思路：MyReadonly返回一个对象，in遍历Todo类的联合类型即mapped操作(keyof：将原接口用keyof映射为联合类型, 即lookup操作），将原接口每个属性设置到当前接口（取每个接口的值：indexed），并添加readonly修饰
type MyReadonly<T> = {
  readonly [P in keyof T]: T[P] // P in keyof 接口，可以遍历接口
}
```


**题目4，实现元组转对象**：
传入一个元组类型，将这个元组类型转换为对象类型，这个对象类型的键/值都是从元组中遍历出来
 
示例：
```typescript
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const
type result = TupleToObject<typeof tuple> // expected { tesla: 'tesla', 'model 3': 'model 3', 'model X': 'model X', 'model Y': 'model Y'}
// 请写出 TupleToObject 的实现
type TupleToObject = any
```

**代码：**
```typescript
type TupleToObject<T extends readonly (symbol | number | string)[]> = { // 让T约束在可输入symbol、number、string的只读（readonly）的元组类型下
  [P in T[number]]: P // P in T[number] 可以遍历元组
}
```


**题目5，第一个元素**：
实现一个通用First<T>，它接受一个数组T并返回它的第一个元素的类型。
本题记录它的四种解法
 
示例：
```typescript
type arr1 = ['a', 'b', 'c']
type arr2 = [3, 2, 1]
type arr3 = []
type head1 = First<arr1> // expected to be 'a'
type head2 = First<arr2> // expected to be 3
type head3 = First<arr3> // expected to be never
// 请写出 TupleToObject 的实现
type First<T extends any[]> = any
```

**代码：**
```typescript
// 解法1：直接判断元组是否为空，是的话取第一项会是never，否则取返回第一项
type First<T extends any[]> = T extends [] ? never : T[0]
// 解法2：判断元组长度是否为0
type First<T extends any[]> = T['length'] extends 0 ? never : T[0] // 求length 是 indexed操作
// 解法3：遍历元组 -> 若元组为空，T[0]为undefined，而T[number]的遍历结果是never；若元祖非空，则T[0] 会存在于T[number]的遍历的union类型里；
type First<T extends any[]> = T[0] extends T[number] ? T[0] : never
// 解法4：使用infer模拟元组的解构操作
type First<T extends any[]> = T extends [infer first, ...infer rest] ? first : never

```


**题目6，获取元组长度**：
创建一个通用的`Length`，接受一个`readonly`的数组，返回这个数组的长度
 
示例：
```typescript
  type tesla = ['tesla', 'model 3', 'model X', 'model Y']
  type spaceX = ['FALCON 9', 'FALCON HEAVY', 'DRAGON', 'STARSHIP', 'HUMAN SPACEFLIGHT']
  type teslaLength = Length<tesla> // expected 4
  type spaceXLength = Length<spaceX> // expected 5
  // 请写出 TupleToObject 的实现
  type Length<T> = any
```

**代码：**
```typescript
  type Length<T extends readonly any[]> = T['length']
```


**题目7，Exclude**：
  实现内置的Exclude <T, U>类型，但不能直接使用它本身。
  > 从联合类型T中排除U的类型成员，来构造一个新的类型。
 
示例：
```typescript
  type Result = MyExclude<'a' | 'b' | 'c', 'a'> // 'b' | 'c'
  // 请写出 Exclude 的实现
  type MyExclude<T, U> = any
```

**代码：**
```typescript
  // union1 extends union2 -> 代表两层遍历union；
  // 三元表达式的 ？ 则表示在union2遍历到了union1中存在的映射（即成功匹配），则本题中剔除 返回never；
  // 否则return 不存在于union1中的元素
  // 即 Union1 extends Union2 时，它的行为是循环遍历匹配；若匹配上，则返回匹配上的Union类型
  type MyExclude<T, U> = T extends U ? never : T 
```


**题目8，Awaited**：
示例：
```typescript
  type ExampleType = Promise<string>
  type Result = MyAwaited<ExampleType> // 希望输出：string
  // 请写出 Awaited 的实现
  type MyAwaited<T> = any
```

**代码：**
```typescript
  type MyAwaited<T extends Promise<unknown>> = T extends Promise<infer X> 
    ? X extends Promise<unknown> // 判断内部参数是否promise
      ? MyAwaited<X>
      : X
    : T // 入参不是promise 直接返回
```


**题目9，If**：
实现一个 `IF` 类型，它接收条件类型 `C` ，一个判断为真时的返回类型 `T` ，以及判断为假时的返回类型 `F`。 C 只能是 true 或者 false， T 和 F 可是任意类型。
示例：
```typescript
  type A = If<true, 'a', 'b'>  // expected to be 'a'
  type B = If<false, 'a', 'b'> // expected to be 'b'
  // 请写出 If 的实现
  type If<C, T, F> = any
```

**代码：**
```typescript
  type If<C extends (true | false), T, F> = C extends true ? T : F
```


**题目10，Concat**：
在类型系统里实现 JavaScript 内置的 `Array.concat` 方法，这个类型接受两个参数，返回的新数组类型应该按照输入参数从左到右的顺序合并为一个新的数组。
示例：
```typescript
  type Result = Concat<[1], [2]> // expected to be [1, 2]
  // 请写出 Concat 的实现
  type Concat<T, U> = any
```

**代码：**
```typescript
  type Concat<T extends any[], U extends any[]> = [...T, ...U]
```