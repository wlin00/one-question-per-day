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