# Learn React

## 1、用React实现一个井字游戏
`入口文件App.tsx` 
```tsx
  import { useState, useCallback, useMemo } from 'react';
  import './App.scss'
  import TimeTravelListItem from './components/TimeTravelListItem';
  import { BOARDSQUARE } from './enums';
  import Board from './components/Board';
  // APP
  export default function Game() {
    const [history, setHistory] = useState([new Array(BOARDSQUARE).fill(null)]) // [ [LENGTH = 9], [], []... ]
    const [currentIndex, setCurrentIndex] = useState<number>(0)
    const isNextStepX = currentIndex % 2 === 0 // 下一步棋是否是 ‘X’
    // const currentSquares = history[currentindex]
    const currentSquares = useMemo(() => history[currentindex], [history, currentIndex]) 
    const handleJumpTo = useCallback((index: number) => {
      setHistory(index)
    }, [])
    const handleBoardClick = (squaresSlice: string[]) => { // 根据当前currentIndex，追加历史记录
      const historyCopy = history.slice(0, currentIndex + 1) // 确保当前history时间历史数组长度和当前时间线下标匹配，即history.length - 1 = currentIndex
      setHistory([...historyCopy, squaresSlice])
      setCurrentIndex(currentIndex + 1)
    }
    const TimeTravelList = history.map((item, index) => {
      return (
        <TimeTravelListItem 
          key={index} 
          index={index} 
          jumpTo={handleJumpTo} 
        /> 
      )
    })
    return (
      <div className="game">
        <div className="game-board">
          <Board 
            xIsNext={isNextStepX} 
            squares={currentSquares} 
            currentIndex={currentIndex}
            onPlay={handleBoardClick} 
          />
        </div>
        <div className="game-info">
          { TimeTravelList }
        </div>
      </div>
    )
  }

```

`棋盘中的某个格子组件` 
```tsx
  import { ISquare } from '../interface';
  // 棋盘中的某个格组件 - props：1、当前展示值'X' or 'O'；2、
  const Square: React.FC<ISquare> = ({ value, onSquareClick }) => {
    return (
      <button onClick={onSquareClick} className='square'>{ value }</button>
    )
  }
  export default Square
```

`时间旅行列表元素组件` 
```tsx
  import { ITimeTravelListItem } from '../interface';
  // 时间旅行列表元素组件
  const TimeTravelListItem: React.FC<ITimeTravelListItem> = ({ index, jumpTo }) => {
    const description = index > 0 ? `Go to step${index}` : 'Go to game start'
    return (
      <li key={index}>
        <button onClick={() => jumpTo(index)}>{ description }</button>
      </li>
    )
  }
  export default TimeTravelListItem
```

`棋盘组件` 
```tsx
  import { IBoard } from '../interface';
  import Square from './Square';
  import { BOARDSQUARE } from '../enums';
  // 棋盘组件
  const Board: React.FC<IBoard> = ({ xIsNext, squares, onPlay, currentIndex }) => {
    const winner = calculateWinner(squares)
    const status = winner
      ? `Winner is ${winner}`
      : (currentIndex === BOARDSQUARE ? 'Ended In A Draw' : `Next Player Is ${xIsNext ? 'X' : 'O'}`)
    const handleClick = (index: number) => {
      if (squares[index] || calculateWinner(squares)) { // 若已经出现胜者，或者当前格已有棋子，则return
        return
      }
      const squareSlice = squares.slice()
      squaresSlice[index] = xIsNext ? 'X' : 'O'
      onPlay(squaresSlice)
    }  
    function caculateWinner (squares: string[]) {
      const arr = squares.slice()
      const winnerResultArr = [ // 胜利者下标数组
        [0, 1, 2],
        [3, 4, 5],
        [6, 7, 8],
        [0, 3, 6],
        [1, 4, 7],
        [2, 5, 8],
        [0, 4, 8],
        [2, 4, 6],
      ]
      for (let i = 0; i < winnerResultArr.length; i++) {
        const [a, b, c] = winnerResultArr[i]
        if (arr[a] && arr[a] === arr[b] && arr[a] === arr[c]) {
          return arr[a]
        }
      }
      return null
    }
    return (
      <>
        <div className="status">{status}</div>
        <div className="board-row">
          <Square value={squares[0]} onSquareClick={() => handleClick(0)} />
          <Square value={squares[1]} onSquareClick={() => handleClick(1)} />
          <Square value={squares[2]} onSquareClick={() => handleClick(2)} />
        </div>
        <div className="board-row">
          <Square value={squares[3]} onSquareClick={() => handleClick(3)} />
          <Square value={squares[4]} onSquareClick={() => handleClick(4)} />
          <Square value={squares[5]} onSquareClick={() => handleClick(5)} />
        </div>
        <div className="board-row">
          <Square value={squares[6]} onSquareClick={() => handleClick(6)} />
          <Square value={squares[7]} onSquareClick={() => handleClick(7)} />
          <Square value={squares[8]} onSquareClick={() => handleClick(8)} />
        </div>
      </>
    )
  }
  export default Board
```

`sh脚本部署到github-pages`

```sh
  # deploy.sh
  rm -rf dist &&
  npm run build &&
  cd dist &&
  git init &&
  git add . &&
  git commit -m 'update' &&
  git branch -M gh-pages && # 打包dist文件推送至部署分支，从而在github-pages部署
  git remote add origin 《git仓库ssh地址》 &&
  git push -f -u origin gh-pages &&
  cd -
```


## 2、React Render & Commit 过程描述
可以想象一下，组件像`厨房里的厨师`，从食材中组装出美味的菜肴，在这种情况下，`React`就像是`服务员`，接受来自`顾客的请求`并将他们的`订单`交给`厨房`，UI请求和服务的过程有三个步骤：
```ts
  1、触发一个渲染：react将客人的订单（UI变化的请求）交给厨房（组件处理）
  2、渲染组件：react在厨房（组件处理）准备订单（UI变化）
  3、提交到DOM：react将订单（UI变化请求）放在桌子上（更新到Dom）
```

## 3、React批处理快照
实现一个`getFinalState`方法来模拟react状态的批量更新，`getFinalState`方法接收两个参数：
1、`baseState`, 即一个起始状态如 `const [baseState, setBaseState] = useState(0)`
2、存储状态批量更新的队列`queue`，如：`[1, 1, 1]` 或 `[1, n => n + 1]` 或 `[5, n => n+1, 42]`
```ts
  三个输入示例代码的期望输出是：
  (1) baseState = 0, queue = [1, 1, 1]：输出1，因为三次快照都是1，则直接将原数据baseState赋值为1
  (2) baseState = 0, queue = [1, n => n + 1]：输出2，队列第一项是1，则将原数据baseState赋值为1；第二项为函数，则输入当前状态执行函数获得2
  (3) baseState = 0, queue = [5, n => n + 1, 42]：输出42，队列第一项是1，则将原数据baseState赋值为1；第二项为函数，则输入当前状态执行函数获得2；队列第一项是42，则将原数据baseState赋值为42
```
下面实现`getFinalState(baseState, queue)`方法，来模拟react中的批量更新处理，该方法可获取最新的状态
```ts
  export function getFinalState(baseState, queue) {
    let finalState = baseState
    queue.forEach((update) => {
      if (typeof update === 'function') { // Apply the updater function.
        finalState = update(finalState)
      } else { // Replace the next state.
        finalState = update
      }
    })
    return finalState
  }
```

## 4、React更新Object中的状态
```tsx
  // 方法1，使用解构运算符更新
  import { useState } from 'react'
  export default function Form() {
    const [obj, setObj] = useState({
      name: 'Niki de Saint Phalle',
      artwork: {
        title: 'Blur Nana',
        city: 'Hamburg',
        image: 'https: //i.imgur.com/Sd1AgUOm.jpg',
      }
    })
    // 更新Object中的状态
    const handleTitleChange = (e) => {
      setObj({
        ...obj,
        artwork: {
          ...obj.artwork,
          name: e.target.value,
        }
      })
    }
    return (
      <label>Title:
        <input
          value={obj.artwork.title}
          onChange={handleTitleChange}
        />
      </label>
    )
  }
  // 方法2，使用《useImmer》库快速修改
  import { useImmer } from 'react'
  export default function Form() {
    const [obj, setObj] = useImmer({
      name: 'Niki de Saint Phalle',
      artwork: {
        title: 'Blue Nana',
        city: 'Hamburg',
        image: 'https://i.imgur.com/Sd1AgUOm.jpg',
      }
    })
    // 更新Object中的状态
    const handleTitleChange = (e) => {
      setObj((draft) => {
        draft.artwork.title = e.target.value // draft指向当前obj状态的引用，可以直接修改
      })
    }
    return (
      <label>Title:
        <input
          value={obj.artwork.title}
          onChange={handleTitleChange}
        />
      </label>
    )
  }
```

## 4、React更新Array中的状态
```tsx
  // 方法1，使用解构运算符更新
  import { useState } from 'react'
  let nextId = 3
  const initialList = [
    { id: 0, title: 'title1', seen: false },
    { id: 1, title: 'title2', seen: false },
    { id: 2, title: 'title3', seen: true },
  ]
  export default function List() {
    const [list, setList] = useState(initialList)
    const handleToggle = (artId, status) => {
      setList(list.map((item) => {
        if (item.id === artId) {
          return { ...item, seen: status }
        } else {
          return item
        }
      }))
    }
    return (
      <>
        <h1>List Title</h1>
        <ItemList 
          artworks={list}
          onToggle={handleToggle}
        />
      </>
    )
  }
  function ItemList({ artworks, onToggle }) => {
    return (
      <ul>
        {artworks.map(artwork => (
          <li key={artwork.id}>
            <input
              type="checkbox"
              checked={artwork.seen}
              onChange={e => {
                onToggle(
                  artwork.id,
                  e.target.checked
                )
              }}
            />
          </li>
        ))}
      </ul>
    )
  }

  // 方法2，使用《useImmer》库快速修改
  import { useImmer } from 'react'
  let nextId = 3
  const initialList = [
    { id: 0, title: 'title1', seen: false },
    { id: 1, title: 'title2', seen: false },
    { id: 2, title: 'title3', seen: true },
  ]
  export default function List() {
    const [list, setList] = useState(initialList)
    const handleToggle = (artId, status) => {
      setList((draft) => {
        const findItem = draft.find((item) => item.id === artId)
        findItem.seen = status
      })
    }
    return (
      <>
        <h1>List Title</h1>
        <ItemList 
          artworks={list}
          onToggle={handleToggle}
        />
      </>
    )
  }
  function ItemList({ artworks, onToggle }) => {
    return (
      <ul>
        {artworks.map(artwork => (
          <li key={artwork.id}>
            <input
              type="checkbox"
              checked={artwork.seen}
              onChange={e => {
                onToggle(
                  artwork.id,
                  e.target.checked
                )
              }}
            />
          </li>
        ))}
      </ul>
    )
  }
```