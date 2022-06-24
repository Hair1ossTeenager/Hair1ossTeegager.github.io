---
title: "k8s 单节点 mongodb 数据备份及恢复"
date: 2021-02-21T17:31:47+08:00
draft: false
---

### 背景
        最近为测试同学搭建测试平台用到 mongo，但是阿里云的 mongo 服务收费太贵。
    因此使用 k8s 自建单节点 mongo 服务，那就不得不考虑数据备份的问题。
 
### 思路
- 数据备份

	建立 cronjob，通过 mongodump 全量备份数据，ossutil 上传至阿里云。
	
- 数据恢复

	建立 job ，通过 ossutil 下载 oss 备份的数据，mongorestore 全量恢复数据。
	
### Docker 镜像
- Dockerfile 目录结构
```
docker
|── Dockerfile
|── mongo-manager.sh
|── ossutil64
```
- Dockerfile
```bash
# mongo 版本和 mongo 服务使用的版本保持一致
FROM mongo:4.4.3

COPY ossutil64 /usr/local/bin/ossutil64
RUN chmod 755 /usr/local/bin/ossutil64
COPY mongo-manager.sh /usr/local/bin/mongo-manager.sh
RUN chmod 755 /usr/local/bin/mongo-manager.sh

WORKDIR /
```
- ossutil64，对 ossutil 版本有需求的同学可以下载不同的版本，然后重新打包镜像。
```
wget http://gosspublic.alicdn.com/ossutil/1.7.1/ossutil64                           
下载到 Dockerfile 同级目录下
```
- mongo-manager.sh，有不同需求的同学可以修改此文件，然后重新打包 docker 镜像。
```bash
#!/bin/bash

function create_backup_dir() {
    echo "创建备份文件"
    date=`date +"%Y-%m-%d"`
    backup_dir="/mongo-back-$date"
    mkdir $backup_dir
}

function backup_mongo() {
    echo "备份 mongo 数据"
    mongodump -h $MONGO_HOST:$MONGO_PORT -u $MONGO_USER -p $MONGO_PASSWORD --authenticationDatabase $AUTHENTICATION_DB --gzip --out $backup_dir
}

function upload_to_oss() {
    echo "上传至 OSS"
    ossutil64 config -e $OSS_ENDPOINT -i $OSS_ACCESSID -k $OSS_SECRET
    ossutil64 cp -r $backup_dir oss://$OSS_BUCKET/$OSS_DIR$backup_dir
}

function restore_mongo() {
    echo "恢复 mongo 数据"
    mongorestore -h $MONGO_HOST:$MONGO_PORT -u $MONGO_USER -p $MONGO_PASSWORD --authenticationDatabase $AUTHENTICATION_DB --drop --gzip /$RESTORE_DIR
}

function download_from_oss() {
    echo "从 OSS 下载备份文件"
    ossutil64 config -e $OSS_ENDPOINT -i $OSS_ACCESSID -k $OSS_SECRET
    ossutil64 cp -r oss://$OSS_BUCKET/$OSS_DIR/$RESTORE_DIR /
}

function help() {
    cat <<EOF
options:
    backup 全量备份数据并上传到 oss
    restore 全量恢复 oss 的数据
EOF
}

if [[ $1 == "backup" ]]; then
    create_backup_dir
    backup_mongo
    upload_to_oss
elif [[ $1 == "restore" ]]; then
    download_from_oss
    restore_mongo
else
    help
    exit 1
fi
```

### k8s mongo 数据备份 cronjob & 数据恢复 job
- mongo 数据备份 cronjob
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: mongo-backup
spec:
  schedule: "30 02 * * *"
  successfulJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mongo-manager
            image: registry.cn-hangzhou.aliyuncs.com/k_k/mongo-manager:v2
            env:
            - name: MONGO_HOST # mongo 地址
              value: mongo.yapi.svc.cluster.local 
            - name: MONGO_PORT # mongo 端口
              value: "27017"
            - name: MONGO_USER # mongo 进行备份的用户
              value: root
            - name: MONGO_PASSWORD # 用户的密码
              value: mongo 
            - name: AUTHENTICATION_DB # 鉴权的 DB
              value: admin
            - name: OSS_ENDPOINT # oss 的 endpoint
              value: oss-cn-hangzhou.aliyuncs.com
            - name: OSS_ACCESSID # 访问 oss 的 access id
              value: xxxxxx
            - name: OSS_SECRET # 访问 oss 的 secret
              value: xxxxxx
            - name: OSS_BUCKET # 存放备份数据的 oss bucket
              value: yapi-mongo-backup
            - name: OSS_DIR # 存放备份数据的 oss bucket 下的目录
              value: mongo-backup
            command: ["mongo-manager.sh", "backup"]
          restartPolicy: OnFailure
```
- mongo 数据恢复的 job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mongo-backup
spec:
  template:
    spec:
      containers:
      - name: mongo-manager
        image: registry.cn-hangzhou.aliyuncs.com/k_k/mongo-manager:v2
        env:
        - name: MONGO_HOST # mongo 地址
          value: mongo.yapi.svc.cluster.local 
        - name: MONGO_PORT # mongo 端口
          value: "27017"
        - name: MONGO_USER # mongo 进行备份的用户
          value: root
        - name: MONGO_PASSWORD # 用户的密码
          value: mongo 
        - name: AUTHENTICATION_DB # 鉴权的 DB
          value: admin
        - name: OSS_ENDPOINT # oss 的 endpoint
          value: oss-cn-hangzhou.aliyuncs.com
        - name: OSS_ACCESSID # 访问 oss 的 access id
          value: xxxxxx
        - name: OSS_SECRET # 访问 oss 的 secret
          value: xxxxxx
        - name: OSS_BUCKET # 存放备份数据的 oss bucket
          value: yapi-mongo-backup
        - name: OSS_DIR # 存放备份数据的 oss bucket 下的目录
          value: mongo-backup
        - name: RESTORE_DIR # 上传至 oss 中的备份文件夹
          value: mongo-back-2021-02-21
        command: ["mongo-manager.sh", "restore"]
      restartPolicy: Never
```
