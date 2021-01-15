## 介绍 
 react开发中，我们经常封装 form 组件，将表单抽离成一个模块方便管理，例如常用的 antd 提供的 form.create() 来包裹封装的组件。经由Form.create() 包装过的组件会自带 this.props.form 属性，方便对表单进行操作，Form.create() 也是一个高阶组件。

> 高阶组件（HOC）：通过包裹（wrapper）被传入的组件，经过一系列处理，返回一个相对增强（enhanced）的组件的函数。
```
  高阶组件成立条件：
   1、接受组件作为参数
   2、返回一个组件
   二者满足其一即可
```


  ## 应用
  当我们基于 antd 的 form 封装好一个组件，可能会需要在外部调用该组件的方法，所以想通过 ref 来调用子组件方法。

**子组件**
```Jsx
const Child = forwardRef(({...props}, ref) =>  {
  // 代理组件的方法
  useImperativeHandle(ref, () => ({
    handleSubmit,
  }))

  const handleSubmit = async () => {
    ... 
    // 校验表单，并提交的相关处理
  }

  return(
    <Form
      className="Child"
    >
      ...
    </Form>)
  })

  // 由form.create包裹组件
  export default Form.create()(Child)
```

子组件我使用了函数组件来开发，所以会结合 React.forwardRef、和 useImperativeHandle 来自定义想要暴露给父组件的实例，让外层组件可以调用 ref 来控制组件内部。（通过ref来访问违背了设计的原则，尽可能少的使用这样的命令式代码）

**父组件**
```Jsx
const Parent = () => {
  let childRef = useRef()

  // 通过ref访问子组件， 调用子组件内部方法
  const handleClick = () => {
    if (useRef && useRef.handleSubmit) {
      let res = await useRef.handleSubmit()
      ...
    }
  }

return  (
  <div className="parent">
    <Child ref={ childRef } />
    <button onClick = { handleClick }>提交</button>
  </div>)
}

  export default Parent
```
然而点击父组件按钮后，会发现提交函数并没有被触发，于是我们打印出获取到的 ref

![image](https://user-images.githubusercontent.com/48883217/98462590-5234ed00-21f0-11eb-9f0e-fe13ec67c2db.png)

发现当前获取到的是经过 **form.create()** 处理后的**新对象**，不是我们想要的初始组件。
于是经过查阅，在antd官网上发现了用于正确获取初始组件的方法wrappedComponentRef

![image](https://user-images.githubusercontent.com/48883217/98462659-df784180-21f0-11eb-976d-20fdf0f0210e.png)


于是修改父组件的获取 ref 方式，即可成功调用子组件方法。
```Jsx
  <Child wrappedComponentRef={(inst)=>childRef = inst} />
```

[参考链接](https://www.cnblogs.com/krisZzz/p/11158003.html)