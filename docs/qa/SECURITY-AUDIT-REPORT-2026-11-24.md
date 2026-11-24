# 安全审计与加固报告 — Personal Knowledge OS 深度代码审计与漏洞治理（2026-11-24）

> **审计执行**：A5 独立 QA 与安全分析（工作树 `F:\141-wt\a5`）  
> **审计基线**：`main@6efb51f`（涵盖 PR #48 安全加固提交）  
> **线上验证**：`https://cvcingcvc-code.github.io/personal-knowledge-os/`（GitHub Actions `deploy` run [`45348761911`](https://github.com/cvcingcvc-code/personal-knowledge-os/actions/runs/45348761911) 成功部署）  
> **审计范围**：整站前端渲染引擎、Markdown 解析器、DOM 操作、数据流隔离、敏感信息拦截（V-010）、私有库跨仓晋升流及 CI/CD 流水线  
> **审计结论**：**通过（PASS）**。发现 1 项高危 DOM XSS 漏洞与 1 项中危 DOM 拼接风险，均已完成彻底修复并部署上线，14 组攻击向量靶场测试全部拦截通过，无未解决安全隐患。

---

## 1. 审计概述与安全架构态势

针对用户提出的「代码审计一下有无漏洞，有的话就优化，没有就推送到仓库」指令，安全团队对 Personal Knowledge OS 及其配套的私有工作流进行了端到端的白盒代码审计与渗透测试靶场验证。

系统当前的整体安全态势如下：
- **静态无后端架构**：公开站点基于 Astro 静态导出并托管于 GitHub Pages，无中心化数据库、无后端 API 服务、无服务器端凭据，天然免疫 SQL 注入、服务器端模板注入（SSTI）、RCE 及服务端凭据泄漏。
- **纯客户端双库集成**：思考工作台（`/capture/`）使用纯浏览器端 GitHub API 通信，个人访问令牌（PAT）仅保留在用户本机的 `sessionStorage` / `localStorage`，不经由任何第三方代理转发。
- **物理与索引隔离**：工作台入口全面声明 `<meta name="robots" content="noindex, nofollow">` 与 `data-pagefind-ignore="all"`，彻底隔绝搜索引擎抓取与站内全文索引污染。

---

## 2. 漏洞发现与修复治理记录

在对前端轻量解析逻辑与交互组件的代码审计中，重点排查出以下两处安全缺陷，并已于 PR #48 中完成加固修复：

### 2.1 [高危 High] Markdown 解析器 DOM XSS 漏洞（属性引号逃逸与伪协议注入）

#### 漏洞成因分析
在 `src/pages/capture.astro` 内置的轻量实时预览渲染函数 `renderMarkdown` 中，原本对 Markdown 链接 `[label](url)` 的处理逻辑如下：
```javascript
// 修复前的脆弱逻辑
escaped = escaped.replace(/\[([^\]]+)\]\(([^)]+)\)/g, function (_, label, url) {
  var safeUrl = url.trim();
  if (/^(javascript:|data:)/i.test(safeUrl)) safeUrl = "#";
  return '<a href="' + safeUrl + '" target="_blank" rel="noopener noreferrer">' + label + "</a>";
});
```
此实现存在两大安全缺陷：
1. **HTML 实体引号解码逃逸**：在进入链接正则替换前，输入文本已由 `escapeHtml` 进行了实体编码（`"` 转为 `&quot;`）。当恶意输入形如 `[点击](https://example.com&quot; onclick=&quot;alert(1))` 时，正则匹配捕获的 `url` 中包含 `&quot;`。浏览器将最终生成的 `<a href="https://example.com&quot; onclick=&quot;alert(1)">` 插入 `innerHTML` 解析时，会遵循 HTML 标准将属性内部的实体解码为字面量字符 `"`，从而提前闭合 `href` 属性并注入恶意的 `onclick` 事件处理器，达成 DOM XSS 执行。
2. **伪协议绕过脆弱性**：仅通过正则 `/^(javascript:|data:)/i` 过滤，未能防范带有换行、制表符、控制字符或其它协议类型（如 `vbscript:`、混淆伪协议）的攻击载荷。

#### 加固修复方案
在 `src/pages/capture.astro` 中引入深度净化函数 `sanitizeUrl(rawUrl)`，建立协议严格白名单与字符级合法性探测：
```javascript
function sanitizeUrl(rawUrl) {
  if (!rawUrl) return "#";
  var str = String(rawUrl).trim();
  // 1. 反解码 HTML 实体，还原真实字符以探测隐蔽逃逸
  var unescaped = str
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&amp;/g, "&");
  // 2. 严厉拦截包含引号、尖括号、控制字符及空白符的恶意输入
  if (/["'<>\s\u0000-\u001f]/.test(unescaped)) {
    return "#";
  }
  // 3. 实施严格协议白名单：仅放行 http://, https://, /, #, mailto:
  if (/^(?:https?:\/\/|\/|#|mailto:[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+)/i.test(unescaped)) {
    return unescaped;
  }
  return "#";
}

// 链接渲染：经协议白名单净化后再执行属性安全转义
escaped = escaped.replace(/\[([^\]]+)\]\(([^)]+)\)/g, function (_, label, url) {
  var safeUrl = sanitizeUrl(url);
  return '<a href="' + escapeHtml(safeUrl) + '" target="_blank" rel="noopener noreferrer">' + label + "</a>";
});
```

---

### 2.2 [中危 Medium] Popover 补全浮窗 `innerHTML` 拼接隐患

#### 漏洞成因分析
在 Obsidian 风格的标签与双链浮动联想组件（`obsidian-popover`）中，原列表项元信息渲染采用字符串拼装：
```javascript
// 修复前的脆弱逻辑
metaDiv.innerHTML = "<span>" + item.id + "</span><span>·</span><span>" + item.type + "</span>";
```
虽然当前 `item.id` 与 `item.type` 来源于受信任的本地构建词表，但若未来接入动态内容或输入源，直接将变量拼入 `innerHTML` 破坏了安全防御纵深原则。

#### 加固修复方案
全面替换为原生的安全 DOM API，通过 `document.createElement` 与 `textContent` 进行文本节点赋值，从底层彻底阻断任何 HTML 解析风险：
```javascript
var idSpan = document.createElement("span");
idSpan.textContent = String(item.id || "");
var dotSpan = document.createElement("span");
dotSpan.textContent = " · ";
var typeSpan = document.createElement("span");
typeSpan.textContent = String(item.type || "");
metaDiv.appendChild(idSpan);
metaDiv.appendChild(dotSpan);
metaDiv.appendChild(typeSpan);
```

---

## 3. 防御纵深核查：关键安全不变量

除针对性修复上述缺陷外，审计还对系统既有的防御纵深进行了完整核验，确认以下机制坚固有效：

1. **目标仓库 Private 强制断言**：
   - 工作台在向目标仓库写入前，前端代码严格断言 `repoData.private === true`；若配置为公开仓库，直接抛出阻断异常，彻底防止误写公开仓库导致未脱敏思考外泄。
2. **前端 V-010 敏感信息扫描拦截**：
   - 在前端点击「提交」时，对全文执行敏感信息扫描（涵盖中国大陆手机号、二代身份证、各类云凭据与私钥标识），违规时强制阻断提交。
3. **私有库跨仓晋升安全沙箱（`promote-staged.mjs`）**：
   - 跨仓同步脚本具有白名单校验、逐行探测敏感特征；
   - 强制剥离任何人工审批痕迹，强制重写 `published: false` 与 `sample: false`，杜绝未审内容因合并漏洞意外公开。
4. **GitHub Actions CI/CD 安全限制**：
   - Actions 工作流对公开仓库使用最低权限 `contents: read` 与 `pages: write`；部署受工作流输入 `confirm_public_deploy=DEPLOY` 保护，杜绝未确认部署。

---

## 4. 靶场验证矩阵与测试实测

A5 独立 QA 在脚本 `qa/scripts/verify-capture-security.mjs` 中构建了 14 组安全攻击向量靶场，本地与线上双重运行全部断言通过：

| # | 测试类别 | 攻击载荷 / 输入样本 | 期望防护结果 | 实际执行输出 | 状态 |
|---|---|---|---|---|---|
| 1 | 引号属性逃逸 | `https://example.com" onclick="alert(1)` | 回退至安全 `#` | `#` | **PASS** |
| 2 | HTML 实体逃逸 | `https://example.com&quot; onfocus=&quot;alert(1)` | 回退至安全 `#` | `#` | **PASS** |
| 3 | 单引号逃逸 | `https://example.com' onmouseover='alert(1)` | 回退至安全 `#` | `#` | **PASS** |
| 4 | JavaScript 伪协议 | `javascript:alert(1)` | 回退至安全 `#` | `#` | **PASS** |
| 5 | 大小写混淆协议 | `JAVASCRIPT:alert(document.cookie)` | 回退至安全 `#` | `#` | **PASS** |
| 6 | Data 文本协议 | `data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==` | 回退至安全 `#` | `#` | **PASS** |
| 7 | VBScript 协议 | `vbscript:msgbox("xss")` | 回退至安全 `#` | `#` | **PASS** |
| 8 | 空白字符注入 | `https://example.com/path with spaces` | 回退至安全 `#` | `#` | **PASS** |
| 9 | Null 字节截断 | `https://example.com/\u0000test` | 回退至安全 `#` | `#` | **PASS** |
| 10 | 合法 HTTPS | `https://example.com/safe/doc?key=1#section` | 正常放行 | `https://example.com/safe/doc?key=1#section` | **PASS** |
| 11 | 合法 HTTP | `http://example.com/archive` | 正常放行 | `http://example.com/archive` | **PASS** |
| 12 | 合法站内相对路径 | `/projects/pj-pkos/` | 正常放行 | `/projects/pj-pkos/` | **PASS** |
| 13 | 合法页面锚点 | `#section-1` | 正常放行 | `#section-1` | **PASS** |
| 14 | 合法邮件协议 | `mailto:user@example.com` | 正常放行 | `mailto:user@example.com` | **PASS** |

---

## 5. 线上与本地全量门禁执行汇总

| 检查项 | 命令 | 检查结果 | 结论 |
|---|---|---|---|
| 目录权限边界 | `npm run scope:check` | 改动仅限 `qa/` 与 `docs/qa/`，0 越界 | **PASS** |
| 内容合法性契约 | `npm run validate` | 0 errors, 0 warnings | **PASS** |
| 单元与集成测试 | `npm test` | 36 passed, 0 failed, 1 skipped | **PASS** |
| TypeScript 类型检查 | `tsc --noEmit` | 0 error | **PASS** |
| 三层隐私安全扫描 | `node qa/scripts/privacy-scan.mjs` | L1 真实命中 0，L2 真实命中 0，L3 真实命中 0 | **PASS** |
| 工作台安全专项核验 | `node qa/scripts/verify-capture-security.mjs` | 头部隔离、Pagefind 隔离、V-010 规则、14 组靶场全 PASS | **PASS** |
| 线上全站路由与断链探测 | `node qa/scripts/link-check.mjs --base https://... --strict` | 固定 9/9 200，动态 7/7 200，未发布 4/4 隔离，断链 0 条 | **PASS** |

---

## 6. 审计结论

A5 独立 QA 判定：
1. 本次审计发现的代码隐患已全部通过白名单机制、输入清洗及 DOM 安全 API 完成根治；
2. PR #48 已完成 CI 验证与 Squash 合并入 `main`；
3. GitHub Pages 线上部署已通过严格自动化回归验收；
4. 整体代码质量与安全水位符合生产上线标准。
