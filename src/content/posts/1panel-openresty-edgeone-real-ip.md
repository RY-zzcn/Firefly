---
title: EdgeOne+OpenResty获取用户真实IP教程
published: 2026-07-22
tags: [EdgeOne, OpenResty, 1Panel, 真实IP, Nginx, realip]
category: 教程
draft: false
slug: edgeone-openresty-real-ip
---

# EdgeOne+OpenResty获取用户真实IP教程

把网站接入腾讯 EdgeOne 之后，最容易被忽略、也最容易踩坑的一件事，就是 **Nginx 拿到的客户端 IP 变成了 EdgeOne 回源节点的地址**。本文用一份可直接照抄的方案，带你把真实访客 IP 正确地还原出来，并附带一个自动同步回源 IP 段的脚本，最后还会讲清楚一个会让 OpenResty 直接起不来的隐藏雷。

## 一、问题背景

EdgeOne 作为反向代理/CDN，会替访客向你的源站（OpenResty）发起请求。此时 OpenResty 看到的 TCP 连接来源，是 EdgeOne 的某个回源节点，而不是真实用户。

后果很直接：

- 访问日志里的 `$remote_addr` 全是回源段，无法做访问分析；
- 基于 IP 的限流、封禁、Fail2ban 全部失效；
- 一些按地域处理请求的逻辑会判断错误。

解决它的标准做法，是让 Nginx 的 `realip` 模块知道：「来自 EdgeOne 节点的请求，去它带来的请求头里取真实 IP」。

## 二、核心原理

EdgeOne 回源时会在请求头中写入访客原始地址，字段名为 `EO-Connecting-IP`。我们要做两件事：

1. 用 `set_real_ip_from` 声明「哪些 IP 是可信的 EdgeOne 回源节点」；
2. 用 `real_ip_header` 告诉模块「真实 IP 在 `EO-Connecting-IP` 这个头里」。

`real_ip_recursive on` 则用于处理多层代理，确保一路回溯到第一个可信节点以外的那个地址。

> 关键点：`realip` 模块在 OpenResty/Nginx 中是默认编译进来的，通常无需额外安装。

## 三、创建全局配置片段

在 1Panel 环境中，站点配置目录位于宿主机：

```
/opt/1panel/www/conf.d/
```

这个目录会被挂载到 OpenResty 容器内：

```
/usr/local/openresty/nginx/conf/conf.d/
```

为了让真实 IP 配置对**所有站点**生效，我们在 `conf.d` 放一个全局文件。`set_real_ip_from` / `real_ip_header` 属于 `http` 上下文指令，在 `conf.d` 中引入后会作用于全部 `server`，因此文件名只需语义清晰即可：

```nginx
# /opt/1panel/www/conf.d/edgeone-real-client-ip.conf
real_ip_header EO-Connecting-IP;
real_ip_recursive on;

# 下面的 set_real_ip_from 由脚本自动维护，请勿手动编辑
```

此时如果直接重启，OpenResty 仍然不知道 EdgeOne 的回源段长什么样，所以 `set_real_ip_from` 暂时为空，真实 IP 还取不到。下一步补上这部分。

## 四、编写自动同步脚本

EdgeOne 的回源节点段会变动，手动维护不现实。官方提供了两个纯文本接口，分别返回 IPv4 与 IPv6 的段清单：

- `https://api.edgeone.ai/ips?version=v4`
- `https://api.edgeone.ai/ips?version=v6`

为避免「同步出错反而把正常配置弄坏」，脚本遵循三个原则：

- 先写临时文件，校验通过后再原子替换正式文件；
- 两个接口都失败、或合法行数过少时，**不覆盖**现有配置；
- 写入前对每一行做严格的 IP/CIDR 校验。

新建脚本 `/root/edgeone-realip-refresh.sh`：

```bash
#!/bin/bash
set -euo pipefail

OUT="/opt/1panel/www/conf.d/edgeone-real-client-ip.conf"
TMP="$(mktemp)"
BAK="${OUT}.bak"
umask 022

URL_V4="https://api.edgeone.ai/ips?version=v4"
URL_V6="https://api.edgeone.ai/ips?version=v6"

fetch() {
  local url="$1"
  curl -fsSL --http1.1 \
    --retry 12 --retry-delay 1 --retry-all-errors \
    --connect-timeout 8 --max-time 25 \
    -H "User-Agent: realip-refresh/1.0" \
    -H "Accept: text/plain" \
    "$url"
}

is_cidr_token() {
  local s="$1"
  # IPv4：四组点分十进制，可选 /掩码
  if echo "$s" | grep -Eq '^([0-9]{1,3}\.){3}[0-9]{1,3}(/[0-9]{1,2})?$'; then
    return 0
  fi
  # IPv6：必须由十六进制与冒号组成，且必须包含冒号，杜绝纯 hex 串被误判
  if echo "$s" | grep -Eq '^[0-9A-Fa-f:]+((/[0-9]{1,3})?)$' && echo "$s" | grep -q ':'; then
    return 0
  fi
  return 1
}

gen_block() {
  local label="$1"
  local url="$2"
  echo "# - ${label}"

  local data
  data="$(fetch "$url")"

  if [ -z "${data//[[:space:]]/}" ]; then
    echo "WARN: empty response for $label" >&2
    return 1
  fi

  for i in $data; do
    if is_cidr_token "$i"; then
      echo "set_real_ip_from $i;"
    fi
  done
  return 0
}

ok_any=0
{
  echo "# EdgeOne real client IP (auto-generated)"
  echo ""

  if gen_block "IPv4" "$URL_V4"; then ok_any=1; else echo "WARN: v4 fetch failed" >&2; fi
  echo ""
  if gen_block "IPv6" "$URL_V6"; then ok_any=1; else echo "WARN: v6 fetch failed" >&2; fi

  echo ""
  echo "real_ip_header EO-Connecting-IP;"
  echo "real_ip_recursive on;"
} > "$TMP"

if [ "$ok_any" -eq 0 ]; then
  echo "ERROR: both v4 and v6 fetch failed; not updating $OUT" >&2
  rm -f "$TMP"
  exit 1
fi

COUNT="$(grep -c '^set_real_ip_from ' "$TMP" || true)"
if [ "$COUNT" -lt 5 ]; then
  echo "ERROR: too few IP ranges ($COUNT); not updating $OUT" >&2
  rm -f "$TMP"
  exit 1
fi

if [ -f "$OUT" ]; then
  cp -f "$OUT" "$BAK"
fi

mv "$TMP" "$OUT"
chown root:root "$OUT"
chmod 644 "$OUT"

echo "OK: updated $OUT (set_real_ip_from lines: $COUNT)"
```

赋予执行权限并手动跑一次：

```bash
chmod +x /root/edgeone-realip-refresh.sh
/root/edgeone-realip-refresh.sh
```

正常输出类似：

```text
OK: updated /opt/1panel/www/conf.d/edgeone-real-client-ip.conf (set_real_ip_from lines: 597)
```

## 五、加入定时任务

回源段会更新，建议每天自动同步一次。编辑 `crontab -e`：

```cron
30 2 * * * /root/edgeone-realip-refresh.sh >>/var/log/edgeone-realip-refresh.log 2>&1
```

每天凌晨 2:30 重新拉取并刷新配置。即便某天接口异常，因为脚本有「失败不覆盖」的保护，现有配置也不会被破坏。

## 六、一个会让 OpenResty 起不来的隐藏雷

这是本方案最容易翻车的地方，单独拿出来讲。

如果你看到 OpenResty 容器重启后立刻退出，日志里反复出现：

```text
[emerg] host not found in set_real_ip_from "be" in
/usr/local/openresty/nginx/conf/conf.d/edgeone-real-client-ip.conf:4
```

说明生成的配置里混进了一条非法的 `set_real_ip_from be;`。Nginx 在启动阶段解析该指令时，要求后面必须是可识别的 IP、网段或主机名，`be` 显然不是，于是直接 `[emerg]` 退出，容器进入无限重启。

### 成因

问题出在校验函数对 IPv6 的判断过宽。把校验写成下面这样就会中招：

```bash
# 有缺陷的写法
if echo "$s" | grep -Eq '^[0-9A-Fa-f:]+(/[0-9]{1,3})?$'; then
  return 0
fi
```

它的本意是匹配 IPv6，但实际匹配的是「任意由十六进制字符和冒号组成的串」。`be` 两个字符都在十六进制范围（`b`、`e` 均合法），于是被当成合法 IPv6 放行写进了配置。同理，`abc`、`face`、`deadbeef` 这类纯十六进制串也都会过这一关，而它们都会让 Nginx 崩溃。

至于 `be` 从哪来：接口返回内容中偶尔会混入非 IP 的结构性文本片段，脚本按空白分词后把它切成了独立 token，宽松的正则又放它过了关。「接口返回空才拦截」这一层保护挡不住它，因为它夹在几百条正常数据里，并不为空。

### 修复

给 IPv6 判断加一条硬性约束：**必须包含冒号**。纯十六进制串没有冒号，立即被拒；真正的 IPv6（如 `240e::/64`）一定带冒号，正常通过：

```bash
if echo "$s" | grep -Eq '^[0-9A-Fa-f:]+((/[0-9]{1,3})?)$' && echo "$s" | grep -q ':'; then
  return 0
fi
```

本文第四节给出的脚本已经是修复后的版本，无需再改。部署后确认没有脏数据、再重启容器即可：

```bash
/root/edgeone-realip-refresh.sh
grep -n 'be;' /opt/1panel/www/conf.d/edgeone-real-client-ip.conf   # 无输出即正常
docker restart <openresty容器名>
```

由于修复落在脚本层面，之后每天的定时任务产出的都是干净文件，问题不会复发。

> 补充：早期有人会想「直接注释掉那一行 `set_real_ip_from be;` 不就行了」。这是治标不治本——第二天定时任务重新生成文件，`be` 又回来了，容器再次崩。必须从脚本校验层面根治。

## 七、关于 curl 的 HTTP/2 报错

脚本运行时 stderr 偶尔会冒出：

```text
curl: (92) HTTP/2 stream 1 was not closed cleanly: INTERNAL_ERROR (err 2)
```

这是访问官方接口时 HTTP/2 流不稳定的现象，本身不致命——脚本中的 `--retry-all-errors` 会重试并最终拿到完整数据。如果你希望更干净，可以给 curl 加上 `--http1.1` 强制走 HTTP/1.1（本文脚本已默认加上）。海外机器拉取该接口时建议保留此项。

## 八、验证真实 IP 是否生效

配置能加载，不代表日志里就是真实 IP。等站点有一些外部访问后，查看访问日志：

```bash
tail -n 20 /opt/1panel/www/sites/<你的域名>/log/access.log
```

如果 `$remote_addr` 显示的是访客的公网 IP，而不是 `1.14.x`、`43.x` 这类 EdgeOne 节点段，说明整条链路已经打通。

## 九、小结

- EdgeOne 之后还原真实 IP，依赖 `realip` 模块 + `EO-Connecting-IP` 请求头，用一个 `00-` 前缀的全局 conf 即可一次覆盖所有站点。
- 回源 IP 段需要自动同步，但**写入前必须严格校验 IP/CIDR**，否则接口里的杂数据（如 `be`）会变成 `set_real_ip_from be;`，导致 Nginx 启动即崩。
- 一行根治：IPv6 校验要求「必须包含冒号」。
- 海外拉取官方接口建议 `curl --http1.1`，规避 HTTP/2 不稳定。

按本文步骤操作，即可在 1Panel 的 OpenResty 上稳定获取 EdgeOne 后的真实访客 IP。
