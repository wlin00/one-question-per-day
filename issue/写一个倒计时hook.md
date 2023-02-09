**题目：**
 写一个useCountDown的倒计时hook，要求能够让用户手动开关倒计时

> `Vue`版本
1、代码
 ```typescript
  <button :disabled="pending" @click="handleClick">{{ display }}</button>
  import useCountDown from '@/utils/useCountDown'
  export default {
    setup (props, context) {
      const COUNT_NUM = 5
      const { count, pending, startCountDown } = useCountDown(COUNT_NUM, true)
      const handleClick = () => {
        startCountDown()
      }
      const display = computed(() => {
        return count.value === COUNT_NUM ? '倒计时按钮' : count.value
      })
      return {
        pending,
        handleClick,
        display,
      }
    }
  }
 ```

 2、实现自定义hook - useCountDown.ts
 ```typescript
  // useCountDown.ts
  import { ref, onMounted, onUnmounted } from 'vue'
  function useCountDown (seconds: number = 60, isManual: boolean = true) {
    const num = ref<Number>(seconds)
    const timer = ref(null)
    const pending = ref<Boolean>(false)
    function startCountDown () {
      pending.value = true
      timer.value = setTimeout(() => {
        cleatTimeout(timer.value)
        if (num.value > 1) {
          num.value -= 1
          startCountDown()
        } else {
          num.value = seconds
          pending.value = false
        }
      }, 1000)
    }
    onMounted(() => {
      !isManual && startCountDown()
    })
    onUnmounted(() => {
      timer.value = null
      clearTimeout(timer.value)
    })
  }
  export default useCountDown
 ```

> `React`版本
1、代码
```typescript
  import { useState, useEffect } from 'react'
  export const useCountDown = (seconds: number = 60) => {
    const [count, setCount] = useState<number>(seconds)
    const [pending, setPending] = useState<boolean>(false)

    const startCountDown = () => {
      setPending(true)
    }
    const stopCountDown = () => {
      setPending(false)
    }
    const resetCountDown = () => {
      setPending(false)
      setCount(seconds)
    }

    useEffect(() => {
      let timerId: NodeJS.Timeout | null = null
      if (!pending) {
        return
      }
      if (count < 1) {
        resetCountDown()
        return
      }
      timerId = setTimeout(() => {
        setCount(count - 1)
      }, 1000)
      return () => clearTimeout(timerId!)
    }, [count, pending])

    return {
      count,
      pending,
      startCountDown,
      stopCountDown,
      resetCountDown,
    }
  }
```
