# TypeScript特殊符号

## ??=（空值合并赋值运算符）

`??=` 是逻辑空值合并赋值运算符（Logical Nullish Assignment），当变量的值为 `null` 或 `undefined` 时，才将右侧的值赋给它；否则保持原值不变。

### 基本语法

```typescript
a ??= b;
// 等价于：
a = a ?? b;      // 注意：并非 a === null || a === undefined 的完整写法，见下方说明
```

### 行为规则

| 变量当前值 | `??=` 右侧值 | 赋值结果 |
| ---------- | ------------- | -------- |
| `null`     | `42`          | `42`     |
| `undefined`| `"hello"`     | `"hello"`|
| `0`        | `99`          | `0`（不变）|
| `""`       | `"default"`   | `""`（不变）|
| `false`    | `true`        | `false`（不变）|

> **关键区别**：`??=` 只在 `null` / `undefined` 时赋值，与 `||=` 不同 —— `||=` 在所有假值（`0`、`""`、`false`、`null`、`undefined`）时都会赋值。

### 使用示例

```typescript
// 1. 基础用法
let count: number | undefined = undefined;
count ??= 10;
console.log(count); // 10

let name: string = "TypeScript";
name ??= "JavaScript";
console.log(name); // "TypeScript"（原值不是 null/undefined，不赋值）

// 2. 对象属性默认值初始化
interface Config {
    host?: string;
    port?: number;
    debug?: boolean;
}

function applyDefaults(config: Config): Config {
    config.host ??= "localhost";
    config.port ??= 3000;
    config.debug ??= false;
    return config;
}

const userConfig: Config = { port: 8080 };
const finalConfig = applyDefaults(userConfig);
console.log(finalConfig);
// { port: 8080, host: "localhost", debug: false }

// 3. 与 ||= 的对比
let zero = 0;
zero ??= 100;  // zero 仍为 0（?? 只判断 null/undefined）
zero ||= 100;  // zero 变为 100（|| 判断所有假值）

// 4. 惰性求值（右侧表达式仅在需要时才执行）
function expensive(): number {
    console.log("expensive() called");
    return 42;
}

let value = 1;
value ??= expensive(); // 不会调用 expensive()，因为 value 不是 null/undefined
```

### 惰性求值特性

`??=` 具有短路（惰性）特性：如果左侧不是 `null` / `undefined`，右侧表达式**不会被执行**。这在右侧是函数调用或有副作用的表达式时非常重要。

### 适用场景总结

1. **为可选属性提供默认值**：避免重复的 `if (x === null || x === undefined)` 判断
2. **配置对象初始化**：用户传入的配置中缺失的字段用默认值补全
3. **保留合法的假值**：当 `0`、`""`、`false` 是有效值时，使用 `??=` 而非 `||=`
4. **链式默认值**：多级对象属性安全赋值

```typescript
// 链式默认值示例
interface AppConfig {
    server?: {
        host?: string;
        timeout?: number;
    };
}

const app: AppConfig = { server: {} };
app.server ??= {};
app.server.host ??= "127.0.0.1";
app.server.timeout ??= 5000;
```

## ?.（可选链运算符）

`?.` 是可选链运算符（Optional Chaining），用于安全地访问可能为 `null` 或 `undefined` 的对象属性、方法或索引元素。当链中的某个引用为 `null` / `undefined` 时，表达式立即短路返回 `undefined`，而不会抛出 `TypeError`。

### 基本语法

```typescript
// 属性访问
obj?.prop;

// 方法调用
obj?.method(args);

// 索引访问
arr?.[index];
```

### 行为规则

| 左侧值 | 表达式 | 返回值 |
| ------ | ------ | ------ |
| `null` | `null?.prop` | `undefined` |
| `undefined` | `undefined?.prop` | `undefined` |
| 有效对象 | `{ a: 1 }?.a` | `1` |
| 有效对象 | `{ a: 1 }?.b` | `undefined` |
| 有效数组 | `[1,2]?.[0]` | `1` |

> **短路特性**：一旦链中某处为 `null` / `undefined`，后续所有属性访问、方法调用都会跳过，整个表达式返回 `undefined`。

### 使用示例

```typescript
// 1. 安全访问嵌套属性
interface User {
    name: string;
    address?: {
        city?: string;
        zip?: string;
    };
}

const user: User = { name: "张三" };
console.log(user.address?.city);     // undefined（不会报错）
console.log(user.address?.zip);      // undefined
// console.log(user.address.city);   // ❌ TypeError: Cannot read property 'city' of undefined

// 2. 安全调用可能不存在的方法
interface Service {
    connect?: () => void;
    disconnect?: () => void;
}

const svc: Service = { connect: () => console.log("connected") };
svc.connect?.();       // "connected"
svc.disconnect?.();    // 什么都不做（方法不存在，返回 undefined）

// 3. 安全访问数组元素
const items: string[] | undefined = undefined;
console.log(items?.[0]);  // undefined（不会报错）

const list = ["a", "b", "c"];
console.log(list?.[1]);   // "b"

// 4. 多层链式调用
interface DeepConfig {
    level1?: {
        level2?: {
            level3?: string;
        };
    };
}

const cfg: DeepConfig = {};
console.log(cfg.level1?.level2?.level3);  // undefined（在 level1 处短路）

const fullCfg: DeepConfig = { level1: { level2: { level3: "value" } } };
console.log(fullCfg.level1?.level2?.level3);  // "value"
```

### 与其他运算符配合

`?.` 常与 `??` 组合使用，在访问失败时提供默认值：

```typescript
const city = user.address?.city ?? "未知城市";
const port = config.server?.port ?? 3000;
```

### 适用场景总结

1. **访问可选属性**：接口中用 `?` 标记的可选属性，读取时用 `?.` 避免报错
2. **处理 API 返回数据**：外部数据可能缺少某些层级，用 `?.` 安全访问
3. **调用可选方法**：回调函数或策略模式中，方法可能未实现
4. **替代冗余的 `if` 判断**：简化多层嵌套的空值检查

```typescript
// 传统写法 vs 可选链
// 传统：
let result;
if (obj && obj.prop && obj.prop.value) {
    result = obj.prop.value;
}
// 可选链：
const result2 = obj?.prop?.value;
```

## ??（空值合并运算符）

`??` 是空值合并运算符（Nullish Coalescing），当左侧操作数为 `null` 或 `undefined` 时返回右侧的值，否则返回左侧的值。

### 基本语法

```typescript
const result = left ?? right;
// 等价于：
const result2 = (left !== null && left !== undefined) ? left : right;
```

### 行为规则

| 左侧值 | 右侧值 | `??` 结果 | `\|\|` 结果（对比） |
| ------ | ------ | --------- | -------------- |
| `null` | `"default"` | `"default"` | `"default"` |
| `undefined` | `42` | `42` | `42` |
| `0` | `99` | `0` | `99` |
| `""` | `"fallback"` | `""` | `"fallback"` |
| `false` | `true` | `false` | `true` |
| `NaN` | `0` | `NaN` | `0` |

> **核心区别**：`??` 只将 `null` / `undefined` 视为“空”，而 `||` 将所有假值都视为“空”。当 `0`、`""`、`false` 是合法业务值时，应使用 `??`。

### 使用示例

```typescript
// 1. 基础用法
const username: string | null = null;
console.log(username ?? "匿名用户");  // "匿名用户"

const count = 0;
console.log(count ?? 100);   // 0（保留合法假值）
console.log(count || 100);   // 100（|| 会覆盖 0）

// 2. 函数参数默认值
function setVolume(level?: number): void {
    const vol = level ?? 50;  // level 为 0 时不会被覆盖
    console.log(vol);
}
setVolume(0);    // 0
setVolume();     // 50
setVolume(80);   // 80

// 3. 与可选链配合
interface Response {
    data?: {
        items?: string[];
    };
}

const res: Response = {};
const items = res.data?.items ?? [];
console.log(items);  // []

// 4. 多级回退
const primary: string | undefined = undefined;
const secondary: string | undefined = "backup";
const final = primary ?? secondary ?? "default";
console.log(final);  // "backup"
```

### 使用限制

`??` 不能直接与 `||` 或 `&&` 混合使用，必须用括号明确优先级：

```typescript
// ❌ 语法错误
// const x = a ?? b || c;
// const y = a || b ?? c;

// ✅ 用括号明确意图
const x = (a ?? b) || c;
const y = a || (b ?? c);
```

### 适用场景总结

1. **参数默认值**：替代 `||` 避免 `0`、`""`、`false` 被意外覆盖
2. **安全访问后的回退**：与 `?.` 组合提供缺失值的默认方案
3. **配置项读取**：用户配置可能显式传入假值，需正确区分“未设置”与“设置为空”
4. **多级回退链**：按优先级从多个候选值中取第一个非空值

## ||=（逻辑或赋值运算符）

`||=` 是逻辑或赋值运算符（Logical OR Assignment），当变量的值为假值（falsy）时，才将右侧的值赋给它；否则保持原值不变。

### 基本语法

```typescript
a ||= b;
// 等价于：
a || (a = b);  // 注意：并非 a = a || b，具有短路特性
```

### 行为规则

| 变量当前值 | `\|\|=` 右侧值 | 赋值结果 |
| ---------- | -------------- | -------- |
| `null`     | `42`           | `42`     |
| `undefined`| `"hello"`      | `"hello"`|
| `0`        | `99`           | `99`     |
| `""`       | `"default"`    | `"default"`|
| `false`    | `true`         | `true`   |
| `NaN`      | `0`            | `0`      |
| `"exist"`  | `"other"`      | `"exist"`（不变）|
| `1`        | `100`          | `1`（不变）|

> **假值列表**：`false`、`0`、`-0`、`0n`、`""`、`null`、`undefined`、`NaN`

### 使用示例

```typescript
// 1. 基础用法
let title = "";
title ||= "无标题";
console.log(title);  // "无标题"（空字符串是假值）

let score = 85;
score ||= 0;
console.log(score);  // 85（原值为真，不赋值）

// 2. 初始化对象属性
interface Options {
    verbose: boolean;
    retries: number;
    label: string;
}

const opts: Partial<Options> = { retries: 3 };
opts.verbose ||= false;
opts.label ||= "untitled";
opts.retries ||= 5;
console.log(opts);  // { retries: 3, verbose: false, label: "untitled" }

// 3. 与 ??= 的对比
let port = 0;
port ||= 8080;   // port 变为 8080（0 是假值）
port = 0;
port ??= 8080;   // port 仍为 0（??= 只处理 null/undefined）

// 4. 惰性求值
function getDefault(): string {
    console.log("getDefault() called");
    return "default";
}

let existing = "value";
existing ||= getDefault();  // 不会调用 getDefault()
```

### 适用场景总结

1. **确保变量有“有意义”的值**：当空字符串、`0` 等假值不应被保留时使用
2. **简化条件赋值**：替代 `if (!x) { x = defaultValue; }`
3. **与 `??=` 的选择原则**：当 `0`、`""`、`false` 是合法业务值时用 `??=`；当它们是“未设置”的标记时用 `||=`

## &&=（逻辑与赋值运算符）

`&&=` 是逻辑与赋值运算符（Logical AND Assignment），当变量的值为真值（truthy）时，才将右侧的值赋给它；否则保持原值不变。

### 基本语法

```typescript
a &&= b;
// 等价于：
a && (a = b);  // 短路：左侧为假时不执行赋值
```

### 行为规则

| 变量当前值 | `&&=` 右侧值 | 赋值结果 |
| ---------- | -------------- | -------- |
| `"hello"`  | `"world"`      | `"world"` |
| `1`        | `2`            | `2`      |
| `true`     | `false`        | `false`  |
| `null`     | `42`           | `null`（不变）|
| `undefined`| `"x"`          | `undefined`（不变）|
| `0`        | `99`           | `0`（不变）|
| `""`       | `"default"`    | `""`（不变）|
| `false`    | `true`         | `false`（不变）|

> **与 `||=` 互为反向**：`||=` 在假值时赋值，`&&=` 在真值时赋值。

### 使用示例

```typescript
// 1. 基础用法
let msg = "hello";
msg &&= "world";
console.log(msg);  // "world"（原值为真，赋值）

let empty = "";
empty &&= "replaced";
console.log(empty);  // ""（原值为假，不赋值）

// 2. 条件更新：只在已有值时才更新
interface State {
    username: string | null;
    lastLogin: string | null;
}

const state: State = { username: null, lastLogin: null };
state.username &&= state.username.trim();      // username 为 null，不执行
console.log(state.username);                   // null

state.username = "  admin  ";
state.username &&= state.username.trim();      // username 为真，执行 trim
console.log(state.username);                   // "admin"

// 3. 防止对未初始化对象赋值
let config: Record<string, string> | undefined = undefined;
config &&= { ...config, debug: "true" } as any;  // config 为 undefined，不赋值
console.log(config);  // undefined

config = { host: "localhost" };
config &&= { ...config, port: "3000" };  // config 为真，赋值
console.log(config);  // { host: "localhost", port: "3000" }

// 4. 惰性求值
let flag = 0;
flag &&= (() => { console.log("side effect"); return 1; })();
// 不会打印 "side effect"，因为 flag 为 0（假值）
```

### 适用场景总结

1. **条件更新**：只在变量已有有效值时才执行更新操作
2. **避免空值上的操作**：防止对 `null` / `undefined` 调用方法或赋值
3. **链式安全更新**：多个属性只在父对象存在时才更新

## 三个赋值运算符对比总结

| 运算符 | 赋值条件 | 等价写法 | 典型用途 |
| ------ | -------- | -------- | -------- |
| `\|\|=` | 左侧为假值时赋值 | `a \|\| (a = b)` | 为“空”变量提供默认值 |
| `&&=` | 左侧为真值时赋值 | `a && (a = b)` | 只在变量已初始化时更新 |
| `??=` | 左侧为 `null`/`undefined` 时赋值 | `a ?? (a = b)` | 精确处理空值，保留假值 |

```typescript
// 综合示例
let a: number | undefined = undefined;
let b: number = 0;
let c: number = 5;

a ??= 10;   // a = 10（undefined → 赋值）
b ??= 10;   // b = 0（0 不是 null/undefined，不变）
b ||= 10;   // b = 10（0 是假值 → 赋值）
c &&= 20;   // c = 20（5 是真值 → 赋值）
a &&= 30;   // a = 30（10 是真值 → 赋值）
```
