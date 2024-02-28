kubectl get -获取资源列表

```
#获取类型为Deployment的资源列表
kubectl get deployments

#获取类型为Pod的资源列表
kubectl get pods

#获取类型为Node的资源列表
kubectl get nodes
```

> 使用-A或--all-namespaces可以查看所有命名空间中的对象 -n指定命名空间
>
> 并非所有对象都在命名空间中

**kubectl describe** - 显示有关资源的详细信息

```
# kubectl describe 资源类型 资源名称

#查看名称为nginx-XXXXXX的Pod的信息
kubectl describe pod nginx-XXXXXX	

#查看名称为nginx的Deployment的信息
kubectl describe deployment nginx	
```

**kubectl logs** - 查看pod中的容器的打印日志（和命令docker logs 类似）

```
# kubectl logs Pod名称

#查看名称为nginx-pod-XXXXXXX的Pod内的容器打印的日志
#本案例中的 nginx-pod 没有输出日志，所以您看到的结果是空的
kubectl logs -f nginx-pod-XXXXXXX
```

**kubectl exec** - 在pod中的容器环境内执行命令(和命令docker exec 类似)

```
# kubectl exec Pod名称 操作命令

# 在名称为nginx-pod-xxxxxx的Pod中运行bash
kubectl exec -it nginx-pod-xxxxxx /bin/bash
```

