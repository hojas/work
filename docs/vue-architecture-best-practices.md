# Vue 3 项目架构最佳实践

> 公司内部技术分享 · 2026
>
> 面向有 Vue 基础、希望构建可维护大型项目的开发者

---

## 目录

1. [项目目录结构设计](#一项目目录结构设计)
2. [组件架构设计](#二组件架构设计)
3. [可组合函数 Composables 设计](#三可组合函数-composables-设计)
4. [状态管理架构](#四状态管理架构)
5. [路由与导航架构](#五路由与导航架构)
6. [TypeScript 集成最佳实践](#六typescript-集成最佳实践)
7. [构建与工程化](#七构建与工程化)
8. [测试策略](#八测试策略)
9. [性能优化](#九性能优化)
10. [安全实践](#十安全实践)
11. [架构演进案例](#十一架构演进案例)
12. [推荐工具链](#十二推荐工具链)

---

## 一、项目目录结构设计

### 📌 核心理念

**按领域/功能组织，而非按文件类型。** 当项目增长到 20+ 组件时，把所有 `.vue` 文件堆在 `components/` 下会迅速失控。

### ✅ 推荐做法

采用 **feature-based + 分层混合** 结构：

```text
src/
├── pages/                    # 路由级页面 —— 只做数据获取和布局编排
│   ├── dashboard/
│   │   ├── DashboardPage.vue
│   │   └── widgets/
│   │       ├── StatsWidget.vue
│   │       └── ChartWidget.vue
│   └── settings/
│       ├── SettingsPage.vue
│       └── tabs/
│           ├── ProfileTab.vue
│           └── BillingTab.vue
│
├── components/               # 跨页面共享的通用组件
│   ├── ui/                   #   UI 原语 (Button, Input, Modal, Card …)
│   │   ├── Button.vue
│   │   ├── Modal.vue
│   │   └── SurfaceCard.vue
│   └── layout/               #   布局组件 (Header, Sidebar, Footer …)
│       ├── AppHeader.vue
│       └── AppSidebar.vue
│
├── composables/              # 可复用的状态逻辑 (use 前缀)
│   ├── useAuth.ts
│   ├── useFetch.ts
│   └── useMediaQuery.ts
│
├── stores/                   # Pinia 全局状态
│   ├── auth.ts
│   └── notification.ts
│
├── api/                      # API 请求封装
│   ├── client.ts             #   基础请求实例 (axios / fetch 封装)
│   ├── endpoints.ts          #   接口路径常量
│   └── services/             #   按领域拆分的 api 函数
│       ├── userService.ts
│       └── orderService.ts
│
├── lib/                      # 纯函数工具 (无副作用，无 Vue 依赖)
│   ├── format.ts             #   日期/金额格式化
│   ├── validation.ts         #   校验规则
│   └── constants.ts          #   业务常量
│
├── types/                    # 共享 TypeScript 类型
│   ├── user.ts
│   └── api.ts
│
├── styles/                   # 全局样式 & Design Tokens
│   ├── tokens.css            #   CSS 自定义属性
│   ├── typography.css
│   └── global.css
│
├── assets/                   # 静态资源 (图片、字体、图标)
│   ├── images/
│   └── fonts/
│
├── router/
│   └── index.ts
│
├── App.vue
└── main.ts
```

**命名规范速查：**

| 类型 | 规范 | 示例 |
|------|------|------|
| 组件文件 | PascalCase | `UserProfile.vue` |
| Composable | camelCase + `use` 前缀 | `useAuth.ts` |
| 工具函数 | camelCase | `formatCurrency.ts` |
| 类型文件 | camelCase | `user.ts` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| CSS 类 | kebab-case 或 utility | `.user-profile`, `.text-muted` |

### ❌ 常见反模式

```text
# 反模式 1: 按文件类型堆放
src/
├── components/   # 50+ 个组件，无任何分组
│   ├── Button.vue
│   ├── UserList.vue
│   ├── OrderForm.vue
│   └── ...
├── hooks/        # 与 components/ 无对应关系
└── views/        # 与 components/ 无对应关系
```

```text
# 反模式 2: 过度嵌套
src/pages/dashboard/widgets/charts/line/multi-series/Tooltip.vue
# 7 层深度——没人记得住路径
```

### 💡 实践建议

- **从简单开始**：项目只有 5 个页面时，一个扁平的 `pages/` 完全可以接受。目录结构随复杂度演进，不要提前过度设计。
- **就近放置**：只被一个页面使用的组件放在该页面的目录下，被多处使用的才提升到 `components/`。
- **边界清晰**：`lib/` 不依赖 Vue；`composables/` 依赖 Vue；`api/` 只关心数据获取。

---

## 二、组件架构设计

### 📌 核心理念

**组件是 UI 的基本单元，好的组件设计 = 明确的输入输出 + 单一职责 + 可组合。**

### ✅ 推荐做法

#### 2.1 组件分层

```
Page                    —— 路由入口，获取数据，编排布局
  └─ Container          —— 连接数据和行为，不渲染 DOM 细节
       └─ Presentational —— 接收 props，纯渲染
            └─ UI Primitive —— 通用基础组件 (Button, Input...)
```

**Page 组件示例：**

```vue
<!-- pages/dashboard/DashboardPage.vue -->
<script setup lang="ts">
import { useDashboardData } from './composables/useDashboardData'
import DashboardHeader from './components/DashboardHeader.vue'
import StatsGrid from './components/StatsGrid.vue'
import ChartPanel from './components/ChartPanel.vue'

// Page 层只负责：获取数据 + 编排子组件
const { stats, chartData, isLoading, error } = useDashboardData()
</script>

<template>
  <div class="dashboard-page">
    <DashboardHeader />
    <StatsGrid :stats="stats" :loading="isLoading" />
    <ChartPanel :data="chartData" :loading="isLoading" />
    <p v-if="error" class="error">{{ error }}</p>
  </div>
</template>
```

#### 2.2 Props 设计原则

```vue
<script setup lang="ts">
// ✅ 类型驱动 + withDefaults
interface Props {
  title: string
  items: Item[]
  loading?: boolean
  size?: 'sm' | 'md' | 'lg'
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  size: 'md',
})
</script>

<template>
  <div :class="['panel', `panel--${props.size}`]">
    <h2>{{ props.title }}</h2>
    <slot :items="props.items" />
    <Spinner v-if="props.loading" />
  </div>
</template>
```

**Props 设计原则：**
- 必选 props 放前面，可选放在后面并给默认值
- 用字面量联合类型而非宽泛的 `string`（`'sm' | 'md' | 'lg'` 比 `string` 安全得多）
- 复杂对象考虑拆分为多个独立 prop，或提供一个组合 prop 加单独覆盖项
- 组件 props 数量一般不超过 8 个，超过时考虑拆分子组件

#### 2.3 Emits 设计原则

```vue
<script setup lang="ts">
// ✅ 类型安全的 emit
const emit = defineEmits<{
  submit: [data: FormData]
  cancel: []
  'update:modelValue': [value: string]
}>()

function handleSubmit(data: FormData) {
  emit('submit', data)
}
</script>
```

#### 2.4 Slots 设计

```vue
<!-- components/ui/Modal.vue -->
<script setup lang="ts">
defineProps<{ open: boolean; title: string }>()
defineEmits<{ close: [] }>()
</script>

<template>
  <Teleport to="body">
    <div v-if="open" class="modal-overlay" @click.self="$emit('close')">
      <div class="modal-content">
        <header class="modal-header">
          <h2>{{ title }}</h2>
          <slot name="actions" />
        </header>
        <div class="modal-body">
          <slot />
        </div>
        <footer class="modal-footer">
          <slot name="footer" :close="() => $emit('close')">
            <button @click="$emit('close')">关闭</button>
          </slot>
        </footer>
      </div>
    </div>
  </Teleport>
</template>
```

#### 2.5 Compound Components 模式

当子组件之间需要共享状态时，用 Provide/Inject 替代 prop drilling：

```vue
<!-- Tabs.vue —— 父组件掌管状态 -->
<script setup lang="ts">
import { ref, provide, computed, type InjectionKey, type Ref } from 'vue'

interface TabsContext {
  activeTab: Ref<string>
  selectTab: (id: string) => void
}
export const TABS_KEY: InjectionKey<TabsContext> = Symbol('Tabs')

const props = defineProps<{ defaultTab: string }>()
const activeTab = ref(props.defaultTab)

provide(TABS_KEY, {
  activeTab,
  selectTab: (id) => { activeTab.value = id },
})
</script>

<template>
  <div class="tabs"><slot /></div>
</template>
```

```vue
<!-- Tabs.Trigger.vue —— 子组件通过 inject 消费 -->
<script setup lang="ts">
import { inject, computed } from 'vue'
import { TABS_KEY } from './Tabs.vue'

const props = defineProps<{ value: string }>()
const ctx = inject(TABS_KEY)!
const isActive = computed(() => ctx.activeTab.value === props.value)
</script>

<template>
  <button :class="['tab-trigger', { active: isActive }]" @click="ctx.selectTab(props.value)">
    <slot />
  </button>
</template>
```

#### 2.6 组件粒度参考

| 信号 | 行动 |
|------|------|
| 组件超过 300 行 | 提取子组件或 composable |
| 单个 `ref` 被 5+ 处使用 | 提取为 composable |
| 模板中重复 3+ 次的 UI 片段 | 提取为子组件 |
| props 超过 8 个 | 检查是否可以合并或拆分 |

### ❌ 常见反模式

```vue
<!-- ❌ 巨型组件：一个组件做所有事 —— 200 行状态 + 100 行逻辑 + 150 行模板 -->

<!-- ❌ Props 爆炸 -->
<script setup lang="ts">
defineProps<{
  title: string; subtitle: string; avatar: string; email: string
  phone: string; address: string; bio: string; twitter: string
  github: string; linkedin: string; website: string
}>()
</script>
<!-- 应抽一个 user: UserProfile 对象 prop -->

<!-- ❌ 直接修改 prop -->
<script setup lang="ts">
const props = defineProps<{ user: User }>()
function rename() { props.user.name = 'new name' } // 错误！
</script>
<!-- 正确：emit 事件让父组件修改，或用 v-model -->
```

### 💡 实践建议

- **优先组合而非继承**。Vue 3 Composition API 天然支持组合——用 composables 和插槽，别想着 extend 组件。
- **SFC 块顺序**：`<script setup>` → `<template>` → `<style scoped>`，团队内保持一致。
- **`provide`/`inject` 必须带 `InjectionKey<T>` 类型**（详见第六章），确保类型安全。

---

## 三、可组合函数 Composables 设计

### 📌 核心理念

**Composable 是 Vue 3 的第一公民——将响应式状态逻辑从组件中提取为独立、可测试、可复用的函数。**

### ✅ 推荐做法

#### 3.1 基本结构

```typescript
// composables/useFetch.ts
import { ref, watchEffect, onUnmounted, type Ref } from 'vue'

interface UseFetchOptions {
  immediate?: boolean
  refetchInterval?: number
}

export function useFetch<T>(
  url: Ref<string> | string,
  options: UseFetchOptions = {},
) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const isLoading = ref(false)

  const urlRef = isRef(url) ? url : ref(url)
  let intervalId: ReturnType<typeof setInterval> | undefined

  async function execute() {
    isLoading.value = true
    error.value = null
    try {
      const res = await fetch(urlRef.value)
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      data.value = await res.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      isLoading.value = false
    }
  }

  watchEffect(() => {
    if (options.immediate !== false) execute()
  })

  if (options.refetchInterval) {
    intervalId = setInterval(execute, options.refetchInterval)
  }

  // ✅ CRITICAL: 清理副作用
  onUnmounted(() => {
    if (intervalId) clearInterval(intervalId)
  })

  return { data, error, isLoading, refetch: execute }
}
```

#### 3.2 输入输出约定

```typescript
import { unref, type MaybeRef } from 'vue'

// ✅ 同时接受 Ref 和原始值
export function useMediaQuery(query: MaybeRef<string>) {
  const mql = window.matchMedia(unref(query))
  const matches = ref(mql.matches)

  const handler = (e: MediaQueryListEvent) => { matches.value = e.matches }
  mql.addEventListener('change', handler)

  onUnmounted(() => mql.removeEventListener('change', handler))

  return { matches }
}
```

#### 3.3 Composable 常见分类

```typescript
// 1. 状态型 —— 封装特定状态管理
export function useLocalStorage<T>(key: string, fallback: T) { /* ... */ }
export function useDarkMode() { /* ... */ }
export function useMediaQuery(query: MaybeRef<string>) { /* ... */ }

// 2. 数据获取型 —— 封装 API 请求逻辑
export function useFetch<T>(url: MaybeRef<string>) { /* ... */ }
export function useInfiniteScroll(fetchFn: () => Promise<Page>) { /* ... */ }

// 3. 事件型 —— 封装事件监听
export function useEventListener<E extends Event>(
  target: MaybeRef<EventTarget | null>,
  event: string,
  handler: (e: E) => void,
) { /* ... */ }

// 4. 动画型 —— 封装动画/交互状态
export function useScrollProgress(target: MaybeRef<HTMLElement>) { /* ... */ }
export function useTransition(source: Ref<number>, options: { duration: number }) { /* ... */ }
```

#### 3.4 关于 VueUse

`@vueuse/core` 是 Vue 生态中覆盖面最广的 composable 库，涵括上述四类场景中的绝大多数常见需求。

**使用原则：**
- **优先使用，但理解原理** —— 看过源码再决定是直接用还是定制
- **不要为用而用** —— 一个组件只用了 `useTitle`，不需要引入整个 VueUse
- **注意 tree-shaking** —— `import { useDark } from '@vueuse/core'` 只打包用到的函数

### ❌ 常见反模式

```typescript
// ❌ composable 中直接操作 DOM
export function useAlert() {
  const el = document.querySelector('.alert') // composable 不应知道 DOM 结构
}

// ❌ 忘记清理副作用
export function useClock() {
  const now = ref(Date.now())
  setInterval(() => { now.value = Date.now() }, 1000) // 永不清理 —— 内存泄漏
  return { now }
}

// ❌ composable 内部创建或挂载 Vue 组件实例
// composable 是纯逻辑，不应 createApp / mount
```

### 💡 实践建议

- **一个 composable 只做一件事**。`useAuth()` 管认证逻辑，`usePermissions()` 管权限检查，别混在一起。
- **composable 应该是可测试的**。如果测试需要 mount 整个组件才能用，说明耦合太重。
- **返回对象而非数组**（与 React hooks 不同的惯例）。对象解构可以按需取用，顺序无关。

---

## 四、状态管理架构

### 📌 核心理念

**不是所有状态都应该放在全局 store。按状态的生命周期和来源选择正确的工具。**

### ✅ 推荐做法

#### 4.1 四类状态分类

| 状态类型 | 来源 | 生命周期 | 工具 |
|----------|------|----------|------|
| **服务端状态** | API 响应 | 服务端决定 | TanStack Query (Vue Query) |
| **客户端全局状态** | 用户交互 | 跨页面/组件 | Pinia |
| **URL 状态** | URL 参数 | 与 URL 绑定 | Vue Router |
| **表单/局部状态** | 组件内部 | 组件卸载时销毁 | `ref` / `reactive` |

#### 4.2 Pinia Setup Store（推荐写法）

```typescript
// stores/auth.ts
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'
import { api } from '@/api/client'

export const useAuthStore = defineStore('auth', () => {
  // State —— 用 ref
  const user = ref<User | null>(null)
  const token = ref<string | null>(localStorage.getItem('token'))

  // Getters —— 用 computed
  const isAuthenticated = computed(() => !!token.value && !!user.value)
  const displayName = computed(() => user.value?.name ?? '未登录')

  // Actions —— 普通异步函数
  async function login(credentials: { email: string; password: string }) {
    const res = await api.post('/auth/login', credentials)
    token.value = res.data.token
    user.value = res.data.user
    localStorage.setItem('token', res.data.token)
  }

  function logout() {
    token.value = null
    user.value = null
    localStorage.removeItem('token')
  }

  return { user, token, isAuthenticated, displayName, login, logout }
})
```

**Setup Store 优势**：与 Composition API 语法完全一致、可直接使用 composables、类型推导更自然。

#### 4.3 服务端状态 —— 不要手动同步

```vue
<!-- ❌ 手动同步模式 —— 大量样板，双数据源 -->
<script setup lang="ts">
const store = useUserStore()
const users = ref<User[]>([])
const loading = ref(true)

onMounted(async () => {
  loading.value = true
  const res = await fetch('/api/users')
  users.value = res.data
  store.setUsers(res.data)  // 重复存储
  loading.value = false
})
</script>
```

```vue
<!-- ✅ TanStack Query —— 单一数据源，自动缓存 -->
<script setup lang="ts">
import { useQuery } from '@tanstack/vue-query'

const { data: users, isLoading, error } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then(r => r.json()),
  staleTime: 5 * 60 * 1000, // 5 分钟内不重新请求
})
</script>
```

#### 4.4 URL 状态作为真实的页面状态

```typescript
const route = useRoute()
const router = useRouter()

// 从 URL 读取 —— 不做二次存储
const currentFilter = computed(() => route.query.status ?? 'all')
const currentPage = computed(() => Number(route.query.page) || 1)

function setFilter(status: string) {
  router.push({ query: { ...route.query, status, page: undefined } })
}
```

### ❌ 常见反模式

```typescript
// ❌ 所有状态都放 Pinia —— 包括局部 UI 状态
export const useUIStore = defineStore('ui', () => {
  const isModalOpen = ref(false)       // 应该留在组件里
  const hoveredItemId = ref<string>()  // 同上
  const formInputValue = ref('')       // 同上
})

// ❌ Store 中复制 API 响应
// 数据同时存在于 TanStack Query 缓存和 Pinia —— 到底信哪个？

// ❌ 没有持久化的 token 写在 Pinia
// 刷新后 store 丢失，token 需配合 localStorage / cookie 做持久化
```

### 💡 实践建议

- **Pinia store 按领域拆分**（`auth`, `orders`, `notifications`），避免单个巨型 store。
- **区分 Server State 和 Client State** —— 这是前端架构中最容易被忽视的边界。
- **乐观更新**用 TanStack Query 的 `onMutate` + `onError`，先改 UI，失败回滚。

---

## 五、路由与导航架构

### 📌 核心理念

**URL 就是页面状态——筛选、排序、分页、当前 tab 都放 URL 里，页面才可分享、可书签、可后退。**

### ✅ 推荐做法

#### 5.1 路由懒加载

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/dashboard',
      component: () => import('@/pages/dashboard/DashboardPage.vue'),
      meta: { requiresAuth: true },
    },
    {
      path: '/settings',
      component: () => import('@/pages/settings/SettingsLayout.vue'),
      meta: { requiresAuth: true },
      children: [
        { path: '', redirect: { name: 'settings-profile' } },
        {
          path: 'profile',
          name: 'settings-profile',
          component: () => import('@/pages/settings/tabs/ProfileTab.vue'),
        },
        {
          path: 'billing',
          name: 'settings-billing',
          component: () => import('@/pages/settings/tabs/BillingTab.vue'),
        },
      ],
    },
    {
      path: '/:pathMatch(.*)*',
      name: 'not-found',
      component: () => import('@/pages/NotFoundPage.vue'),
    },
  ],
})
```

#### 5.2 路由元信息

```typescript
// 扩展 RouteMeta 类型
declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
    title?: string
    breadcrumb?: string
    permissions?: string[]
  }
}
```

#### 5.3 导航守卫

```typescript
// 全局前置守卫 —— 认证检查
router.beforeEach((to) => {
  const auth = useAuthStore()
  if (to.meta.requiresAuth && !auth.isAuthenticated) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})

// 全局后置钩子 —— 页面标题
router.afterEach((to) => {
  document.title = `${to.meta.title ?? '首页'} | MyApp`
})
```

```vue
<!-- 组件内守卫 —— 防止未保存数据丢失 -->
<script setup lang="ts">
import { onBeforeRouteLeave } from 'vue-router'

const formDirty = ref(false)

onBeforeRouteLeave((_to, _from, next) => {
  if (formDirty.value) {
    const answer = window.confirm('有未保存的更改，确认离开？')
    next(answer)
  } else {
    next()
  }
})
</script>
```

#### 5.4 路由参数双向绑定

```vue
<script setup lang="ts">
const route = useRoute()
const router = useRouter()

// ✅ 用 computed get/set 将 URL 参数当作响应式状态
const activeTab = computed({
  get: () => (route.query.tab as string) ?? 'overview',
  set: (val) => router.push({ query: { ...route.query, tab: val } }),
})

const page = computed({
  get: () => Number(route.query.page) || 1,
  set: (val) => router.push({ query: { ...route.query, page: String(val) } }),
})
</script>

<template>
  <Tabs v-model="activeTab">
    <Tab value="overview">概览</Tab>
    <Tab value="details">详情</Tab>
  </Tabs>
  <Pagination v-model="page" :total="100" />
</template>
```

### ❌ 常见反模式

```typescript
// ❌ 路由组件同步导入 —— 所有页面打包进首屏
import Dashboard from '@/pages/Dashboard.vue'

// ❌ 路由组件内部做大量数据转换 —— 应该放在 composable 或子组件里

// ❌ 用 watch(route) 代替 computed 监听 URL 参数
watch(() => route.query.tab, (tab) => {
  activeTab.value = tab as string // 多余的赋值
})
```

### 💡 实践建议

- **命名路由优于路径字符串**。`router.push({ name: 'settings-billing' })` 比 `'/settings/billing'` 更不易出错。
- **嵌套路由 + `<RouterView>` 实现布局嵌套**，比在每个页面里包 `<AppLayout>` 更干净。
- **params 是资源标识（`/user/:id`），query 是展示选项（`?tab=profile`）**。

---

## 六、TypeScript 集成最佳实践

### 📌 核心理念

**TypeScript 不只是给变量加类型标注——它是架构的约束层，防止整类运行时错误。**

### ✅ 推荐做法

#### 6.1 Strict 是底线

```json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "isolatedModules": true
  }
}
```

**`strict: true` 不是负担，是安全带。**

#### 6.2 组件全链路类型

```vue
<script setup lang="ts">
interface Props {
  items: Item[]
  selectedId?: string
  loading?: boolean
}
const props = withDefaults(defineProps<Props>(), { loading: false })

const emit = defineEmits<{
  select: [id: string]
  delete: [id: string]
}>()

defineSlots<{
  default: (props: { item: Item }) => unknown
  header: () => unknown
  empty: () => unknown
}>()

const listEl = ref<HTMLElement>()
const childComp = ref<InstanceType<typeof ChildComponent>>()
</script>
```

#### 6.3 Provide/Inject 类型安全

```typescript
// types/injectionKeys.ts
import type { InjectionKey, Ref } from 'vue'

export interface ThemeContext {
  theme: Ref<'light' | 'dark'>
  toggleTheme: () => void
}
export const THEME_KEY: InjectionKey<ThemeContext> = Symbol('theme')
```

```vue
<!-- 后代组件中 injection 自动获得类型 -->
<script setup lang="ts">
import { inject } from 'vue'
import { THEME_KEY } from '@/types/injectionKeys'

const ctx = inject(THEME_KEY)
if (!ctx) throw new Error('ThemeContext not provided')
// ctx.theme 自动推导为 Ref<'light' | 'dark'>
</script>
```

#### 6.4 API 响应运行时校验

```typescript
import { z } from 'zod'

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'viewer']),
})

export type User = z.infer<typeof UserSchema>

export async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`)
  const json = await res.json()
  return UserSchema.parse(json) // 在边界处暴露问题
}
```

### ❌ 常见反模式

```typescript
// ❌ as any —— 类型安全的逃逸舱
const data = ref<any>(null)
const result = (await api.get()).data as any

// ❌ API 函数返回 any
async function getUsers() {
  const res = await api.get('/users')
  return res.data // 隐式 any
}

// ❌ 裸 Symbol 做 InjectionKey —— 没有类型信息
export const KEY = Symbol() // 类型推导为 symbol，不是 InjectionKey
```

### 💡 实践建议

- **`vue-tsc --noEmit` 放入 CI**。类型错误 = 构建失败。这是成本最低的 bug 预防。
- **用 `z.infer<typeof Schema>` 派生 TS 类型**，避免类型定义与校验逻辑不一致。
- **`InjectionKey<T>` 是必须的**，裸 `Symbol()` 不带类型信息。

---

## 七、构建与工程化

### 📌 核心理念

**Vite 是标配，但工程化不止于 `vite.config.ts` —— Bundle 分析、代码分割、环境变量、Git Hooks 共同构成质量基线。**

### ✅ 推荐做法

#### 7.1 Vite 配置

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],

  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },

  build: {
    target: 'es2020',
    rollupOptions: {
      output: {
        manualChunks: {
          vue: ['vue', 'vue-router', 'pinia'],
          vueuse: ['@vueuse/core'],
          query: ['@tanstack/vue-query'],
        },
      },
    },
  },
})
```

#### 7.2 环境变量

```bash
# .env                  —— 所有环境共享
VITE_APP_TITLE=MyApp

# .env.development      —— 仅 dev
VITE_API_BASE_URL=http://localhost:3000/api

# .env.production       —— 仅 prod
VITE_API_BASE_URL=https://api.example.com
```

```typescript
// env.d.ts —— 类型安全的环境变量
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_TITLE: string
}
interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

**⚠️ 只有 `VITE_` 前缀暴露给客户端。不要把密钥放在这里。**

#### 7.3 CSS 工程化：Design Tokens

```css
/* styles/tokens.css */
:root {
  /* 调色板 */
  --color-surface: oklch(98% 0 0);
  --color-surface-alt: oklch(95% 0 0);
  --color-text: oklch(18% 0 0);
  --color-text-muted: oklch(50% 0 0);
  --color-accent: oklch(65% 0.21 250);

  /* 间距 */
  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --space-lg: 1.5rem;
  --space-xl: 2rem;
  --space-section: clamp(3rem, 3rem + 5vw, 8rem);

  /* 排版 */
  --text-base: clamp(1rem, 0.9rem + 0.4vw, 1.125rem);
  --text-lg: clamp(1.25rem, 1.1rem + 0.8vw, 1.5rem);

  /* 动效 */
  --duration-fast: 150ms;
  --duration-normal: 300ms;
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
}
```

**组件中引用变量，永远不硬编码颜色值。**

#### 7.4 Git Hooks

```json
{
  "simple-git-hooks": {
    "pre-commit": "pnpm lint-staged"
  },
  "lint-staged": {
    "*.{vue,ts,js}": ["eslint --fix"],
    "*.css": ["stylelint --fix"]
  }
}
```

### ❌ 常见反模式

- ❌ 不分析 bundle —— 引入了 moment.js 却浑然不觉
- ❌ 生产构建带 `sourcemap: true` —— 暴露源码
- ❌ 所有依赖打成一个 vendor chunk —— 一个库更新就要全量重新下载
- ❌ 每个组件里重复硬编码颜色/间距 —— 改一次等于搜全局

### 💡 实践建议

- **`rollup-plugin-visualizer` 定期分析，找到最大的 chunk**。
- **`vue-tsc --noEmit` 作为 `pnpm build` 的前置步骤**。类型错误必须阻塞构建。
- **Design Tokens 是最小投入、最大回报的 CSS 工程化措施**。

---

## 八、测试策略

### 📌 核心理念

**分层测试，金字塔模型：大量单元测试 + 适量集成测试 + 少量 E2E 覆盖关键用户流程。**

```
         /\
        /E2E\         Playwright — 核心用户流程
       /------\
      / 集成   \      Vue Test Utils — 组件交互
     /----------\
    /   单元测试  \    Vitest — composables / stores / utils
   /--------------\
```

### ✅ 推荐做法

#### 8.1 Composable 测试

```typescript
// composables/__tests__/useCounter.test.ts
import { describe, it, expect } from 'vitest'
import { useCounter } from '../useCounter'

describe('useCounter', () => {
  it('starts at 0 by default', () => {
    const { count } = useCounter()
    expect(count.value).toBe(0)
  })

  it('increments the count', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
  })
})
```

#### 8.2 Pinia Store 测试

```typescript
import { setActivePinia, createPinia } from 'pinia'
import { beforeEach, describe, it, expect } from 'vitest'
import { useAuthStore } from '../auth'

describe('authStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('is unauthenticated by default', () => {
    expect(useAuthStore().isAuthenticated).toBe(false)
  })
})
```

#### 8.3 组件测试 —— 测行为不测实现

```typescript
import { mount } from '@vue/test-utils'
import { describe, it, expect } from 'vitest'
import Counter from '../Counter.vue'

describe('Counter', () => {
  it('increments count on button click', async () => {
    const wrapper = mount(Counter, { props: { initial: 0 } })

    expect(wrapper.text()).toContain('0')
    await wrapper.find('button').trigger('click')
    expect(wrapper.text()).toContain('1')
  })

  it('emits overflow when count reaches max', async () => {
    const wrapper = mount(Counter, { props: { initial: 9, max: 10 } })
    await wrapper.find('button').trigger('click')
    expect(wrapper.emitted('overflow')).toBeTruthy()
  })
})
```

#### 8.4 E2E 测试

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test'

test('user can log in and see dashboard', async ({ page }) => {
  await page.goto('/login')
  await page.fill('[data-testid="email-input"]', 'test@example.com')
  await page.fill('[data-testid="password-input"]', 'password')
  await page.click('[data-testid="login-button"]')

  await expect(page).toHaveURL('/dashboard')
  await expect(page.locator('h1')).toContainText('Dashboard')
})
```

### ❌ 常见反模式

- ❌ **测试实现细节**：不要测 `wrapper.vm.internalCount` 的值——重构会破坏测试
- ❌ **过度 mock**：mock 整个 vue-router 让路由测试失去意义
- ❌ **覆盖率 100% 崇拜**：100% 覆盖率 ≠ 零 bug。关注复杂逻辑和边界条件

### 💡 实践建议

- **测试命名描述行为**：`'returns empty array when no results'` 而非 `'test getResults'`
- **用 `data-testid` 隔离测试选择器**，不与 CSS 类名或 DOM 结构耦合
- **E2E 只覆盖核心流程**（登录、支付、核心 CRUD），不试图穷举所有分支

---

## 九、性能优化

### 📌 核心理念

**以 Core Web Vitals 为指南：LCP < 2.5s、INP < 200ms、CLS < 0.1。优化从构建配置覆盖到运行时细节。**

### ✅ 推荐做法

#### 9.1 异步组件 + Suspense

```vue
<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const ChartPanel = defineAsyncComponent({
  loader: () => import('./ChartPanel.vue'),
  loadingComponent: () => <Spinner />,
  errorComponent: () => <ErrorCard message="图表加载失败" />,
  delay: 200,      // 200ms 后才显示 loading（避免闪烁）
  timeout: 10000,
})
</script>

<template>
  <Suspense>
    <ChartPanel :data="chartData" />
    <template #fallback><Spinner /></template>
  </Suspense>
</template>
```

#### 9.2 列表优化

```vue
<!-- v-memo：只重新渲染实际上变化的行 -->
<div v-for="item in items" :key="item.id" v-memo="[item.updatedAt]">
  <ExpensiveRow :item="item" />
</div>
```

1000+ 行列表使用虚拟滚动（`vue-virtual-scroller` 或 `@tanstack/vue-virtual`）。

#### 9.3 响应式优化

```typescript
// shallowRef —— 大数据不需要深层响应式
const largeDataset = shallowRef<DataPoint[]>([])
// 只追踪 .value 的替换，适合整体更新场景（API 响应、图表数据）

// shallowReactive —— 仅第一层属性是响应式
const config = shallowReactive({ theme: 'light', locale: 'zh' })
```

#### 9.4 图片与媒体

```html
<!-- 首屏 hero 图片 —— eager -->
<img src="/hero.webp" alt="" width="1200" height="600"
     loading="eager" fetchpriority="high" />

<!-- 非首屏图片 —— lazy + 显式尺寸防止 CLS -->
<img :src="imageUrl" :alt="alt" width="640" height="360"
     loading="lazy" decoding="async" />
```

#### 9.5 动画性能

**只动画化 compositor-friendly 属性：`transform`、`opacity`、`clip-path`、`filter`（谨慎）。**

```css
/* ✅ transform + opacity —— 跳过 Layout 和 Paint 阶段 */
.card-enter {
  opacity: 0;
  transform: translateY(20px);
  transition: opacity var(--duration-normal) var(--ease-out-expo),
              transform var(--duration-normal) var(--ease-out-expo);
}
.card-enter-active {
  opacity: 1;
  transform: translateY(0);
}

/* ❌ 不要动画化 width、height、top、left、margin、padding —— 触发 Layout */
```

### ❌ 常见反模式

- ❌ `deep: true` watch 大型对象 —— 递归遍历全部属性
- ❌ 全量注册全局组件 —— 首屏 JS 膨胀
- ❌ `watchEffect` 中产生无条件的副作用链 —— 可能触发无限循环

### 💡 实践建议

- **用 Lighthouse 和 DevTools Performance 面板验证，不凭感觉优化**。
- **`v-memo` 是 Vue 3 最被低估的性能特性**，长列表中效果显著。
- **`manualChunks` 把稳定的核心依赖（vue, pinia, vueuse）单独分包**，提高浏览器缓存命中率。

---

## 十、安全实践

### 📌 核心理念

**安全是架构的一部分，不是后补的 checklist。前端安全核心是 XSS 防护 + 输入校验 + CSP。**

### ✅ 推荐做法

#### 10.1 XSS 防护

```vue
<!-- ✅ Vue 模板自动转义 —— 默认安全 -->
<p>{{ userInput }}</p>

<!-- ❌ v-html 必须 sanitized -->
<div v-html="sanitizedHtml" />

<script setup lang="ts">
import DOMPurify from 'dompurify'
const raw = '<p>User <script>alert(1)<\/script> content</p>'
const sanitizedHtml = DOMPurify.sanitize(raw)
// 结果: '<p>User  content</p>'
</script>
```

#### 10.2 API 安全

```typescript
// ✅ CSRF Token（如后端要求）
const csrfToken = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content')

// ✅ 状态变更用 POST，不要用 GET
// ❌ GET /api/delete-user?id=123  — 可被 <img> 触发
// ✅ POST /api/users/123/delete
```

#### 10.3 敏感信息

```typescript
// ❌ 密钥绝对不要出现在客户端代码中
const STRIPE_SECRET = 'sk_live_xxxx' // 禁止！

// ⚠️ VITE_* 变量在构建时被内联，任何人都能从 bundle 看到
// 真正需要保密的 key 只能放在后端
```

```bash
# 定期审计依赖漏洞
pnpm audit
pnpm audit --audit-level=high # CI 中作为检查门禁
```

### ❌ 常见反模式

- ❌ `v-html` 直接渲染用户输入
- ❌ 前端校验替代后端校验 —— 攻击者直接调 API
- ❌ 用户输入拼接到 URL：`<a :href="userInput">` —— 可能是 `javascript:alert(1)`

### 💡 实践建议

- **CSP Header 是最后一道防线**，限制脚本来源。
- **对用户输入永远做最坏假设**，前后端都要校验。
- **`<a>` 的 `href` 来自用户输入时，校验协议为 `http:` 或 `https:`**。

---

## 十一、架构演进案例

### 📌 核心理念

**架构是演进的，不是设计的。从简单开始，只在复杂度真正出现时才引入新模式。**

### 演进阶段

#### 阶段一：项目启动期

**特征**：1-5 个页面，1-2 人，验证方向

```text
src/
├── pages/          # 平铺页面
├── components/     # 少量共享组件
├── router/
├── App.vue
└── main.ts
```

**选型**：Vue 3 + Vite + Vue Router。不需要 Pinia，`ref` 足够。

**关键决策**：商定目录命名和 lint 规则。这些后来改起来最贵。

---

#### 阶段二：页面增多

**特征**：10-20 页，路由复杂，少量跨页面共享状态

**引入**：路由懒加载、Pinia（仅用于跨页面状态）、`api/` 层

```text
src/
├── pages/          # 子目录出现，页面内组件就近存放
├── components/     # ui/ 和 layout/ 开始分化
├── composables/    # 首个 useAuth / useFetch
├── stores/         # 1-2 个 store
├── api/            # client.ts + service 文件
└── ...
```

---

#### 阶段三：数据复杂度上升

**特征**：分页、筛选、缓存、乐观更新需求

**引入**：TanStack Query、zod 校验、`types/` 独立为目录

---

#### 阶段四：团队扩大

**特征**：3+ 人并行开发，组件复用强烈

**引入**：Design System / 组件库、测试体系 (Vitest + Playwright)、CI 强制检查

---

#### 阶段五：生产优化

**特征**：用户量增长，性能/稳定性优先

**引入**：错误监控 (Sentry)、Web Vitals 上报、灰度发布、Bundle 持续优化、CSP

---

### 演进原则

| 原则 | 说明 |
|------|------|
| **不一定走到阶段五** | 很多内部工具停在阶段二就够用了 |
| **不要提前引入复杂度** | 1 页的项目不需要 TanStack Query |
| **但要预留扩展空间** | `api/` 目录第一天就可以建 |
| **重构是正常的** | 每次阶段升级都是合理的重构 |

### 💡 实践建议

- 每个变化由**具体痛点**驱动："每次切页面都要重新请求用户信息" → Pinia；"列表页回来总是 loading" → TanStack Query。
- 提前建立 `api/` 和 `types/` 的边界约定，这两个目录的职责从第一天就该清晰。

---

## 十二、推荐工具链

### 📌 一页速查

| 领域 | 推荐 | 说明 |
|------|------|------|
| **框架** | Vue 3 Composition API | 当前标准 |
| **构建** | Vite | 极速 HMR，Rollup 生产构建 |
| **路由** | Vue Router 4 | 官方路由 |
| **全局状态** | Pinia | 官方推荐，Vuex 维护模式 |
| **服务端状态** | @tanstack/vue-query | 缓存/去重/乐观更新 |
| **工具集** | @vueuse/core | 200+ composables |
| **类型校验** | zod / valibot | 运行时校验，派生 TS 类型 |
| **单元测试** | Vitest | 与 Vite 共用配置 |
| **组件测试** | @vue/test-utils | 官方工具 |
| **E2E** | Playwright | 跨浏览器，稳定 |
| **Lint** | ESLint + @antfu/eslint-config | Vue/TS 一站规则集 |
| **文档** | VitePress | Vite + Vue 文档站 |
| **组件库** | Naive UI / Ant Design Vue / Element Plus | 按需 |
| **部署** | Vercel / GitHub Pages / Docker | 视项目 |

### 快速起步

```bash
pnpm create vite my-app --template vue-ts
```

---

## 附录：架构自检清单

在项目关键节点（上线、交接、重构）逐项检查：

- [ ] 目录按功能域组织，非文件类型平铺
- [ ] 组件分层明确（Page / Container / Presentational / UI）
- [ ] 无超 400 行的单个组件
- [ ] 可复用逻辑提取为 composable，有副作用清理
- [ ] Pinia store 按领域拆分；服务端状态用 TanStack Query
- [ ] 路由全懒加载；筛选/排序/分页状态在 URL
- [ ] TypeScript strict 开启，无 `as any` 滥用
- [ ] API 响应有 zod/valibot 校验
- [ ] Design Tokens 在 CSS 变量定义
- [ ] `vue-tsc --noEmit` 在 CI 通过
- [ ] 关键用户流程有 E2E 测试
- [ ] 无 `v-html` 渲染未 sanitized 输入
- [ ] `VITE_*` 变量中无密钥
- [ ] 构建产物经过 bundle 分析

---

> **架构没有银弹。好的架构来自对项目规模的诚实评估、对复杂度的审慎管理、以及持续的、小步的重构。**
