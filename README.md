# 毕业去向地图模板

一个纯静态的高中毕业生大学去向可视化模板。页面以天地图卫星影像为底图，将不同班级学生的大学去向显示在全国地图上，支持口令验证、班级筛选、姓名/大学搜索、数据统计和缺失数据展示。

当前仓库已移除原始真实班级信息，内置数据均为演示用虚构姓名。示例高校名称使用中国真实高校，便于演示地理编码和地图落点效果。

## 当前示例配置

- 示例页面标题：`杭州第二中学毕业去向地图模板`
- 示例浏览器标签页标题：`毕业去向地图模板`
- 示例班级数量：5 个班
- 示例口令明文：`杭州第二中学`
- 示例口令 SHA-256：`2e03f3251d70c28a76fd1b9a2aa366b823391da7d0818608146d292ec8107ebf`
- 地图服务：天地图 JavaScript API v4.0

> 口令明文只写在 README 中便于模板使用者调试。生产使用时，请替换为你自己的口令哈希，并按需要删除这段明文说明。

## 功能特性

- 口令验证：浏览器端使用 Web Crypto API 计算 SHA-256，与 `js/config.js` 中的哈希列表比对。
- 班级筛选：右下角动态生成 1 到 `CONFIG.classCount` 班的筛选项。
- 地图标记：按大学聚合学生；同一大学若来自多个班级，自动合并为灰色标记。
- 搜索：支持按学生姓名、大学名称和城市名称检索。
- 数据统计：展示总人数、各班人数、城市分布 Top 8、热门院校 Top 8。
- 缺失数据：支持在 `classNMissingData` 中记录尚未收集到去向的学生。
- 坐标优先：数据中提供 `latitude` / `longitude` 时直接落点，否则通过天地图地理编码查询。

## 项目结构

```text
.
├── index.html
├── README.md
├── favicon.ico
├── css/
│   └── style.css
├── js/
│   ├── config.js
│   └── main.js
└── data/
    ├── class1.js
    ├── class2.js
    ├── class3.js
    ├── class4.js
    └── class5.js
```

## 快速预览

本项目不需要构建步骤。建议通过本地静态服务器打开，避免浏览器对本地脚本加载的限制。

```powershell
python -m http.server 8080
```

然后访问：

```text
http://localhost:8080
```

默认口令为：

```text
杭州第二中学
```

## 自定义模板

### 1. 修改页面标题、浏览器标签页标题和班级数量

编辑 `js/config.js`：

```javascript
pageTitle: '你的学校毕业去向地图',
browserTitle: '毕业去向地图',
classCount: 5,
```

如果修改 `classCount`，需要同步调整：

- `data/class1.js` 到 `data/classN.js`
- `CONFIG.classColors`
- `CONFIG.classNames`

`index.html` 会根据 `CONFIG.classCount` 自动加载对应数据文件，无需手写多个 `<script>` 标签。

### 2. 修改口令

在浏览器控制台或 Node.js 中计算新口令的 SHA-256。

浏览器控制台：

```javascript
crypto.subtle.digest('SHA-256', new TextEncoder().encode('你的口令'))
  .then(function (buf) {
    return Array.from(new Uint8Array(buf))
      .map(function (b) { return b.toString(16).padStart(2, '0'); })
      .join('');
  })
  .then(console.log);
```

Node.js：

```javascript
require('crypto')
  .createHash('sha256')
  .update('你的口令', 'utf8')
  .digest('hex');
```

将结果写入 `js/config.js`：

```javascript
validPasswordHashes: [
  '你的64位sha256哈希'
],
```

### 3. 修改天地图 TK

访问 [天地图开放平台](https://lbs.tianditu.gov.cn/) 申请 JavaScript API 密钥，然后替换：

```javascript
tiandituTK: '你的天地图TK',
```

### 4. 替换班级数据

每个 `data/classN.js` 文件导出两个数组：

```javascript
var class1Data = [
  {
    "name": "张三",
    "university": "北京大学",
    "city": "北京",
    "latitude": 39.9928,
    "longitude": 116.3109
  }
];

var class1MissingData = [
  { "name": "李四" }
];
```

字段说明：

| 字段           | 是否必填 | 说明                         |
| -------------- | -------- | ---------------------------- |
| `name`       | 是       | 学生姓名                     |
| `university` | 是       | 大学全名                     |
| `city`       | 是       | 大学所在城市，不需要写“市” |
| `latitude`   | 否       | 纬度，范围 -90 到 90         |
| `longitude`  | 否       | 经度，范围 -180 到 180       |

当目标大学无法被天地图自动定位到/定位不准确时，可手动提供坐标。若不提供坐标，系统会使用 `city + university` 调用天地图地理编码。

## 部署

这是纯静态项目，可部署到任意静态托管服务：

- GitHub Pages
- Vercel
- Netlify
- Nginx
- Apache
- 任意对象存储静态网站

部署前建议检查：

- 已替换 `js/config.js` 中的标题、口令哈希和天地图 TK。
- 已确认 `data/class*.js` 中不包含不应公开的真实个人信息。
- 已按实际数据数量设置 `CONFIG.classCount`。
- README 中不再保留临时口令或内部说明。

## 代码说明

- `index.html`：页面结构、模态框、动态数据脚本加载、天地图 API 加载。
- `css/style.css`：页面布局、地图覆盖层、响应式样式、信息窗样式。
- `js/config.js`：页面标题、口令哈希、班级数量、班级颜色、天地图 TK 和性能参数。
- `js/main.js`：口令验证、地图初始化、地理编码队列、标记渲染、搜索、统计和缺失数据逻辑。
- `data/classN.js`：班级演示数据。

## 浏览器要求

推荐使用现代浏览器：

- Chrome 90+
- Edge 90+
- Firefox 90+
- Safari 15+

关键依赖：

- Web Crypto API
- CSS 变量
- SVG data URI
- 天地图 JavaScript API v4.0

## 隐私与数据安全提示

这个模板是前端静态页面。所有数据文件都会被浏览器下载，口令验证只能降低误访问概率，不能作为真正的数据访问控制。若数据包含真实个人信息，请在发布前确认：

- 已获得授权。
- 数据脱敏或最小化。
- 不在公开仓库提交敏感信息。
- 不把私人口令、内部说明或未公开名单写入前端代码。

如需严格访问控制，应将页面部署在带有服务端鉴权的系统之后。

## 许可

本项目采用 MIT License。详见 [LICENSE](LICENSE)。

内置演示姓名均为虚构，仅用于展示数据格式和页面效果。
