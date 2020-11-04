> 开发一个小需求时，为了缩小项目体积没有引用组件库，所以需要实现一个分页组件，这里参考了Antd的带省略号交互模式。

**Pagination.jsx**
```Jsx
  import React, { useCallback, useState, useEffect } from 'react'
  import './Pagination.scss'

  function Pagination({ 
    current = 1, // 当前值
    total = 100, // 分页大小
    onChange = null, // 页数改变时的回调
    onPageSizeChange, // 分页大小改变时的回调
   }) {

  const [value, setValue] = useState(current) // 当前值
  const [list, setList] = useState([]) // 分页数组
  const [inputValue, setInpValue] = useState('') // 分页器输入框

  useEffect(() => {
    watcher(value)
   }, [])

  // 数组去重
  const unique = (arr) => {
    let map = {}
    arr.forEach(item => {
      map[item] = true
    })
    return Object.keys(map).map(e => parseInt(e, 10))
  }
    
  // 更新当前分页数组
  const watcher = useCallback((val) => {
    // 合法数据筛选, 排序
    let filter = unique([
      val,
      val - 1,
      val - 2,
      val - 3,
      val - 4,
      val + 1,
      val + 2,
      val + 3,
      val + 4,
    ].filter(item => {
    return item >= 1 && item <= total
  }).sort((a, b) => a-b))

  // 边界判断
  let temp = val > 3 && val < total - 2 ? 
    [val - 2, val - 1, val, val + 1, val + 2] :
    (val <= 3 ? filter.slice(0, 5) : filter.slice(-5)) 

  // 首尾添加数据并去重，数组差值大于1时添加省略号
  let pages =unique([1, ...temp, total]).reduce((prev, current, index, arr) => {
    prev.push(current)
    // 判断前后差值大于一时，添加省略号
    arr[index + 1] !== undefined &&
    arr[index + 1] - arr[index] > 1 &&
    prev.push('•••')
    return prev
  }, [])
    setList(pages)
    return pages
  })
    
   // 页码改变的回调
  const pageChange = useCallback((page) => {
    if(page === value || !page || page === '•••') return
    setValue(page)
    watcher(page)
    onChange && onChange(page)
  })

    // 上一页
  const onPrev = () => {
    if (value <= 1) return
      pageChange(value - 1)
    }
    
  // 下一页
  const onNext = () => {
    if (value >= total) return
      pageChange(value + 1)
    }

  // 组件pagesize改变区域
   const handleSelect = (e) => {
     var currentSelect = Number(e.target.value)
     setValue(1)
     onChange && onChange(1)
     onPageSizeChange && onPageSizeChange(currentSelect)
   }

   // 分页器跳转
   const handleJump = () => {
     let page = inputValue === "" ? 1 : Number(inputValue)
     if (1 <= page && page <= total) {
       setValue(page)
       onChange && onChange(page)
     }
   }

    // 分页器输入回调
  const handleInpChange = (e) => {
    setInpValue(e.target.value)
  }

  return (
    <div className="wlin-pager">
      <div className={`wlin-pager-nav prev ${value === 1 && 'wlin-pager-default'}`}>
        <button onClick={onPrev} disabled={value === 1}>{'<'}</button>   
       </div>
        {
          list && list.map((item, index) => (
            <span className={`wlin-pager-item ${item === value && 'current'} ${item === '•••' && 'icon'}`} onClick= 
              {pageChange.bind(null, item)}  key={index}>{item}</span>
           ))
         }
         <div className={`wlin-pager-nav next ${value === total && 'wlin-pager-default'}`}>
           <button onClick={onNext} disabled={value === total}>{'>'}</button>   
          </div>
          <div className="wlin-pager-select wlin-pager-box">
            <select onChange={handleSelect}>
              <option value="10">10</option>
              <option value="20">20</option>
              <option value="30">30</option>
              <option value="40">40</option>
            </select>
          </div>
          <div className="wlin-pager-jump">
            <input type="text" value={inputValue} onChange={handleInpChange} />
            <span onClick={handleJump}>跳转</span>
          </div>            
        </div>
      )
  }
  export default Pagination
```

**Pagination.scss**
```scss
.wlin-pager {
    user-select: none;
    display: flex;
    justify-content: flex-start;
    align-items: center;
    &-default{
        cursor: default !important;
        >button{
            cursor: default !important;
        }
    }
    &-item{
        border: 1px solid #e1e1e1;
        border-radius: 4px;
        padding: 0 4px;
        color: #000;
        display: inline-flex;
        justify-content: center;
        align-items: center;
        font-size: 12px;
        min-width: 23px;
        height: 20px;
        margin: 0 4px;
        cursor: pointer;
        &:hover{
            border-color: rgb(56, 134, 165)
        }
        &.current {
            cursor: default;
            border-color: rgb(56, 134, 165)
        }
        &.icon {
            border: none;
            cursor: default !important;
        }
    }

    &-box {
        display: inline-block;
        >select{
          margin-left: 20px;
        }
      }
      &-select {
        width: 70px;
        > select {
          width: 100%;
        }
      }
      &-jump {
        display: inline-block;
        margin-left: 40px;
        >input{
          margin-right: 5px;
          display: inline-block;
          width: 40px;
          box-sizing: border-box;
          padding: 0 10px;
        }
        >span{
          display: inline-block;
          width: 36px;
          color: #fff;
          background-color: rgb(20, 127, 214);
          line-height: 20px;
          font-size: 14px;
          cursor: pointer;
          text-align: center;
          height: 20px;
          border-radius: 4px;
        }
        >span:hover{
          opacity: 0.7;
        }
      }
      &-nav {
        cursor: pointer;
        display: inline-flex;
        justify-content: center;
        align-items: center;
        background: rgb(104, 122, 141);
        height: 20px;
        border-radius: 4px;
        font-size: 12px;
        margin: 8px 4px;
        >button{
          min-width: 23px;
          &:not(.disabled){
              cursor: pointer;
          }
        }
        >button.disabled {
          fill: rgb(148, 162, 175);
          background: rgb(110, 122, 136);
          cursor: default;
        }
      }
}
```

使用方式

```Jsx
  <Pagination 
    current={2}
    total={50}
    onPageSizeChange={handleSizeChange}
    onChange={handleChange}
  ></Pagination>
```

效果图

![image](https://user-images.githubusercontent.com/48883217/97953148-9a13d880-1dda-11eb-8492-753a6401daa4.png)
