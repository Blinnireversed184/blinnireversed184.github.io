# 班级网盘 · 前端

班级共享网盘的门户前端（纯静态，**零第三方依赖**：不引 CDN、字体用系统栈），可直接部署到 GitHub Pages。

线上地址（部署后）：https://eternal2526.github.io/

## 现在能跑什么

现在跑的是 **演示模式（mock）**，不依赖任何后端：

- 演示账号：任意名字（推荐 `林小满` / `陈小雨` / `王浩` / `赵一鸣`），**密码 `2526`**
- 已实现：登录、左侧目录树、网格/列表切换、排序、按文件名/上传者搜索、
  缩略图（图片显示缩略图，视频在后端接好后显示首帧）、点击预览、下载、删除（只能删自己传的）、
  拖拽上传、**分片上传 + 断点续传 + 进度/速度/剩余时间**、移动端抽屉式目录

## 接真实后端（只改一个开关）

`assets/js/api.js` 顶部：

```js
const API = {
  MOCK: true,                        // ← 改成 false
  base: 'https://你的后端域名:8443',  // ← 填后端地址
  ...
```

## 后端契约（门户服务要实现这 9 个接口）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/login` | `{username,password}` → `{token,user:{id,name,nick,role}}` |
| POST | `/api/register` | `{username,password,invite}` → 同上（**用户名支持中文，且兼作显示名**；邀请码 = 班级口令） |
| POST | `/api/logout` | Bearer token |
| GET | `/api/me` | → `{user}` |
| GET | `/api/list?path=/学习资料` | → `{path,items:[Item]}` |
| GET | `/api/search?q=关键词&scope=all\|mine` | → `{items:[Item]}` |
| POST | `/api/mkdir` | `{path}` |
| POST | `/api/delete` | `{paths:[...]}`，**服务端必须校验"只能删自己传的"** |
| POST | `/api/upload?uploadId=&path=&chunk=&chunks=&total=` | body=分片原始字节 → `{done:false}` \| `{done:true,item}` |
| GET | `/api/upload/status?uploadId=` | → `{received:[0,2,3]}` ← 断点续传 |
| GET | `/api/raw?path=/a.jpg` | 302 直链或文件流（下载/预览） |
| GET | `/api/thumb?path=&w=480` | 缩略图字节（视频封面后端 ffmpeg 生成） |

```ts
Item = {
  id: string, name: string, path: string, parent: string,
  isDir: boolean, size: number, mtime: number,
  uploader: string,                        // 上传者名字，界面要显示
  kind: 'image'|'video'|'audio'|'pdf'|'doc'|'sheet'|'slide'|'code'|'archive'|'text'|'other'|'dir',
  thumb?: string|null                      // 后端可给缩略图 URL
}
```

分片约定：前端按 **2MB** 切片，`uploadId` 由「用户+目录+文件名+大小+修改时间」哈希得出（断点续传靠它）。
服务端按 `uploadId` 记录已收分片，`GET /api/upload/status` 返回已收到的下标数组。

## 目录

```
index.html            单页（登录 + 网盘界面）
assets/style.css      绿色主题 / 卡片式 / 移动优先 / 自动深色模式
assets/js/api.js      ★ 接口层：mock 后端 + 真实后端 + API 契约
assets/js/app.js      界面逻辑（无依赖）
```

## 本地预览

```bash
python3 -m http.server 8080     # 然后打开 http://127.0.0.1:8080/
```

## 后端部署现状（家里服务器 N1）

```
systemd 服务 classdrive  →  /opt/classdrive/portal.py（纯 Python 标准库，无依赖）
                          环境变量写在 /etc/systemd/system/classdrive.service
存储根（移动硬盘）       →  /mnt/disk/ClassDisk/
监听                     →  0.0.0.0:8081（局域网；公网入口待解决）
内存占用                 →  ~14 MB    开机自启：已启用
```

目录结构：

```
/mnt/disk/ClassDisk/
├── 共享/ 学习资料/ 照片/ 视频/ 作业/     ← 共享目录树（可以直接在盘上手动放）
└── .portal/                              ← 元数据，别手动改
    ├── portal.db    账号 / 归属 / 分片进度（SQLite，备份它就是备份账号体系）
    ├── thumbs/      缩略图封面（前端生成 JPEG 后回传，服务端不装 ffmpeg）
    ├── tmp/         上传中的分片
    └── trash/       覆盖上传时旧版本的去处
```

常用运维：

```bash
systemctl status classdrive        # 看状态
systemctl restart classdrive       # 重启
journalctl -u classdrive -f        # 看日志

# 改存储根：改服务单元里的 CD_ROOT 后 daemon-reload + restart
# 改班级口令：改 CD_INVITE，或在库里 UPDATE settings SET v='新口令' WHERE k='invite'

# 改管理员密码（在 N1 上跑）：
python3 - <<'PY'
import hashlib,secrets,sqlite3
c=sqlite3.connect("/mnt/disk/ClassDisk/.portal/portal.db")
salt=secrets.token_hex(16)
h=hashlib.pbkdf2_hmac("sha256",b"新密码",salt.encode(),120000).hex()
c.execute("UPDATE users SET pw=?,salt=? WHERE name='admin'",(h,salt))
c.execute("DELETE FROM sessions"); c.commit(); print("已改")
PY
```

> 备份 = 复制 `/mnt/disk/ClassDisk/`（文件 + `.portal/`）即可；只复制文件会丢账号和"谁传的"。

## 待办

- [x] 门户后端（账号 / 归属 / 权限 / 上传落盘 / 缩略图）——已上线 N1，见下节
- [x] 分片上传 + 断点续传 + Range 下载的真机联调
- [ ] **公网入口（唯一大石头）**：IPv6+DDNS+证书，或对象存储。现在后端是局域网 http，页面上了 GitHub Pages（HTTPS）会被浏览器按"混合内容"拦掉
- [ ] 改密码页、管理端（删别人的文件、重置口令、看上传日志）
- [ ] 视频封面已实现（前端抓首帧回传）；注意只支持浏览器能解的编码（mp4/H.264 最稳）
