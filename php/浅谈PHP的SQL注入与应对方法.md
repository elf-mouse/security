# 浅谈PHP的SQL注入与应对方法

SQL注入攻击是黑客攻击网站最常用的手段。如果你的站点没有使用严格的用户输入检验，那么常容易遭到SQL注入攻击。SQL注入攻击通常通过给站点数据库提交不良的数据或查询语句来实现，很可能使数据库中的纪录遭到暴露，更改或被删除。下面来谈谈SQL注入攻击是如何实现的，又如何防范。

看这个例子：

```php
// supposed input
$name = "ilia'; DELETE FROM users;";
mysql_query("SELECT * FROM users WHERE name='{$name}'");
```

很明显最后数据库执行的命令是：

```php
SELECT * FROM users WHERE name=ilia; DELETE FROM users
```

这就给数据库带来了灾难性的后果--所有记录都被删除了。

不过如果你使用的数据库是MySQL，那么还好，`mysql_query()`函数不允许直接执行这样的操作（不能单行进行多个语句操作），所以你可以放心。如果你使用的数据库是SQLite或者PostgreSQL，支持这样的语句，那么就将面临灭顶之灾了。

上面提到，SQL注入主要是提交不安全的数据给数据库来达到攻击目的。为了防止SQL注入攻击，PHP自带一个功能可以对输入的字符串进行处理，可以在较底层对输入进行安全上的初步处理，也即Magic Quotes。(php.ini `magic_quotes_gpc`)。如果`magic_quotes_gpc`选项启用，那么输入的字符串中的单引号，双引号和其它一些字符前将会被自动加上反斜杠。

但Magic Quotes并不是一个很通用的解决方案，没能屏蔽所有有潜在危险的字符，并且在许多服务器上Magic Quotes并没有被启用。所以，我们还需要使用其它多种方法来防止SQL注入。

许多数据库本身就提供这种输入数据处理功能。例如PHP的MySQL操作函数中有一个叫`mysql_real_escape_string()`的函数，可将特殊字符和可能引起数据库操作出错的字符转义。

看这段代码：

```php
//如果Magic Quotes功用启用
if (get_magic_quotes_gpc()) {
  $name = stripslashes($name);
}else{
  $name = mysql_real_escape_string($name);
}
mysql_query("SELECT * FROM users WHERE name='{$name}'");
```

注意，在我们使用数据库所带的功能之前要判断一下Magic Quotes是否打开，就像上例中那样，否则两次重复处理就会出错。如果MQ已启用，我们要把加上的去掉才得到真实数据。

除了对以上字符串形式的数据进行预处理之外，储存Binary数据到数据库中时，也要注意进行预处理。否则数据可能与数据库自身的存储格式相冲突，引起数据库崩溃，数据记录丢失，甚至丢失整个库的数据。有些数据库如 PostgreSQL，提供一个专门用来编码二进制数据的函数`pg_escape_bytea()`，它可以对数据进行类似于Base64那样的编码。

如：

```php
// for plain-text data use:
pg_escape_string($regular_strings);
// for binary data use:
pg_escape_bytea($binary_data);
```

另一种情况下，我们也要采用这样的机制。那就是数据库系统本身不支持的多字节语言如中文，日语等。其中有些的ASCII范围和二进制数据的范围重叠。

不过对数据进行编码将有可能导致像LIKE abc%　这样的查询语句失效。
