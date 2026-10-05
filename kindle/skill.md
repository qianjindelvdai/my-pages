---
name: kindle-jailbreak
description: "在受限网络（国内）下给旧款 Kindle 越狱并安装 KOReader，全程离线化 + USB 侧载 + telnet 系统级调试。解决 WinterBreak 卡死、KPM 包被墙掐断、图书库 .sh Scriptlet 不显示等疑难。Trigger phrases: kindle 越狱, kindle 装 koreader, WinterBreak, KPM, sh_integration, 图书库看不到脚本, kindle jb."
agent_created: true
---

# Kindle 越狱 + KOReader 安装（受限网络实战）

面向国内网络环境（GitHub / kindlemodding.org 被墙）的 Kindle 越狱全流程。
所有步骤都在设备型号 **Kindle 7 代 (PW2)、固件 5.12.2.2** 上实测通过。

## 0. 关键前提：网络

- `kindlemodding.org` 主站**直连被墙**，`repo.kindlemodding.org`（包仓库）**多数时候直连可达**——下载包优先走它。
- GitHub raw / releases 走镜像前缀：`https://gh-proxy.com/` 或 `https://ghfast.top/`。
- 会下载到 **Git LFS 指针（133 字节）** 时，换 `media.githubusercontent.com/media/...`；仍被拦则直接试 `repo.kindlemodding.org` 原始路径。
- **任何图片/HTML 形式的"下载结果"都要先看文件头**：曾经下到的所谓 `.kpkg` 其实是运营商中文拦截页（`<html>` 开头）。用 `head -c 16 | xxd` 或 `file` 验证魔数（真包 = gzip `1f 8b`）。

## 1. WinterBreak 越狱（离线化改造是核心）

1. 解压 `WinterBreak.tar.gz` → 根目录放三件套：`.active_content_sandbox`、`apps`、`mesquito`。
2. **默认流程会失败**：`apps/tech.hackerdude.winterbreak/dialoger.html` 里的 `source_command` 默认是运行时 `curl -L https://kindlemodding.org/jb.sh | sh`（netid:1 强制联网），国内必卡 "Please Wait"。
3. **离线化**：
   - 从 `KindleModding/jb.sh` GitHub releases 下最新 `jb.sh`（v2.x，约 5.3MB 自包含，无后续联网），拷到 Kindle 根目录；
   - 改 `dialoger.html` 的 `source_command` 为：
     `sh -c 'JB_HEADER="Winterbreak Jailbreak" sh /mnt/us/jb.sh'`
4. **触发文件（都放 Kindle 根目录）**：
   - `jb.sh.runmode1` —— 强制完整重装（否则检测到已越狱会静默 exit，见下）；
   - `jb.sh.debug` —— 详细日志写 `/mnt/us/jb.sh.log` + **自动开 telnetd:23 root shell**。
5. 打开**商店** → WinterBreak 界面弹出 → 点运行。成功标志：屏幕出现 "Finished installing jailbreak / ready to install the hotfix"，日志里可读到 `Installing libkh` / `Setting up hotfix and emergency runners` / `OTA blocking` / `Restarting GUI`；根目录生成 `documents/JAILBROKEN.txt`（"You are jailbroken!"）。
6. **新版无需手动装 Hotfix**：jb.sh v2.x 完整运行时已自动装 patch_system.sh（即旧版 Hotfix）+ KPM + OTA 阻断。"Update Your Kindle" 菜单灰色是正常的，别纠结。
7. `update.bin.tmp.partial`（0 字节，根目录）是 **jb.sh 写的 OTA 阻断文件**，**保留勿删**。

### 反复踩的坑
- **商店每次开 WiFi 都会重写 `.active_content_sandbox`**，把特制缓存刷回官方版 → WinterBreak 弹窗消失。重试前必须把特制缓存从 PC 覆盖回设备（含删掉 `store/resource/LocalStorage/` 里亚马逊写的"弹窗已看过"状态文件）。
- `jb.sh` 逻辑：`if [ $RUN_MODE -eq 0 ] && [ $JAILBROKEN -eq 1 ]; then exit; fi`——已越狱后普通运行会秒退，屏幕显示 "Device already jailbroken - exiting"。
- **电子墨水屏残影**：屏幕上的大字可能是上一次运行留下的残影，不代表本次真的跑了。以 `jb.sh.log` 时间戳为准。

## 2. 安装 KOReader（KPM 路线 + 手动兜底）

- 设备端命令（搜索框输入）：`;kpm update` → `;kpm install koreader` → `;kpm launch koreader`。
- **`;kpm install` 会先联网更新索引**，网络卡则"没反应"；`file://` 本地路径**不被支持**。
- `.kpkg` 真包地址：`https://repo.kindlemodding.org/packages/koreader/artifacts/koreader_<版本>_kindlepw2.kpkg`（约 41.7MB，gzip）。
- **被墙掐断的后果**：KPM 会解出残缺包（缺 `koreader.sh`/`frontend/`/`install.sh`），但 DB 里照样登记"已安装" → `launch` 无入口。
- **手动兜底部署**（本质 = install.sh 干的事，PC 上直接做）：
  1. 官方 `koreader-kindlepw2-vX.zip` 或真 `.kpkg` 解压，把 `koreader/` 整体覆盖到 `/mnt/us/koreader/`；
  2. 把包里的 `scriptlets/KOReader.sh`（**120KB，带 Name/Author/Icon 元数据头**）放进 `/mnt/us/documents/`——**不要**手写裸脚本（42 字节的裸 `#!/bin/sh` 不会被索引成书）；
  3. （KPM 登记可选）`installed_packages` 表结构见 KPM 源码 `src/kpm.c`。

## 3. Scriptlet / 图书库不显示 .sh 的排查（最难的一环）

**机制**（hdnext 栈，KUAL 已废弃）：
- `sh_integration` 在 `/var/local/appreg.db` 注册 `tech.hackerdude.shell_integration.extractor` 提取器 + launcher，`sh` → `MT:text/x-shellscript`。
- `documents/*.sh` 会被索引成图书库里的"书"，点开即运行。

**诊断用 telnet（有 `jb.sh.debug` 时 192.168.x.x:23 是 root shell）**：
- `sqlite3 /var/local/appreg.db "SELECT * FROM properties"` 验证注册（应有 launcher/extractor/mimetype 行）。
- 图书库真正的内容库是 **`/var/local/cc.db`**（日志里 "CC DB"），表 = `Entries`。

### ⚠️ 头号杀手：设备 sqlite3 未编译 ICU
- `cc.db` 的文本列用 ICU 排序；设备端 `sqlite3` 无 ICU → 任何触碰 collation 列的 `UPDATE`/`INSERT`/`DELETE` 报 **`no such collation sequence: icu`**。
- **规避法：只按 `rowid`（整数主键）定位，只改安全字段**；不改 `p_titles_0_collation` / `p_lastAccess` / `p_credits_*_collation`；不新建行、不删行。
- 本地可复现：PC 上 `sqlite3.connect(db)` **不要** `create_collation('icu',...)`，即模拟设备环境逐句试跑。

### 正确字段（照一条正常条目克隆）
| 字段 | 值 |
|---|---|
| `p_type` | `Entry:Item`（**不能**是 `Entry:Item:Feedback` 等非法值） |
| `p_mimeType` | `text/x-shellscript` |
| `p_cdeType` | `PDOC` |
| `p_cdeKey` | `'*' + sha1(文件绝对路径)`（实测：sha1("/mnt/us/documents/JAILBROKEN.txt") = `31f9f9b0…`） |
| 标题 | `j_titles` + `p_titles_0_nominal` |
| 大小 | `p_diskUsage` / `p_contentSize` / `p_totalContentSize` |

### 绕开设备限制的两个手段
1. **注入 dialoger**：把修复脚本前置到 `dialoger.html` 的 `source_command`：`sh -c 'sh /mnt/us/fix_cc.sh; JB_HEADER=... sh /mnt/us/jb.sh'`。
2. **WebKit 缓存是隐形杀手**：改了 `dialoger.html` 但设备仍跑旧命令 → 因为 Mesquito 框架按 URL 缓存。**建一个全新 id 的应用**（如 `apps/tech.hackerdude.fixer/`，含 `manifest.json`/`index.html`/`dialoger.html`/`icon.png`）即可绕开；或改 `unique_id`。
3. **把数据库搬出来解剖**：修复脚本里 `cp /var/local/cc.db /mnt/us/_cc.db`，PC 上细看真实表结构再写准 SQL。

## 4. Windows 侧文件锁怪象（务必遵守）
- Kindle U 盘上某些文件（`jb.sh`/`dialoger.html`/`kpm.db`/`fix_cc.sh`）会被 Windows 诡异锁住：**写操作静默无效**（cp / Copy-Item / Python 原地覆写都"成功"但内容没变），但 **`rm` 删除通常能成功**。
- **突破法：「删了再拷」**（`rm -f` 后 sleep 再 cp）。
- **铁律：每次拷贝后用 sha256 复核**（`python -c "hashlib.sha256(...)"` 比对 PC 与设备），否则会在"以为改好了"的假象里绕圈。
- 盘符探测：`for d in d e f g h i j k; do [ -d "/$d/documents" ] && [ -d "/$d/system" ] && echo "/$d"; done`。
- 写 `.sh` 必须 **LF 换行**（先 `d.replace(b'\r\n', b'\n')` 再拷）。

## 5. 装微信读书（= KOReader 插件，不是安卓 App）

**结论先行**：微信读书官方「墨水屏版」是 **Android APK**（`com.tencent.weread.eink`），Kindle 是 Linux 系统，**装不了**。Kindle 上读微信读书只有三条路：
1. **网页版（0 元，体验差）**：Kindle 体验版浏览器开 `r.qq.com`，扫码登录。能读、能同步进度/时长，但翻页有"网页感"、需联网、无评论区。
2. **越狱 + KOReader + `weread.koplugin` 插件（0 元，推荐，完整体验）**：扫码登录、书架同步、整本离线下载、划线想法、书评、时长上报统计。
3. 刷 CrackDroid（安卓）系统 / 买官方墨水屏设备——前者坑深（老版 WeRead 1.5.2、无背光调节、卡顿、可能要激活码），不推荐。

**插件安装（已实测可行）**：
- 仓库 `finlater/weread.koplugin`（AGPL-3.0），GitHub Releases 下 `weread.koplugin-vX.Y.Z.zip`（镜像 `https://gh-proxy.com/https://github.com/...`）。
- **要求 KOReader ≥ 2026.03**（旧版"工具"菜单里不出现"微信读书"；2026.7.2 ✅）。
- 解压后得到 `weread.koplugin/` 文件夹，**注意只能有一层目录**：正确 `koreader/plugins/weread.koplugin/main.lua`；错误 `.../weread.koplugin/weread.koplugin/main.lua`（多套一层就不显示）。
- 拷到设备 `/mnt/us/koreader/plugins/weread.koplugin/`（= KOReader 安装目录下的 `plugins/`，同目录还有 archiveviewer.koplugin 等）。
- 重启 KOReader → 菜单 **工具 → 微信读书**。
- 登录：手机微信读书 App → 我 → 设置 → **微信读书 Skill** → 获取 API Key；再在 Kindle 上 `工具→微信读书→微信扫码登录`，微信扫码，手机显示四位验证码时在 Kindle 输入。
- 备选：`miumiupy98-art/weread.koplugin-fixed`（增强 fork，支持插件内扫码、OTA、带划线/想法的 EPUB）。
- **实测记录（2026-10-05）**：PW2 / 5.12.2.2 + KOReader 2026.7.2 装 `weread.koplugin` **v1.5.3（91 个文件）成功**，拷完用递归 sha256 全量比对（PC vs 设备）确保无静默丢文件；目标目录不存在时 `cp -r` 不会触发 Windows 文件锁。
- 后续升级：插件内「微信读书 → 设置 → 更新管理」可在线更新（默认优先走代理，代理失败自动回退 GitHub）。

## 6. 使用注意（交付给用户时务必说明）
- **KOReader 运行中插 USB → Kindle 只读挂载**，写入全拒。必须先退出 KOReader、重启，看到 "USB 驱动器模式" 画面再插。
- **OTA 已阻断**，但别手滑去点系统更新。
- 开 WiFi 开商店会刷掉特制缓存（WinterBreak 弹窗消失）——属正常，不影响已装的 KOReader。
- 收尾清理：删 `jb.sh.debug`、删根目录 `.kpkg`（40+MB）和 `fix_cc*.sh`。
- ⚠️ **调试补丁必须显式还原（血泪教训）**：为远程调试给 `jb.sh` 注入过 telnetd 后门（`telnetd -l /bin/sh -p 23 &` 无条件启动）。**只删 `jb.sh.debug` 关不掉它**——只要脚本被再次执行（点一下 WinterBreak 运行），后门照开。**必须用官方原版整体回写 `jb.sh`**（打补丁前先 `cp jb.sh jb.sh.pwr` 备份，收尾时 `rm -f /mnt/us/jb.sh` 再 `cp jb.sh.pwr`，并 sha256 复核 + `grep -n telnetd` 确认只剩官方 debug 分支那一处）。给任何越狱脚本打补丁，都要在收尾时把本体换回净版。
- 保留 `update.bin.tmp.partial`（OTA 阻断文件，删了会恢复自动升级）；根目录可能存在用户自己的旧文件（如 `screenshot_*.png`），清理前先甄别、勿误删。
