# Dotfiles 工程与发布

> 时间：2026年6月24日
> 设备：宿舍台式机（Ryzen 7 9700X，RTX 5070）
> 系统：Windows 11 + WSL2 Ubuntu
> 背景：MIT Missing Semester / Command-line Environment / Aliases and Dotfiles
> 目标：把 zsh/tmux/vim/git 配置收编成可一键安装的版本控制仓库，两台设备（台式机 + 老家笔记本）同步

---

## 〇、核心理解（先记结论）

1. **dotfiles 的本质是软链接，不是复制。** 仓库里的真实文件 → `~/.xxx` 建软链接。这样"编辑配置"和"管理版本"变成同一件事；用 `cp` 则两边脱钩，版本控制名存实亡。
2. **dotfiles 管的是"配置文件（指令）"，不是"配置依赖的软件（建材）"。** `zshrc` 只是"去加载 OMZ、用 p10k 主题"的指令，OMZ / p10k / 插件本身是另外几千个文件，不在仓库里。所以一份 zshrc 搬到新机器，软链接能建、但一 `source` 就炸——install 脚本除了建链接，还要负责装依赖。
3. **命名约定决定脚本能不能优雅。** 仓库存不带点的名字（`zshrc`、`tmux.conf`），靠 install 脚本统一链到 `~/.xxx`。规整的命名让一个 for 循环就能处理所有文件。

---

## 一、建仓库与软链接接管

```bash
mkdir ~/dotfiles && cd ~/dotfiles
git init

cp ~/.zshrc ~/dotfiles/zshrc        # 复制进仓库（不带点命名）。注意是 cp 不是 mv
```

install 脚本（用循环，加新配置只需往 in 后面加名字）：

```bash
#!/usr/bin/env bash
BASEDIR=$(cd "$(dirname "$0")" && pwd)   # 动态定位仓库根目录，与运行位置无关

for name in zshrc tmux.conf vimrc gitconfig; do
    ln -sf "$BASEDIR/$name" "$HOME/.$name"
done
```

**要理解的点：**

- `BASEDIR` 用 `dirname $0` + `cd && pwd` 拿绝对路径 → 脚本搬到任何机器、任何路径都能跑（可移植性的关键）。
- `ln -sf` 的 `-f`：目标已存在时**强制覆盖**。有破坏性——会先删掉新机器上原有的 `~/.zshrc`。生产级脚本应先备份（`mv ~/.zshrc ~/.zshrc.bak`）。
- 软链接源必须用**绝对路径**，否则解析相对于链接所在位置（`~/`），容易断。

验证接管成功：

```bash
ls -l ~/.zshrc          # 开头是 l（link），末尾有 -> 仓库路径 的箭头
readlink ~/.zshrc       # 直接打印指向，应为仓库内绝对路径
```

---

## 二、.gitignore（迁配置前先立防线）

dotfiles 目录天生混着"配置"和"运行时垃圾/凭据"，`.gitignore` 是主动防线，不是美化。

```gitignore
# 运行时/缓存，不是配置本身
*.swp
*~
.DS_Store
.netrwhist

# 历史与补全缓存，可能含敏感信息
.zsh_history
.zcompdump*

# 凭据类，永远不进仓库（哪怕私有仓库也别 commit，git 历史难彻底清除）
.git-credentials
*.key
*.pem
id_rsa
id_ed25519
```

---

## 三、在干净环境测安装脚本

不必开真 VM，**新建一个 Linux 用户**即可——新用户的家目录就是"全新机器"，隔离粒度刚好匹配"测软链接"这个目的。

```bash
sudo useradd -m -s /bin/bash testuser
sudo passwd testuser            # 给 testuser 设密码（不要旧密码，直接输两遍新的）
su - testuser                   # 切过去；sudo 用自己的密码，su 用 testuser 的密码——别混

# 在 testuser 下，从 GitHub clone（需仓库临时 public，或本地拷贝）
git clone https://github.com/<user>/dotfiles.git
cd dotfiles && ./install
readlink ~/.zshrc               # 应指向 /home/testuser/...，证明脚本不依赖原机器路径

# 测完拆干净
exit
sudo userdel -r testuser        # -r 连家目录一起删
```

**实测暴露的关键一课**：在 testuser 里 `zsh` 启动报
`source:84: no such file or directory: /home/testuser/.oh-my-zsh/oh-my-zsh.sh`
——软链接成功了，但 OMZ 不在仓库里所以 source 失败。这就是〇.2"配置 vs 依赖"的现场。install 脚本还缺"装依赖"那一半（clone OMZ、装 p10k/插件、装 tmux TPM 等），见 WSL setup.md 第二节的安装命令。

---

## 四、发布到 GitHub

```bash
git remote add origin git@github.com:<user>/dotfiles.git   # 用 SSH，免密且不走易污染的 https 路径
git branch -M main            # 本地 master 改名 main，对齐 GitHub 默认
git push -u origin main       # -u 关联本地与远程，之后直接 git push
```

**可见性的不对称**：public ↔ private 可随时切，但**公开过就要当已泄露**——公开期间可能已被爬虫/fork 抓走，改回 private 挡不住已有副本。所以拿不准先 private，确认无凭据再转 public。（迁 git/ssh 配置时最易手滑带进敏感信息，更要在 private 下做。）

---

## 五、踩坑记录

### 5.1 DNS 二次复发（6-14 修复失效）

> 详见 WSL setup.md 第一节。此处是 6-24 复发，根因不同，需补充。

**症状**：`Temporary failure in name resolution`，所有域名解析失败（不只 github）。

**根因**：6-14 写的"`/etc/resolv.conf` 静态 nameserver 8.8.8.8 + `generateResolvConf=false`"**被 systemd-resolved 二次覆盖**。重启后 resolv.conf 被改回指向 `127.0.0.53` 的符号链接，而该 stub 无可用上游（`resolvectl status` 显示 `Current Scopes: none`）。**`generateResolvConf=false` 只挡 WSL 自动生成，挡不住 systemd-resolved 接管**——这是 6-14 没意识到的盲区。

**诊断顺序**：

```bash
cat /etc/resolv.conf        # 是否又变回指向 127.0.0.53 的符号链接
resolvectl status           # 看 Scopes 是否 none、有无 DNS Servers
ip route                    # 确认默认路由走哪块网卡（本机是 eth1，mirrored 模式）
```

**解法**：

```bash
sudo rm /etc/resolv.conf
sudo bash -c 'printf "nameserver 8.8.8.8\nnameserver 223.5.5.5\n" > /etc/resolv.conf'
# 若仍被覆盖，彻底停掉接管者：
sudo systemctl disable --now systemd-resolved
```

**实测教训**：`dig @8.8.8.8 github.com` vs `dig @223.5.5.5 github.com`，本机网络下 **8.8.8.8 秒回（7ms）、阿里 223.5.5.5 全超时**（与 Clash 分流有关）。别凭"国内用阿里更稳"的常识，`dig` 实测为准。两个 DNS 都写进 resolv.conf，8.8.8.8 在前、223.5.5.5 兜底，换网络环境时自动 fallback。

### 5.2 git push 报 Permission denied (publickey)

**这不是网络问题**——`ssh -T git@github.com` 已经连上（返回了 host key），卡在"连上之后 GitHub 不认你是谁"，属于**身份认证层**，与 DNS（解析层）是两回事。

**根因**：本机 SSH key 已生成（`~/.ssh/id_ed25519`），但**公钥没加到 GitHub 账号**。

**解法**：`cat ~/.ssh/id_ed25519.pub` → 整行（`ssh-ed25519 ... 注释`，三段都要）贴到 GitHub Settings → SSH and GPG keys → New SSH key。验证 `ssh -T git@github.com` 显示 `Hi <user>!` 即通。

### 5.3 排查"连不上"的分层习惯

| 层次 | 问什么 | 典型报错 | 测法 |
| --- | --- | --- | --- |
| 解析层 | 找得到对方吗 | `Temporary failure in name resolution` | `dig` / `ping` 能否解析出 IP |
| 认证层 | 对方认你吗 | `Permission denied (publickey)` | 解析正常但 ssh/git 报 auth |

先分清在哪层，再对症。解析不出是 DNS，解析出来但被拒是身份——别把两层搅在一起。

---

## 六、待办（install 装依赖那一半）

当前 install 只建软链接，缺装依赖。回老家笔记本配置时按真实报错驱动补全：检测/安装 zsh → clone OMZ → 装 p10k 主题 + syntax-highlighting/autosuggestions 插件 → 装 tmux TPM → vim-plug。命令见 WSL setup.md 第二、三节。
