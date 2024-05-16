```dockerfile
FROM mysql:5.7 as builder

# 初始化 mysql 设置并且运行 mysql，通过删除最后一行使其志初始化
RUN ["sed", "-i", "s/exec \"$@\"/echo \"not running $@\"/", "/usr/local/bin/docker-entrypoint.sh"]
# 配置密码和数据库
ENV MYSQL_ROOT_PASSWORD="webide@123"
ENV MYSQL_DATABASE="webide"

ADD webide.sql /docker-entrypoint-initdb.d/
# 修改 /var/lib/mysql 到其他目录，因为其已经被挂载为 volume
RUN ["/usr/local/bin/docker-entrypoint.sh", "mysqld", "--datadir", "/initialized-db"]

FROM mysql:5.7

COPY --from=builder /initialized-db /var/lib/mysql
```

