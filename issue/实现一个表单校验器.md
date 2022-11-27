**介绍：**
```
  表单校验是业务开发中经常遇到的需求，本文章来记录和总结下使用ts实现表单校验的过程；本文我们使用的开发环境是vue3+tsx
```

## 1、首先我们会有一个表单的html如下：
```tsx
  <form onSubmit={handleSubmit}>
    <div class={s.formRow}>
      <div class={s.formItem}>
        <input 
          type="text" 
          v-model={formData.name}
          class={[s.formItem, s.input, `${errors['name']?.length ? s.error : ''}`]}
          onInput={() => handleFormCheck('name')} 
        >
      </div>
      <div class={[s.formItem_errorHint]}>
        <span>{errors['name']?.length ? errors['name'][0] : ''}</span>
      </div>
    </div>
    <div class={s.formRow}>
      <div class={s.formItem}>
        <EmojiSelect 
          v-model={formData.emoji}
          class={[s.formItem, s.emojiList, `${errors['emoji']?.length ? s.error : ''}`]} 
          onChange={() => handleFormCheck('emoji')} 
        />
      </div>
      <div class={[s.formItem_errorHint]}>
        <span>{errors['emoji']?.length ? errors['emoji'][0] : ''}</span>
      </div>
    </div>
  </form>
```

## 2、在实现表单校验前，我们先定义好需要的数据结构，并结合Typescript进行类型约束
```tsx
  // 1、首先在表单页面create.tsx, form本身的数据定义为formData，它的类型是FormData
  type FormData = {
    name: string
    emoji: string
  }
  const formData = reactive<FormData>({
    name: '',
    emoji: '',
  })

  // 2、在validate.ts中，我们向外暴露通用的FormError错误类型、Rule和Rules的规则类型
  // 重点：泛型约束 - 我们知道的是Rule规则的主键key，它的范围是约束在FormData的key范围内，即Rule的key的范围是《FormData的联合类型范围内》；同理FormError的主键key也一样，所以我们在定义时，考虑传入T的泛型来表示FormData类型
  export type FormError<T> = {
    [key in keyof T]?: string[]
  }
  export type Rule<T> = {
    key: keyof T
    message: string
  } & ( // 这里我们的表单校验初版，具备两种形式，一是校验是否必填；二是传入正则校验，所以这里交上两种情况的并集
    { type: 'required' } |
    { type: 'pattern', regex: RegExp }
  )
  export type Rules<T> = Rule<T>[]
```

## 3、在页面上，按上述类型来定义rules和errors的数据结构 & 表单校验方法：
```ts
  // 定义rules 和 errors
  import { Rules, validate, FormError, Rule } from '../../../../utils/validate';
  const errors = reactive<FormError<FormData>>({
    name: [],
    emoji: []
  })
  const rules: Rules = [
    { key: 'name', type: 'required', message: '请输入标签名' }, // 必填校验
    { key: 'name', type: 'pattern', message: '标签名称长度不大于5', regex: /^.{1,5}$/ }, // 标签名称长度正则校验
    { key: 'name', type: 'required', message: '请选择符号' },
  ]
  // 表单提交后的方法，终止页面的默认刷新事件 & 表单校验
  const handleSubmit = (e: Event) => {
    e.preventDefault()
    // 调用表单校验方法
    handleFormCheck()
  }

  const handleFormCheck = (validateField?: keyof FormData) => { // 可以选择传入validateField进行局部校验
    // 表单校验后，我们希望能得到校验后的新error对象，如果error对象中没有错误则通过；
    // 如果错误对中存在错误（数组长度>0)则会终止后续提交 & 页面上标红
    if (!validateField) { // 若是全量表单校验
      Object.assign(errors, { // 每次校验前清空现有错误
        name: [],
        emoji: [],
      })
      Object.assign(errors, validate(formData, rules))
    } else { // 若是局部表单校验, 则只校验传入的key对应规则
      Object.assign(errors, { ...errors, [`${validateField}`]: [] }) // 只清空当前局部key的错误，再更新它为校验后的值
      const filterRules: Rules<FormData> = rules.filter((item: Rule<FormData>) => item.key === validateField)
      Object.assign(errors, { ...errors, [`${validateField}`]: validate(formData, filterRules)[validateFiled] })
    }
  }
```

## 4、最后实现validate方法
```ts
  // validate.ts
  interface IFormData {
    [key: string]: number | string | null | undefined | IFormData
  }
  export const validate = <T extends IFormData>(formData: T, rules: Rules<T>) => { // 把rules的key范围约束在FormData的联合类型范围内
    // validate方法返回一个新的errors对象，表示本次校验结果，返回给外部
    const validateError: FormError<T> = {}
    // 遍历规则，如果出现错误，更新到错误对象里
    rules.map((rule: Rule<T>) => {
      const { key, type, message } = rule
      const value = formData[key] // 获取对应formItem的value
      switch (type) {
        case 'required': // 必填型校验
          if (value === null || value === undefined || value === '') {
            if (!validateError[key]) {
              validateError[key] = []
            }
            validateError[key].push(message)
          }
          break;
        case 'pattern': // 必填型校验
          const regex = (rule as Rule<T> & { regex: RegExp }).regex
          if (value && !regex.test(value.toString())) {
            if(!validateError[key]) {
              validateError[key] = []
            }
            validateError[key].push(message)
          }
          break;
        default:
          return  
      }
    })
    return validateError
  }
```

> tip：该版本实现了全量校验功能和部分校验功能，还可以继续拓展实现异步校验 和 自定义校验；即可结合async/await改造validate方法，也可以通过传入一个type: 'custom'的规则和一个自定义方法 + 用户自行调用callback来实现自定义校验