Ubuntu 24.04 官方源已移除 `fcitx-googlepinyin`，**推荐用 Fcitx5 + 云拼音 / 智能拼音替代**（体验接近谷歌拼音）；若必须用原版谷歌拼音，需添加第三方 PPA。下面分两种完整方案：

### 一、推荐方案：Fcitx5 + 云拼音（最稳、接近谷歌拼音）

#### 1. 安装 Fcitx5 框架与中文组件

bash

运行

```bash
sudo apt update
sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 fcitx5-frontend-qt5 fcitx5-module-cloudpinyin
```

#### 2. 设置 Fcitx5 为默认输入法框架

bash

运行

```bash
im-config -n fcitx5
```

弹出确认时选 **Yes**，再选 **fcitx5**，确认退出。

#### 3. 配置环境变量（Wayland/Xorg 兼容）

bash

运行

```bash
echo "GTK_IM_MODULE DEFAULT=fcitx5" >> ~/.pam_environment
echo "QT_IM_MODULE DEFAULT=fcitx5" >> ~/.pam_environment
echo "XMODIFIERS DEFAULT=@im=fcitx5" >> ~/.pam_environment
echo "SDL_IM_MODULE DEFAULT=fcitx5" >> ~/.pam_environment
```

#### 4. 重启 / 注销生效

必须**注销重新登录**（或重启），让框架与环境变量生效。

#### 5. 添加并启用拼音输入法

1. 打开 Fcitx5 配置：终端输入 `fcitx5-configtool`，或从应用列表打开 **Fcitx5 Configuration**
2. 点击左下角 **+** → 取消勾选 **Only Show Current Language**
3. 搜索 **Pinyin**（智能拼音）或 **Cloud Pinyin**（云拼音，词库更全），添加到列表
4. 调整顺序，把拼音放在前面；切换快捷键默认 **Ctrl + 空格**，可在配置里修改



