---
title: git commits 统计分析
date: 2016-08-27 12:45:57
categories:
  - 工具
tags:
  - git commits
  - statistic logs
---

随着一个项目的推进，项目的代码越来越多，团队成员也可能越来越多。这时候，如何跟踪项目的演进对于项目管理者来说就成了一个问题。项目管理者往往需要知道项目代码数量随着时间的推移发生的变化、每天/每周/每月/每年的提交情况以及每个贡献者的提交情况等等。而这些统计信息无法直接从 git log 中直接获取，最近发现了一个叫[GitStats](http://gitstats.sourceforge.net/)的工具可以从 git 提交历史中自动生成这些统计信息，并生成 HTML 形式的报告。下面简要介绍一下这个工具。

<!-- more -->

GitStats 生成的统计信息主要有四个方面：

## 一般统计信息
包括以下几项
* 项目名称
* 统计的时间段
* 总共的文件数目
* 总共的代码行数
* 总共提交次数、每天平均提交次数
* 贡献者数量、平均每个贡献者的提交次数

下图是一个例子：

![一般统计信息](/images/blogs/statistic-git-commits/general_info.png)

## 活动信息
包括：
* 过去32周每周的提交次数
![weekly_activity](/images/blogs/statistic-git-commits/weekly_activity.png)
* 按小时统计的提交次数
![hours_of_day](/images/blogs/statistic-git-commits/hours_of_day.png)
* 按星期统计的提交次数
![days_of_week](/images/blogs/statistic-git-commits/days_of_week.png)
* 按小时和星期统计的提交次数
![hours_of_week](/images/blogs/statistic-git-commits/hours_of_week.png)
* 按月份统计的提交次数
![month_of_year](/images/blogs/statistic-git-commits/month_of_year.png)
* 按年统计的提交次数
![year](/images/blogs/statistic-git-commits/year.png)
* 按月份和年统计的提交次数
![weekly_activity](/images/blogs/statistic-git-commits/year_month.png)

## 贡献者信息
* 贡献者列表（包括名字、提交次数以及占总提交次数的百分比，第一次提交日期，最后一次提交日期，加入项目的天数、活跃的天数等等
![authors](/images/blogs/statistic-git-commits/authors.png)

* 每位贡献者累积添加的代码行数曲线
![lines_per_authors](/images/blogs/statistic-git-commits/lines_per_authors.png)

* 每位贡献者的提交次数曲线
![commits_per_author](/images/blogs/statistic-git-commits/commits_per_author.png)

* 每位贡献者每个月的提交次数
![authors_per_months](/images/blogs/statistic-git-commits/authors_per_months.png)

* 每位贡献者每年的提交次数
![authors_per_year](/images/blogs/statistic-git-commits/authors_per_year.png)

## 文件信息
包括
* 总文件数
* 代码总行数
* 平均文件大小
* 文件数量随时间变化曲线
* 各种类型文件的数量及占的百分比


## 代码信息
* 总代码行数
* 代码行数随时间的变化曲线

总之，自动化地使用 GitStats 生成项目的统计信息能够大大减轻了对多个代码仓库各项指标的跟踪工作，易于管理者统计一段时间来仓库的变化以及每一位贡献者的提交统计。

看了这么多，如果你对 GitStats 感兴趣并想尝试一下，可以 clone 我的[代码](https://github.com/cartosquare/git-commits-vis.git)来批量生成你的所有仓库的统计信息。
