# Mac 微信双开教程

这篇教程教你保留原版微信，再做一个叫 **wechat-dev** 的副本，用来登录第二个账号。

已在微信 **4.1.13** 上验证创建、启动和 Finder 名称、图标；其他版本可能不同。按顺序操作，每段命令整段粘贴到终端后按回车。遇到报错先停下，不要继续下一步。

## 第一步：安装并打开原版微信

从官方渠道安装微信，放到 Mac 的“应用程序”文件夹，先打开一次，确认能正常使用。

下面的命令默认原版位于：

```text
/Applications/WeChat.app
```

**为什么先做这一步？** 后面要从这个官方 App 复制副本，先确认原版本身没有问题。

## 第二步：复制一个微信

按 **Command + 空格**，搜索“终端”并打开，执行：

```bash
/bin/bash <<'BASH'
set -euo pipefail
src="/Applications/WeChat.app"
dst="$HOME/Applications/wechat-dev.app"

if [ ! -d "$src" ]; then
  echo "找不到原版微信，请先完成第一步。"
  exit 1
fi
if [ -e "$dst" ]; then
  echo "副本已经存在，停止创建，避免覆盖。"
  exit 1
fi

mkdir -p "$HOME/Applications"
ditto "$src" "$dst"
echo "副本已创建，可以继续第三步。"
BASH
```

**这一步做了什么？** 把原版完整复制到个人的“Applications”文件夹，并将文件命名为 `wechat-dev.app`。原版保持原样。

看到“副本已创建”再继续。如果提示已经存在，不要覆盖；已完成设置的副本直接按第六步打开。

## 第三步：修改副本的应用标识和名称

**为什么复制后还要修改？** 改文件名不会改变应用内部的身份。需要给副本一个不同的 **Bundle ID（应用标识）**，才能让它以另一个应用的身份运行。

执行：

```bash
/bin/bash <<'BASH'
set -euo pipefail
dst="$HOME/Applications/wechat-dev.app"

/usr/libexec/PlistBuddy -c 'Set :CFBundleIdentifier com.tencent.xinWeChat.dual' "$dst/Contents/Info.plist"
/usr/bin/plutil -replace CFBundleName -string wechat-dev "$dst/Contents/Info.plist"
/usr/bin/plutil -replace CFBundleDisplayName -string wechat-dev "$dst/Contents/Info.plist"

for file in "$dst"/Contents/Resources/*.lproj/InfoPlist.strings; do
  [ -f "$file" ] || continue
  /usr/bin/plutil -replace CFBundleName -string wechat-dev "$file"
  /usr/bin/plutil -replace CFBundleDisplayName -string wechat-dev "$file"
done
echo "副本标识和名称已修改。"
BASH
```

这里也修改了微信各语言的名称配置。**只改主配置，中文或英文配置仍可能把显示名称覆盖回 WeChat。**

完成后，副本的应用名称为 `wechat-dev`；微信内部的窗口标题和菜单仍可能显示“微信”。

## 第四步：重新签名，并让系统识别副本

**为什么要重新签名？** 上一步改动了 App 内的文件，原来的签名不再匹配。下面会给副本重新做本地签名、检查完整性，再向系统注册它。

这是非官方的多开方法：本地签名会替换副本的原厂签名，本次实测也未保留原厂沙盒权限；原版不受影响。

```bash
/bin/bash <<'BASH'
set -euo pipefail
dst="$HOME/Applications/wechat-dev.app"

/usr/bin/codesign --force --deep --sign - --timestamp=none "$dst"
/usr/bin/codesign --verify --deep --strict "$dst"
"/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister" -f "$dst"
echo "签名检查通过，可以启动。"
BASH
```

出现 `replacing existing signature` 是替换签名的正常提示。看到“签名检查通过”再继续。

## 第五步：打开两个微信，分别登录

执行：

```bash
open "/Applications/WeChat.app"
open "$HOME/Applications/wechat-dev.app"
```

原版登录第一个账号，副本登录第二个账号。如果副本显示的是同一个账号，点 **“切换账号”**，再用第二个账号扫码。

**完成标准：两个窗口分别登录不同账号，并且都保持在线。** 本教程已验证副本能打开登录页，账号扫码和手机确认需要你自己完成。

## 第六步：以后怎么打开

以后不用再执行创建命令。

1. 打开 Finder，按 **Command + Shift + G**。
2. 输入 `~/Applications`，按回车。
3. 双击 **wechat-dev**；原版微信照常打开。
4. 副本启动后，右键 Dock 中对应图标，选择 **“选项 → 在程序坞中保留”**。

注意：Finder 侧栏的“应用程序”通常打开的是 `/Applications`，而副本放在个人目录 `~/Applications`，所以要按上面的路径找。

## 第七步：打开副本的通知

第一次启动副本后，打开 **系统设置 → 通知 → wechat-dev**。

把 **允许通知** 打开，并确认 **桌面**、**通知中心**、**锁屏** 和 **播放通知的声音** 已打开。

**为什么要单独设置？** `wechat-dev` 是一个新的应用标识，macOS 会把它当成新的 App，通知权限不会自动沿用原版微信。

如果系统设置里没有 `wechat-dev`，先打开一次副本，再回到这里。

> 提醒：副本使用本地临时签名，没有腾讯原版的推送凭证。副本在打开或后台运行时，通知可以正常工作；完全退出副本后，推送可能不稳定。这是双开副本的技术限制，不能靠 macOS 通知开关彻底解决。

## 更新或换电脑时

原版更新后，副本不会跟着更新。需要更新副本时，先备份重要聊天记录、退出副本，只将旧的 `wechat-dev.app` 移到废纸篓，再从第二步重新创建；不要删除微信的数据目录。

换电脑从第一步开始。**这套操作只创建第二个微信，不会迁移聊天记录**，记录需另外迁移。

如果启动器或 Dock 仍是旧名称、灰色图标，先从第六步的 Finder 路径打开正确副本。本机 Finder 名称和绿色图标已确认正常，但启动器/Dock 的灰色图标问题没有确认解决，不要反复重置整个系统的图标缓存。
