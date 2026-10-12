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
监听                     →  [::]:8081（**IPv4+IPv6 双栈**，`DualStackHTTPServer`）
内存占用                 →  ~14 MB    开机自启：已启用
```

> ⚠️ **为什么必须双栈**：Python 的 `ThreadingHTTPServer((host, port))` 默认 `AF_INET`，只监听
> `0.0.0.0` —— 公网 IPv6 连进来会失败，现象和"路由器拦入站"一模一样（本项目为此白查了一轮）。
> 现在改成 `address_family = socket.AF_INET6` + `IPV6_V6ONLY=0`、bind `::`，`ss` 显示 `*:8081`。

## DDNS（dynv6 · 家宽 IPv6 动态解析）

电信下发的 /64 前缀会变（实测换过一次），所以用 DDNS 把域名顶到当前地址：

```
域名      classdisk.dynv6.net  （AAAA → N1 的公网 IPv6）
更新器    /opt/classdrive/ddns.py      环境变量 /etc/classdrive-ddns.env（600，只放 token）
定时器    classdrive-ddns.timer        开机后 90 秒起，每 5 分钟一次
状态文件  /var/lib/classdrive-ddns/state.json
```

选地址的逻辑：**按网卡 MAC 算 EUI-64 接口标识，只认 IID 匹配的那个全球地址**。
前缀变了内核会在新前缀下自动生成同一个 IID 的地址，脚本每次挑到的就是当前正确的那个，
不会误推旧前缀的残留地址。地址没变且一小时内有推送过就跳过（dynv6 会回收长期不更新的 zone，
所以有一小时一次的保活重推）。

```bash
# 手动跑一次（带 --dry 只打印不推送）
set -a; . /etc/classdrive-ddns.env; set +a; python3 /opt/classdrive/ddns.py
systemctl list-timers classdrive-ddns.timer        # 看下次何时跑
journalctl -u classdrive-ddns.service -n 20        # 看历史
# 换 token / 换域名：改 /etc/classdrive-ddns.env 后不用重启，下次触发即生效
```

## HTTPS 公网入口（nginx 9443 + Let's Encrypt DNS-01）

```
公网 HTTPS   https://classdisk.dynv6.net:9443   ← 前端 api.js 就指这里
             ↓ nginx（sites-enabled/classdrive，模板见 backend/nginx-classdrive.conf）
后端         http://127.0.0.1:8081              ← 8081 现在只对局域网开放
证书         /etc/nginx/ssl/classdisk.dynv6.net.{crt,key}   ECDSA P-256，到期 2027-01-10
签发/续期    acme.sh v3.1.7 + 自写 dynv6 钩子（纯 shell + python3，N1 上不编译）
             cron 每天 2/8/14/20 点检查，续期后自动 `systemctl reload nginx`
```

防火墙边界（ufw）：

```bash
9443/tcp             ALLOW  Anywhere        ← 公网 HTTPS（v4+v6）
8081/tcp             ALLOW  192.168.1.0/24  ← 只有局域网能直连后端
8081/tcp             ALLOW  192.168.2.0/24
```

签发相关文件（都在 N1 上，**仓库里只有不含凭据的脚本和模板**）：

```
/opt/classdrive/acme_dynv6.py                     # 用 dynv6 API 增删 _acme-challenge TXT
/root/.acme.sh/dnsapi/dns_classdynv6.sh           # acme.sh 的 DNS-01 钩子（调上面那个）
/root/.acme.sh/classdisk.dynv6.net_ecc/           # acme.sh 的证书工作目录
/etc/nginx/sites-enabled/classdrive               # 站点配置
```

> ⚠️ **踩过的坑：dynv6 API v2 的 `name` 必须是相对名。** apex 记录是空串，子域写 `_acme-challenge`
> 而不是 `_acme-challenge.classdisk.dynv6.net`；写全名的话记录不会出现在 DNS 上，Let's Encrypt
> 会直接报 `No TXT record found at …`（我第一次就是这么失败的）。

```bash
# 看证书还有多久过期
openssl x509 -in /etc/nginx/ssl/classdisk.dynv6.net.crt -noout -dates -subject
# 手动触发续期检查（不到期会 Skip）
/root/.acme.sh/acme.sh --cron --home /root/.acme.sh
# 强制立刻续期
/root/.acme.sh/acme.sh --renew -d classdisk.dynv6.net --force
```

## 目录结构

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
- [x] **公网入口**：IPv6 直连 + dynv6 DDNS + Let's Encrypt 证书 + nginx 9443 反代（见上文三节）。前端 `api.js` 的 `base` 已指向 `https://classdisk.dynv6.net:9443`
- [ ] 剩下的都是前端体验项：改密码页、管理端（删任意文件 / 重置口令 / 看日志）、上传队列刷新后续传
- [ ] 视频封面已实现（前端抓首帧回传）；注意只支持浏览器能解的编码（mp4/H.264 最稳）
