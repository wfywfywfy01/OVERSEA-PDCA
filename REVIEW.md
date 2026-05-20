# M9 CRM index.html 代码审查

## 审查总览
- 文件：`index.html`（~3300 行，含 ~1200 行 JS）
- 阻塞问题：6 处
- 建议改进：7 处
- 讨论项：2 项

---

## [阻塞] 必须修复

### 1. index.html:2267 — JSON.parse 无错误保护

**问题**：`initStorage()` 直接调用 `JSON.parse(raw)`，若 localStorage 数据损坏或被截断，会抛出未捕获异常，页面白屏。

**修改方式**：用 try/catch 包裹，捕获异常后清空缓存并降级注入 mock 数据：

```js
// 当前代码（有问题）
const d = JSON.parse(raw);
this.users = d.users;
// ...

// 改为
try {
  const d = JSON.parse(raw);
  this.users         = d.users;
  this.customers     = d.customers;
  this.interactions  = d.interactions;
  this.receivables   = d.receivables;
  this.managerTodos  = d.managerTodos  || [];
  this.superiorTodos = d.superiorTodos || this._defaultSuperiorTodos();
  this.storageStatus = '当前数据已存入 localStorage（从缓存恢复）';
} catch {
  localStorage.removeItem('m9_data');
  const mock = this._mockData();
  this.users         = mock.users;
  this.customers     = mock.customers;
  this.interactions  = mock.interactions;
  this.receivables   = mock.receivables;
  this.superiorTodos = this._defaultSuperiorTodos();
  this.managerTodos  = [];
  this._save();
  this.storageStatus = '缓存数据损坏，已重置为 Mock 数据';
}
```

---

### 2. index.html:3086 — getLatestInteraction 返回最早记录而非最新（Bug）

**问题**：`getCustomerInteractions` 按 `b.date.localeCompare(a.date)` 降序排列，`list[0]` 是最新记录，`list[list.length - 1]` 是最旧记录。函数名是 `getLatestInteraction`，但当前返回的是最旧的一条。

**修改方式**：

```js
// 当前代码（有问题）
getLatestInteraction(customerId) {
  const list = this.getCustomerInteractions(customerId);
  return list.length > 0 ? list[list.length - 1] : null;
},

// 改为
getLatestInteraction(customerId) {
  const list = this.getCustomerInteractions(customerId);
  return list.length > 0 ? list[0] : null;
},
```

---

### 3. index.html:3269-3270 — getConversionRate 使用了回复率阈值（copy-paste bug）

**问题**：`getConversionRate` 中判断红/黄状态时用了 `this.rules.responseRateRed` 和 `this.rules.responseRateYellow`，这是回复率的阈值字段，不是转化率的，语义错误且无法分别配置。

**修改方式**：

第一步，在 `defaultRules` 中增加转化率专属阈值：
```js
defaultRules: {
  newbieDays:           30,
  newbieTaskCount:      5,
  veteranTaskCount:     2,
  responseRateRed:      30,
  responseRateYellow:   50,
  conversionRateRed:    30,    // 新增
  conversionRateYellow: 50,    // 新增
  receivableYellowLine: 7,
},
```

第二步，修改 `getConversionRate` 中的引用：
```js
// 当前代码（有问题）
const status = rate < this.rules.responseRateRed ? 'red' : rate < this.rules.responseRateYellow ? 'yellow' : 'green';

// 改为
const status = rate < this.rules.conversionRateRed ? 'red' : rate < this.rules.conversionRateYellow ? 'yellow' : 'green';
```

---

### 4. index.html:2472–2796 — getTodayTasks 与 _getTasksForUser 业务逻辑不一致

**问题**：两个函数都计算"某销售员今日任务"，但逻辑不同：
- `getTodayTasks`：所有人都拿 S 类客户 + Potential（新人多老人少）
- `_getTasksForUser`：新人只拿 Potential，老人只拿 S 类（互斥）

主管看板的触达进度调用 `_getTasksForUser`，与销售员自己看到的任务数不一致，导致进度统计错误。

**修改方式**：统一逻辑，`getTodayTasks` 直接委托 `_getTasksForUser`：

```js
getTodayTasks() {
  if (!this.currentUser || this.currentUser.role === 'director') return [];
  return this._getTasksForUser(this.currentUser);
},
```

同时将 `_getTasksForUser` 中的业务逻辑对齐到正确版本（由团队确认新人/老人规则后统一到此处）。

---

### 5. index.html:3096–3195 — getHermesMsgs 函数超 99 行，违反函数长度规则

**问题**：单函数处理 4 个 tab 的所有洞察逻辑，99 行，远超 50 行上限。

**修改方式**：拆分为 4 个私有方法：

```js
getHermesMsgs(tab) {
  if (!this.currentUser) return [];
  const dispatch = {
    tasks:       () => this._hermesTasksMsgs(),
    funnel:      () => this._hermesFunnelMsgs(),
    receivables: () => this._hermesReceivablesMsgs(),
    manager:     () => this._hermesManagerMsgs(),
  };
  return dispatch[tab]?.() ?? [];
},

_hermesTasksMsgs() {
  const msgs = [];
  // 把原来 if (tab === 'tasks') { ... } 里的内容移到这里
  return msgs;
},

_hermesFunnelMsgs() {
  const msgs = [];
  // 把原来 if (tab === 'funnel') { ... } 里的内容移到这里
  return msgs;
},

_hermesReceivablesMsgs() {
  const msgs = [];
  // 把原来 if (tab === 'receivables') { ... } 里的内容移到这里
  return msgs;
},

_hermesManagerMsgs() {
  const msgs = [];
  // 把原来 if (tab === 'manager') { ... } 里的内容移到这里
  return msgs;
},
```

---

### 6. index.html:2741 — getTeamName 硬编码回退，第三组会返回错误结果

**问题**：`return u.teamId === 'team_a' ? 'A组' : 'B组'` 对任何非 `team_a` 的组都返回 `'B组'`，扩展性差。

**修改方式**：

```js
getTeamName(userId) {
  const u = this.users.find(x => x.id === userId);
  if (!u || !u.teamId) return '';
  const teamNameMap = { team_a: 'A组', team_b: 'B组' };
  return teamNameMap[u.teamId] ?? u.teamId;
},
```

---

## [建议] 推荐改进

### 1. index.html:7-9 — CDN 依赖无版本锁定，无 SRI 校验

`alpinejs@3.x.x` 使用浮动版本，CDN 内容可能悄然变化，存在供应链风险。

**修改方式**：固定版本号，并添加 integrity 属性（SRI hash 从 cdn.jsdelivr.net 获取）：
```html
<script defer
  src="https://cdn.jsdelivr.net/npm/alpinejs@3.14.1/dist/cdn.min.js"
  integrity="sha384-XXXXXXXXXXXXXXXX"
  crossorigin="anonymous"></script>
```

---

### 2. index.html:2577,2629,2644,2909 — ID 生成存在碰撞风险

`'I'+Date.now()` 在同一毫秒内调用两次即产生重复 ID。

**修改方式**：增加一个统一的 ID 生成辅助函数：
```js
_genId(prefix = '') {
  return prefix + Date.now().toString(36) + Math.random().toString(36).slice(2, 7);
},
```

所有 `'I'+Date.now()`、`'R'+Date.now()`、`'T'+Date.now()` 统一替换为：
```js
id: this._genId('I'),
id: this._genId('R'),
id: this._genId('T'),
```

---

### 3. index.html:3177-3180 — getHermesMsgs manager tab 存在 O(n×m) 重复计算

`getSalesUsers().filter(u => getReceivablesByStatusAll('red').some(...))` 在每次渲染都执行，且 `getReceivablesByStatusAll` 在内层循环里被反复调用。

**修改方式**：在函数开头缓存一次：
```js
_hermesManagerMsgs() {
  const msgs      = [];
  const redList   = this.getReceivablesByStatusAll('red');
  const redOwners = new Set(redList.map(r => this.getCustomerOwner(r.customerId)));
  const anomalyRed = this.getSalesUsers().filter(u =>
    redOwners.has(u.id) ||
    this.getResponseRate(u.id).status === 'red' ||
    this.getConversionRate(u.id).status === 'red'
  ).length;
  // ...
},
```

---

### 4. index.html:2906 — confirmImport 中冗余二次过滤

`this.importModal.parsed.filter(r => r.name)` 是多余的，`parsed` 已经通过 `.filter(Boolean)` 去除了 null 行。

**修改方式**：
```js
// 当前代码
const valid = this.importModal.parsed.filter(r => r.name);
valid.forEach(r => { ... });

// 改为
this.importModal.parsed.forEach(r => { ... });
```

---

### 5. index.html:3073-3076 — getRelationshipTemp 魔术数字

`2`、`6`、`13` 天阈值散落在代码中，无法通过规则配置调整。

**修改方式**：提取为具名常量（或并入 `defaultRules`）：
```js
getRelationshipTemp(customer) {
  const HOT_DAYS  = 2;
  const WARM_DAYS = 6;
  const COOL_DAYS = 13;
  if (!customer.lastInteract) return { icon:'🥶', label:'从未触达', days:null, barCls:'bg-slate-200' };
  const days = Math.floor((new Date(this.todayStr()) - new Date(customer.lastInteract)) / 86400000);
  if (days <= HOT_DAYS)  return { icon:'🔥', label:`${days}天前`, days, barCls:'bg-red-400' };
  if (days <= WARM_DAYS) return { icon:'😊', label:`${days}天前`, days, barCls:'bg-amber-400' };
  if (days <= COOL_DAYS) return { icon:'😐', label:`${days}天前`, days, barCls:'bg-slate-300' };
  return { icon:'🥶', label:`${days}天前`, days, barCls:'bg-blue-200' };
},
```

---

### 6. index.html:2545 — getFunnelCustomers 包含无效角色 'manager'

`role === 'director' || role === 'manager'` 中的 `'manager'` 不是数据模型中的有效角色（有效值为 `director/team_lead/sales`），是残留死代码。

**修改方式**：删除 `|| role === 'manager'` 这个条件。

---

### 7. index.html:2933-2936 — downloadTemplate URL 释放时机有风险

`a.click()` 后立即 `URL.revokeObjectURL(url)` 在部分浏览器中可能导致下载失败。

**修改方式**：延迟释放：
```js
downloadTemplate() {
  const csv  = '客户名称,级别,来源,联系人,电话\n示例公司,A,市场活动,张总,13800000001\n';
  const blob = new Blob(['﻿' + csv], { type: 'text/csv;charset=utf-8;' });
  const url  = URL.createObjectURL(blob);
  const a    = document.createElement('a');
  a.href = url;
  a.download = 'M9名单导入模板.csv';
  a.click();
  setTimeout(() => URL.revokeObjectURL(url), 300);
},
```

---

## [讨论] 需团队决定

### 1. 所有状态和逻辑集中在单个 appData() 对象

当前 `appData()` 同时承担数据持久化、业务规则计算、UI 辅助、弹窗状态管理，共 70+ 方法。随功能增长，可维护性下降。可考虑按职责拆分，但需要团队对架构方向达成一致。

### 2. 客户数据明文存储在 localStorage

姓名、电话、应收金额等商业数据以 JSON 明文形式存在浏览器本地。需评估是否满足数据合规要求（如 GDPR / 国内数据安全法），以及是否需要迁移到服务端存储。

---

## 测试覆盖评估

- **现有测试**：无
- **最高优先级缺失测试**（按风险排序）：
  1. `getLatestInteraction` — 验证返回的是最新记录（当前有 Bug）
  2. `getReceivableStatus` — 红/黄/绿三色边界日期（今天、明天、第 N 天）
  3. `getTodayTasks` vs `_getTasksForUser` — 同一用户任务数一致性
  4. `getConversionRate` — 验证阈值与回复率相互独立
  5. `confirmImport` — 空行、非法 level、重复名称的 Excel 输入
