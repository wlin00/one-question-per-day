## 跨路由动画
> 本文记录如何用Vue3实现跨路由的组件动画

### 首先我们知道Vue的组件在路由切换时会卸载，若想要保存组件状态，我们可以：
```
1、全局存储组件状态：但这样切换路由时，还是会进行重新渲染，不够优雅；
2、使用keep-alive来持久化组件，但这样不同的路由可能不方便去控制我们的动画组件；
3、市面上的一种跨路由动效解决方案：FLIP，它是以关键帧的方式，即 first、Last、Invert来区分实现不同路由的动画；但它的一个缺陷就是，开始和结束帧的组件不是一个组件，它们不具备相同的状态；
```

综上，我们希望用一个新的思路来实现一个跨路由的动画组件，它在不同路由下具备相同的状态，首先我们实现一个全局唯一的单例组件：

![image](https://s1.ax1x.com/2022/08/29/vWxDpt.png)

> 可见，假设我们的 `App` 下有两个路由，分别是 `/index` 和 `/foo`, 若我们想有一个跨路由组件，能从不同路由下的 `Proxy` 容器中获取到不同的状态信息，然后这个状态可以同步给 `App` 下一个 `游离状态` 的动画组件，这样就能实现效果。

实现细节点：
```typescript
  1、不同路由下，我们注册 Proxy 组件来收集对应路由的信息，即收集当前路由信息，到一个全局的响应式数据：
    import { metaData, proxyEl } from '../composables/data'
    setup(props, context) {
      // 收集当前路由下，proxy容器的props和attrs
      metaData.props = props
      metaData.attrs = context.attrs
      // 注册当前路由下，proxy容器的位置信息
      const proxyRef = ref(null)
      onMounted(() => {
        proxyEl.value = proxyRef.value
      })
    }

  2、收集完 Proxy 容器的位置，创建图片的容器组件 ImageContainer 来动态计算 Proxy 容器的位置信息并赋予自身，并且将全局的状态同步给内部的图片组件
    1) 位置信息同步：
      // 使用 useElementBounding 结合reative 来动态获取当前 Proxy 容器的位置
      import { metaData, proxyEl } from '../composables/metaData'
      import { useElementBounding } from '@vueuse/core'
      setup () {
        const rect = reactive(useElementBounding(proxyEl))
        // 计算一个computeStyle赋予给自身，改变位置和宽高等信息
        const computeStyle = compute(() => {
          return {
            left: `${rect?.left ?? 0}px`,
            top: `${rect?.top ?? 0}px`,
            width: `${rect?.width ?? 0}px`,
            height: `${rect?.height ?? 0}px`,
            transition: `all .3s linear`
          }
        })
        return {
          metaData,
          computeStyle
        }
      }

    2) 将 Proxy 容器的attrs同步给容器内部图片：
      // template
      <div :style="computeStyle" class="ImageContainer">
        <slot v-bind="metaData.attrs" />
      </div>

      // 在App.vue 里，使用作用域插槽给内部图片组件传递状态，需手动绑定
      <template>
        <main class="main">
          <router-view></router-view>
        </main>
        <ImageContainer v-slot="props">
          <!-- 作用域插槽：将全局状态同步给图片 -->
          <TheImage v-bind="props" /> 
        </ImageContainer>
      </template>

  3、创建图片组件 TheImage，来进行图片展示

  4、不同路由里，我们除了给 Proxy 容器传递不同的样式信息，还可以对当前容器的 size 做持久化处理。实现方法是：useStorage结合ref 来创建一个持久化数据：
    // /foo 路由对应组件里:
    import { ref, computed } from 'vue'
    import { useStorage } from '@vueuse/core'
    setup() {
      const size = ref(useStorage('size', 300))
      const computeStyle = computed(() => {
        return {
          width: `${size.value ?? 0}px`,
          height: `${size.value ?? 0}px`,
          borderRadius: `50%`,
        }
      })
    }
  补充：vue-use中的，useStorage的实现原理是：（将本地存储变成响应式数据）
    1、首先本地存储如localStorage、sessionStorage 它们有get和set的api，可以用于更新或者是获取某个值；
    2、如果要把本地存储绑定在一个响应式的数据上，那在初始化的时候，我们需要先拿到本地存储的get的值，去把它放到ref的第一个参数去做一个初始化；
    3、如果这个ref改变的时候，我们需要用一个watch 去侦听这个ref，然后调用本地存储的set方法，来更新本地的数据为当前改变后的响应式数据
    
        
```
