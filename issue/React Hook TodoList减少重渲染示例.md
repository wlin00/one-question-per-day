### 1、本文分享一下，使用 React Hook 实现TodoList

> 实现一个简单的列表小demo，使用 `useCallback`、`React.memo` 等优化手段最小化列表重渲染

```ts
  1、首先，我们的目的是封装两个组件：《List.tsx》列表组件，和《ListItem.tsx》列表元素子组件；
  2、如果每个《ListItem.tsx》子组件，都有操作按钮如：删除、更新，那么假设我们什么都不处理，只是单纯删除某一个《列表元素》，那默认整个列表都会进行重渲染;
  3、为了减少没有必要的重渲染，例如我们只希望让《点击的某一个列表元素》进行重渲染，其余列表元素没有修改可以保持不变，并且我们新增元素其余的列表元素也不用额外重渲染，接下来我们来实现这样的一个简单小demo。
```

### 2、前置处理

为了更清楚直观的看到 `组件是否渲染`，有一些不同的方法来识别,最简单的就是勾选 `React dev tools` 中的` Highlight Updates` 选项即可：

![image](https://img2018.cnblogs.com/blog/1588732/202001/1588732-20200104153408704-1992752131.png)

勾选图中第一个checkbox，这样当 `刷新页面` 或者进行 `各种业务交互` 时，如果组件的边框进行`闪烁提示`，就代表组件进行重渲染了，这可以让我们发现不需要的重渲染。

## 3、实现List.tsx列表组件
```tsx
  // React Hook TodoList场景，使用useCallback、React.memo等手段最小化列表重渲染
  import * as React from 'react'
  import s from './List.module.scss'
  import { useState, useCallback } from 'react';
  import { MemoizedListItem } from './ListItem';
  interface IListProps {
  }
  export type Item = {
    name: string,
    id: string,
    isCompleted?: boolean
  }
  // 列表组件
  export const List: React.FC<IListProps> = () => {
    // 维护列表数组
    const [list, setList] = useState<Item[]>([
      { name: 'Apple', id: '1', isCompleted: false },
      { name: 'Banana', id: '2', isCompleted: false },
      { name: 'Orange', id: '3', isCompleted: false },
      { name: 'Noodle', id: '4', isCompleted: false },
      { name: 'Watermelon', id: '5', isCompleted: false },
    ])
    // 删除列表元素
    const handleDelete = useCallback((id: string) => {
      setList((prev) => prev.filter((item) => item.id !== id))
    }, [])
    // 更新列表元素状态
    const handleUpdate = useCallback((id: string) => {
      // 由于 useState 的更新是异步的, 所以这里使用setList回调是确保拿到更新后的list数组
      setList((prev) => {
        const listSlice = [...prev]
        const updateIndex = listSlice.findIndex((item) => item.id === id)
        const updateItem = listSlice.find((item) => item.id === id)!
        listSlice.splice(updateIndex, 1, {
          ...updateItem,
          isCompleted: true
        })
        return listSlice
      })
    }, [])
    // 新增列表元素
    const handleAdd = () => {
      setList((prev) => [...prev, {
        id: Math.random().toString(36).slice(2, 7), // 每次新增列表元素，新增的id是一个随机的6位字符串
        name: 'AddItem'
      }])
    }

    return (<>
      <div className={s.list}>
        {list.map((item: Item) => (
          <MemoizedListItem 
            key={item.id} 
            item={item} 
            onDelete={handleDelete} 
            onUpdate={handleUpdate} 
          />
        ))}
      </div>
      <div className={s.addWrap}>
        <span onClick={handleAdd}>add</span>
      </div>
    </>
    )
  }
```

## 4、实现ListItem.tsx列表元素子组件
```tsx
import s from './List.module.scss'
import { Item } from './List';
import React from 'react';
interface IListItemProps {
  onDelete: (id: string) => any,
  onUpdate: (id: string) => any,
  item: Item
}
// 列表元素组件
export const ListItem: React.FC<IListItemProps> = ({ item, onDelete, onUpdate }) => {
  return (
    <div className={s.listItem}>
      <span className={item.isCompleted ? s.isCompleted: ''}>{item.name}</span>
      <span className={s.buttonWrap}>
        <span className={[s.button, s.normalStyle].join(' ')} onClick={() => onUpdate(item.id)}>update</span>
        <span className={s.button} onClick={() => onDelete(item.id)}>delete</span>
      </span>
    </div>
  )
}
// 普通模式 - 添加/删除/更新 任意一个ListItem会造成整个列表重渲染
// export const MemoizedListItem = ListItem 

// React.memo模式，默认写法，根据key比对，如果listItem的key相同则不进行重渲染
// export const MemoizedListItem = React.memo(ListItem)

// React.memo模式，使用参数2来自定义比对，当前后的props满足：id、和isCompleted字段都不变时可以不进行重渲染
export const MemoizedListItem = React.memo(ListItem, 
  (prevProps: any, nextProps: any) => 
    prevProps.item.id === nextProps.item.id && 
    prevProps.item.isCompleted === nextProps.item.isCompleted
)
```

## 4、添加样式
```scss
  // List.module.scss文件
  .list {
    width: 100%;
  }
  .listItem {
    display: flex;
    height: 50px;
    align-items: center;
    justify-content: flex-start;
    font-size: 16px;
    padding: 0 20px;
    box-sizing: border-box;
    color: #333;
    border: 1px solid #ddd;
    margin: 20px;
  }
  .buttonWrap {
    margin-left: auto;
  }
  .button {
    display: inline-flex;
    height: 32px;
    border-radius: 8px;
    padding: 0 20px;
    cursor: pointer;
    font-size: 14px;
    background: #39f;
    color: #fff;
    align-items: center;
    justify-content: center;
    &:hover {
      opacity: .7;
    }
  }
  .normalStyle {
    background-color: #ddd;
    color: #fff;
    margin-right: 5px;
  }
  .addWrap {
    display: flex;
    padding: 20px 0;
    align-items: center;
    justify-content: center;
    > span {
      background: #39f;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      padding: 0 20px;
      height: 32px;
      font-size: 14px;
      color: #fff;
      border-radius: 8px;
    }
  }
  .isCompleted {
    text-decoration: line-through;
  }
```

## 5、总结
```ts
  1、上述代码可以看到，其实就是用React.memo包裹了子组件《ListItem.tsx》，来减少了它不必要的渲染，主要是通过在React.memo第二个参数的回调里自定义了重渲染条件，即当列表元素前后的Props里的id和isCompleted状态两个字段都不发生变化时，来记忆我们的ListItem组件从而减少重渲染；
  2、同时我们使用useCallback来缓存了我们的删除、更新方法；当函数作为 props 传递给子组件时，useCallback 可以缓存函数，避免不必要的重新渲染子组件。
```

## 6、改造前后对比

（1）改造前，即子组件不使用 `React.memo` 包裹，那么每次 `添加/删除/更新` 某个列表元素，所有列表元素都会进行重渲染
 
 > 示例图：点击第三行元素Orange的《update》更新按钮


 ![image](https://s2.loli.net/2023/05/27/U8OaEhfbjKRn6WT.png)

（2）改造前，子组件使用 `React.memo` 包裹，每次 `添加/删除/更新` 某个列表元素，其余列表元素会被memo住，避免多余重渲染

 > 示例图：点击第三行元素Orange的《update》更新按钮

 ![image](https://s2.loli.net/2023/05/27/Gd1Umeb7rluPHVq.png)



