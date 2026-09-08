# 自研后台组件库实施方案

> 目标：把后台项目中反复出现的搜索栏、列表、分页、增删改弹窗和表单布局抽成可配置、可复用的内部组件库。页面只保留业务字段、接口、权限和少量特殊插槽。

## 1. 先说结论

这个方向值得做。当前项目里的列表页通常要重复编写查询表单、分页状态、加载状态、表格列、弹窗、校验、提交和刷新逻辑；`game_admin` 使用的 `cc1-form` 本质上就是把这些固定流程做成了一个“配置驱动的 CRUD 页面引擎”。

建议不要一开始照着 `cc1-*` 拆出五个包。第一版先做一个包 `@your-scope/admin-kit`，内部按模块分层，对外通过子路径导出：

- `@your-scope/admin-kit/core`：类型、数据适配、通用函数，不依赖 Vue。
- `@your-scope/admin-kit/form`：配置式表单和紧凑弹窗。
- `@your-scope/admin-kit/crud`：搜索、表格、分页、增删改查流程。
- `@your-scope/admin-kit/styles`：小号控件、间距、表单无冒号等全局设计变量。

当这个包被两个以上项目稳定使用后，再按需要拆成独立包。这样第一阶段的发布、版本和依赖管理都简单很多。

## 2. 工期估算

以下按一名熟悉 Vue 3、TypeScript、Element Plus 和当前业务的开发者估算，已有可直接参考的列表和弹窗模板，后端分页结构基本统一。

| 目标                        | 内容                                                                                | 预计时间       |
| --------------------------- | ----------------------------------------------------------------------------------- | -------------- |
| 可演示原型                  | 紧凑表单、弹窗、列表、分页，使用本地假数据                                          | 3～5 个工作日  |
| CRUD MVP                    | 接真实接口，支持查询、重置、分页、新增、编辑、删除、刷新、插槽，在 3 个真实页面落地 | 8～12 个工作日 |
| 可长期使用的内部 v1         | 全局配置、权限、请求适配、错误处理、类型、测试、文档、发布流程、迁移 5～10 个页面   | 3～5 周        |
| 接近当前 `cc1-*` 的完整包族 | 动态字段、自定义控件注册、导出、拖拽、行内编辑、国际化、自动导入、Node 工具等       | 6～10 周       |

最现实的说法是：**一周左右可以看到能用的版本，约三周可以做出能真正减少大部分页面冗余的内部版本；完整复刻 `cc1` 不是一两周的工作。**

下列情况会增加工期：后端返回结构不统一、每个页面权限规则不同、复杂联动表单很多、上传/富文本/多语言组件尚未统一、需要同时兼容多个 UI 框架。

## 3. `cc1-*` 到底是什么

这些目录不是某种 Vue 特殊语法，而是普通的 npm 包。作者在源码仓库中开发，然后通过 Vite/Rollup 构建出 `dist`，发布时只把 `dist` 放进 npm。项目执行 `pnpm install` 后，包被下载到 `node_modules`，再通过 `import` 使用。

基本链路如下：

```text
组件库源码
  -> pnpm build
  -> dist/*.js + dist/*.css + dist/*.d.ts
  -> 发布到 npm/私有仓库
  -> 业务项目 pnpm add 包名
  -> import 或 app.use()
```

几个关键字段：

- `main` / `module` / `exports`：告诉构建工具从哪里加载 JavaScript 和 CSS。
- `types`：告诉 TypeScript 从哪里读取类型声明 `.d.ts`。
- `peerDependencies`：要求业务项目自己安装 Vue、Element Plus 等宿主依赖，组件库不再打包一份，避免出现两个 Vue 实例。
- `files: ["dist"]`：发布时只带构建产物，所以目前在 `node_modules/cc1-form` 看不到完整源码。
- `app.use(plugin)`：调用包导出的 `install(app)`，可以注册全局组件和全局默认配置。

不要直接修改 `node_modules`。重新安装依赖后修改会消失，而且这里是构建产物。正确方式是建立自己的组件库源码项目，通过 workspace、`pnpm link` 或私有 npm 包接入。

## 4. 五个参考包的职责

根据 `/Users/code/game_admin/node_modules` 中的包清单、类型声明和 `game_admin` 的实际使用情况，可得到下面的结构。

| 包         | 主要职责                                                        | 当前观察                                                  | 自研第一版是否需要                                      |
| ---------- | --------------------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------- |
| `cc1-js`   | 浏览器侧通用工具：数组、Cookie、HTTP、事件、DOM、时间、字符串等 | 基础工具包，API 较杂                                      | 只收录真正复用的纯函数，不要整包照抄                    |
| `cc1-vue3` | Vue 相关工具：Store、Router、Scope、校验、倒计时、Mixin         | `game_admin` 主要使用 `Scope`、`CStore`                   | 先用 composable 和 Pinia；有明确需求再抽                |
| `cc1-ui`   | 小型视觉组件：Button、Icon、Switch、Mask、Select、Scale 等      | 支持全局安装和自动导入                                    | 当前已有 Element Plus/PureAdmin，第一版不重复造基础按钮 |
| `cc1-form` | 配置驱动的搜索、表格、分页、表单、弹窗、CRUD、导出、拖拽        | 核心包；在参考项目约 233 个非 `node_modules` 文件中被引用 | 是本次最应该借鉴的部分                                  |
| `cc1-node` | Node 环境的文件、HTTP、字符串、系统工具                         | 参考项目主要在 Vite 插件里使用                            | 前端运行时不需要，构建工具成熟后再考虑                  |

参考包之间可以理解为：

```text
cc1-js          cc1-node（仅 Node/构建阶段）
   |
cc1-vue3
   |
cc1-form  ------ Vue + Element Plus + Vue Router

cc1-ui   ------- Vue 视觉组件（相对独立）
```

最有价值的并不是 `cc1` 这个名字，而是它的分层思想：纯工具、Vue 工具、视觉组件、业务型 CRUD 组件、Node 工具分别管理。

## 5. `cc1-form` 的核心工作方式

`cc1-form` 对外注册了 `TCurd`、`TFormList`、`TColumn`，其中最重要的是 `TCurd`。名称里的 `Curd` 实际表达的是常见的 `CRUD`：Create、Read、Update、Delete，只是字母顺序写成了 C-U-R-D。

一个字段配置同时描述它在搜索、表格和弹窗表单中的行为：

```ts
{
  key: "status",
  label: "状态",
  type: "switch",
  value: 1,
  show: {
    search: true,
    table: true,
    form: true
  },
  rules: true,
  options: {
    switch: {
      activeValue: 1,
      inactiveValue: 0
    }
  }
}
```

组件再接收一组 API：

```ts
api: {
  list: params => getList(params),
  create: form => createItem(form),
  update: form => updateItem(form),
  delete: ({ items }) => deleteItems(items)
}
```

它内部统一完成：

1. 初始化查询参数和分页参数。
2. 请求列表并从响应中取出 `records`、`total`。
3. 根据字段配置渲染搜索区和表格列。
4. 打开新增、编辑、查看弹窗并初始化表单。
5. 校验和提交，成功后提示并刷新列表。
6. 通过插槽处理图片、复杂联动、特殊操作列等非标准内容。

参考项目还通过 `TFormConfig.setConfig()` 一次性设置了全局规则，例如：

- 控件统一大小。
- 请求分页字段使用 `current` 和 `size`。
- 响应列表字段使用 `records`。
- 表格行主键使用 `id`。
- 每页默认 20 条。
- Switch 的启用/禁用值和文字。
- 弹窗、表格、表单的默认布局。

这就是“页面直接调用库”后代码变少的原因：流程代码只写一次，业务页面只提供差异。

## 6. 适合当前项目的目标 API

建议最终页面写成下面这样：

```vue
<script setup lang="ts">
import { AdminCrud, defineCrud } from "@your-scope/admin-kit/crud";
import { rewardApi } from "@/api/reward";

type Reward = {
  id: number;
  name: string;
  rewardType: "cash" | "points";
  amount: number;
  status: 0 | 1;
};

const crud = defineCrud<Reward>({
  rowKey: "id",
  request: {
    list: rewardApi.list,
    create: rewardApi.create,
    update: rewardApi.update,
    remove: rewardApi.remove
  },
  fields: [
    {
      key: "name",
      label: "奖励名称",
      type: "input",
      search: true,
      table: { minWidth: 160 },
      form: { span: 24 },
      required: true
    },
    {
      key: "rewardType",
      label: "奖励类型",
      type: "select",
      search: true,
      options: [
        { label: "现金", value: "cash" },
        { label: "积分", value: "points" }
      ]
    },
    {
      key: "amount",
      label: "奖励金额",
      type: "number",
      defaultValue: 0
    },
    {
      key: "status",
      label: "状态",
      type: "switch",
      search: true,
      defaultValue: 1
    }
  ]
});
</script>

<template>
  <AdminCrud :config="crud">
    <template #table-name="{ row }">
      <el-link type="primary">{{ row.name }}</el-link>
    </template>
  </AdminCrud>
</template>
```

这里要保留两个逃生口：

- **插槽**：解决图片、组合控件、特殊操作列等复杂 UI。
- **生命周期钩子**：解决打开前加载详情、提交前转换数据、提交后联动等复杂流程。

不要追求所有页面都做到零模板、零函数。能覆盖 70%～85% 的普通后台页面，就已经有很高收益；特殊页面继续使用原生 Vue 和 Element Plus。

## 7. 建议的数据和类型设计

### 7.1 不直接绑定后端响应格式

不同后端可能返回：

```ts
{ data: { records: [], total: 0 } }
```

也可能返回：

```ts
{ list: [], count: 0 }
```

通过适配器统一成组件库内部格式：

```ts
export type PageResult<T> = {
  items: T[];
  total: number;
};

export type CrudRequest<T, Query, CreateDto, UpdateDto> = {
  list(query: Query): Promise<unknown>;
  create?(data: CreateDto): Promise<unknown>;
  update?(data: UpdateDto): Promise<unknown>;
  remove?(rows: T[]): Promise<unknown>;
};

export type ResponseAdapter<T> = {
  toPageResult(response: unknown): PageResult<T>;
  isSuccess(response: unknown): boolean;
  getMessage(response: unknown): string;
};
```

全局注册默认适配器，个别页面可以覆盖。这样换后端字段时不需要修改组件源码。

### 7.2 字段类型用可辨识联合，少用 `any`

```ts
type BaseField<T> = {
  key: keyof T & string;
  label: string;
  search?: boolean;
  table?: false | TableFieldOptions;
  form?: false | FormFieldOptions;
};

type InputField<T> = BaseField<T> & {
  type: "input";
  props?: InputProps;
};

type SelectField<T> = BaseField<T> & {
  type: "select";
  options: Option[] | (() => Promise<Option[]>);
};

type CrudField<T> =
  InputField<T> | SelectField<T> | NumberField<T> | SwitchField<T>;
```

这样配置写错时编辑器能直接提示，而不是运行后才发现。

### 7.3 权限由业务传入

组件库只定义能力，不读取当前项目的路由 Store：

```ts
permissions: {
  create: hasAuth("reward:create"),
  update: row => hasAuth("reward:update") && row.status === 0,
  remove: hasAuth("reward:delete")
}
```

动态菜单和动态路由属于应用壳层，不要耦合进第一版 CRUD 包。以后确实有多个项目共用时，可以单独增加 `router` 模块，用适配器接收后端菜单并转换为 Vue Router 路由。

## 8. 全局小号样式怎么做

当前需要的“小号弹窗、间隔窄、label 不带冒号”应该做成全局默认值，但允许页面覆盖：

```ts
app.use(AdminKit, {
  density: "compact",
  form: {
    size: "small",
    labelWidth: 116,
    labelSuffix: "",
    rowGap: 8,
    columnGap: 10
  },
  dialog: {
    width: 780,
    bodyPadding: 12,
    footerPadding: 10
  },
  table: {
    size: "small",
    pageSize: 20
  }
});
```

CSS 使用变量，避免在每个页面复制选择器：

```css
:root {
  --admin-page-gap: 12px;
  --admin-dialog-body-padding: 12px 16px;
  --admin-form-row-gap: 8px;
  --admin-form-column-gap: 10px;
  --admin-control-height: 28px;
}
```

组件内部使用命名空间，例如 `.admin-kit__form`，不要直接全局覆盖所有 `.el-form-item`，否则会误伤登录页和其他第三方组件。

## 9. 推荐工程结构

第一版可以直接在当前仓库中作为 workspace 包开发，稳定后再移动到独立仓库：

```text
vue-pure-admin/
├─ packages/
│  └─ admin-kit/
│     ├─ src/
│     │  ├─ core/
│     │  │  ├─ types.ts
│     │  │  ├─ response-adapter.ts
│     │  │  └─ defaults.ts
│     │  ├─ form/
│     │  │  ├─ AdminForm.vue
│     │  │  ├─ AdminFormField.vue
│     │  │  └─ field-registry.ts
│     │  ├─ dialog/
│     │  │  └─ AdminFormDialog.vue
│     │  ├─ crud/
│     │  │  ├─ AdminCrud.vue
│     │  │  ├─ AdminSearch.vue
│     │  │  ├─ AdminTable.vue
│     │  │  └─ useCrud.ts
│     │  ├─ styles/
│     │  │  ├─ tokens.css
│     │  │  └─ compact.css
│     │  ├─ plugin.ts
│     │  └─ index.ts
│     ├─ package.json
│     ├─ tsconfig.json
│     └─ vite.config.ts
├─ src/
│  └─ views/schema-form/       # 演示和业务验证页面
└─ pnpm-workspace.yaml
```

`pnpm-workspace.yaml` 增加：

```yaml
packages:
  - packages/*
```

业务项目依赖写为：

```json
{
  "dependencies": {
    "@your-scope/admin-kit": "workspace:*"
  }
}
```

第一阶段使用 workspace 就够了，不必为了测试每次发布 npm。

## 10. 分阶段实施步骤

### 阶段 0：盘点重复代码，冻结模板（1～2 天）

1. 选 3 个有代表性的页面：普通列表、带新增编辑的列表、含复杂自定义字段的列表。
2. 标记重复部分：查询状态、分页、加载、表格工具栏、打开弹窗、校验、提交、刷新。
3. 标记不可抽象部分：复杂联动、特殊权限、业务数据转换、自定义单元格。
4. 以当前 `/form/index` 中的“紧凑弹框”和“紧凑列表”为视觉基准，冻结间距、字号、控件高度、弹窗宽度。

验收：得到字段清单、交互清单和至少 3 个真实页面样本。

### 阶段 1：搭建包和演示环境（1 天）

1. 新建 `packages/admin-kit`。
2. 配置 Vite library mode、TypeScript 声明文件和 CSS 输出。
3. 把 `vue`、`element-plus`、`@pureadmin/table` 放入 `peerDependencies` 或构建 external。
4. 配置 `exports`：主入口、`./crud`、`./form`、`./styles.css`。
5. 当前项目通过 `workspace:*` 引用它。

验收：业务项目能导入一个测试按钮或空组件，并且构建时没有重复打包 Vue。

### 阶段 2：先做配置式表单和紧凑弹窗（2～3 天）

1. 定义 `FieldSchema` 和 `FormSchema` 类型。
2. 支持第一批字段：input、select、number、switch、radio、date/datetime。
3. 实现 `AdminFormField` 字段分发器。
4. 实现两列/整行栅格、分组标题、必填校验、label 无冒号。
5. 实现 `AdminFormDialog`，统一打开、关闭、提交、loading。
6. 加入 `form-{key}` 插槽和自定义字段注册表。

验收：用配置复刻当前紧凑弹窗，页面中不再手写每个 `el-form-item`。

### 阶段 3：实现 CRUD 主链路（3～5 天）

1. 实现 `useCrud`：query、loading、rows、pagination、selection、load、reset、refresh。
2. 实现 `AdminSearch`、`AdminTable` 和分页。
3. 复用同一份字段 schema 生成搜索字段、表格列和表单字段。
4. 接入新增、编辑、删除和查看弹窗。
5. 支持操作列、批量删除、状态切换和确认弹窗。
6. 加入响应适配器和请求前后钩子。

验收：普通 CRUD 页面只需要 API、字段配置和少量插槽。

### 阶段 4：处理真实业务差异（3～5 天）

1. 动态 options 和缓存，例如会员等级、分组、字典。
2. 字段联动显示/禁用。
3. `openBefore`、`submitBefore`、`submitAfter` 等钩子。
4. 权限适配。
5. 图片、上传、多语言等自定义控件注册。
6. 后端排序、分页字段、成功状态和错误信息适配。

验收：至少迁移 3 个真实页面，其中 1 个包含复杂字段。

### 阶段 5：质量和发布（3～5 天）

1. 使用 Vitest 测试 composable、数据适配和字段转换。
2. 使用组件测试覆盖加载、空数据、失败、校验、提交、删除确认。
3. 建立独立 playground 或 Storybook/Histoire 风格的组件示例。
4. 生成 `.d.ts`，执行构建和类型检查。
5. 加入 Changesets 或等价版本发布流程。
6. 发布到公司私有 npm，或继续使用 Git/workspace 依赖。

验收：新项目只需安装、引入样式、`app.use()` 三步即可使用。

## 11. 安装和调用的最终形态

发布后，业务项目的接入方式应保持简单：

```bash
pnpm add @your-scope/admin-kit
```

```ts
// src/main.ts
import { createApp } from "vue";
import AdminKit from "@your-scope/admin-kit";
import "@your-scope/admin-kit/styles.css";

const app = createApp(App);

app.use(AdminKit, {
  density: "compact",
  responseAdapter,
  page: {
    currentField: "current",
    sizeField: "size",
    defaultSize: 20
  }
});
```

之后每个页面使用 `AdminCrud` 或按需导入，不需要再复制全局样式和通用流程。

## 12. 第一版明确做什么、不做什么

### 第一版必须做

- 搜索、重置、刷新。
- 表格、分页、loading、空数据。
- 新增、编辑、查看、删除。
- 紧凑表单、label 无冒号。
- 输入、数字、选择、开关、单选、日期。
- 全局默认配置和单页覆盖。
- 插槽、自定义字段注册、请求适配。
- TypeScript 类型和基础测试。

### 第一版暂缓

- 自己重写按钮、输入框、Select 等基础 UI。
- 动态路由和后端菜单。
- Excel 导出、拖拽排序、行内编辑。
- 富文本、复杂上传、多语言编辑器内置实现。
- `cc1-node` 一类 Node 工具包。
- 同时兼容 Element Plus 之外的 UI 框架。

先把 80% 高频 CRUD 场景做稳定，再根据真实需求添加能力。

## 13. 容易踩的坑

1. **把业务规则写进组件库。** 例如组件库直接读取某个项目的 Pinia、路由、token 或接口文件，会导致另一个项目无法使用。
2. **过度配置化。** schema 比原来的 Vue 模板更难读时，抽象已经失去价值；复杂区域应使用插槽。
3. **大量使用 `any`。** 短期写得快，长期配置改名无法发现，组件库收益会被类型问题抵消。
4. **直接透传全部 Element Plus 属性但不做边界。** 对外 API 会被底层组件版本锁死，应只稳定承诺常用字段，额外属性放 `props`。
5. **全局 CSS 污染。** 样式必须有命名空间或作用域。
6. **把 Vue 和 Element Plus打进库。** 应使用 peer dependency 和 external，避免重复实例及包体积膨胀。
7. **只做演示，不迁移真实页面。** 没有复杂业务验证的组件库往往第二个页面就需要重写。
8. **一次性复制 `cc1-form` 的全部 API。** 参考它的思想和公开行为即可；先根据当前项目重新设计更小、更清晰的 API。

## 14. 迁移策略

不要一次重构所有页面，采用“新页面优先、旧页面顺手迁移”：

1. 用当前 `/form/index` 的两个示例确定视觉和 API。
2. 抽出库后先迁移一个普通列表页。
3. 再迁移一个带编辑弹窗的页面。
4. 最后迁移一个复杂页面，补齐插槽和钩子。
5. 三类页面稳定后发布 `0.1.0`。
6. 后续每迁移 3～5 个页面，总结一次共性；同一需求出现至少两次再进入库。

建议统计迁移前后的代码量和重复逻辑。一个普通 CRUD 页面如果能从数百行模板与状态代码收敛到约 80～150 行业务配置，说明抽象方向基本正确。

## 15. 推荐的版本路线

- `0.1.0`：紧凑表单、弹窗、静态表格和分页。
- `0.2.0`：真实 CRUD 请求、响应适配、生命周期钩子。
- `0.3.0`：权限、动态选项、自定义字段、3 个真实页面落地。
- `0.5.0`：测试、文档、主题变量、5～10 个页面迁移。
- `1.0.0`：API 稳定、变更策略明确、至少两个项目使用。

版本升级遵循语义化版本：修复用 patch，向后兼容功能用 minor，破坏调用方式用 major。

## 16. 下一步执行清单

第一周建议只做下面这些：

- [ ] 确认包名和是否放在当前仓库 workspace。
- [ ] 选定 3 个真实业务页面作为验收样本。
- [ ] 固定紧凑模式的间距、弹窗宽度、控件尺寸、label 样式。
- [ ] 新建 `packages/admin-kit` 和 playground。
- [ ] 完成 `FieldSchema`、字段渲染器、`AdminForm`。
- [ ] 完成 `AdminFormDialog`，复刻当前紧凑弹窗。
- [ ] 完成最小 `useCrud` 和列表请求。
- [ ] 周末评审 API；不满意时在发布前调整，此时修改成本最低。

如果目标只是当前一个项目减少重复，workspace 单包就是最佳起点；如果明确要供多个完全独立项目使用，则在 `0.3.0` 之后迁移到独立 Git 仓库并发布私有 npm 包。

## 17. 本次参考依据

- `/Users/code/game_admin/node_modules/cc1-form/package.json`
- `/Users/code/game_admin/node_modules/cc1-form/dist/index.d.ts`
- `/Users/code/game_admin/node_modules/cc1-form/dist/components/TCurd/indexType.d.ts`
- `/Users/code/game_admin/node_modules/cc1-form/dist/utils/TFormConfig.d.ts`
- `/Users/code/game_admin/node_modules/cc1-js/dist/index.d.ts`
- `/Users/code/game_admin/node_modules/cc1-vue3/dist/index.d.ts`
- `/Users/code/game_admin/node_modules/cc1-ui/dist/index.d.ts`
- `/Users/code/game_admin/node_modules/cc1-node/dist/index.d.ts`
- `/Users/code/game_admin/src/plugins/index.js`
- `/Users/code/game_admin` 中各业务页面对 `t-curd` 的实际调用
- 当前项目 `/form/index` 下的紧凑弹窗与紧凑列表示例

## 18. 最快方案：复用 `cc1-form`，对外使用自己的包名

如果当前目标是尽快投入项目，可以先做一个自己的“门面包”，例如 `@your-scope/admin-kit`。业务项目只导入这个名字，门面包内部依赖和初始化 `cc1-form`。后续再逐步把内部实现替换为自己的组件，业务页面的调用名称不需要跟着修改。

> 注意：门面包只解决“业务代码使用自己的包名”，其底层依然会安装并执行 `cc1-*`。如果要求依赖树、源码和构建产物里也完全没有 `cc1-*`，应执行第 19 节的独立化方案。

### 18.1 为什么适合先包一层

- `cc1-form` 已经实现了搜索、表格、分页、表单、弹窗、CRUD、插槽和全局配置。
- 它的 `package.json` 声明为 MIT，Vue 和 Element Plus 使用 peer dependency。
- 当前项目的 Node 22、Vue 3.5 和 Element Plus 2.14 在它声明的版本范围内。
- 自己的门面包可以固定视觉风格、请求字段和响应字段。
- 以后替换底层实现时，业务页面继续使用 `AdminCrud` 或自己的包名。

但不能只复制 `cc1-form/dist` 后改一个名字：它运行时还引用 `cc1-vue3`，内部使用的 `Timer`、`StrUtil`、`ObjectUtil` 等全局工具由 `cc1-js` 的副作用导入提供。最小运行依赖至少包括：

```text
cc1-form
├─ cc1-vue3
├─ cc1-js（需要在入口优先执行 import "cc1-js"）
├─ vue
├─ vue-router
└─ element-plus
```

`cc1-ui` 不是 `cc1-form` 当前构建文件的直接运行依赖，只有确实要复用它的 Button、Icon、Mask 等组件时再接入。`cc1-node` 只放在 Node/Vite 构建侧，普通页面不需要。

### 18.2 推荐的门面包入口

```ts
// packages/admin-kit/src/index.ts
import "cc1-js";
import cc1Form, {
  TFormConfig,
  TSys,
  type CurdOption,
  type curdConfType
} from "cc1-form";
import "cc1-form/index.css";
import type { App } from "vue";
import type { Router } from "vue-router";

export type AdminCrudOption<T = unknown> = CurdOption<T>;
export type AdminCrudRef = curdConfType;

export type AdminKitOptions = {
  router: Router;
  response?: {
    listField?: string;
    totalField?: string;
  };
};

export const createAdminKit = (options: AdminKitOptions) => ({
  install(app: App) {
    TSys.router = options.router;

    TFormConfig.setConfig({
      size: {
        table: "small",
        form: "small",
        search: "small"
      },
      dialog: {
        width: "780px",
        closeOnClickModal: false
      },
      form: {
        labelWidth: "116px"
      },
      field: {
        page: {
          num: "current",
          size: "size"
        },
        result: {
          list: options.response?.listField ?? "records",
          total: options.response?.totalField ?? "total"
        }
      },
      pagination: {
        size: 20,
        pageSizes: [20, 50, 100]
      },
      table: {
        rowKey: "id"
      }
    });

    app.use(cc1Form);
  }
});

export {
  TCurd as AdminCrud,
  TFormList as AdminFormList,
  TColumn as AdminFormColumn
} from "cc1-form";
```

应用中只出现自己的命名：

```ts
// src/main.ts
import { createAdminKit } from "@your-scope/admin-kit";
import router from "./router";

app.use(
  createAdminKit({
    router,
    response: {
      listField: "records",
      totalField: "total"
    }
  })
);
```

```vue
<script setup lang="ts">
import { AdminCrud, type AdminCrudOption } from "@your-scope/admin-kit";

const option: AdminCrudOption = {
  api: {
    list: params => rewardApi.list(params)
  },
  column: [
    {
      key: "rewardName",
      label: "奖励名称",
      type: "input",
      show: { search: true, table: true, form: true }
    }
  ]
};
</script>

<template>
  <AdminCrud :option="option" />
</template>
```

### 18.3 包依赖建议

门面包自身使用固定版本，不使用 `*`：

```json
{
  "name": "@your-scope/admin-kit",
  "version": "0.1.0",
  "type": "module",
  "dependencies": {
    "cc1-form": "1.4.7",
    "cc1-js": "1.0.9",
    "cc1-vue3": "1.1.0"
  },
  "peerDependencies": {
    "element-plus": "^2.9.0",
    "vue": "^3.5.0",
    "vue-router": "^4.0.0 || ^5.0.0"
  }
}
```

上面的 `cc1-form` 版本只是按参考项目锁文件举例，正式接入时应选择一个实际安装并通过测试的版本，然后精确锁定。当前参考目录存在版本漂移：`game_admin/package.json` 声明 `1.4.8`，`package-lock.json` 记录 `1.4.7`，本地 `node_modules/cc1-form/package.json` 显示 `1.4.6`。因此不要复制现有 `node_modules` 作为正式来源。

### 18.4 三种复用方式与时间

| 方式                          | 首次可用  | 稳定接入       | 说明                                                 |
| ----------------------------- | --------- | -------------- | ---------------------------------------------------- |
| 业务项目直接安装并使用原名    | 0.5～1 天 | 2～4 天        | 最快，但页面直接依赖 `cc1-*` API                     |
| 创建自己的门面包并重命名导出  | 1～2 天   | 4～7 天        | 最推荐；能加全局主题、适配器并逐步替换底层           |
| 拿到完整源码后 Fork、批量改名 | 3～5 天   | 1～2 周        | 适合确定长期维护且能取得源码的情况                   |
| 只拿 `dist` 复制后批量改名    | 2～4 天   | 后续成本不可控 | 能短期跑起来，但调试、升级、类型和依赖修复都很费时间 |

与从零完成 CRUD MVP 的 8～12 个工作日相比，门面包方案通常可以缩短到 4～7 个工作日，节省约 40%～65%。与完整自研内部 v1 的 3～5 周相比，先复用再逐步替换，第一阶段通常可以节省约 2～3 周。

### 18.5 推荐迁移顺序

1. 先在临时分支直接安装精确版本的 `cc1-form`、`cc1-js`、`cc1-vue3`。
2. 在当前 `/form/index` 新增一个真实 `TCurd` 测试页，验证 Vue 3.5、Vue Router、Element Plus 和 CSS。
3. 验证列表请求、分页响应、弹窗、表单校验、Switch 和插槽。
4. 确认运行稳定后建立 `@your-scope/admin-kit` 门面包。
5. 业务页面改用 `AdminCrud` 等自己的导出名称。
6. 在门面包内增加紧凑主题和后端响应适配。
7. 后续优先替换最需要修改的底层模块，不必一次重写全部功能。

这条路线的关键是“自己的 API 稳定、底层实现可替换”。包名改掉只是表面，真正能保护后续维护成本的是门面层和适配器。

## 19. 完全独立方案：安装、源码和构建产物都是自己的

“完全变成自己的”应满足以下验收条件：

- 业务项目执行 `pnpm add @your-scope/admin-kit`，只使用自己的包名。
- `pnpm why cc1-form cc1-js cc1-vue3 cc1-ui` 查不到运行依赖。
- 组件库仓库中有可维护的 `.vue`、`.ts`、样式和测试源码。
- 构建产物由自己的源码生成，不是直接把 `cc1-form.js` 改文件名。
- 全局组件名、CSS 命名空间、类型名和文档全部使用自己的品牌。

### 19.1 当前已有材料能做到什么

目前 `/Users/code/game_admin/node_modules/cc1-*` 中包含：

- 构建后的 JavaScript。
- 构建后的 CSS。
- TypeScript 声明文件 `.d.ts`。
- `package.json` 和少量 README/LICENSE 文件。

其中没有组件的 `.vue/.ts` 原始源码，也没有 source map。因此当前材料适合用来：

- 了解公开 API 和配置结构。
- 对照验证界面和行为。
- 编写兼容层和迁移测试。
- 作为重新实现时的功能清单。

它不适合作为长期源码仓库。压缩后的单文件代码即使格式化，也已经丢失原始组件边界、变量名、注释和构建结构，后续维护成本接近重新开发。

### 19.2 路线 A：能取得原始源码时

这是最快的完全独立路线。

1. 取得 `cc1-form`、`cc1-js`、`cc1-vue3`、`cc1-ui`、`cc1-node` 的原始 Git 仓库或源码压缩包。
2. 确认源码版本和当前业务实际使用版本一致，不要混用 `1.4.6/1.4.7/1.4.8`。
3. 新建自己的 monorepo，不在 `node_modules` 中开发。
4. 保留适用的原始许可证与版权声明；现有包清单声明为 MIT。
5. 先让原始源码在新仓库中完整构建和通过测试，建立迁移基线。
6. 修改 package scope 和内部依赖名。
7. 修改组件、类型、全局变量、CSS 命名空间和自动导入代码。
8. 去掉所有 `cc1-*` 运行依赖并执行依赖扫描。
9. 在当前 PureAdmin 项目的 3 个页面中做回归测试。
10. 发布自己的 `0.1.0` 版本。

推荐包名映射：

| 原包       | 自己的包                 |
| ---------- | ------------------------ |
| `cc1-js`   | `@your-scope/admin-core` |
| `cc1-vue3` | `@your-scope/admin-vue`  |
| `cc1-ui`   | `@your-scope/admin-ui`   |
| `cc1-form` | `@your-scope/admin-crud` |
| `cc1-node` | `@your-scope/admin-node` |

内部依赖也必须修改，例如：

```diff
- import { Scope } from "cc1-vue3";
+ import { Scope } from "@your-scope/admin-vue";
```

```diff
- import "cc1-form/index.css";
+ import "@your-scope/admin-crud/index.css";
```

### 19.3 路线 B：拿不到原始源码时

按公开类型和当前业务模板重新实现，先保证自己的 API 稳定，不追求一次覆盖全部 `cc1` 能力。

建议顺序：

1. 以 `indexType.d.ts` 为功能清单，但重新设计更小的 `AdminCrudOption`。
2. 从当前已经做好的紧凑列表和紧凑弹窗提取视觉组件。
3. 先实现 `admin-core`：类型、分页、响应适配、深拷贝等必要纯函数。
4. 实现 `admin-crud`：字段渲染、搜索、表格、分页、弹窗、增删改查。
5. 使用 Vue composable 代替旧式全局工具和隐式变量。
6. 特殊组件通过 registry 和 slot 扩展，不塞进 CRUD 核心。
7. 等真实页面提出需求后，再实现 `admin-ui` 和 `admin-vue`。
8. `admin-node` 最后做，且只在确有构建任务复用时创建。

这一条路线的目标不是逐行还原压缩代码，而是实现相同的业务能力，并让类型、依赖和维护方式更适合当前项目。

### 19.4 独立 monorepo 目录

```text
your-admin-kit/
├─ packages/
│  ├─ core/
│  ├─ vue/
│  ├─ ui/
│  ├─ crud/
│  └─ node/
├─ playgrounds/
│  └─ pure-admin-demo/
├─ tests/
├─ package.json
├─ pnpm-workspace.yaml
├─ tsconfig.base.json
└─ LICENSE
```

第一阶段只创建 `core` 和 `crud`，其余目录按实际需求增加。

### 19.5 从源码 Fork 后需要改哪些位置

不能只改根目录 `package.json` 的 `name`，至少检查这些位置：

1. 所有包的 `name`、`repository`、`exports`、`main`、`module`、`types`。
2. 所有源码中的 `from "cc1-*"` 和动态 `import("cc1-*")`。
3. 自动导入插件里写死的包名；`cc1-ui/dist/autoimport.js` 中存在 `cc1-ui` 字符串。
4. Vue 全局组件名，例如 `TCurd`、`TColumn`、`TFormList`。
5. 对外类型名，例如 `CurdOption`、`curdConfType`。
6. CSS 类名前缀、CSS 变量和构建后的 CSS 文件名。
7. UMD 全局变量名和 Vite library `name`。
8. README、示例、错误提示和注释中的旧名称。
9. 包之间的 peer dependency/dependency 名称。
10. License、NOTICE、CHANGELOG 和版本发布配置。

建议新的公开名称使用正确的 `Crud` 顺序：

```text
TCurd       -> AdminCrud
CurdOption  -> AdminCrudOptions
curdConfType -> AdminCrudExpose
TFormConfig -> AdminKitConfig
TSys        -> AdminRuntime
```

先提供一层兼容别名，旧页面迁移完成后再删除旧名字。

### 19.6 去掉隐式全局变量

现有 `cc1-form` 构建文件中会直接使用 `Timer`、`StrUtil`、`ObjectUtil`，参考项目靠入口处的 `import "cc1-js"` 把这些对象放到全局。自己的实现应改成明确导入：

```ts
import { createTimer, deepMerge, createId } from "@your-scope/admin-core";
```

这样组件依赖能被 TypeScript 和构建工具追踪，也不会因为入口导入顺序不同而报错。

### 19.7 构建配置重点

`vue`、`vue-router` 和 `element-plus` 留给业务项目提供：

```ts
// vite.config.ts
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
  plugins: [vue()],
  build: {
    lib: {
      entry: "src/index.ts",
      name: "YourAdminKit",
      formats: ["es"],
      fileName: "index"
    },
    rollupOptions: {
      external: ["vue", "vue-router", "element-plus"]
    }
  }
});
```

每个包至少输出：

```text
dist/index.js
dist/index.d.ts
dist/index.css（有样式时）
```

### 19.8 发布步骤

1. 在 npm、公司私有 npm、GitHub Packages 或其他私有仓库创建自己的 scope。
2. 配置仓库级 `.npmrc`，token 放环境变量，不提交到 Git。
3. 先发布 `0.1.0-alpha.1`。
4. 在 PureAdmin playground 安装 alpha 版本并完成构建、类型和页面测试。
5. 修复后发布 `0.1.0`。
6. 业务项目执行：

```bash
pnpm add @your-scope/admin-crud
```

7. 验证依赖树：

```bash
pnpm why cc1-form cc1-js cc1-vue3 cc1-ui
```

预期没有输出相关运行依赖。

### 19.9 完全独立后的时间

| 前提                                     | 第一版可用       | 比从零节省                                     |
| ---------------------------------------- | ---------------- | ---------------------------------------------- |
| 有五个包的完整、可构建源码               | 5～10 个工作日   | 约 50%～70%                                    |
| 只有 `cc1-form` 完整源码，其他包需要替换 | 2～3 周          | 约 30%～50%                                    |
| 只有当前 `dist` 和 `.d.ts`               | CRUD MVP 2～4 周 | 主要节省需求分析和 API 设计时间                |
| 完整重做五个包的全部能力                 | 6～10 周         | 参考包主要帮助减少试错，不会把实现工期压到几天 |

如果你手里的“模板”包含真正的 `.vue/.ts` 源码，工期可以从原先的 3～5 周压缩到约 1～2 周；如果模板只是业务页面和 `node_modules`，比较稳妥的估算仍是 2～4 周完成独立 CRUD MVP。

### 19.10 最推荐的执行决策

1. 先找原始源码，限定半天时间。
2. 找到源码：走 Fork 独立化，第一周完成改名、构建和基础回归。
3. 没找到源码：直接重写 `admin-core + admin-crud`，不要花数天整理压缩后的 `dist`。
4. 第一版只覆盖当前紧凑列表、紧凑弹窗和常用 CRUD。
5. 用 3 个真实页面验收后再发布自己的包。

这样最终安装的是自己的包，依赖树中没有 `cc1-*`，并且后续每一行核心代码都能正常维护。
