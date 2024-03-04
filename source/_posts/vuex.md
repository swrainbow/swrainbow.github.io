---
title: Vuex & VueRouter
date: 2022-07-06 23:02:21
tags: [vue]
categories: [vue]
---
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