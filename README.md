# eyyb-video-download
下载 E英语宝(eyyb.vip)教材课文动画高清视频。当用户说"帮我下载 XX教材 XX年级XX册 XX单元 的课文动画/视频"时使用本技能。

注意：
若本次会话还没有令牌:引导用户在已登录的 eyyb.vip 页面按 F12 → Console 执行:
```js
localStorage.getItem('authorization')
```
把输出(`ApiGateToken eyJ...` 格式)粘贴到对话。令牌只用于本次会话调用,不写入任何文件。
