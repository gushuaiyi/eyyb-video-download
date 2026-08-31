---
name: eyyb-video-download
description: 下载 E英语宝(eyyb.vip)教材课文动画高清视频。当用户说"帮我下载 XX教材 XX年级XX册 XX单元 的课文动画/视频"时使用本技能。
---

# E英语宝课文动画高清视频下载

按用户需求从 eyyb.vip(湘少版/人教PEP/湘鲁版等教材配套教师资源站)下载课文动画,输出 1080p 高清 mp4。

## 工作原理(背景知识)

- 网页播放器用的是 KMS 加密 HLS,**只有 480p**;而单元资源 API 返回的 `filePath` 是 oss.eyyb.vip 上的 **1080p 明文 mp4 直链**,画质更高且免解密——脚本优先走直链。
- 直链不可用时,脚本自动走 KMS 兜底链路:`kms/geturl` 拿 m3u8(自带 token)→ `kms/getkey` 拿 AES-128-CBC 密钥 → 解密 ts 分片 → ffmpeg 封装 mp4(已验证可用)。
- API 基址 `https://api-teacher.eyyb.vip/`,所有接口需要请求头 `authorization`(浏览器登录令牌)。`getunits` 接口的响应体 `obj` 是 AES-128-CBC(base64)加密,密钥 `eyyb201811438611`、IV `iveyyb2018456209`,脚本自动解密。

## 执行步骤

### 1. 定位脚本

工具脚本 `download_kwdh.js` 与本项目同目录(桌面文件夹 `31abc99d1a01bb7b78a171dc464a19dc`)。若当前目录没有,用 Glob 搜索 `**/download_kwdh.js` 定位,或直接使用该桌面文件夹路径。

### 2. 获取登录令牌(必需)

若本次会话还没有令牌:引导用户在已登录的 eyyb.vip 页面按 F12 → Console 执行:

```js
localStorage.getItem('authorization')
```

把输出(`ApiGateToken eyJ...` 格式)粘贴到对话。令牌只用于本次会话调用,不写入任何文件。

### 3. 把用户需求映射为参数

| 用户说的 | 环境变量 | 取值映射 |
|---|---|---|
| 教材版本 | `EYYB_VERSION` | `xsbnew`=湘少版(2024审定,默认);`xsb2`=湘少版旧教材;`pepnew`/`rjbnew`=人教PEP;`xlbnew`=湘鲁版(2024审定);`xlb`=湘鲁版旧;`pep`=人教三起旧;`rjxqd`=人教一起旧;`jkb`=教科版;`gzb`=广州版;`wyxbz`=外研版三起 |
| 年级/学期 | `EYYB_BOOK` | 书名关键词,如 `五年级上册`、`四年级下册`(默认 `五年级上册`) |
| 单元 | `EYYB_UNIT` | 单元名关键词,如 `Unit 1`、`Unit 3`(默认 `Unit 1`) |
| 部分 | `EYYB_PART` | 部分名关键词,如 `Let's Chant`、`A Look`(默认空=单元第一个部分;`EYYB_ALL=1`=该单元全部 A–H 部分) |
| 输出文件名 | `EYYB_OUT` | 可选,仅单文件模式生效 |

教材版本无法确定时优先试 `xsbnew`;失败时让脚本打印候选列表(见步骤 5)再换版本重试。

### 4. 运行下载

```bash
node download_kwdh.js '<令牌>'          # 单元第一个部分
EYYB_ALL=1 node download_kwdh.js '<令牌>'                 # 全部部分
EYYB_PART="Let's Chant" node download_kwdh.js '<令牌>'    # 指定部分
```

多文件模式输出到子目录 `<教材名>_<单元名>/`,文件名即部分名(如 `A Look, Listen and Say.mp4`)。

### 5. 验证与交付

- 用 `ffprobe` 抽查输出 mp4:分辨率应为 1920×1080(直链模式),时长与各部分相符(20~100 秒量级)
- 报告文件位置与大小给用户

### 6. 常见问题处理

- **返回 401 / 请求失败**:令牌过期或无效 → 回到步骤 2 重新取令牌
- **找不到书/单元**:脚本会打印候选列表(教材列表存 `_work/books.json`、单元列表存 `_work/units.json`)→ 按候选名调整 `EYYB_BOOK`/`EYYB_UNIT` 关键词重跑
- **部分名不匹配**:脚本会打印该单元全部可用部分名(如 A Look, Listen and Say / B Listen and Learn / C Let's Practise …)→ 用更精确的 `EYYB_PART` 关键词重跑
- **getkey 406**:仅 KMS 兜底路径可能遇到,新会话的 m3u8 自带 token 通常可解;仍失败则请用户在播放页 Network 面板复制 getkey 请求为 cURL 交给 Claude 处理
- **用户只说了教材名没给版本/年级**:先用默认版本+关键词尝试,脚本的候选列表会帮助收敛;无法确定时向用户确认

## 注意

- 视频仅限个人教学使用,勿二次分发
- 令牌视为敏感凭证,不落盘、不外发
