---
layout: post
title: "实时更新OSM数据库"
date: 2016-05-07
categories:
  - GIS
tags:
  - OSM
---

巧妇难为无米之炊。对于制图者，地图数据便是一切的基础。[OpenStreetMap](http://www.openstreetmap.org/)（简称 OSM ）是一个全球路网数据（不仅仅是路网数据，还包括行政区划、自然要素等数据）的众包平台，所有人都可以免费得到这份全球的数据。由于众包的性质，OSM 的数据每分钟都在发生变化，因此，维护一个实时（分钟级的频率）更新的 OSM 数据源能够使我们的地图有更好的时效性。

<!-- more -->

## 依赖的工具

* [osm2pgsql](https://github.com/openstreetmap/osm2pgsql) osm2pgsql  可以将 OSM 的数据导入 PostgreSQL 中，转化成易于渲染的结构，并且支持增量更新。

* [osmosis](https://github.com/openstreetmap/osmosis) osmosis 是一个 java 的命令行工具，主要用来进行 OSM 数据的各种格式之间的转换，这里用来从 OSM 远程服务器中获取更改集，从而 osm2pgsql 可以将此更改集增量应用到数据库中。

* PostgreSQL, 带 postgis 拓展



## 初始化数据库

下载[全球](http://planet.openstreetmap.org/)或者[某个区域](http://download.geofabrik.de/index.html)的 osm 数据，最好选择 pbf 格式，相比于 xml 格式，pbf 格式会小很多。下载的时候记下数据的生产时间，下面选择同步起始时间的时候会用到，这个时间在下载的页面里会有说明。

使用下述 sql 创建一个数据库，并且建立拓展。

```
createdb osm
psql -d osm -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
```

这里 osm 是我的数据库名，如果不习惯用命令行，可以用 pgAdmin 来进行上述操作。总之，现在我们有了一个全新的带有空间拓展的数据库了。

使用下面的命令导入之前下载好的数据

```
osm2pgsql -c -d osm --slim -C <75% Mem> --flat-nodes <flat nodes> -U gis -W -H localhost -P 5432 you.osm.pbf
```

这里有几个地方要注意，一个是*-C*选项最好指定你电脑的内存的75%，单位为 Mb，并且最大只能为30000，*--flat-nodes* 用来指定一个文件路径，存放这个文件的位置至少要有20G的空闲空间（如果导入全球数据的话）。
执行这个命令可能会耗费一段时间，特别是导入全球数据，可能需要几天，如果导入全国的数据，我用8G内存的 Macbook Pro 只需要不到一个小时。注意虽然我们下载的数据量可能不是很大，但是导入到数据库后会占据很可观的数据磁盘空间，我的笔记本里全国的数据占据了接近10G的空间，因此导入比较大的范围的数据需要保证有足够的磁盘空间可用。

## 设定更新频率

首先指定更新的工作目录。下面的这行可以放到主目录下的 .bash_profile 里，如果是 windows 用户可以新建一个系统的环境变量。

```
export WORKDIR_OSM=$HOME/.osmosis
```

初始化工作目录

```
mkdir $WORKDIR_OSM
osmosis --read-replication-interval-init workingDirectory=$WORKDIR_OSM
```

上面的命令告诉 *osmosis* 从哪个目录寻找更新的信息，以及下载数据到哪里。*osmosis* 会在这个目录里创建 *configuration.txt* 和 *download.lock* 这两个文件。*download.lock* 文件用来确保同一时刻只会进行一个更新任务。*configuration.txt* 文件则用来指定更新的频率。默认情况下，*osmosis* 会提取每分钟的更改集，如果想改为提取每小时或者是天的，可以把 *configuration.txt* 里的 *baseurl* 中的 *replication/minute/* 部分改为 *replication/hour/* 或者 *replication/day/*。默认情况下每次执行更新任务最多只会提取1分钟的更改集，可以把 *maxInterval=3600* 设为0，这样子就可以一次提取所有的更改集。

为了让*osmosis*知道从哪个时刻开始进行更新，我们还要告诉它我们刚刚导入的数据的时间，访问[Peter Körner's website tool](https://osm.mazdermind.de/replicate-sequences/)，输入我们的数据的时间，可以得到一个UTC格式的时间文件，把它保存到工作目录中，并命名为 *state.txt*。

执行更新任务

```
osmosis -q --rri --bc --simc --bc --write-xml-change "-" | osm2pgsql -s -a -b "73,3,136,54" -U gis -d osm -P 5432 -H localhost -e 15 -o expire.list -
```

上述的命令中，如果导入的是全球的数据就不需要有 *-b* 参数。*expire.list* 中包含的是第15级的 dirty tile（即数据有更新的瓦片），这些瓦片需要重新生成。

linux 上可以使用 crontab 命令定期执行上述命令，这样就可以得到一个以分钟级的频率与 OSM 数据保持同步的本地数据库了。

如果使用 *-b* 选项指定了更新的范围，可能会使数据库变大，此时可以使用下面的 sql 删除掉无关的 way 和 relation。

```
DELETE FROM planet_osm_ways AS w WHERE 0 = (SELECT count(1) FROM planet_osm_nodes AS n WHERE n.id = ANY(w.nodes));

DELETE FROM planet_osm_rels AS r WHERE
  0=(SELECT count(1) FROM planet_osm_nodes AS n WHERE n.id = ANY(r.parts))
AND
  0=(SELECT count(1) FROM planet_osm_ways AS w WHERE w.id = ANY(r.parts));
REINDEX TABLE planet_osm_ways;
REINDEX TABLE planet_osm_rels;
VACUUM FULL;
```


