#H2 数据库

The H2 in memory database is very convenient for development because your evolutions are run from scratch when play is restarted. If you are using anorm you probably need it to closely mimic your planned production database. To tell h2 that you want to mimic a particular database you add a parameter to the database url in your application.conf file, for example:

```scala
db.default.url="jdbc:h2:mem:play;MODE=MYSQL"
```


##目标数据库
* H2 does not have a uuid() function. You can use random_uuid() instead. Or insert the following line into your 1.sql file:

```
CREATE ALIAS UUID FOR "org.h2.value.ValueUuid.getNewRandom";
```

* Text comparison in MySQL is case insensitive by default, while in H2 it is case sensitive (as in most other databases). H2 does support case insensitive text comparison, but it needs to be set separately, using SET IGNORECASE TRUE. This affects comparison using =, LIKE, REGEXP.

MySql	MODE=MYSQL	
DB2	MODE=DB2	
Derby	MODE=DERBY	
HSQLDB	MODE=HSQLDB	
MS SQL	MODE=MSSQLServer	
Oracle	MODE=Oracle	
PostgreSQL	MODE=PostgreSQL	


##防止内存数据库重置
H2 drops your database if there are no connections. You probably don’t want this to happen. To prevent this add `DB_CLOSE_DELAY=-1` to the url (use a semicolon as a separator) eg: `jdbc:h2:mem:play;MODE=MYSQL;DB_CLOSE_DELAY=-1`


##H2 浏览器
You can browse the contents of your database by typing `h2-browser` at the play console. An SQL browser will run in your web browser.


##H2 文档
More H2 documentation is available [from their web site](http://www.h2database.com/html/features.html).