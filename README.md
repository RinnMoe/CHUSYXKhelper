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

## 一键完成

预习任务里有两处"一键完成"，都出现在右下角气泡里，点按钮才执行。

### 视频（`syyx_playvideo.html`）

```
本任务可一键完成（视频）            [一键完成]  [×]
```

点「一键完成」会按视频总时长补报一次进度上报，等价于把进度条拖到结尾。成功后气泡变成：

```
已上报完成（视频）：1009.28 秒
```

- 视频总时长取播放器写入的全局变量 `spsc`（拿不到时退回 `<video>` 的 `duration`）
- 拿不到时长就不显示按钮
- 想关掉：`oneClickVideoComplete: false`

### 资料（`syyx_4.html`）

```
本任务可一键完成（资料）            [一键完成]  [×]
```

点「一键完成」会逐个补报 `addDownjl`，过程中气泡显示 `正在上报资料进度 2/3…`，结束后：

```
已上报完成（资料）：3 项
```

- **上报单位是项目，不是资料行。** `addDownjl` 的请求体是 `{jxrwid, ID}`，其中 `ID` 取的是列表行的 `XMID`（整个项目），不是资料自己的 `ID`。所以同一项目下的多份资料只发一次，脚本按 `XMID` 去重。
- 一个资料页通常只有一个项目 → 实际只发 1 个请求。
- 串行发送，前一个成功（或失败）后才发下一个，避免并发打服务端。
- 想关掉：`oneClickMaterialComplete: false`

> 两处都只是在预习进度上打"已完成"的标记，**不会真的看视频或看资料**。是否需要看内容自己判断。

## 答题防重复提交

答题页点选项后经常弹出「请勿重复提交」，原因是前端在用户不知情的情况下反复发了同一个请求：

- `tjks`（逐题提交）和 `PaperSubmit`（交卷）**没有设置 timeout**，服务端慢时要等几分钟（实测最长 295 秒）
- 等待期间**没有任何反馈**——站点自己的 `alertLoading()` 被注释掉了
- 选项的 `onclick` 只在**成功之后**才清除

于是用户以为没点上，反复点同一题，服务端收到一串相同请求，除第一次外全部回「请勿重复提交」。

**脚本的处理：在发出去之前就去重。** 同一次提交（URL + 请求体完全相同）还在路上时，重复的点击直接丢弃，不发给服务端；第一个请求完成（成功或失败）后自动解锁。

> 注意：**没有**去改 `timeout`。给这两个接口加超时反而更糟——超时中止后用户再点，服务端照样判重复。

想关掉：`dedupeAnswerSubmit: false`

## 修改已答题目

站点在题目已答（`StuAnSwer` 非空）时**不再给选项绑 onclick**，所以答过的题点不动：

```js
if (Data[0].tminfo[0].StuAnSwer != "") { flagIsTrue = "1"; }   // 已答
if (flagIsTrue == 1) {
  if (StuAnSwer.indexOf("A") > -1) { strCss = "tjdx"; }       // 只上色，不绑点击
} else {
  onlickdiv = "onclick=\"tjks(...)\"";                        // 未答才绑
}
```

脚本在每道题渲染后，把站点原本会生成的那句 onclick 补回去：

```js
tjks('KSID','tmid','tmtype','选项字母','该题分值')
```

点击后走的是**站点自己的提交逻辑**——发 `tjks`、成功后清 onclick 并自动翻到下一题、失败弹服务端消息。脚本没有另写一套提交。

- 只接管**单选（A）**和**判断（C）**；多选题要走"勾选 + 提交按钮"，不在此列
- 未答的题站点自己绑了 onclick，脚本不碰
- **服务端是否接受改答案取决于它自己**。不接受时返回「请勿重复提交」，页面重新渲染后答案还原
- 分值取 `#AScore` / `#CScore`，和站点生成 onclick 时用的是同一处
- 想关掉：`allowAnswerChange: false`

## 选工位弹窗关窗不刷新课表

移动端选课课表（`syxk_3.html`）里，带工位的行点开是选工位弹窗：

```js
function chooseGW(xmmc, kbid, jxrwid, xmid) {
  var url = "chooseGW.html?kbid=" + kbid + "&jxrwid=" + jxrwid + "&xmid=" + xmid;
  layer.open({
    type: 2, title: xmmc, area: ['95%', '95%'],
    content: [url],
    end: function () { AddSYKB(1); },   // ← 关窗就重拉整个课表
  })
}
```

`AddSYKB(1)` 会重新发 `Getxkkbbyxmid`，而它是同步请求——一拉就把页面冻住。脚本接管 `layer.open`，把这类弹窗的 `end` 摘掉，关窗不再刷新。

- 只认 `content` 里含 `chooseGW.html` 的弹窗，其它弹窗的 `end` 原样保留
- 代价：关窗后列表里的名额数字是打开弹窗之前的，想更新手动刷新页面
- 想关掉：`keepListAfterChooseGW: false`

## 工作原理

### 两套前端

| 前端 | 入口 | 请求方式 |
|---|---|---|
| 桌面版 | `chdsyxk.chd.edu.cn/LMSMini/teacher/UI/...` | `$.ajax` + `$Post` 加密隧道 |
| 移动 / APK 版 | `chdsyxk.chd.edu.cn:8088/apk/App/...` | 只有 `$.ajax` |

`@match` 按主机匹配、不区分端口，另用 `@include` 显式覆盖 `:8088`。移动端没有 `$Post`，所以两者分开接管、分开报告——移动版控制台看不到 `已接管 $Post` 是正常的。

**除接管方式外，两端行为一致**：同样的缓存、同样的分页放宽、同样的等待提示。两端是不同 origin（端口不同），`sessionStorage` 天然隔离，不会互相污染。

**但"哪些请求能转异步"的策略相反**：

| | 策略 | 原因 |
|---|---|---|
| 桌面端 | 默认转异步，`keepSync` 里的是例外 | 全量读过站点 JS，能逐个确认依赖 |
| 移动端 | **默认保持同步**，只有 `asyncSafeMobile` 里验证过的才转 | 移动端普遍是"`$.ajax` 之后同步读回调变量"，按名字黑名单兜不住 |

移动端的判断依据是**页面地址**（端口 8088 或路径含 `/apk/`），不是请求 URL——移动端页面用的是 `../../../wxInterface/...` 这种相对路径，URL 里看不出前端。

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

所以移动端不能用"黑名单"策略——漏掉一个未知端点就会让对应列表变空白（预习模块的 `AddYXXM` 就是这么被漏掉的）。改用"白名单"：**移动端默认全部保持同步**，只有下面三个验证过"结果只在 `success` 里使用"的才转异步：

| `asyncSafeMobile` 里的端点 | 用途 | 为什么安全 |
|---|---|---|
| `getPdXmSl` | 名额校验 | `success` 里只做跳转 |
| `getBgTjXs` | 报告提交限制 | 同上 |
| `getBgSc` | 报告上传检查 | 同上 |
| `addDownjl` | 下载记录上报 | `success` 里只开预览层或跳转 |

其余全部保持同步，包括两套列表查询（选课的 `GetSyyxRwlistXK` / `GetSyyxRwXMlistXK` / `Getxkkbbyxmid`，预习的 `GetSyyxRwlistXK` / `GetSyyxRwXMlist` / `GetSYXMVideo` / `GetSYXMKeJian` / `GetXmExamList`）、选课提交与取消选课。

注意预习的 `GetSyyxRwXMlist` **没有 `XK` 后缀**，和选课的 `GetSyyxRwXMlistXK` 是两个端点——早期按名字列黑名单时正是漏掉了它，导致预习列表整片空白。

代价是移动端保留了一部分同步 XHR，页面在接口慢时仍会短暂无响应——但列表渲染正确优先于这点体验。想放开某个端点，把它的名字加进 `asyncSafeMobile` 即可（前提是确认它没有同步依赖）。

### 为什么同步请求看不到等待气泡

等待气泡靠 `setInterval` 每秒重绘。同步 XHR 两条路都堵死：

1. **jQuery 在 `$.ajax` 返回前就已 resolve**，`always` 立刻执行把请求从 `pending` 里删掉，1 秒阈值根本走不到
2. 就算走到了，同步 XHR 冻住主线程，定时器回调也不会运行

所以 `Getxkkbbyxmid`、`GetSyyxRwXMlist` 这类必须保持同步的请求**不会有等待提示**，页面就是直接卡住。脚本干脆不为同步请求建定时器。

这是"列表渲染正确"和"有等待提示"只能二选一的地方——脚本选了前者。能看到的等待提示都来自**异步**请求（桌面端列表、答题提交等）。

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
