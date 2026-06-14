# WSL 开发环境配置记录

> 时间：2026年6月14日  
> 设备：宿舍台式机（Ryzen 7 9700X，RTX 5070）  
> 系统：Windows 11 + WSL2 Ubuntu

---

## 一、网络问题排查

**症状**：curl/git 访问 GitHub 全部失败

**根因**：两层问题叠加
1. Steam++（Watt Toolkit）修改了 Windows hosts 文件，把 `github.com` 等大量域名指向 `127.0.0.1`
2. WSL 的 DNS 继承了 Windows 的错误解析结果

**解决**：
- 删除 hosts 文件中 `# Steam++ Start` 到 `# Steam++ End` 的全部内容
- 卸载 Watt Toolkit
- 固定 WSL DNS：`/etc/resolv.conf` 写入 `nameserver 8.8.8.8`，`/etc/wsl.conf` 加 `generateResolvConf=false`
- WSL 切换镜像网络模式（`~/.wslconfig` 加 `networkingMode=mirrored`）
- 代理指向本机 Clash：`http_proxy=http://127.0.0.1:7890`

**验证链路**：
```bash
ping 8.8.8.8          # 第一层：基础网络
nslookup github.com 8.8.8.8   # 第二层：DNS
curl -v http://127.0.0.1:7890  # 第三层：代理连通性
curl -I https://github.com     # 第四层：全链路
```

---

## 二、Shell 环境

**安装 zsh + Oh My Zsh**
```bash
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

**插件**
```bash
# 语法高亮
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# 自动补全
git clone https://github.com/zsh-users/zsh-autosuggestions.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

`~/.zshrc` 中启用：
```
plugins=(git zsh-syntax-highlighting zsh-autosuggestions)
```

**主题：Powerlevel10k**
```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
```

`~/.zshrc` 中设置：
```
ZSH_THEME="powerlevel10k/powerlevel10k"
```

字体：Windows Terminal 字体改为 `MesloLGS NF`（需提前安装四个字重）

**代理写入 `~/.zshrc`**
```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```

---

## 三、Vim 配置

**插件管理器：vim-plug**
```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
  https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

**`~/.vimrc` 配置**
```vim
set number
set relativenumber
syntax on
colorscheme gruvbox
set background=dark

call plug#begin('~/.vim/plugged')
Plug 'vim-airline/vim-airline'
Plug 'vim-airline/vim-airline-themes'
call plug#end()

let g:airline_theme='simple'
```

配色：gruvbox（需提前下载到 `~/.vim/colors/gruvbox.vim`）

---

## 四、Git 配置

```bash
git config --global user.email "邮箱"
git config --global user.name "名字"
git config --global core.editor vim
git config --global credential.helper store
```

**apt 走代理**
```bash
echo 'Acquire::http::Proxy "http://127.0.0.1:7890";' | sudo tee /etc/apt/apt.conf.d/proxy.conf
echo 'Acquire::https::Proxy "http://127.0.0.1:7890";' | sudo tee -a /etc/apt/apt.conf.d/proxy.conf
```

**GitHub 推送**：密码用 Personal Access Token（勾选 repo 权限，建议 No expiration）

---

## 五、踩坑记录

| 问题 | 原因 | 解法 |
|------|------|------|
| github.com 解析到 127.0.0.1 | Steam++ 污染 hosts | 删 hosts 条目，卸载 Steam++ |
| HTTPS 出不去 | 无代理配置 | Clash 开 Allow LAN，配 http_proxy |
| WSL localhost 代理警告 | NAT 模式限制 | 切镜像网络模式 |
| apt 装包失败 | apt 不走系统代理 | 写 /etc/apt/apt.conf.d/proxy.conf |
| git commit 进了 nano | 默认编辑器未设置 | git config core.editor vim |
| g++ 找不到 cout | 用了 gcc 编译 cpp | 改用 g++ |