# 作为来源层
FROM wisdgod/cursor-api:latest AS src

# 最终镜像仍保持极简
FROM scratch

# 关键：重新拷贝并修正 owner/权限（0755 够用）
COPY --from=src --chown=10014:10014 --chmod=0755 /app /app

WORKDIR /app
ENV PORT=3000
EXPOSE 3000

USER 10014
ENTRYPOINT ["/app/cursor-api"]
