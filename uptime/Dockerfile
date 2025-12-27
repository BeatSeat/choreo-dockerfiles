# === 阶段 1: 构建阶段 (Builder) ===
FROM node:20-alpine AS builder

# 安装构建所需的依赖
RUN apk add --no-cache git python3 py3-pip make g++ build-base

# 统一使用 /app 目录
WORKDIR /app

# 1. 优化：使用 --depth 1 只克隆最新提交
RUN git clone https://github.com/louislam/uptime-kuma.git .

# 2. 优化：安装依赖
RUN npm run setup && \
    npm cache clean --force

# 3. 优化：在 /app 目录下创建 Python 虚拟环境，保持路径一致性
ENV VIRTUAL_ENV=/app/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"
RUN pip install --no-cache-dir requests webdavclient3

# === 阶段 2: 运行阶段 (Runner) ===
FROM node:20-alpine

WORKDIR /app

# 从构建阶段拷贝所有内容
COPY --from=builder /app /app

# 安装最小化运行时依赖
RUN apk add --no-cache python3 curl tzdata

# 设置环境变量
ENV PATH="/app/venv/bin:$PATH"
ENV NODE_ENV=production

# 直接在 Dockerfile 中生成优化后的 sync_data.sh
RUN printf '#!/bin/sh\n\
if [ -z "$WEBDAV_URL" ] || [ -z "$WEBDAV_USERNAME" ] || [ -z "$WEBDAV_PASSWORD" ]; then\n\
    echo "Starting without backup functionality - missing WebDAV config"\n\
    exec node server/server.js\n\
fi\n\
\n\
FULL_WEBDAV_URL="${WEBDAV_URL}${WEBDAV_BACKUP_PATH:+/$WEBDAV_BACKUP_PATH}"\n\
\n\
restore_backup() {\n\
    echo "Checking for latest backup..."\n\
    python3 -c "\n\
import sys, os, tarfile, requests, shutil\n\
from webdav3.client import Client\n\
options = {\"webdav_hostname\": \"$FULL_WEBDAV_URL\", \"webdav_login\": \"$WEBDAV_USERNAME\", \"webdav_password\": \"$WEBDAV_PASSWORD\"}\n\
client = Client(options)\n\
try:\n\
    backups = sorted([f for f in client.list() if f.endswith(\".tar.gz\") and f.startswith(\"uptime_kuma_backup_\")])\n\
    if not backups: return\n\
    latest = backups[-1]\n\
    print(f\"Downloading {latest}...\")\n\
    r = requests.get(f\"$FULL_WEBDAV_URL/{latest}\", auth=(\"$WEBDAV_USERNAME\", \"$WEBDAV_PASSWORD\"), stream=True)\n\
    if r.status_code == 200:\n\
        with open(f\"/tmp/{latest}\", \"wb\") as f: \n\
            for c in r.iter_content(8192): f.write(c)\n\
        if os.path.exists(\"data\"): shutil.rmtree(\"data\")\n\
        with tarfile.open(f\"/tmp/{latest}\", \"r:gz\") as tar: tar.extractall(\".\")\n\
        print(\"Restore complete.\")\n\
except Exception as e: print(f\"Restore failed: {e}\")" \n\
}\n\
\n\
sync_data() {\n\
    while true;\ndo\n\
        sleep ${SYNC_INTERVAL:-600}\n\
        if [ -d "data" ]; then\n\
            ts=$(date +%%Y%%m%%d_%%H%%M%%S)\n\
            file="uptime_kuma_backup_${ts}.tar.gz"\n\
            tar -czf "/tmp/${file}" data\n\
            if curl -fs -u "$WEBDAV_USERNAME:$WEBDAV_PASSWORD" -T "/tmp/${file}" "$FULL_WEBDAV_URL/${file}"; then\n\
                python3 -c "\n\
from webdav3.client import Client\n\
c = Client({\"webdav_hostname\": \"$FULL_WEBDAV_URL\", \"webdav_login\": \"$WEBDAV_USERNAME\", \"webdav_password\": \"$WEBDAV_PASSWORD\"})\n\
bs = sorted([f for f in c.list() if f.endswith(\".tar.gz\") and f.startswith(\"uptime_kuma_backup_\")])\n\
if len(bs) > 5:\n\
    for f in bs[:-5]: c.clean(f)"\n\
            fi\n\
            rm -f "/tmp/${file}"\n\
        fi\n\
    done\n\
}\n\
\n\
restore_backup\n\
sync_data &\n\
exec node server/server.js\n\
' > /app/sync_data.sh && chmod +x /app/sync_data.sh

# --- 权限修正 ---
# 将整个工作目录的所有权授予 UID 10014
RUN chown -R 10014:0 /app && \
    chmod -R g=u /app

# 切换到指定用户
USER 10014

EXPOSE 3001

# 启动
CMD ["/bin/sh", "/app/sync_data.sh"]
