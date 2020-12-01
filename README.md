

本项目为华中科技大学数据中心技术实验，主要内容分为两个部分，即：

* Kubernetes容器编排技术
* 对象存储技术

相应的技术文档参考https://github.com/cs-course。这里对kubernetes和对象存储实验进行总结

Kubernetes虚拟机集群搭建见：https://github.com/cs-course/k3s-tutorial

## Kuberbetes入门实践

### 使用Kubernetes部署Typecho博客系统

* 部署应用

```bash
# 将typecho文件夹从宿主机上传到node0
$ cd ~/k3s-tutorial
$ vagrant upload typecho node0
# 连接node0
$ vagrant ssh node0
# 进入到node0
vagrant@node0:~$ ls
conf node0.log typecho
# 部署typecho
vagrant@node0:~$ cd typecho/
vagrant@node0:~/typecho$ ls
mysql typecho_deploy.yaml typecho_service.yaml
vagrant@node0:~/typecho$ sudo k3s kubectl create -f .
deployment.apps/typecho-app created
service/typecho-service created
# 部署mysql
vagrant@node0:~/typecho$ cd mysql
vagrant@node0:~/typecho/mysql$ sudo k3s kubectl create -f .
deployment.apps/mysql-deploy created
service/mysql-service created
# 查看是否部署成功
vagrant@node0:~/typecho/mysql$ sudo k3s kubectl get pods
NAME READY STATUS RESTARTS AGE
typecho-app-5bd56599-r5rx5 1/1 Running 0 59s
mysql-deploy-7c94d7bfc9-fkcs9 1/1 Running 0 50s
```

* 创建数据库

```bash
# 获取mysql容器的名称
vagrant@node0:~/typecho/mysql$ sudo k3s kubectl get pods | grep mysql
mysql-deploy-7c94d7bfc9-fkcs9 1/1 Running 0 2m33s
# 进入到mysql容器内部
vagrant@node0:~/typecho/mysql$ sudo k3s kubectl exec -it mysql-deploy-
7c94d7bfc9-fkcs9 -- /bin/bash
root@mysql-deploy-7c94d7bfc9-fkcs9:/#
# 创建数据库
root@mysql-deploy-7c94d7bfc9-fkcs9:/# mysql -u root -p
# 密码是123456
Enter password:
mysql> create database typecho;
Query OK, 1 row affected (0.00 sec)
# 退出即可
mysql> exit
连接数据库
Bye
root@mysql-deploy-7c94d7bfc9-fkcs9:/# exit
exit
vagrant@node0:~/typecho/mysql$
```

* 连接数据库

```bash
# 查看mysql数据库的地址,此处为 10.43.85.181:80
vagrant@node0:~/typecho/mysql$ sudo k3s kubectl get svc
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 10.43.0.1 <none> 443/TCP 7d2h
typecho-service NodePort 10.43.249.246 <none> 80:30010/TCP 6m44s
mysql-service NodePort 10.43.85.181 <none> 80:30011/TCP 6m35s
# 查看typecho部署到的哪一个节点，此处为node2，即192.168.33.22
vagrant@node0:~/typecho/mysql$ sudo k3s kubectl get pods -o wide | grep typecho
typecho-app-5bd56599-r5rx5 1/1 Running 0 8m2s 10.42.2.8
node2 <none> <none>
# 在浏览器中打开网页： 192.168.33.22:30010 ，按照指示连接数据库即可
```

### 搭建Kubernetes Dashboard插件

[k8s集群之部署DASHBOARD应用(连载)](https://zhuanlan.zhihu.com/p/130473678)

注意该教程的使用k8s搭建，本次实验是使用k3s，因此对应指令处要将 `k8s` 改成 `sudo k3s`

## 对象存储实践与性能指标分析

### 配置使用minio

下载 `minio.exe`， `mc.exe`，使用命令行打开，输入`run-minio.cmd`打开minio服务器。成功后在dos界面显示 endpoint、accessKey和secretKey：

```bash
Endpoint:  http://169.254.232.196:9000  http://10.11.74.141:9000  http://169.254.238.110:9000  http://169.254.0.61:9000  http://169.254.62.42:9000  http://192.168.33.1:9000  http://192.168.211.1:9000  http://192.168.56.1:9000  http://192.168.92.1:9000  http://127.0.0.1:9000
AccessKey: hust
SecretKey: hust_obs

Browser Access:
   http://169.254.232.196:9000  http://10.11.74.141:9000  http://169.254.238.110:9000  http://169.254.0.61:9000  http://169.254.62.42:9000  http://192.168.33.1:9000  http://192.168.211.1:9000  http://192.168.56.1:9000  http://192.168.92.1:9000  http://127.0.0.1:9000

Command-line Access: https://docs.min.io/docs/minio-client-quickstart-guide
   $ mc.exe alias set myminio http://169.254.232.196:9000 hust hust_obs

Object API (Amazon S3 compatible):
   Go:         https://docs.min.io/docs/golang-client-quickstart-guide
   Java:       https://docs.min.io/docs/java-client-quickstart-guide
   Python:     https://docs.min.io/docs/python-client-quickstart-guide
   JavaScript: https://docs.min.io/docs/javascript-client-quickstart-guide
   .NET:       https://docs.min.io/docs/dotnet-client-quickstart-guide


```

打开端点 `127.0.0.1:9000` ,浏览器启动后，输入 `AccessKey: hust` 和 `SecretKey: hust_obs `登录。

### 配置使用minio client

```bash
# 添加主机
# Minio为主机名称，之后为endpoint、accessKey、secretKey和API签名默认为s3v4。
E:\Data-Center\obs-tutorial>mc config host add minio http://127.0.0.1:9000 hust hust_obs --api s3v4
mc: Configuration written to `C:\Users\jchen\mc\config.json`. Please update your access credentials.
mc: Successfully created `C:\Users\jchen\mc\share`.
mc: Initialized share uploads `C:\Users\jchen\mc\share\uploads.json` file.
mc: Initialized share downloads `C:\Users\jchen\mc\share\downloads.json` file.
Added `minio` successfully.

# 为服务器添加bucket
E:\Data-Center\obs-tutorial>mc mb minio/mybucket
Bucket created successfully `minio/mybucket`.

# 上传文件
E:\Data-Center\obs-tutorial>mc.exe cp e:\Data-Center\obs-tutorial\minio\test.txt minio/mybucket
...l\minio\test.txt:  32 B / 32 B [=====================================================================] 2.60 KiB/s 0s

```

![image-20201128213557440](https://gitee.com/jchenTech/images/raw/master/img/20201128221914.png)

### 配置使用S3 Bench

```bash
E:\Data-Center\obs-tutorial>s3bench.exe     -accessKey=hust     -accessSecret=hust_obs     -bucket=mybucket     -endpoint=http://127.0.0.1:9000     -numClients=8     -numSamples=256     -objectNamePrefix=mybucket     -objectSize=1024
Test parameters
endpoint(s):      [http://127.0.0.1:9000]
bucket:           mybucket
objectNamePrefix: mybucket
objectSize:       0.0010 MB
numClients:       8
numSamples:       256
verbose:       %!d(bool=false)


Generating in-memory sample data... Done (3.0002ms)

Running Write test...

Running Read test...

Test parameters
endpoint(s):      [http://127.0.0.1:9000]
bucket:           mybucket
objectNamePrefix: mybucket
objectSize:       0.0010 MB
numClients:       8
numSamples:       256
verbose:       %!d(bool=false)


Results Summary for Write Operation(s)
Total Transferred: 0.250 MB
Total Throughput:  0.18 MB/s
Total Duration:    1.352 s
Number of Errors:  0
------------------------------------
Write times Max:       0.100 s
Write times 99th %ile: 0.095 s
Write times 90th %ile: 0.061 s
Write times 75th %ile: 0.054 s
Write times 50th %ile: 0.044 s
Write times 25th %ile: 0.028 s
Write times Min:       0.008 s


Results Summary for Read Operation(s)
Total Transferred: 0.250 MB
Total Throughput:  2.55 MB/s
Total Duration:    0.098 s
Number of Errors:  0
------------------------------------
Read times Max:       0.007 s
Read times 99th %ile: 0.006 s
Read times 90th %ile: 0.004 s
Read times 75th %ile: 0.003 s
Read times 50th %ile: 0.003 s
Read times 25th %ile: 0.002 s
Read times Min:       0.002 s


Cleaning up 256 objects...
Deleting a batch of 256 objects in range {0, 255}... Succeeded
Successfully deleted 256/256 objects in 491.3857ms
```

*进一步分析：吞吐率Throughput*、*延迟Latency*，与环境参数：*对象尺寸object size*、*并发性*、*服务器数量*的关系。

思考:

- 对象尺寸如何影响性能?
  - 对于熟悉的某类应用，根据其数据访问特性，怎样适配对象存储最合适?
- I/O 延迟背后的关键影响要素?
  - 首先要采集全面的 I/O 延迟观测数据。
  - 百分位延迟观测需使用s3bench，然后即可分析尾延迟影响因素。
- 如果客户端爆满将怎样?
- 测试项为何出现 '**fail**'? (不是 terminate)
- 横向扩展系统 (Scaling Out) 效果如何 (向系统中追加更多存储服务器)?