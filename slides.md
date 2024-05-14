---
# try also 'default'  start simple
theme: academic

background: https://images.unsplash.com/photo-1455644571951-5733aeae3a81?q=80&w=1470&auto=format&fit=crop&ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D
title: Vuex -> Pinia
info: |
  ## Vuex -> Pinia
  Feature differences, Migration Guide, Quirks

class: text-center
highlighter: shiki
drawings:
  persist: false
transition: slide-left
mdc: true
fonts:
  sans: Inter
canvasWidth: 1050

---

# Vuex -> Pinia

Feature differences, Migration Guide, Quirks

---

# Why migrate to Pinia?
Vuex is still available in Vue3, but it is in maintenance mode, hence it is preferable to use Pinia, the updated replacement.

According to [pinia.vuejs.org](https://pinia.vuejs.org/introduction.html#Comparison-with-Vuex-3-x-4-x)
> Compared to Vuex, Pinia provides a simpler API with less ceremony, offers Composition-API-style APIs, and most importantly, has solid type inference support when used with TypeScript.

---

# Main differences with Vuex

- Mutations no longer exist. They were often perceived as extremely verbose. They initially brought devtools integration but that is no longer an issue.

- No need to create custom complex wrappers to support TypeScript, everything is typed and the API is designed in a way to leverage TS type inference as much as possible.

- No more magic strings to inject, import the functions, call them, enjoy autocompletion!

- No need to dynamically add stores, they are all dynamic by default and you won't even notice. Note you can still manually use a store to register it whenever you want but because it is automatic you don't need to worry about it.

- No more nested structuring of modules. You can still nest stores implicitly by importing and using a store inside another but Pinia offers a flat structuring by design while still enabling ways of cross composition among stores. You can even have circular dependencies of stores.

- No namespaced modules. Given the flat architecture of stores, "namespacing" stores is inherent to how they are defined and you could say all stores are namespaced.

---

# Modules become... normal stores

There are no more modules in Pinia, so generally the best idea is to create separate stores for every module, IE:

```bash
# Vuex example (assuming namespaced modules)
src
└── store
    ├── index.js           # Initializes Vuex, imports modules
    └── modules
        ├── module1.js     # 'module1' namespace
        └── nested
            ├── index.js   # 'nested' namespace, imports module2 & module3
            ├── module2.js # 'nested/module2' namespace

# Pinia equivalent, note ids match previous namespaces
src
└── stores
    ├── index.js          # (Optional) Initializes Pinia, does not import stores
    ├── module1.js        # 'module1' id
    ├── nested-module2.js # 'nestedModule2' id
    └── nested.js         # 'nested' id
```

Each store requires an `id`, similar to a namespace in Vuex. Btw, a suggestion is renaming the `store` directory to `stores`, to emphasize the fact that there are multiple stores and not modules

---

# Creating a store

Everything is defined in the `defineStore` function:

```js
export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0, name: 'Thomas' }),
  getters: {
    doubleCount: (state) => state.count * 2,
  },
  actions: {
    increment() {
      this.count++
    },
  },
})
```

- Every store has to have an `id`
- The state is a function
- Mutations are inlined or become actions

---

# Updating the state

<div style="display: flex; flex-direction: row; gap: 1rem; width: 100%;">

<div>
<p>
Vuex:
<div style="height: 12px"/>
```js
//outside the store
const state = {
  firstName: '',
  lastName: '',
  userId: null,
}
```
</p>

<p>
Pinia:

```js
//inside the store
state: () => ({
  firstName: '',
  lastName: '',
  userId: '' 
}),
```
</p>
</div>

<div>

Subscribing to the state:
```js
cartStore.$subscribe((mutation, state) => {
  // import { MutationType } from 'pinia'
  // 'direct' | 'patch object' | 'patch function'
  mutation.type 
  // same as cartStore.$id
  mutation.storeId // 'cart'
  // only available with 
  // mutation.type === 'patch object'

  // patch object passed to cartStore.$patch()
  mutation.payload 
})
```

<br>

> Tip: you can use `{ detached: true }` as a second parameter to prevent the subscription being removed when the component is unmounted
</div>

<div>

Mutating the state:

```js
store.name = 'John'

//patch object
store.$patch({
  count: store.count + 1,
  age: 34,
  name: 'John',
})

//patch function
store.$patch((state) => {
  state.items.push({ name: 'shoes', quantity: 1 })
  state.hasChanged = true
})
```
</div>


</div>


---

# Updating getters

1. First of all you can remove any getters that return state under the same name (eg. `firstName: (state) => state.firstName`), these are not necessary as you can access any state directly from the store instance.

2. If you need to access other getters, they are on this instead of using the second argument

3. If using rootState or rootGetters arguments, replace them by importing the other store directly

But for the most part they stay the same :)

> Be careful, when not passing `state` to a getter and accessing information directly through `this`, the return type must be explicity set with typescript or JSDoc

---

# Updating actions

1. Remove the first `context` argument from each action. Everything should be accessible from `this` instead

2. If using other stores either import them directly or access them on Vuex, the same as for getters

Vuex:

```js
    async loadUser ({ state, commit }, id: number) {
      if (state.userId !== null) throw new Error('Already logged in')
      const res = await api.user.load(id)
      commit('updateUser', res)
    }
```

Pinia:

```js
    async loadUser (id: number) {
      if (this.userId !== null) throw new Error('Already logged in')
      const res = await api.user.load(id)
      this.updateUser(res)
    },
```

---

# Subscribing to actions
```js
const subscribe = someStore.$onAction(
  ({
    name, // name of the action
    store, // store instance, same as `someStore`
    args, // array of parameters passed to the action
    after, // hook after the action returns or resolves
    onError, // hook if the action throws or rejects
  }) => {}
)
```

> You can always detach the subscription from the component by passing `true` as the second parameter

---

# What to do with the leftover mutations?

1. Just inline them, eg. `userStore.firstName = 'First'`

2. If converting to actions, remove the first `state` argument and replace any assignments with `this` instead

> A common issue I encountered is that oftentimes you would have mutations and actions and mutations with the same names, so when converting, be careful to give them a unique name 

---

# Accessing stores
```js
import { useCounterStore } from '@/stores/counter'

const store = useCounterStore()
```

The state, getters, and actions, are all accessed in the same way:

```js
store.someInformation

store.someGetter

store.someAction()
```

---

# Mapping stores

When using multiple stores in a component or a mixin the best way is to map them:

```js
import { mapStores } from 'pinia'

const useUserStore = defineStore('user', {
  // ...
})

export default {
  computed: {
    ...mapStores(useUserStore, useSomeOtherStore)
  },

  methods: {
    method() {
      this.userStore.authenticate()
    }
  },
}
```
<br>

> When mapping, the stores are accessible as its `id` + *Store*

---

# Mapping state (and getters)

Vuex has `mapState` and `mapGetters`, Pinia also has them both, but `mapGetters` is just a copy of `mapState`, and it is advised to always use `mapState`.

```js
  ...mapState(useCounterStore, {
    n: 'count',
    triple: store => store.n * 3,
    // note we can't use an arrow function if we want to use `this`
    custom(store) {
      return this.someComponentValue + store.n
    },
    doubleN: 'double'
  })
```

If you want the name of the mapped state to be the same you can use a shorter version

```js
  ...mapState(useCounterStore, ['count', 'double'])
```

<br>

> A variation is `mapWritableState`, which also creates computed setters for the state elements. Naturally getters cannot be added to this.
