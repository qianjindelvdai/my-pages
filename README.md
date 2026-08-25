# 发布到 Gitee Pages（稳定永久链接）

本目录是可直接部署的静态网站：
- `index.html`        → 工作台（工作/学习/生活）
- `brain-swap/index.html` → 换脑工程动画

## 一、注册 Gitee（约 2 分钟）
1. 打开 https://gitee.com ，点右上角「注册」
2. 用手机号 + 验证码注册（国内手机号即可），填用户名/邮箱
3. 登录后，点右上角「+」→「新建仓库」
   - 仓库名随意，如 `my-pages`
   - **可见性**：想公开分享选「公开」；工作台含内部信息务必选「私有」（私有仓库开 Pages 需 Gitee 会员/企业版）
   - 勾选「使用 Readme 初始化」可不勾，下面会直接推

## 二、部署（注册完在本机执行）
把下面 `你的用户名` 和 `my-pages` 换成你自己的，在 Git Bash 里跑：

```bash
cd "C:/Users/yuanzhiqian/WorkBuddy/Claw/发布-gitee"
git init
git add .
git commit -m "init pages"
git branch -M main
git remote add origin https://gitee.com/你的用户名/my-pages.git
git push -u origin main
```

## 三、开 Gitee Pages
1. 进仓库 → 顶部「服务」→「Gitee Pages」
2. 部署分支选 `main`，部署目录选「根目录」
3. 点「启动」

生成的稳定链接：
- 工作台：  https://你的用户名.gitee.io/my-pages/
- 换脑动画：https://你的用户名.gitee.io/my-pages/brain-swap/

链接永久不变（只要仓库不删）。以后改内容，本地改完再 `git add . && git commit -m u && git push` 即可，Gitee Pages 会自动更新。
