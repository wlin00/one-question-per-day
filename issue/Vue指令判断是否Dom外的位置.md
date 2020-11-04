**开发中遇到的小需求，需要监听Dom以外位置的点击事件**
> 全局使用

```javascript
    Vue.directive('clickOutside', {
      bind (el, binding, node) {
        el.event = function (event) {
        // 判断当前点击是否非Dom范围
          if (el !== event.target && !el.contains(event.target)) {
          node.context[binding.expression](event)
          }
        }
        document.body.addEventListener('click', el.event, true)
      },
      unbind (el) {
        document.body.removeEventListener('click', el.event, true)
      }
    })
```

> 组件范围内使用

```javascript
     directives: {
        clickOutside: {
          bind (el, binding, node) {
            el.event = function (event) {
              // 判断当前点击是否非Dom范围
              if (el !== event.target && !el.contains(event.target)) {
                node.context[binding.expression](event)
              }
            }
            document.body.addEventListener('click', el.event, true)
          },
          unbind (el) {
            document.body.removeEventListener('click', el.event,true)
          }
        }
      }
```
**使用**
```html
    <div v-click-outside="switchMode"></div>
 ```

[ReactHooks版本](https://github.com/always-on-the-road/one-question-per-day/issues/65)