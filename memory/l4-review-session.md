# M0 L4 人工复审 — 操作记录

> 开始时间：2026-06-03
> Dev Server：http://localhost:5173/
> 进程 PID：后台 `pnpm dev`
> 复审人：用户手动测试

---

## 操作日志

| #   | 时间  | 操作             | 页面/URL         | 数据流                                                                        | 结果 |
| --- | ----- | ---------------- | ---------------- | ----------------------------------------------------------------------------- | :--: |
| 1   | 19:55 | Dev server 启动  | `localhost:5173` | Vite → main.ts → App.vue → router-view → NotebookHome                         |  ✅  |
| 2   | —     | 打开首页         | `/`              | GET / → index.html → main.ts → App.vue 渲染 "MarkLuck" + "M0 scaffold ready." |  ✅  |
| 3   | —     | F12 Console      | `/` → DevTools   | 浏览器控制台打开，无任何 console 输出                                         |  ✅  |
| 4   | —     | 导航到设置页     | `/settings`      | vue-router 匹配 /settings → SettingsPage.vue（空占位），App 壳层照常渲染      |  ✅  |
| 5   | —     | 导航到不存在路径 | `/*`             | vue-router 匹配 /:pathMatch(._)_ → NotFoundPage.vue（空占位）                 |  ✅  |

---

## L4 复审清单

| #   | 检查项                                     | 状态 | 备注                                  |
| --- | ------------------------------------------ | :--: | ------------------------------------- |
| 1   | `pnpm dev` 启动 → 浏览器可以访问           |  ✅  | Vite :5173 正常                       |
| 2   | 页面显示 "MarkLuck" + "M0 scaffold ready." |  ✅  | App.vue 壳层 — M0 预期行为            |
| 3   | 浏览器 Console 无错误                      |  ✅  | ESLint `no-console: error` 生效       |
| 4   | `pnpm build` 构建成功                      |  ✅  | 35 modules, 89.62 kB (35.04 kB gzip)  |
| 5   | 路由导航 `/settings` 正常                  |  ✅  | Router 正确跳转，页面组件为 M0 空占位 |
| 6   | 路由导航不存在的路径 → 404                 |  ✅  | Router catch-all 工作正常             |

---

## 数据流记录（实际运行验证）

```
浏览器请求 HTTP GET /
  → Vite Dev Server :5173
    → packages/app/index.html
      → <script type="module" src="/src/main.ts">
        → main.ts:
            createApp(App)           ← Vue 应用实例
            createPinia()            ← Pinia 状态管理（M0 无 store）
            router                   ← vue-router 实例
            app.use(pinia)
            app.use(router)
            app.mount('#app')
        → App.vue（根布局壳层）
            <h1>MarkLuck</h1>         ← 所有页面共享
            <p>M0 scaffold ready.</p>
            <router-view />           ← 页面内容注入点
              ├── /          → NotebookHome.vue   → <div />（M1+ 实现）
              ├── /settings  → SettingsPage.vue   → <div />（M5 实现）
              └── /*         → NotFoundPage.vue   → <div />（M0 占位）
```

**验证结论**：

- Vue 3 + Pinia + vue-router 三件套正确初始化，无运行时错误
- 路由系统正确匹配不同路径并渲染对应组件
- App.vue 作为布局壳层正确包裹了 `<router-view />`
- 所有页面组件的占位骨架已就绪，M1 可直接填充内容
- L1 ESLint `no-console` 规则在生产代码中生效（控制台清洁）

---

## 发现的问题

无。所有行为符合 M0 脚手架预期。

"M0 scaffold ready." 文本在 App 壳层中，后续里程碑中 AppLayout 三联画结构（M1-20）会替换此占位壳层。

---

## 复审结论

✅ **M0 L4 复审通过。** 可以推进 M1。

---

## 构建验证

```
$ pnpm build
  → vue-tsc --noEmit    ✅ 零错误
  → vite build          577ms

dist/index.html                       0.32 kB │ gzip: 0.24 kB
dist/assets/NotebookHome-CpHZ4Ubo.js  0.15 kB │ gzip: 0.15 kB
dist/assets/SettingsPage-C3tSJijC.js  0.15 kB │ gzip: 0.15 kB
dist/assets/NotFoundPage-CWNsGcsJ.js  0.15 kB │ gzip: 0.15 kB
dist/assets/index-D4m1pP4D.js        89.62 kB │ gzip: 35.04 kB

✅ Bundle 体积在 100 kB gzip 限制内
```
