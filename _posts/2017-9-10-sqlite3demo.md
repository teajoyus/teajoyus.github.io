---
layout: post
title: 轻量级数据库Sqlite3基础使用
date: 2016-11-20 
tags: 工具    
---

 **使用SQLite3本地数据库，运行一个demo,记录一下配置与运行结果。SQLite数据库广泛用于嵌入式系 统、桌面软件等作为本地数据库** 

本文有参考网络Blog完成


#准备工作
1、新建工程，内容如下
![这里写图片描述](http://img.blog.csdn.net/20171123110415952?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3RhcmVsZWdhbnQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
2、配置库目录 lib目录
![这里写图片描述](http://img.blog.csdn.net/20171123110508837?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3RhcmVsZWdhbnQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20171123110453848?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3RhcmVsZWdhbnQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#源码

```
#include <iostream>
#include "sqlite3.h"

#define _DEBUG_ 1P

using namespace std;

int main(int argc, char* argv[])
{
	
	sqlite3 *db = NULL;
	char *zErrMsg = 0;

	int rc;
	rc = sqlite3_open("mydemo.db", &db); //打开指定的数据库文件,如果不存在将创建一个同名的数据库文件
	if (rc)
	{
		
		cout << stderr << "Can't open database: " << sqlite3_errmsg(db) << endl;
		sqlite3_close(db);
		return (1);
	}
	else 
		cout<<"create or open Sqlite3 sucess" <<endl;
	
	//创建表  TABLE ，字段有:ID,SensorID ,SiteNum ,Time, SensorParameter
	char *sql = " CREATE TABLE SensorData(ID INTEGER PRIMARY KEY,SensorID INTEGER,SiteNum INTEGER,Time VARCHAR(12),SensorParameter REAL);";
	sqlite3_exec(db, sql, 0, 0, &zErrMsg);


	//插入数据 
	sql = "INSERT INTO SensorData VALUES(NULL , 4 , 1 , '201602011203', 18.9 );";
	sqlite3_exec(db, sql, 0, 0, &zErrMsg);

	sql = "INSERT INTO SensorData VALUES(NULL , 24 , 40 , '201602011303', 16.4 );";
	sqlite3_exec(db, sql, 0, 0, &zErrMsg);

	sql = "INSERT INTO SensorData VALUES(NULL , 34 , 15 , '201602011308', 15.4 );";
	sqlite3_exec(db, sql, 0, 0, &zErrMsg);


	int nrow = 0, ncolumn = 0;
	char **azResult; //二维数组存放结果

	//查询数据
	sql = "SELECT * FROM SensorData ";
	sqlite3_get_table(db, sql, &azResult, &nrow, &ncolumn, &zErrMsg);
	int i = 0;
	printf("row:%d column=%d  ", nrow, ncolumn);
	printf(" The result of querying is :  \r\n");
	for (i = 0; i < (nrow + 1) * ncolumn; i++)
		printf("azResult[%d] = %s  \r\n", i, azResult[i]);

	//删除数据
	sql = "DELETE FROM SensorData WHERE SensorID =0 ;";
	sqlite3_exec(db, sql, 0, 0, &zErrMsg);


#ifdef _DEBUG_
	printf("zErrMsg = %s  ", zErrMsg);
#endif
	sql = "SELECT * FROM SensorData ";
	sqlite3_get_table(db, sql, &azResult, &nrow, &ncolumn, &zErrMsg);
	printf(" row:%d column=%d ", nrow, ncolumn);
	printf(" After deleting , the result of querying is :  ");
	for (i = 0; i < (nrow + 1) * ncolumn; i++)
		printf("azResult[%d] = %s \r\n", i, azResult[i]);


	//释放掉 azResult 的内存空间
	sqlite3_free_table(azResult);

#ifdef _DEBUG_
	printf("zErrMsg = %s  ", zErrMsg);
#endif

	sqlite3_close(db); //关闭数据库



	system("pause");
	return 0;
}
```

#问题处理
1、编译不成功，主要是配置问题
![这里写图片描述](http://img.blog.csdn.net/20171123110731523?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3RhcmVsZWdhbnQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
注意根目录下放着 sqlite3.dll 
其中lib文件夹下有sqlite3.lib

2、注意使用debug x86模式，笔者编译工程时候，工程默认为x64 编译不通过，
错误提示为link 2019 slqlite3的函数未定义

工程已经上传github
[选择chapter1工程](https://github.com/Elegant2011/ElegantBlog/tree/master/chapter1/SqliteDemo "选择chapter1工程")

![微信公众号](http://photo-elegant.oss-cn-shanghai.aliyuncs.com/18-8-22/88408382.jpg)