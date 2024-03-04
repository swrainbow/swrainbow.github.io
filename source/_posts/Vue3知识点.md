---
title: Vue3知识点
date: 2023-11-06 23:12:27
tags: [vue]
categories: [vue]
---
## 根组件全局错误捕捉
```
app.config.errorHandler = (err) => {

}
```

## 全局注册组件
app.component（‘Childcp’， childCp）;

## 多个应用实例
```
const app1 = createApp({
  /* ... */
})
app1.mount('#container-1')

const app2 = createApp({
  /* ... */
})
app2.mount('#container-2')
```

## 调用函数
绑定在表达式中的方法在组件每次更新都会被重新调用，因此不应该产生副作用，比如改变数据或者触发异步操作

## 深层响应式
```
import { ref } from 'vue'

const obj = ref({
  nested: { count: 0 },
  arr: ['foo', 'bar']
})

function mutateDeeply() {
  // 以下都会按照期望工作
  obj.value.nested.count++
  obj.value.arr.push('baz')
}
```
vue2和vue3 vue2数组这种响应式麻烦

## 给代理对象添加属性，这个对象的值依然是代理对象

## reactive的局限性
1. 基本类型
2. 替换对象

## 计算属性缓存 vs 方法

## watch
reactive 对象隐适的创建深层监听
watch(()=> person.name, (newVal, oldVal) => {console.log("newval")})
监视的是对象里的属性， 最好写成函数式 若对象监视的是地址值，需要关注对象内部，需要手动开启深度

## defineExpose

## vue2的生命周期
创建
挂载
更新
销毁 v-show不会触发销毁

## vue3生命周期
创建阶段 setup

## vueRouter

history模式后端配置
```
location / {
    root /;
    index index.html;
    try_files $uri $uri /index.html
}
```

> 传递params参数时 若使用to的对象写法， 必须使用name配置项 不能使用path
> 传递params参数时， 需要提前在规则中占位

1. 第一种写法： 将路由的params参数作为props传给路由组件  props: true;
2. 第二种写法： 函数写法 可以自己决定将什么作为props给路由组件 props（route）{ return route.query }
3. 第三种写法： 对象写法， 可以自己决定将什么作为props给路由组件 props:{ }

## pinia
符合直觉
直接修改
1. store.sum += 1;
2. store.patch({ sum:88, school: 'iii'}) 批量修改
3. store.方法  actions里面实现方法

toRefs 会把store中所有的方法都变成响应式
storeToRefs() 修改store中保存的数据

## vue组件通信
 1. props
 2. custom event defineEmits()
 3. mitt
 4. v-model  :modelValue="username" @update:modelValue="username = $event"
 子组件
 ```js
@input="emit('update:modelValue', $event.targe.value)"

defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
 ```
可以写多个v-model
5. $attrs
6. $refs - $parent
7. provide-inject
8. pinia类似的
9. 插槽 slot
    > 默认插槽 具名插槽 作用域插槽 v-slot: name(#name) <slot name="name"> </slot>
<slot :youxi="games" x="test1"></slot> 数组在子组件那边，但根据数据生成的结构 由父亲决定
```js
    <template v-slot="params">
        <ul>
            <li v-for="y in params.youxi" :ket = "y.id">
Í
            </li>
        </ul>
    </template>
```

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240226084449.png)

## shallowRef && shallowReactive

## readOnly && shadowReadOnly

## toRaw && markRaw
在需要将响应式对象传递给非Vue的库或外部系统， 使用toRaw可以确保他们收到的是普通对象
markRow ==== v_skip = true

## customRef 自定义Ref
```js
export default function(initValue: string, delay: number) {
    let timer: number;

    let msg = customRef((track, trigger) => {
        return {
            get() {
                track();
                return initValue;
            },
            set(value) {
                clearTimeout(timer);
                timer = setTimeout(() => {
                    initValue = value;
                    trigger()
                }, delay)
            }
        }
    })
}

```

## Teleport
position: fixed 被影响的组件可以使用Teleport

## Suspense
子组件在setup中有异步任务
```js
<Suspense>
    <template v-slot:default>
        <Child/>
     </template> 
    <template v-slot:fallback>
        <h2>加载中 </h2>
    </template>

</Suspense>
```

## 全局API转移到应用对象
- app.component
- app.config
- app.directive
- app.mount
- app.unmount
- app.use