---
title: opensips设置独立的日志文件
---

## opensips设置独立的日志文件

### 系统环境 centos7

- 配置 opensips.cfg 文件

```
log_facility=LOG_LOCAL0
```

- 创建日志的配置文件

```bash
echo "local0.* /var/log/opensips.log" > /etc/rsyslog.d/opensips.conf
```

- 创建日志文件

```bash
touch /var/log/opensips.log
```

- 重启 `rsyslog` 和 `opensips`

```bash
service rsyslog restart
opensipsctl restart
```

- 验证结果

```bash
tail -f /var/log/opensips.log //实时查看日志
```