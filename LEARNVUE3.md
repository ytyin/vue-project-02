# MarkDown 
[markdown使用方法](https://shd101wyy.github.io/markdown-preview-enhanced/#/zh-cn/markdown-basics?id=%e9%93%be%e6%8e%a5)
## Vue2和Vue3
[vue2](https://cn.vuejs.org/v2/guide/index.html)
[vue3](https://v3.cn.vuejs.org/guide/introduction.html#vue-js-%E6%98%AF%E4%BB%80%E4%B9%88)

### 组合式API
**基础**
- setup组件选项
`setup在组件创建之前执行。setup的调用发生在data property、computed property或methods被解析之前，所以在setup中应该避免使用this，因为它不会找到组件实例。setup选项是一个接收props和context的函数，后续讨论。此外，我们将setup返回的所有内容都暴露给组件的其余部分（计算属性、方法、生命周期钩子等等）以及组件的模板`
- 带ref的响应式变量
`在Vue 3.0中，我们可以通过一个新的ref函数使任何响应式变量在任何地方起作用，即ref函数可以是变量变为响应式变量，类似于在data中声明的那样。换句话说，ref为我们的值创建了一个响应式引用`
`将值封装在一个对象中，看似没有必要，但是为了保持JavaScript中不同数据类型的行为统一，这是必须的。这是因为在JavaScript中,Number或String等基本类型是通过值而非引用传递的。在任何值周围都有一个封装对象，这样我们就可以在整个应用中安全的传递它，而不必担心在某个地方失去它的响应性。`
- 在setup内部注册生命周期钩子
`Vue中导出的几个新函数是在setup中注册生命周期钩子的方法，它们使得组合式API的功能和选项式API一样完整。组合式API上的生命周期钩子与选项API的名称相同，但前缀为on:即 mounted 看起来会像 onMounted,这些函数接受一个回调，当钩子被组件调用时，该回调将被执行。`
- watch响应式更改
`设置一个监听器，对prop的变化做出反应。就像在组件中使用watch选项并在property上设置侦听器一样，我们可以使用从Vue导入的watch函数执行相同的操作，它接受3个参数：一个想要侦听的响应式引用或getter函数；一个回调；可选的配置选项`
- 独立的computed属性
`与ref和watch类似，也可以使用从Vue导入的computed函数在Vue组件外部创建计算属性`
```javascript
import { ref, computed } from 'vue'
const counter = ref(0)
const twiceTheCounter = computed(()=>counter.value * 2)
counter.value++
console.log(counter.value) // 1
console.log(twiceTheCounter.value) // 2
// 为了访问新建的计算变量的value,我们需要像ref一样使用.value property。
```
`这里我们给computed函数传递了第一个参数，它是一个类似于getter的回调函数，输出的是一个 只读的响应式引用。`

```javascript
// src/components/UserReposities.vue
// toRefs是为了确保我们的侦听器能根据prop的变化做出反应。
import { fetchUserRepositories } from '@/api/repositories'
import { ref, onMounted, watch, toRefs, computed } from 'vue'

setup(props){
    const { user } = toRefs(props) // 使用toRefs创建对props中的user property的响应式引用
    const repositories = ref([])
    const getUserRepositories = async ()=>{
        repositories.value = await fetchUserRepositories(user.value)  // 更新 prop.user 到 user.value 访问引用值
    }

    onMounted(user, getUserRepositories) 
    
    watch(user, getUserRepositories)//在user prop的响应式引用上设置一个侦听器

    const serchQuery = ref('')
    const repositoriesMatchingSearchQuery = computed(()=>{
        return repositories.value.filter(repository => repository.name.includes(searchQuery.value))
    })

    return {
        repositories,
        getUserRepositories,
        searchQuery,
        repositoriesMatchingSearchQuery
    }
}

// 上述代码可以提取到一个独立的组合式函数中

// src/composables/composables.vue
import { fetchUserRepositories } from '@/api/repositories'
import { ref, onMounted, watch, computed } from 'vue'

export const useUserRepositories = (user)=>{
    const repositories = ref([])
    const getUserRepositories = async ()=>{
        repositories.value = await fetchUserRepositories(user.value)
    }
    onMounted(user, getUserRepositories) 
    watch(user, getUserRepositories)
    return {
        repositories,
        getUserRepositories
    }
}

export const useRepositoryNameSearch = (repositories)=>{
    const serchQuery = ref('')
    const repositoriesMatchingSearchQuery = computed(()=>{
        return repositories.value.filter(repository => repository.name.includes(searchQuery.value))
    })
    return {
        searchQuery,
        repositoriesMatchingSearchQuery
    }
}

export const useRepositoryFilters = (repositoriesMatchingSearchQuery)=>{
    const filters = ref('')
    const updateFilters = (repositoriesMatchingSearchQuery) => {...}
    const filteredRepositories = computed(()=>{
        return {...}
    })
    return {
        filters,
        updateFilters,
        filteredRepositories
    }
}


// 现在我们有了连个单独的功能模块，接下来就可以在组件中使用它们了。

// src/components/UserReposities.vue
import { useUserRepositories, useRepositoryNameSearch } from '@/composables/composable.vue'
import { toRefs } from 'vue'
export default {
    components: { RepositoriesFilters, RepositoriesSortBy, RepositoriesList},
    props: {
        user: {
            type: String,
            require: true
        }
    },
    setup(props){
        const { user } = toRefs(props)
        const { repositories, getUserRepositories } = useUserRepositories
        const { searchQuery, repositoriesMatchingSearchQuery } = useRepositoryNameSearch
        
        return {
            repositories: repositoriesMatchingSearchQuery, // 因为我们并不关心未经过滤的仓库，我们可以在`repositories` 名称下暴露过滤后的结果
            getUserRepositories,
            searchQuery
        }
    },
    data(){
        return {
            filters: {...},
        }
    },
    computed: {
        filteredRepositories(){...},
    },
    methods:{
        updateFilters(){...}
    }
}

// 再抽一层，就会变成
// src/components/UserReposities.vue
import { useUserRepositories, useRepositoryNameSearch, useRepositoryFilters } from '@/composables/composable.vue'
import { toRefs } from 'vue'
export default {
    components: { RepositoriesFilters, RepositoriesSortBy, RepositoriesList},
    props: {
        user: {
            type: String,
            require: true
        }
    },
    setup(props){
        const { user } = toRefs(props)
        const { repositories, getUserRepositories } = useUserRepositories
        const { searchQuery, repositoriesMatchingSearchQuery } = useRepositoryNameSearch
        const { filters, updateFilters, filteredRepositories } = useRepositoryFilters(repositoriesMatchingSearchQuery)
        
        return {
            repositories: filteredRepositories, // 因为我们并不关心未经过滤的仓库，我们可以在`repositories` 名称下暴露过滤后的结果
            getUserRepositories,
            searchQuery,
            filters,
            updateFilters
        }
    }
}

```

**setup**
- 参数：
  1. props
  2. ccontext 


**生命周期钩子**
`setup()内部调用生命周期钩子`

| 选项式API | 组合式API(Hook inside `setup`) |
| --- | ---|
| beforeCreate | Not needed* |
| created | Not needed* |
| beforeMount | onBeforeMount |
| mounted | onMounted |
| mounted | onMounted |
| beforeUpdate | onBeforeUpdate |
| updated | onUpdated |
| beforeUnmount | onBeforeUnmount |     beforeDestroy ???
| unmounted | onUnmounted |             destoryed ???
| errorCaptured | onErrorCaptured |
| renderTracked | onRenderTracked |
| activated | onActived |
| deactivated | onDeactivated |


**Provide/Inject**
`我们也可以在组合式API中使用provide/inject。两者都只能在当前活动实例的setup()期间调用`
- 在setup()中使用provide
`先从vue中显示的导入provide方法，provide函数允许通过两个参数定义property:`
    1. name(`<String>`类型)
    2. value
- 在setup()中使用inject
`先从vue中显示的导入inject方法，inject函数有两个参数:`
    1. 要inject的property的name
    2. 默认值（可选）
- 响应性 
`为了增加provide值活动inject值之间的响应性，我们可以在provide值时使用ref或reactive`
- 修改响应式property
`当使用响应式provide/inject值时，建议尽可能将对响应式property的所有修改限制在定义provide的组件内部。如果要确保通过provide传递的数据不会被inject的组件更改，我们建议对提供者的property使用readonly。`

**模板引用**
`在使用组合式API时，响应式引用和模板引用的概念是统一的。为了获得对模板内元素或组件实例的引用，我们可以像往常一样声明ref并从setup()返回。作为模板使用的ref的行为与任何其他ref一样：它们是响应式的，可以传递到（或从中返回）复合函数中`
- JSX中的用法
```javascript
export default{
    setup(){
        const root = ref(null)

        return ()=>{
            h('div', {ref: root})
        }

        // with JSX
        return ()=> <div ref={root}> 
    }
}
```
- v-for中的用法
`组合式API模板引用在 `v-for` 内部使用时没有特殊处理。相反，请使用函数引用执行自定义处理：`
```javascript
<template>
    <div v-for="(item,index) in list" :ref =" el=>{ if(el)  divs[i] = el }"> {{item}} </div>
</template>
<script>
import { ref, reactive, onBeforeUpdate} from 'vue'
export default{
    setup(){
        const list = reactive([1,2,3])
        const divs = ref([])
        // 确保在每次更新之前重置ref
        onBeforeUpdate(()=>{
            divs.value = []
        })

        return {
            list,
            divs
        }
    }
}
</script>
```
- 侦听模板引用
`侦听模板引用的变更可以替代前面例子中演示使用的生命周期钩子。但与生命周期钩子的一个关键区别是，watch()和watchEffect()在DOM挂载或更新之前运行副作用，所以当侦听器运行时，模板引用还未被更新。因此，使用模板引用的侦听器应该用`flush:'post'`选项来定义，这将在DOM更新后运行副作用，确保模板引用与DOM保持同步，并引用正确的元素。`