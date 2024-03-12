---
title: pina-原理
date: 2024-03-05 09:18:22
tags:
---
Pinia符合直觉的 Vue.js 状态管理库 这是官方文档的说法。 注意符合直觉！！！

# 源码常见api
先来个开胃菜 Effect Scope api
## 什么是EffectScope
RFC关于EffectScopeApi的解释
> 在Vue的setup中，响应会在开始初始化的时候被收集，在实例被卸载的时候，响应就会自动的被取消追踪了，这时一个很方便的特性。但是，当我们在组件外使用或者编写一个独立的包时，这会变得非常麻烦。当在单独的文件中，我们该如何停止computed & watch的响应式依赖呢？

实际上EffectScope按我的理解就是副作用生效的作用域。
vue3对响应式的监听是通过effect实现的，当我们的组件销毁的时候vue会自动取消该组件的effect。
那么如果我们想要自己控制effect生效与否呢？ 比如我只想在莫种特定情况下才监听摸个ref，其他情况下不想监听该怎么做？

```js
//（vue-RFC示例代码）
const disposables = []

const counter = ref(0)
const doubled = computed(() => counter.value * 2)

disposables.push(() => stop(doubled.effect))

const stopWatch1 = watchEffect(() => {
  console.log(`counter: ${counter.value}`)
})

disposables.push(stopWatch1)

const stopWatch2 = watch(doubled, () => {
  console.log(doubled.value)
})

disposables.push(stopWatch2)

```

EffectScope如何实现
```js
// effect, computed, watch, watchEffect created inside the scope will be collected

const scope = effectScope()

scope.run(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(doubled.value))

  watchEffect(() => console.log('Count: ', doubled.value))
})

// to dispose all effects in the scope
scope.stop()

```

## markRaw
> 标记一个对象，使其永远不会转换为 proxy。返回对象本身。
```js
const foo = markRaw({})
console.log(isReactive(reactive(foo))) // false
​
// 嵌套在其他响应式对象中时也可以使用
const bar = reactive({ foo })
console.log(isReactive(bar.foo)) // false

```

## toRaw
> toRaw可以获取一个响应式对象的原始属性
```js
const foo = {};
const reactiveFoo = reactive(foo);
console.log("toRaw", toRaw(reactiveFoo) === foo); // true
​
const foo1 = {};
const refFoo1 = ref(foo1);
console.log("toRaw", toRaw(refFoo1.value) === foo1); // true

```

## toRefs 
将一个响应式对象转换为一个普通对象，这个普通对象的每个属性都是指向源对象相应属性的 ref。每个单独的 ref 都是使用 toRef() 创建的。
当从组合式函数中返回响应式对象时，toRefs 相当有用。使用它，消费者组件可以解构/展开返回的对象而不会失去响应性：
```js
function useFeatureX() {
  const state = reactive({
    foo: 1,
    bar: 2
  })

  // ...基于状态的操作逻辑

  // 在返回时都转为 ref
  return toRefs(state)
}

// 可以解构而不会失去响应性
const { foo, bar } = useFeatureX()
```

# 正文

使用pina首先需要注册到vue中
```js
const pinia = createPinia();
app.use(pinia);
```
> createPinia函数

我们就可以看到通过effectScope声明了一个ref，并赋值给了state，这里的effectScope是高级API 可以将其简单理解为声明了一个ref并赋值给state。
```js
export function createPinia(): Pinia {
    const scope = effectScope(true);
    const state = scope.run<Ref<Record<string, StateTree>>>(() =>
       ref<Record<string, StateTree>>({})
    )!;
    // 简化理解
    // const state = ref({})
    
    // ...
}
```

```js
export function createPinia(): Pinia {
  // ...
  let _p: Pinia["_p"] = []; // 所有需要安装的插件
  let toBeInstalled: PiniaPlugin[] = []; // install之前保存的待安装插件
​
  // 使用markRaw标记pinia使其不会被响应式
  const pinia: Pinia = markRaw({
    // vue.use实际执行逻辑
    install(app: App) {
      setActivePinia(pinia); // 设置当前使用的 pinia
      if (!isVue2) { // 如果是vue2，全局注册已经在PiniaVuePlugin完成，所以这段逻辑将跳过
        pinia._a = app; // 保存app实例
        app.provide(piniaSymbol, pinia); // 通过provide传递pinia实例，提供给后续使用
        app.config.globalProperties.$pinia = pinia; // 设置全局属性 $pinia
        toBeInstalled.forEach((plugin) => _p.push(plugin)); // 处理未执行插件
        toBeInstalled = [];
      }
    },
    use(plugin) {
      if (!this._a && !isVue2) { // 如果use阶段为初始化完成则暂存toBeInstalled中
        toBeInstalled.push(plugin);
      } else {
        _p.push(plugin);
      }
      return this;
    },
    _p, // 所有pinia的插件
    _a: null, // app实例，在install的时候会被设置
    _e: scope, // pinia的作用域对象，每个store都是单独的scope
    _s: new Map<string, StoreGeneric>(),  // store缓存 key为pinia的id value为pinia的对外暴露数据
    state, // pinia所有state的合集 key为pinia的id value为store下的所有state（所有可访问变量）
  });
  return pinia;
}

```

## defineStore
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240305134321.png)
在defineStore声明中，我们需要传入三种的参数。

- id：定义store的唯一id，单独传参或者通过options.id进行传参
- options：具体配置信息包含如果传参是对象，则可以传，state，getters，action，id，例如上图1 2 种声明方式；如果传参是Function，则自主声明变量方法，例如上图第三种声明方式
- storeSetup：仅限第三种store的声明方式，传入函

## dfineStore执行逻辑
```js
export function defineStore(
  // TODO: add proper types from above
  idOrOptions: any,
  setup?: any,
  setupOptions?: any
): StoreDefinition {
  let id: string;
  let options: // ....
  
  // 对三种store创建形式进行兼容。
  const isSetupStore = typeof setup === "function";
  if (typeof idOrOptions === "string") {
    id = idOrOptions;
    options = isSetupStore ? setupOptions : setup;
  } else {
    options = idOrOptions;
    id = idOrOptions.id;
  }
  function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
      //.....下面单独分析
  }
  useStore.$id = id;
  // 将useStore执行结果返回，在该store在使用之前被返回函数不会执行。
  // 所以defineStore早于在Vue种注册pinia也不会出现错误。
  return useStore;
}

```
只有在store被执行的时候才会运行被返回的函数useStore，useStore才是核心store的创建逻辑

# useStore逻辑分析
## useStore之前的逻辑执行顺序
1. defineStore初始化
2. main.ts -> createPinia -> vue.use -> install（注册逻辑）
3. 执行useStore（页面逻辑)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240305135252.png)

```js

// useStore接受两个参数，一个是pinia的实例，另一个与热更新相关。
function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
		// 首先会通过getCurrentInstance获取当前组件实例，并处理参数pinia，组件实例可以被正常获取，接下来通过inject(piniaSymbol)获取pinia实例（在install阶段保存）。
    const currentInstance = getCurrentInstance();
    pinia =
        (__TEST__ && activePinia && activePinia._testing ? null : pinia) ||
        (currentInstance && inject(piniaSymbol));
    // 设置当前活跃的pinia，如果存在多个pinia实例，方便后续逻辑获取当前pinia实例
    if (pinia) setActivePinia(pinia);
    // 在dev环境并且全局变量activePinia获取不到当前pinia实例，则说明未全局注册，抛出错误
    if (__DEV__ && !activePinia) {
        throw new Error(
            `[🍍]: getActivePinia was called with no active Pinia. Did you forget to install pinia?\n` +
            `\tconst pinia = createPinia()\n` +
            `\tapp.use(pinia)\n` +
            `This will fail in production.`
        );
    }
    // 获取最新pinia，并断言pinia一定存在（猜测这里主要为了断言，此时两个变量就是一个值）
    pinia = activePinia!;
	// ....
}

```

## 核心store创建
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240305135555.png)
```js
function useStore(pinia?: Pinia | null, hot?: StoreGeneric): StoreGeneric {
    // .....
    // 如果是第一次使用创建store逻辑，后面则跳过
    if (!pinia._s.has(id)) {
        // 如果defineStore的时候第二个参数是函数则为true，否则为false
        if (isSetupStore) {
            createSetupStore(id, setup, options, pinia);
        } else {
            createOptionsStore(id, options as any, pinia);
        }
    }
    // 从_s中获取当前id对应的store信息
    const store: StoreGeneric = pinia._s.get(id)!;
	// 这里返回的值实际上就是我们实际获取到值
    return store as any;
}

```

## createOptionsStore
defineStore的第二个参数使用非Function进行声明将会走入该逻辑
```js
function createOptionsStore<
  Id extends string,
  S extends StateTree,
  G extends _GettersTree<S>,
  A extends _ActionsTree
>(
  id: Id, // storeid
  options: DefineStoreOptions<Id, S, G, A>, // state action getters
  pinia: Pinia, // 当前store实例
  hot?: boolean
): Store<Id, S, G, A> {
  const { state, actions, getters } = options;
  // 获取state中是否已经存在该store实例
  const initialState: StateTree | undefined = pinia.state.value[id];
  console.log("initialState", initialState);
  let store: Store<Id, S, G, A>;
  
  function setup() {
    if (!initialState && (!__DEV__ || !hot)) {
      if (isVue2) {
        set(pinia.state.value, id, state ? state() : {});
      } else {
        // 将数据存储到state中，因为state时通过ref进行创建
        pinia.state.value[id] = state ? state() : {};
      }
    }

    // 避免在 pinia.state.value 中创建状态
    console.log(11, pinia.state.value[id]);
    console.log(22, toRefs(pinia.state.value[id]));

    const localState =
      __DEV__ && hot
        ? // 使用 ref() 来解开状态 TODO 中的 refs：检查这是否仍然是必要的
          toRefs(ref(state ? state() : {}).value)
        : toRefs(pinia.state.value[id]);
    // 经过toRefs的处理后，localState.xx.value 就等同于给state中的xx赋值
    let aa = assign(
      localState, // state => Refs(state)
      actions, //
      Object.keys(getters || {}).reduce((computedGetters, name) => {
        if (__DEV__ && name in localState) {
          // 如果getters名称与state中的名称相同，则抛出错误
          console.warn(
            `[🍍]: A getter cannot have the same name as another state property. Rename one of them. Found with "${name}" in store "${id}".`
          );
        }
        // markRow 防止对象被重复代理
        computedGetters[name] = markRaw(
          // 使用计算属性处理getters的距离逻辑，并且通过call处理this指向问题
          computed(() => {
            setActivePinia(pinia);
            // 它是在之前创建的
            const store = pinia._s.get(id)!;

            // allow cross using stores
            /* istanbul ignore next */
            if (isVue2 && !store._r) return;

            // @ts-expect-error
            // return getters![name].call(context, context)
            // TODO: avoid reading the getter while assigning with a global variable
            // 将store的this指向getters中实现getters中this的正常使用
            return getters![name].call(store, store);
          })
        );

        return computedGetters;
      }, {} as Record<string, ComputedRef>)
    );
    console.log("aa", aa);
    return aa;
  }
  // 使用createSetupStore创建store
  store = createSetupStore(id, setup, options, pinia, hot, true);
  
  // 重写$store方法
  store.$reset = function $reset() {
    const newState = state ? state() : {};
    // 我们使用补丁将所有更改分组到一个订阅中
    this.$patch(($state) => {
      assign($state, newState);
    });
  };
  return store as any;
}

```

createOptionsStore函数在获取defineStore声明的数据后，在其内部构建了setup函数，该函数将option形式的state与getters分别转化为ref与computed，这样就与setup形式声明的store保持一致。

​这一块代码非常核心，初步解释了为何state具备响应式，为何getters具备computed的效果

​最后不论是option方式创建还是setup的形式创建，最后都统一通过createSetupStore完成对store最后的处理

## createSetupStore
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240305140922.png)
​ 经过createOptionsStore的处理，已经将option形式的字段全部转化为setup形式进行返回，现在无论何种创建方式，执行此处的setup函数，都会得到同一个结果。

接下来，就需要对其数据进行处理，获取到所有变量与方法，并对action通过wrapAction进行处理，便于实现后续的订阅发布模式 methods$Action
```js
const setupStore = pinia._e.run(() => {
    scope = effectScope();
    return scope.run(() => setup());
})!;

for (const key in setupStore) {
    const prop = setupStore[key];
    if ((isRef(prop) && !isComputed(prop)) || isReactive(prop)) {
    	// 如果当前props是ref并且不是计算属性与reative
        if (!isOptionsStore) {
			// option结构已经在createOptionsStore将其加入pinia
            if (isVue2) {
                set(pinia.state.value[$id], key, prop);
            } else {
                pinia.state.value[$id][key] = prop;
            }
        }
    } else if (typeof prop === "function") {
        // 如果当前函数是fun
        // wrapAction 会将当前prop也就是函数增加调用错误与正常的回调函数
        const actionValue = __DEV__ && hot ? prop : wrapAction(key, prop);
        if (isVue2) {
            set(setupStore, key, actionValue);
        } else {
            setupStore[key] = actionValue;
        }
        // 将其函数同步到optionsForPlugin中
        optionsForPlugin.actions[key] = prop;
    }
}

```
经过以上逻辑处理后，setupStore方式进行创建的store也会被添加到pinia.state中，而所有的function都会被wrapAction进行包装处理。
​	对state，action进行处理的同时，还需要对当前store可调用API进行处理，例如$reset，$patch

```js
const partialStore = {
    _p:pinia,
    $id,
    $Action,
    $patch,
    $reset,
    $subscribe,
    $dispose
}
const store: Store<Id, S, G, A> = reactive(
    assign(
        __DEV__ && IS_CLIENT
        ? // devtools custom properties
        {
            _customProperties: markRaw(new Set<string>()),
            _hmrPayload,
        }
        : {},
        partialStore
        // must be added later
        // setupStore
    )
) as unknown as Store<Id, S, G, A>;

// ...将变量 方法合并到store中
assign(toRaw(store), setupStore);

```