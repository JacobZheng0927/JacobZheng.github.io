# 第1节：Docker



## 附录

### Docker命令速查

查看docker挂载相关信息

```bash
docker inspect container_id | grep Mounts -A 20
```



### Docker安装MongoDB

1. 拉取最新的docker镜像

   ```bash
   docker pull mongo:latest
   ```

2. 运行容器

   ```bash
   docker run -itd --name mongo -p 27017:27017 mongo --auth
   ```

   参数说明：

   - **-p 27017:27017** ：映射容器服务的 27017 端口到宿主机的 27017 端口。外部可以直接通过 宿主机 ip:27017 访问到 mongo 的服务。
   - **--auth**：需要密码才能访问容器服务。

3. 添加用户设置密码

   ```bash
   docker exec -it mongo mongo admin
   # 创建一个名为 admin，密码为 123456 的用户。
   >  db.createUser({ user:'admin',pwd:'123456',roles:[ { role:'userAdminAnyDatabase', db: 'admin'}]});
   # 尝试使用上面创建的用户信息进行连接。
   > db.auth('admin', '123456')
   ```

   ![image-20200622144220223](https://markdown-image-upload.oss-cn-beijing.aliyuncs.com/img/image-20200622144220223.png)

### Docker安装ArangoDB

ARANGO_NO_AUTH=1 ARANGO_ROOT_PASSWORD=root

docker run -e ARANGO_NO_AUTH=1 -p 8529:8529 -d \
          -v ~/Documents/database/arangodb:/var/lib/arangodb3 \
          arangodb/arangodb:3.5.1