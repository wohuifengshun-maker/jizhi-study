# 如何上传到你的 GitHub（保留版本记录）

本仓库已经初始化好 Git 并包含 3 次提交（对应 v1/v2/v3 三个迭代）。
你只需把它推送到自己的 GitHub 账号即可，**版本历史会原样保留**。

## 一、先在 GitHub 上建一个空仓库

1. 登录 github.com，点右上角 `+` → `New repository`
2. 仓库名填：`jizhi-study`（或你喜欢的名字）
3. 选 **Private**（推荐，含产品思路，先别公开）
4. **不要**勾选 "Add a README"（本地已有，避免冲突）
5. 点 `Create repository`

## 二、把本地仓库推上去

在本仓库文件夹里依次执行（把 `你的用户名` 换成你的 GitHub 用户名）：

```bash
# 关联远程仓库
git remote add origin https://github.com/你的用户名/jizhi-study.git

# 推送主分支与全部历史
git branch -M main
git push -u origin main
```

第一次推送会要求登录。现在 GitHub 不再用密码，需用 **Personal Access Token**：
- GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token
- 勾选 `repo` 权限，生成后复制
- 推送时用户名填 GitHub 用户名，密码处粘贴这个 token

## 三、之后每次改动如何记录版本

```bash
git add -A
git commit -m "说明这次改了什么"
git push
```

每个 commit 就是一个版本节点，在 GitHub 仓库的 "commits" 页能看到完整历史，
和你参考的 CFP-Study 仓库一样。

## 四、查看当前版本历史

```bash
git log --oneline
```

应能看到三条提交，从初版原型到最新的足迹 + 雷达版本。

---

### 提示

- `materials/`、`.env`、课件 PDF/PPTX 已在 `.gitignore` 中排除，**不会上传**，保护版权和密钥。
- 若想让别人也能在线打开演示，可在仓库 Settings → Pages 开启 GitHub Pages，
  指向 `prototypes/latest.html`。
