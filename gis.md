# 拉取镜像

```sh
docker pull kartoza/geoserver

docker pull kartoza/postgis:9.6-2.4
```



# 启动数据库

```shell
docker run --name=postgis -d -e POSTGRES_USER=geogis -e POSTGRES_PASS=fangyc111 -e POSTGRES_DBNAME=gis -e ALLOW_IP_RANGE=0.0.0.0/0 -p 5432:5432 -v pg_data:/var/lib/postgresql --restart=always kartoza/postgis:9.6-2.4
```


> 数据库
>
> 账号：geogis
>
> 密码：fangyc111
>
> 地址：127.0.0.1:5432



# 使用qgis将地图导入数据库

这一步百度



# 数据库操作

1. 进入postgis

```shell
docker exec -it postgis /bin/bash
```

2. 添加pgrouting扩展

```shell
psql -h 127.0.0.1 -p 5432 -U geogis -d gis -c "create extension pgrouting"
```

3. 添加列

```sql
alter table lines.heishan（替换成自己的表名）add column source int;

alter table lines.heishan add column target int;

```

4. 创建拓扑（具体还有什么功能得百度）

```sql
select pgr_createTopology('lines.heishan', 0.000001, 'geom');
//把路线分成一段一段的，用来后面计算距离（只做这一步效果不好，所以这步不做，从下面开始）

select pgr_nodeNetwork('lines.heishan', 0.0000001,'id','geom');
//有时候地图原始的路线不利于分点，这个函数用来生成更多的点

select pgr_createTopology('lines.heishan_noded', 0.000001, 'geom');
//用修改后的地图重新创建拓扑，效果好一点
```

5. 计算路径点之间的距离

```sql
alter table lines.heishan_noded add column cost double precision;

update lines.heishan_noded set cost=ST_Length(ST_TransForm(geom,4326),true)
```

6. 为点添加名字（可选）

```sql
alter table lines.heishan_noded_vertices_pgr add column name int;

update lines.heishan_noded_vertices_pgr set name=id
```



# 查询

```sql
select *from pgr_dijkstra('SELECT id,source,target,cost,cost as reverse_cost FROM lines.heishan_noded', 2,1)as di
join lines.heishan_noded pt
on di.edge = pt.id
```



**可以下载使用pgAdmin便捷查看数据库结构**





# geoserver 操作



1. 获取docker虚拟机ip

```shell
docker inspect --format='{{.NetworkSettings.IPAddress}}' $(docker ps -a -q)
```



2. 启动geoserver

```shell
docker run --name "geoserver" -e USERNAME='geogis' -e PASS='fangyc111'  --link postgis:postgis -p 8080:8080 -d -t kartoza/geoserver
```



> 访问127.0.0.1:8080/geoserver
>
> 账号 Admin
>
> 密码 geoserver
>



3. 添加数据源（百度）



# 前端部分

按照openlayers官网教程进行即可

代码见附件





# docker相关指令（选读）



```shell
# 关闭容器
docker stop geoserver

# 移除关闭的所有容器
docker container prune

# 启动容器
docker start geoserver

# 移除容器
docker rm geoserver
```







# 修改tomcat默认密码（选读）



docker exec -it geoserver /bin/bash



cat $CATALINA_HOME/conf/tomcat-users.xml



apt update

apt install sed

apt install vim



vim $CATALINA_HOME/conf/tomcat-users.xml



\#添加

<role rolename="manager-gui"/>

<role rolename="admin-gui"/>

<user username="tomcat" password=“fangyc111” roles="manager-gui,admin-gui”/>



重启容器



# 为数据库添加扩展的另一种方法（选读）

\#1.进入数据库的docker容器

docker exec -it postgis /bin/bash

\#2.添加用户用以访问数据库（此处添加一个和数据库一样的用户？如果已经有这个用户了就不需要这一步）

adduser geogis

​	fangyc111

\#3.切换用户，添加pgrouting扩展

su - geogis

psql gis -c "create extension pgrouting"



# 引用

[基于PostGIS+PgRouting的最短路径查询的实现（一）：数据库篇](https://blog.csdn.net/u012413551/article/details/85084961)

[基于PostGIS+PgRouting的最短路径查询的实现（二）：Geoserver篇](https://blog.csdn.net/u012413551/article/details/85145966)

[基于PostGIS+PgRouting的最短路径查询的实现（三）：Openlayers篇](https://blog.csdn.net/u012413551/article/details/85217105)

[Geoserver入门操作系列之一：发布地图服务](https://blog.csdn.net/u012413551/article/details/87999686)

[Geoserver入门操作系列之二：图层样式](https://blog.csdn.net/u012413551/article/details/88046986)

[Generating SLD styles with QGIS](https://docs.geoserver.org/latest/en/user/styling/qgis/index.html)

[Building an OpenLayers Application](https://openlayers.org/en/latest/doc/tutorials/bundle.html)



# 下一步工作

在真实的场景中用户进行路径规划的时候都是基于经纬度数据进行路径规划的，因为用户根本不会知道道路上节点的ID。因此查询任意两点间的最短路径是下一步可能遇到的问题。

参考博客：[基于pgrouting的任意两点间的最短路径查询函数](https://blog.csdn.net/longshengguoji/article/details/46051111)
