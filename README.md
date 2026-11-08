# CHUSYXKhelper

长安大学实验选课系统优化脚本

脚本解决三件事：

- **页面不再假死** —— 接口慢的时候还能滚动、点击、切标签
- **不用反复等同一份数据** —— 列表结果会缓存，切走再回来直接显示
- **知道自己在等什么** —— 右下角显示已经等了多少秒

但脚本无法解决：

- 缩短**加载时间**：选课高峰期平均加载用时仍长达4-5分钟，但现在只需等一次。

---

## 安装

1.  [安装 Tampermonkey](https://www.tampermonkey.net/)或 [安装 Violentmonkey](https://violentmonkey.github.io/)。
2. 点击 [安装`CHUSYXKhelper-request-keepalive-hedge.user.js`](https://github.com/RinnMoe/CHUSYXKhelper/raw/refs/heads/main/CHUSYXKhelper-request-keepalive-hedge.user.js)，在弹出的安装页面中点击“安装”即可。
3. 刷新选课页面。右下角出现气泡消息提示说明脚本已经生效。

---

## 工作原理

### 两套前端

| 前端 | 入口 | 请求方式 |
|---|---|---|
| 桌面版 | `chdsyxk.chd.edu.cn/LMSMini/teacher/UI/...` | `$.ajax` + `$Post` 加密隧道 |
| 移动 / APK 版 | `chdsyxk.chd.edu.cn:8088/apk/App/...` | 只有 `$.ajax` |

`@match` 按主机匹配、不区分端口，另用 `@include` 显式覆盖 `:8088`。移动端没有 `$Post`，所以两者分开接管、分开报告——移动版控制台看不到 `已接管 $Post` 是正常的。

**除接管方式外，两端行为完全一致**：同样的缓存、同样的分页放宽、同样的等待提示。两端是不同 origin（端口不同），`sessionStorage` 天然隔离，不会互相污染。

### 卡死的根源：同步 XHR

站点用 `async: false` 发请求。同步 XHR 会锁死渲染主线程——服务端慢多久，页面就死多久。

更糟的是 `timeout` 完全失效：主线程被占住，定时器根本没机会执行，所以设了 30 秒超时也会一直等下去。

这是纯前端问题。改成异步，页面立刻恢复响应，接口该多慢还是多慢。

### 为什么不能全部改成异步

有些同步调用是设计上就依赖的：

| 请求 | 原因 |
|---|---|
| `Handshake` | 返回加密密钥，`$Post` 拿到后立刻用它加密请求体；异步会拿空密钥 |
| `GetUIToken` | 返回的 token 会写进 `$.ajaxSetup` 的默认请求头，后续请求依赖它 |
| 若干 `$Post` 函数 | `GetHiddenColumn`、`getXuanKeSheZhi`、`tableSearchResultLft` 等在 `$Post` 之后直接 `return` 回调里赋的值，转异步会返回 `undefined` |

所以只放行"结果只在 `success` 回调里使用"的功能，其余保持原样。

### 为什么放宽分页

服务端每次都把整个结果集算完再切片。所以 `limit` 调小不会更快，只会让你多等几轮——`limit` 从 20 提到 200 可以把 3 轮压成 1 轮。

移动端同理：`Getxkkbbyxmid` 每页 15 条，一次课表要翻 4 页，把 `PageSize` 提到 200 后一页就够。另外两个移动端列表本来就请求 `PageSize: 999999`，一页拿全，不需要动。

`getRuleItem` 不放宽：它的结果本来就一页装得下。

### 缓存的两个前提

缓存里带着 `RS` / `GW` 这类实时名额，回放的是旧数据。所以：

| 机制 | 作用 |
|---|---|
| 只存 `sessionStorage` | 关标签页即失效，不跨天 |
| `cacheTtlMs` | 过期即丢弃，重新请求 |
| 选课成功（`Addksxmtu`）后立刻清空 | 否则刚抢到的名额会显示成旧数字 |
| 命中时提示 + 「刷新」按钮 | 点按钮清空缓存并重载当前页 |

只缓存**列表**接口——桌面端 2 个（`getItemTime`、`getRuleItem`），移动端 3 个（`GetSyyxRwlistXK`、`GetSyyxRwXMlistXK`、`Getxkkbbyxmid`）。选课、取消、预习、报告等写操作一律不缓存。

回放走的是和真实请求同一条代码路径：layui 的 `success` 里就是 `JSON.parse(res.d)`，把缓存下来的对象原样交回去，渲染结果完全一致。

**回放的时序跟着端点走。** 移动端那三个列表被同步消费（见下一节），命中缓存时也**同步**调用 `success`，否则 `next(lis.join(''), page < count)` 拿到的还是空值。桌面端没有这个约束，回放走 `setTimeout` 异步执行，不阻塞调用方。

### 移动端哪些请求必须保持同步

移动端页面里有这种写法：

```js
setTimeout(function () {
  var count = 0;
  $.ajax({ async: false, success: function (json) { lis.push(kbhtml); count = json.length; } });
  next(lis.join(''), page < count);   // ← 同步读回调里赋的值
  lm_li_action();
}, 500);
```

`next(...)` 在 `$.ajax` 之后同步执行，读的是 `success` 里刚赋的 `lis` 和 `count`。改成异步会让它拿到空列表、`page < 0` 永远为假——**列表直接渲染成空白**。

所以这三个列表查询列入 `keepSync`：

| 端点 | 页面 |
|---|---|
| `GetSyyxRwlistXK` | 选课任务列表 |
| `GetSyyxRwXMlistXK` | 选课项目列表 |
| `Getxkkbbyxmid` | 选课课表 |

另外 `AppInterface.asmx/Addksxmtu`（选课提交）和 `N_QXXK`（取消选课）也保持同步——它们没有同步依赖，但同步 XHR 冻住界面正好挡住连点重复提交，而这是一次性写操作。

**其余移动端同步请求照常转异步**：`getPdXmSl`、`getBgTjXs`、`getBgSc` 的结果都只在 `success` 里使用（多数是直接跳转）。

注意 `Addksxmtu` 用了带服务名的子串 `AppInterface.asmx/Addksxmtu`，只匹配移动端；桌面端的 `syjx.asmx/Addksxmtu` 不受影响。

### 已试过、无效的方向

以下都验证过不成立，代码里没有对应实现，不要重复添加：

| 方向 | 结论 |
|---|---|
| 用 `xqjid` / `zcid` 过滤缩小结果集 | 服务端忽略这两个参数，返回行数不变 |
| 走加密隧道调 `miniYEARDATELIST` | 该端点两个服务前端都失败 |
| 拦截 `JsonSql.js`（89 KB 的 jslinq） | 它被页面混淆代码引用着，拦掉会让页面打不开 |
| 缓存静态资源 / 用 Service Worker | 响应头是 `no-store`；站点是 HTTP，SW 不可用 |
| 并发拆分同一个查询 | 服务端线程有限，并发只会排队 |
| 降低 `limit` 让它更快 | 服务端先算全量再切片，`limit` 小只会多几轮 |

---

## 开发

```bash
node --check CHUSYXKhelper-request-keepalive-hedge.user.js
node --test tests/helper.test.cjs
```

## 许可证

本项目采用 [MIT License](LICENSE) 开源。
