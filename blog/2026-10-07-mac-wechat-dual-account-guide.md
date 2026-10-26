# Mac 微信双开

先把官方微信安装到“应用程序”，然后打开“终端”，粘贴下面整段命令，按回车。

```bash
set -e
src="/Applications/WeChat.app"
dst="$HOME/Applications/wechat-dev.app"

mkdir -p "$HOME/Applications"
ditto "$src" "$dst"
/usr/libexec/PlistBuddy -c 'Set :CFBundleIdentifier com.tencent.xinWeChat.dual' "$dst/Contents/Info.plist"
/usr/bin/plutil -replace CFBundleName -string wechat-dev "$dst/Contents/Info.plist"
/usr/bin/plutil -replace CFBundleDisplayName -string wechat-dev "$dst/Contents/Info.plist"
for f in "$dst"/Contents/Resources/*.lproj/InfoPlist.strings; do
  [ -f "$f" ] || continue
  /usr/bin/plutil -replace CFBundleName -string wechat-dev "$f"
  /usr/bin/plutil -replace CFBundleDisplayName -string wechat-dev "$f"
done
/usr/bin/codesign --force --deep --sign - --timestamp=none "$dst"
/usr/bin/codesign --verify --deep --strict "$dst"
open "$src"
open "$dst"
```

这段命令会复制微信、给副本换一个应用身份、改名为 `wechat-dev`，然后重新签名并打开两个微信。

原版登录账号一，`wechat-dev` 登录账号二。如果副本显示原来的账号，点“切换账号”。

## 通知

第一次打开副本后，进入 **系统设置 → 通知 → wechat-dev**，打开“允许通知”、桌面、通知中心和声音。

macOS 会把 `wechat-dev` 当成新的 App，不会自动使用原版微信的通知权限。

## 下次打开

以后执行：

```bash
open "$HOME/Applications/wechat-dev.app"
```

也可以在 Finder 按 `Command + Shift + G`，输入 `~/Applications`，双击 `wechat-dev`。

## 注意

- 适用于微信 4.1.13，其他版本可能不同。
- 微信更新后需要重新创建副本。
- 聊天记录需要另外迁移。
- 副本完全退出后，通知可能不如原版稳定。
