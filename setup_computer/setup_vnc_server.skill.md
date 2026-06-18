# Setup VNC Server Skill

## 目标
在 Ubuntu 24.04 上配置一个**可开机自启、可远程共享本机图形桌面**的 VNC 服务。

本次最终采用的是：
- `TigerVNC`
- `x0vncserver`
- `GDM 自动登录`
- `Xorg` 而不是 `Wayland`

适用场景：
- 远程看到**和服务器本机显示器完全相同**的桌面
- 服务器重启后，自动进入图形桌面，并可直接 VNC 连接

---

## 两种 VNC 模式要先分清

### 1. `vncserver` / `Xtigervnc`
特点：
- 创建一个**独立虚拟桌面**
- 通常用 `:1`, `:2`，对应 `5901`, `5902`
- 和本机物理显示器不是同一个界面

适合：
- 不需要共享当前桌面
- 希望多人各自用独立桌面

常见问题：
- `:1` 可能被本机已有图形会话占用
- `~/.vnc/xstartup` 写错后会“秒退”

### 2. `x0vncserver`
特点：
- 共享**当前物理桌面**
- 通常共享 `:0`，监听 `5900`
- 远程看到的就是本机正在显示的界面

适合：
- 远程协助
- 远程操作本机桌面

本次最终使用的就是这个模式。

---

## 本次踩过的坑

### 坑 1：`vncserver@1` 起不来
现象：
- 服务失败
- `5901` 没监听

原因：
- `:1` 已被系统图形会话占用
- 应改用别的 display，比如 `:2`

排查命令：
```bash
systemctl status vncserver@1.service
ss -lntp | grep 59
ls -la /tmp/.X11-unix/
ls -la /tmp/.X*-lock
lsof /tmp/.X11-unix/X1
```

结论：
- 如果 `X1` 已被 GNOME/本地图形会话占用，就不要再拿 `:1` 给 TigerVNC。

---

### 坑 2：VNC 能启动，但客户端连不上
排查点：
1. 服务是否真的监听端口
2. 防火墙是否放行对应端口
3. 是否绑定到 `localhost`

排查命令：
```bash
systemctl status vncserver@2.service
ss -lntp | grep 5902
ufw status
```

经验：
- `-localhost no` 或 `-localhost=0` 很关键
- `ufw allow 5900/tcp` / `5902/tcp` 要对应当前模式

---

### 坑 3：客户端报错 `Unable to execute child process "dbus-launch"`
原因：
- 缺少 `dbus-launch`

解决：
```bash
sudo apt-get install -y dbus-x11
```

说明：
- `dbus-launch` 由 `dbus-x11` 提供
- XFCE/GNOME 某些会话启动会依赖它

---

### 坑 4：VNC 登录后是黑屏
这是最关键的经验之一。

原因：
- `x0vncserver` 共享物理桌面时，如果图形会话是 `Wayland`，经常黑屏
- TigerVNC 抓取 `Wayland` 会话通常不稳定

排查命令：
```bash
loginctl show-session <session_id> -p Type
ps -ef | grep -E 'gnome-shell|Xorg|Xwayland'
```

如果看到：
- `Type=wayland`

则需要切换到 `Xorg`。

解决方式：修改 GDM：
```ini
# /etc/gdm3/custom.conf
[daemon]
WaylandEnable=false
```

然后重启 `gdm3` 或重启机器，使桌面改为 Xorg。

---

### 坑 5：重启后 VNC 不能直接登录
原因：
- `x0vncserver` 共享的是**已有桌面**，不是“帮你创建桌面”
- 如果机器启动后没人登录图形界面，就没有桌面可共享

解决：
- 开启 GDM 自动登录

配置：
```ini
# /etc/gdm3/custom.conf
[daemon]
WaylandEnable=false
AutomaticLoginEnable=true
AutomaticLogin=xiping
```

这样机器启动后：
1. 自动进入图形桌面
2. `x0vncserver` 等到桌面起来后自动附着
3. 客户端可直接连接

---

## `vncserver` 虚拟桌面模式的关键经验

### 正确的 `~/.vnc/xstartup`
错误写法示例：
```sh
exec /usr/bin/startxfce4 &
```

问题：
- 主进程后台化，VNC 会判断会话过早退出

正确示例：
```sh
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
if [ -r "$HOME/.Xresources" ]; then
    xrdb "$HOME/.Xresources"
fi
vncconfig -iconic &
exec /usr/bin/startxfce4
```

要点：
- `exec` 的桌面会话不要再加 `&`
- 否则 VNC session 会“cleanly exited too early”

---

## 最终稳定方案

### 安装包
```bash
sudo apt-get update
sudo apt-get install -y \
  tigervnc-standalone-server \
  tigervnc-tools \
  tigervnc-scraping-server \
  dbus-x11
```

说明：
- `tigervnc-standalone-server`：虚拟桌面模式
- `tigervnc-scraping-server`：提供 `x0vncserver`
- `dbus-x11`：提供 `dbus-launch`

---

## 最终采用的文件配置

### 1. GDM 配置
文件：`/etc/gdm3/custom.conf`

推荐内容：
```ini
[daemon]
WaylandEnable=false
AutomaticLoginEnable=true
AutomaticLogin=xiping
```

作用：
- 禁用 Wayland
- 强制使用 Xorg
- 自动登录用户 `xiping`

---

### 2. `x0vncserver` 启动脚本
文件：`/usr/local/bin/start-x0vnc.sh`

推荐逻辑：
- 等待 `gnome-shell` 出现
- 从进程环境中提取：
  - `DISPLAY`
  - `XAUTHORITY`
- 确认 `XAUTHORITY` 文件存在
- 再执行 `X0tigervnc`

核心实现：
```bash
#!/usr/bin/env bash
set -euo pipefail

USER_NAME="xiping"
PASSWD_FILE="/home/xiping/.vnc/passwd"

if [[ ! -f "$PASSWD_FILE" ]]; then
  echo "Missing VNC password file: $PASSWD_FILE"
  exit 1
fi

while true; do
  PID="$(pgrep -u "$USER_NAME" -n gnome-shell || true)"
  if [[ -n "$PID" ]]; then
    DISPLAY_VAL="$(tr '\0' '\n' < "/proc/$PID/environ" 2>/dev/null | sed -n 's/^DISPLAY=//p' | head -n1 || true)"
    XAUTH_VAL="$(tr '\0' '\n' < "/proc/$PID/environ" 2>/dev/null | sed -n 's/^XAUTHORITY=//p' | head -n1 || true)"
    if [[ -z "$DISPLAY_VAL" ]]; then
      DISPLAY_VAL=":0"
    fi
    if [[ -n "$XAUTH_VAL" && -e "$XAUTH_VAL" ]]; then
      export DISPLAY="$DISPLAY_VAL"
      export XAUTHORITY="$XAUTH_VAL"
      exec /usr/bin/X0tigervnc \
        -display "$DISPLAY_VAL" \
        -rfbport=5900 \
        -PasswordFile="$PASSWD_FILE" \
        -SecurityTypes=VncAuth \
        -localhost=0 \
        -AlwaysShared=1 \
        -NeverShared=1
    fi
  fi

  echo "Waiting for GNOME Xorg session for user $USER_NAME..."
  sleep 2
done
```

---

### 3. systemd 服务
文件：`/etc/systemd/system/x0vncserver.service`

推荐内容：
```ini
[Unit]
Description=TigerVNC x0vncserver (share physical desktop)
After=network.target display-manager.service graphical.target
Wants=network.target

[Service]
Type=simple
User=xiping
Group=xiping
ExecStart=/usr/local/bin/start-x0vnc.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启用：
```bash
sudo systemctl daemon-reload
sudo systemctl enable x0vncserver.service
sudo systemctl start x0vncserver.service
```

---

## 防火墙建议

只保留实际使用端口。

如果最终用 `x0vncserver`：
```bash
sudo ufw allow 5900/tcp
sudo ufw delete allow 5901/tcp
sudo ufw delete allow 5902/tcp
```

检查：
```bash
sudo ufw status
```

---

## 标准排查流程

### 1. 看服务状态
```bash
systemctl status x0vncserver.service
systemctl status gdm3
```

### 2. 看端口是否监听
```bash
ss -lntp | grep 5900
```

### 3. 看图形会话类型
```bash
loginctl list-sessions
loginctl show-session <id> -p Type -p Name -p State -p Display
```

### 4. 看当前桌面相关进程
```bash
ps -ef | grep -E 'gnome-shell|Xorg|Xwayland|gdm-x-session' | grep -v grep
```

### 5. 看日志
```bash
journalctl -u x0vncserver.service -b --no-pager
journalctl -u gdm3 -b --no-pager
```

---

## 关键判断口诀

### 想要“同一个桌面”
用：
- `x0vncserver`
- 端口 `5900`
- `Xorg`
- 自动登录

### 想要“独立桌面”
用：
- `vncserver`
- `:1/:2`
- 端口 `5901/5902`
- `~/.vnc/xstartup` 正确写法

---

## 最终结论

在 Ubuntu 24.04 + GNOME 环境中，如果目标是：

> 机器重启后自动进入图形桌面，并且 VNC 连接后看到和本机同一个界面

最稳妥的方案是：

1. 安装 `tigervnc-scraping-server`
2. 使用 `x0vncserver`
3. GDM 设置：
   - `WaylandEnable=false`
   - `AutomaticLoginEnable=true`
   - `AutomaticLogin=<user>`
4. 用 systemd 启动 `x0vncserver`
5. 防火墙仅开放 `5900/tcp`

---

## 本机本次配置结果摘要

- 远程方式：`x0vncserver`
- 共享桌面：物理桌面 `:0`
- 监听端口：`5900`
- 图形协议：`Xorg`
- 登录方式：GDM 自动登录 `xiping`
- 防火墙开放：`5900/tcp`

客户端连接方式：
```text
10.239.82.172:5900
```
