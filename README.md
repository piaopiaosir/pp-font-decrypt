# font-decrypt

雨课堂 + 学习通 **字体混淆解密库**。单文件、零依赖、全内嵌，供油猴脚本通过 `@require` 引用。

解密全程在浏览器里完成，**不依赖任何后端服务**。

## 用法

```js
// ==UserScript==
// @require      https://cdn.jsdelivr.net/gh/piaopiaosir/font-decrypt@v1.0.0/font-decrypt.js
// ==/UserScript==
```

引入后全局可用 `window.FontDecrypt`：

```js
const FD = window.FontDecrypt;
if (!FD) {
  // 库没加载成功，跳过解密，别让主流程挂掉
}
```

### 学习通（同步）

```js
// 判断当前页面有没有加密文字
if (FD.cxDetect(document)) {
  // 直接解密整个文档（会修改传入的 DOM）
  FD.cxDecode(document);
}
```

### 雨课堂（异步预热 + 同步查表）

```js
// 进到题目页时预热一次（下载字体 → 建表）
await FD.yktPrepare(document);

// 之后同步解密
```

### 不动原 DOM 的用法（推荐）

把题目发给服务端前，克隆一份在克隆体上解密，原页面一个字节都不碰：

```js
const plain = FD.cloneDecrypted(questionElement, 'ykt'); // 或 'cx'
const html = plain.outerHTML;
```

## API

| 方法 | 说明 |
|---|---|
| `FD.cxDetect(doc)` | 文档里有没有 `.font-cxsecret` |
| `FD.cxDecode(doc)` | 同步解密整个文档（**会改 DOM**） |
| `FD.yktDetect(doc)` | 文档里有没有 `.xuetangx-com-encrypted-font` |
| `FD.yktPrepare(doc)` | 异步预热雨课堂字体表，返回 `Promise<boolean>` |
| `FD.yktMap()` | 拿当前已就绪的映射表，未就绪返回 `null` |
| `FD.yktStats()` | 诊断：`{exact, geom, fail, total}` |
| `FD.cloneDecrypted(el, platform)` | 克隆并解密，返回克隆体；失败时返回未解密的克隆体 |
| `FD.tables` | 两张表的版本标识 |

## 工作原理

两个平台的字体混淆机制相同：**字形没有变，只是 cmap 被打乱**。所以只要能算出「这个字形长什么样」的指纹，就能反查它是哪个字。

| | 学习通 | 雨课堂 |
|---|---|---|
| 加密节点 | `.font-cxsecret` | `.xuetangx-com-encrypted-font` |
| 字体来源 | `<style>` 内 base64 内嵌 TTF | `@font-face` 远程 `.ttf` |
| 字形指纹 | `md5(Typr.glyphToPath)` 取后 8 位 | `FNV-1a(glyf 原始点坐标)` 取 8 位 |
| 查表方式 | 单级精确匹配 | **两级**：精确坐标 → 几何兜底 |
| 表规模 | 20906 条 | 20975 条 + 881 条兜底 |

## 体积

单个文件约 **656 KB**（gzip 后约 **321 KB**）。表数据占绝大部分，属于一次性加载，油猴会永久缓存。

## 已知限制

- 雨课堂的表是**静态表**。如果官方更换了混淆字体（字形轮廓变化），表会失效，需要用新字体重新生成。运行时可调用 `FD.yktStats()` 观察 `fail` 占比，一旦明显升高就说明字体换版本了。
- 雨课堂精确表 20975 条而非 20976：原表中「墫 / 壿」两个极生僻字的 32 位 FNV 指纹碰撞，合并成了 1 条。影响可忽略；要消除需改用 64 位哈希并重新生成整张表。
- 只处理 BMP 范围内的常用汉字，扩展区生僻字不覆盖。

## License

MIT，见 [LICENSE](LICENSE)。
