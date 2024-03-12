---
title: Vuex
date: 2022-07-06 23:18:21
tags: [vue]
categories: [vue]
---
# vuex 核心概念
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240305084325.png)
- Store
- State
- Getter
- Mutation 状态的变换提交mutation
- Action 异步
- module

store/index.js文件
```js
state: {
    count: 0,
    msg: 'hello Vuex'
},
getters: {
    reverseMsg(state) {
        return state.msg.split('').reverse().join('')
    }
},
mutations: {
    increate(state, payload) {
        state.count += paylod
    }
},
actions: {
    increateAsync(context, payload) {
        setTimeout(()=> {
            context.commit('increate', payload)
        }, 2000)
    }
},
modules: {

}
```
视图使用
```js
computed:
...mapState({num:'count', message:'msg'}),
...mapGetters(['reversemsg'])


methods: {
    ...mapMutations(['increate']),  $store.commit('increate')
    ...mapActions(['increateAsync']) ---> $store.dispatch('increateAsync', 5)
}
```

module 使用
store/modules/cart.js
```js
....
export default {
    namespaced: true, // 开启命名空间
    state,
    getters,
    mutations,
    actions
}
```
在store/index.js 中注册
```js
....
modules: {
    cart
}

```
模版中使用
```js
computed:
    ...mapState('products', ['products'])
methods: {
    ...mapMutations('products', ['setProducts'])
}
```
## 模拟Vuex
模拟结构
```js
let _Vue = null;
class Store {
    constructor(options) {
        const {
            state = {},
            getters = {},
            mutations = {},
            actions = {}
        } = options
    }
    this.state = _Vue.observable(state); // reactive
    this.getters = Object.create(null);
    Object.keys(getters).forEach(key => {
        Object.defineProperty(this.getters, key, {
            get: () => getters[key](state)
        })
    })

    this._mutations = mutations;
    this._actions = actions

    commit (type, payload) {
        this._mutations[type](this.state, payload);
    }

    dispatch(type, payload) {
        this._actions[type](this, payload)
    }
};

function install (Vue) {
    _Vue = Vue;
    _Vue.mixin({
        beforeCreate() { // 混入 当创建根实例的时候就会进行混入
            if(this.$options.store) {
                _Vue.prototype.$store = this.$options.store
            }
        }
    })
}

export default {
    Store,
    install
}
```

## vuex 自定义插件
```js
const myPlugin = store => {
    store.subscribe((mutation, state) => {
        if(mutation.type.startsWidth('cart/')) {
            // 自定义逻辑
            window.localStorage.setItem('', xxxx)
        }
    })
}
```