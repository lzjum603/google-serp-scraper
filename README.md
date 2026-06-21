# SERP API JavaScript 完整指南：如何用 Node.js 抓取 Google 搜索结果？参数怎么设置？哪个方案最省心？（附 ScraperAPI 套餐对比与代码实例）

每隔一段时间，开发圈里都会冒出同一个问题：**我想用 JavaScript 抓 Google 搜索结果，该怎么做？**

然后就开始踩坑——用 `axios` 直接请求 Google，返回 403；加上 headers 伪装浏览器，撑了几百次请求之后 IP 被封；换代理，代理不稳定；改用 Puppeteer 跑无头浏览器，服务器内存扛不住……

折腾了一圈，才发现这条路根本不适合"直接硬来"。

搜索引擎的反爬机制不是吃素的，Google 尤其狠。它会根据请求频率、User-Agent、IP 信誉、行为模式综合判断你是不是机器人。只要其中一环出问题，就是封号。

这时候，很多开发者开始认真考虑 **SERP API**——也就是专门封装好搜索结果抓取能力的 API 服务。今天这篇文章，就来聊聊用 JavaScript（Node.js）接入 SERP API 的完整思路，以及 ScraperAPI 这个工具为什么值得重点考虑。

---

## 什么是 SERP API？JavaScript 开发者为什么需要它？

SERP 是 Search Engine Results Page 的缩写，SERP API 就是把搜索结果页的数据提取出来，封装成结构化 JSON 格式供你调用的 API 服务。

你发一个 HTTP 请求，传个关键词，它返回给你干净的 JSON：标题、链接、摘要、相关问题、知识图谱、视频……该有的全有，不用你自己写一行解析代码。

对于 JavaScript 开发者来说，这个模式非常友好——`fetch` 或者 `axios` 发个请求，`.then()` 处理一下数据，就完事了。比自己维护一套代理池 + 无头浏览器的方案，省下的时间和服务器成本是量级差距。

---

## 用 JavaScript 直接抓 Google 的三条死路

在真正介绍 SERP API 用法之前，先把"自己动手"的方案交代清楚，省得有人还在这条路上耗时间。

**第一条：直接 fetch/axios 请求 Google**

js
const response = await fetch('https://www.google.com/search?q=javascript');


结果：大概率返回 429 或者 302 跳转到人机验证页。Google 一看到来自数据中心 IP 的搜索请求，直接拦截。

**第二条：Puppeteer/Playwright 模拟真实浏览器**

能用，但代价很大。一个无头浏览器实例至少消耗几百 MB 内存，并发跑起来服务器很快撑不住。而且 Google 现在对无头浏览器的检测已经相当成熟，TLS 指纹、Canvas 指纹、行为模式……绕起来很麻烦。

**第三条：买代理自己轮换**

这条路走下去，你会发现代理质量参差不齐，维护成本不低，结果也不稳定。遇到 Google 的动态验证，代理也救不了你。

所以，聪明的做法是——直接用 SERP API，把这些脏活外包出去。

---

## ScraperAPI 的 Google SERP API：JavaScript 用法详解

👉 [免费试用 ScraperAPI，领 5,000 次 API 调用额度](https://www.scraperapi.com/?fp_ref=coupons)

ScraperAPI 提供了一个专门的 Google SERP 结构化数据端点，调用方式极其简单。它会自动处理代理轮换、验证码绕过和浏览器指纹伪装，你只管发请求、拿数据。

### 基础调用：Node.js fetch 写法

javascript
import fetch from 'node-fetch';

const API_KEY = 'YOUR_API_KEY';
const query = 'javascript web scraping';
const country_code = 'us';

fetch(`https://api.scraperapi.com/structured/google/search?api_key=${API_KEY}&query=${encodeURIComponent(query)}&country_code=${country_code}`)
  .then(response => response.json())
  .then(data => {
    console.log(data.organic_results);
  })
  .catch(error => {
    console.error('请求出错：', error);
  });


### async/await 写法（更推荐）

javascript
import fetch from 'node-fetch';

async function fetchSERP(query, countryCode = 'us') {
  const API_KEY = process.env.SCRAPERAPI_KEY;
  const url = `https://api.scraperapi.com/structured/google/search?api_key=${API_KEY}&query=${encodeURIComponent(query)}&country_code=${countryCode}`;
  
  try {
    const response = await fetch(url);
    const data = await response.json();
    return data;
  } catch (err) {
    console.error('SERP 抓取失败:', err);
    throw err;
  }
}

const results = await fetchSERP('best javascript frameworks');
console.log(results.organic_results);


API 密钥放环境变量里，别硬编码在代码里——这是最基本的安全习惯。

### 返回的数据结构长什么样？

json
{
  "search_information": {
    "query_displayed": "javascript web scraping"
  },
  "organic_results": [
    {
      "position": 1,
      "title": "Web Scraping with JavaScript...",
      "snippet": "A complete guide to...",
      "link": "https://example.com/...",
      "displayed_link": "https://example.com › ..."
    }
  ],
  "related_questions": [...],
  "knowledge_graph": {...},
  "videos": [...],
  "pagination": {...}
}


结构很清晰，`organic_results` 是自然搜索结果，`related_questions` 是 People Also Ask，`pagination` 里有翻页 URL。

---

## 关键参数说明

ScraperAPI 的 Google SERP 端点支持以下参数，搞清楚这些，才能抓到你真正想要的数据。

| 参数 | 是否必填 | 说明 |
|------|----------|------|
| `api_key` | ✅ 必填 | 你的 API 密钥 |
| `query` | ✅ 必填 | 搜索关键词，如 `javascript tutorial` |
| `country_code` | 可选 | 双字母国家代码，如 `us`、`gb`、`de`，影响搜索结果的地区版本 |
| `tld` | 可选 | Google 域名后缀，如 `com`、`co.uk`、`com.br`，默认 `com` |
| `hl` | 可选 | 界面语言，如 `en`、`zh-CN` |
| `gl` | 可选 | 结果来源国家倾向，如 `us`、`cn` |
| `start` | 可选 | 翻页起始位置，`start=10` 表示从第 11 条开始（第 2 页） |
| `output_format` | 可选 | 输出格式，支持 `json`（默认）和 `csv` |
| `tbs` | 可选 | 时间过滤，`tbs=d` 过去一天，`tbs=w` 过去一周，`tbs=m` 过去一个月 |
| `uule` | 可选 | 精确地理位置参数（城市级别定位） |

如果你要抓取特定地区的 Google 结果，比如从美国视角抓 google.co.uk 的结果，需要同时指定 `tld=co.uk` 和 `country_code=gb`。

---

## 实战：批量抓取多个关键词的排名数据

SEO 团队最常见的需求之一：给定一批关键词，批量查排名。下面是一个简单的实现思路：

javascript
import fetch from 'node-fetch';

const API_KEY = process.env.SCRAPERAPI_KEY;
const keywords = ['node.js tutorial', 'javascript async await', 'web scraping javascript'];

async function checkRanking(keyword, targetDomain) {
  const url = `https://api.scraperapi.com/structured/google/search?api_key=${API_KEY}&query=${encodeURIComponent(keyword)}&country_code=us`;
  
  const response = await fetch(url);
  const data = await response.json();
  
  const results = data.organic_results || [];
  const found = results.find(r => r.link && r.link.includes(targetDomain));
  
  return {
    keyword,
    position: found ? found.position : 'Not in top results',
    url: found ? found.link : null
  };
}

// 批量查询（注意控制并发速率，避免超出账户限制）
async function bulkCheck(keywords, domain) {
  const results = [];
  for (const kw of keywords) {
    const result = await checkRanking(kw, domain);
    results.push(result);
    console.log(`✓ ${kw}: 排名 ${result.position}`);
    await new Promise(r => setTimeout(r, 500)); // 控制请求间隔
  }
  return results;
}

bulkCheck(keywords, 'yourwebsite.com').then(console.log);


这套逻辑可以直接接进你的 SEO 监控系统或 CI 流程里，定时跑、自动报告。

---

## 翻页：怎么抓第 2、3……N 页的结果？

用 `start` 参数控制起始位置。Google 每页默认 10 条，所以第 2 页是 `start=10`，第 3 页是 `start=20`，以此类推。

javascript
async function fetchPage(query, pageNumber) {
  const start = (pageNumber - 1) * 10;
  const url = `https://api.scraperapi.com/structured/google/search?api_key=${API_KEY}&query=${encodeURIComponent(query)}&start=${start}`;
  const res = await fetch(url);
  return res.json();
}

// 抓前 3 页
for (let page = 1; page <= 3; page++) {
  const data = await fetchPage('javascript frameworks', page);
  console.log(`第 ${page} 页有 ${data.organic_results?.length} 条结果`);
}


---

## ScraperAPI 套餐一览

ScraperAPI 提供 7 天试用，包含 5,000 次免费 API 调用，无需信用卡。付费套餐按月计费，年付可享 9 折优惠。

| 套餐 | 月付价格 | 年付价格 | API Credits | 并发线程 | 地区定向 | 购买链接 |
|------|---------|---------|-------------|---------|---------|---------|
| **Hobby** | $49/月 | $44.10/月 | 100,000 | 20 | 仅美国 & 欧盟 |  [立即购买](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Startup** | $149/月 | $134.10/月 | 1,000,000 | 50 | 仅美国 & 欧盟 |  [立即购买](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Business** | $299/月 | $269.10/月 | 3,000,000 | 100 | 全球 |  [立即购买](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Scaling** ⭐ | $475/月 | $427.50/月 | 5,000,000 | 200 | 全球 |  [立即购买](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Professional** | $975/月 | $877.50/月 | 10,500,000 | 300 | 全球 |  [立即购买](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Advanced** | $1,975/月 | $1,777.50/月 | 21,500,000 | 500 | 全球 |  [立即购买](https://www.scraperapi.com/signup/?fp_ref=coupons) |
| **Enterprise** | 定制 | 定制 | 22,000,000+ | 500+ | 全球 |  [联系销售](https://www.scraperapi.com/contact-sales/?fp_ref=coupons) |

**全部套餐都包含**：JS 渲染、优质代理、JSON 自动解析、代理池轮换、自定义 Headers、验证码处理、无限带宽、99.9% SLA 保障。

> **关于 Credits 消耗**：Google 搜索结果属于高难度目标，每次请求消耗 25 Credits（Google 和 Bing 相关域名统一按此计费）。所以 Hobby 套餐 10 万 Credits 大约对应 4,000 次 Google SERP 查询，Startup 套餐则可以支撑约 40,000 次。根据你的实际需求量选套餐，不要只看 Credits 总数。

👉 [先免费试用，感受一下效果再决定](https://www.scraperapi.com/?fp_ref=coupons)

---

## 常见问题解答

**Q：Google SERP API 返回的结果和我自己在浏览器里搜到的一样吗？**

基本一致，但会有差异。Google 会根据你的 IP 位置、登录状态、历史记录做个性化排序。ScraperAPI 使用的是干净的代理 IP，返回的是更接近"未登录普通用户"视角的搜索结果，适合做竞品分析和排名监控。

**Q：如果 Credits 用完了怎么办？**

Hobby、Startup、Business 套餐用完后可以选择升级套餐，或联系客服定制方案。Scaling 及以上套餐支持 Pay-as-you-go，超出部分按固定单价继续计费，不会中断服务。

**Q：能抓"People Also Ask"和知识图谱吗？**

可以。ScraperAPI 返回的 JSON 里包含 `related_questions`（PAA）、`knowledge_graph`、`videos`、`organic_results` 等字段，一次请求全拿到。

**Q：支持哪些 Google 域名？**

支持 google.com、google.co.uk、google.de、google.fr、google.co.jp、google.com.br 等 20+ 个国家域名，用 `tld` 参数指定即可。

**Q：可以同时跑多少并发请求？**

取决于你的套餐，从 Hobby 的 20 个并发线程到 Enterprise 的 500+ 不等。超出并发限制的请求会返回 429，不额外收费。

---

## 什么时候用 ScraperAPI，什么时候自己建抓取器？

如果你只是偶尔需要几十条搜索结果做研究，用 ScraperAPI 的免费额度就够了，根本不需要自己搭环境。

如果你在做 SEO 工具、竞品监控、内容聚合这类需要持续、大量抓取 Google 数据的产品，ScraperAPI 的成本其实比自己养一套代理池 + 服务器更低——光是省下来的工程师时间，就已经回本了。

ScraperAPI 背后有 4000 万+ IP 的代理池，覆盖 50+ 个国家，Google 封了换 IP 是毫秒级的事，成功率比自建方案高很多。

自己建抓取器的场景通常只剩一个：你需要抓的数据不在 ScraperAPI 的支持范围内，或者有非常特殊的业务逻辑需要高度定制。否则，能用 API 解决的问题，别费那个劲。

---

用 JavaScript 接 SERP API，整个流程其实很轻——注册账号、拿密钥、一行 `fetch`，就开始跑了。剩下的代理、验证码、IP 信誉这些问题，交给 ScraperAPI 去操心。

👉 [注册 ScraperAPI，免费领 5,000 次调用额度](https://www.scraperapi.com/?fp_ref=coupons)
