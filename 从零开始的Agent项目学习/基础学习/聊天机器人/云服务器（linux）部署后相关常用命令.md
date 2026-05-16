
## 一、连接与基础操作

| 操作 | 命令 |
|------|------|
| SSH 登录服务器（本地电脑执行） | `ssh root@你的服务器公网IP` |
| 进入项目目录 | `cd ~/qqbot` |
| 查看当前目录路径 | `pwd` |
| 列出目录下文件（含隐藏） | `ls -la` |
| 编辑文件（nano） | `nano 文件名` （如 `nano docker-compose.yml`） |
| 编辑文件（vi/vim） | `vi 文件名` （按 `i` 编辑，按 `Esc` 输入 `:wq` 保存退出） |
| 查看文件内容 | `cat 文件名` |

---

## 二、Docker 与 Docker Compose（核心）

| 操作 | 命令（在 `~/qqbot` 目录下执行） |
|------|-------------------------------|
| 启动所有容器（后台） | `docker compose up -d` |
| 停止所有容器 | `docker compose down` |
| 重启所有容器 | `docker compose restart` |
| 重启单个容器 | `docker compose restart astrbot` 或 `docker compose restart napcat` |
| 查看容器运行状态 | `docker ps` （加 `-a` 显示所有包括已停止的） |
| 查看容器资源占用 | `docker stats` |
| 更新镜像并重启 | `docker compose pull && docker compose up -d` |
| 查看 docker-compose 配置 | `docker compose config` |

---

## 三、日志与调试

| 操作                        | 命令                                                      |
| ------------------------- | ------------------------------------------------------- |
| 查看 AstrBot 最近日志           | `docker logs astrbot --tail 50`                         |
| 实时跟踪 AstrBot 日志           | `docker logs astrbot -f`                                |
| 查看 NapCat 最近日志            | `docker logs napcat --tail 50`                          |
| 实时跟踪 NapCat 日志            | `docker logs napcat -f`                                 |
| 仅查看 NapCat 中的 WebUI Token | `docker logs napcat 2>&1 \| grep -i "token"`            |
| 查看 AstrBot 配置内容           | `docker exec astrbot cat /AstrBot/data/cmd_config.json` |
| 查看 NapCat 配置文件（需先进入容器）    | 见下文“进入容器”部分                                             |

---

## 四、进入容器内部操作

| 操作 | 命令 |
|------|------|
| 进入 AstrBot 容器 | `docker exec -it astrbot bash` |
| 进入 NapCat 容器 | `docker exec -it napcat bash` |
| 在容器内安装 nano（如果缺失） | `apt update && apt install nano -y` |
| 在容器内编辑 AstrBot 配置文件 | 进入容器后：`cd /AstrBot/data && nano cmd_config.json` |
| 在容器内查看 NapCat 配置文件实际位置 | 进入 NapCat 容器后：`find / -name "onebot11_*.json" 2>/dev/null` |
| 从宿主机直接修改 NapCat 配置（不进入容器） | `docker exec napcat nano /app/napcat/config/onebot11_你的QQ.json` （路径根据实际调整） |

---

## 五、网络与端口检查

| 操作 | 命令 |
|------|------|
| 查看端口监听情况（宿主机） | `ss -tlnp \| grep -E "6185\|6099\|3001\|6199"` |
| 查看容器内端口（AstrBot） | `docker exec astrbot netstat -tlnp 2>/dev/null` |
| 查看容器内端口（NapCat） | `docker exec napcat netstat -tlnp 2>/dev/null` |
| 测试服务器防火墙（ufw）状态 | `ufw status` |
| 放行端口（如 6185） | `ufw allow 6185` |

---

## 六、自动化保活脚本

### 创建监控脚本
```bash
nano /root/check_napcat.sh
```
粘贴内容：
```bash
#!/bin/bash
if ! docker exec napcat pgrep -f "QQ" > /dev/null; then
    echo "$(date): NapCat is dead, restarting..." >> /root/napcat_restart.log
    docker restart napcat
fi
```
### 赋予执行权限
```bash
chmod +x /root/check_napcat.sh
```
### 添加到定时任务（每5分钟检查）
```bash
crontab -e
```
添加一行：
```bash
*/5 * * * * /root/check_napcat.sh
```

---

## 七、常见故障排查指令

| 场景 | 命令 |
|------|------|
| 容器崩溃但 `docker ps` 看不到 | `docker ps -a` 查看退出状态，用 `docker logs 容器名` 看错误 |
| NapCat 提示“已登录无法重复登录” | `docker restart napcat` 然后重新获取 Token 扫码 |
| AstrBot Web 界面打不开 | 检查安全组放行 6185，且 `ss -tlnp \| grep 6185` 有输出 |
| NapCat Web 界面打不开 | 检查安全组放行 6099，且 `ss -tlnp \| grep 6099` 有输出 |
| 机器人发消息超时（retcode=1200） | 通常为风控，可尝试更换 IP、降低频率、使用小号养号 |
| 想要完全重置 NapCat 登录态 | `docker compose down` 然后删除本地挂载的 config 和 QQ 缓存目录（若有），再 `docker compose up -d` |

---

## 八、系统级维护

| 操作 | 命令 |
|------|------|
| 查看系统负载 | `top` 或 `htop` |
| 查看内存使用 | `free -h` |
| 查看磁盘使用 | `df -h` |
| 清理 Docker 无用镜像和容器 | `docker system prune -a`（谨慎，会删除未使用的镜像） |
| 重启服务器 | `reboot` |
| 关闭服务器 | `shutdown now` |

---

## 九、补充：之前我们用过的关键文件路径

| 项目 | 路径 |
|------|------|
| docker-compose.yml | `/root/qqbot/docker-compose.yml` |
| AstrBot 配置文件 | 容器内 `/AstrBot/data/cmd_config.json` |
| NapCat 配置文件（举例） | 容器内 `/app/napcat/config/onebot11_你的QQ.json` |

---

保存这份文档，以后机器人出状况时，按顺序查日志 -> 重启对应容器 -> 重新获取 Token -> 检查网络端口，基本能解决 90% 的问题。如果还有新的报错，随时带着日志来问我。祝你的机器人稳定运行！🤖