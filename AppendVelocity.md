## 背景 ##

> 本人在公司项目中使用了近两年的velocity，根据我自己的实际项目需要，在guzz中增加了很多velocity支持。

> 以前放在halo-cloud工程中，由于具备通用性，在 guzz 1.3.1 版，将其中完全通用的部分挪到了guzz主项目中。


## guzz对velocity的支持 ##

> 默认的动态拼接SQL，基于velocity实现。详细信息 [TutorialTemplatedSQLService](TutorialTemplatedSQLService.md)

> 支持MMVC架构，Velocity也提供了类似JSP taglib的支持。[JSP Taglib介绍](TutorialTaglib.md)

> 对Velocity补充的其他自定义标签【Velocity中称做Directive】。

> 使用方法为将需要的Directive配置到你项目的velocity engine中即可【使用Velocity userdirective参数配置】。

> 可参考的使用例子：http://code.google.com/p/halo-cloud/source/browse/JavaClient/src/main/java/misc/com/guzzservices/velocity/impl/VelocityEngineService.java


## Guzz's Velocity Directive ##

> 如果你用过 [Guzz JSP Taglib](TutorialTaglib.md) ，Velocity Directives 就是翻版。

> 可以直接理解为在Velocity模板中，也可以通过声明的方式读取数据库数据，或是对字符串进行xml转义，过滤javascript等。

**Guzz提供的Directive：**

| **标签**  | **用途**  | **标签体内可否写其他代码**  |
|:--------|:--------|:-----------------|
| isEmpty  | 判断条件是否为空。 为空的条件是：如果 param 为null，算做空；如果 param 为String字符串，且 param.trim().length() == 0，算作空；如果 param 为Collection集合，且 集合为空，则算作空；如果 param 为数组，且数组长度为0，则算作空；其他情况，视作不为空。 | 可以               |
| notEmpty  | 与isEmpty相反| 可以               |
| escapeXml  | 输出标签。对xml进行转义。| 不可以              |
| escapeJS  | 输出标签。过滤html javascript代码| 不可以              |
| utf8encoding  | 输出标签。将参数按照utf-8编码后输出 | 不可以              |
| guzzBoundary  | 定义条件范围。在此标签内的所有标签，都将自动获取其设定的查询条件。  | 可以               |
| guzzAddLimit  | 对当前的g:boundary中增加一个查询条件。 | 不可以              |
| guzzAddInLimit  | 对当前的g:boundary中增加一个in操作查询条件。 | 不可以              |
| guzzCount  | 执行统计操作  | 不可以              |
| guzzGet  | 执行list操作，并读取list的第一个对象。  | 不可以              |
| guzzList  | 读取一个列表。  | 不可以              |
| guzzPage  | 读取一页对象。返回的对象默认为org.guzz.dao.PageFlip 对象。  | 不可以              |
| guzzInc  | 按照主键对域对象的1个字段进行自增操作。g:inc需要配置 [计数器更新队列服务](AppendCoreService.md) 方可生效。  | 不可以              |

> 其中guzzXXX的Directive，与JSP taglib对应的g:XXX标签用途，含义，属性完全一样。请直接阅读 [Guzz JSP Taglib 帮助](TutorialTaglib.md)

> Directive的标签和taglib写法不一样，下面举例说明下实际的使用方法。


**示例指南：**

> 写法大致是
```

#guzzBoundary()
	#guzzAddLimit(true, ["status!=5"])
	#guzzList({"business":"gift", "var": "m_gifts",  "limit" : "toUserId=$user.userId",  "pageNo" : 1, "pageSize" : 6,  "orderBy" :  "id desc" })
#end

```

将执行的sql语句为：
```
Select * from tb_gift where status!=5 and toUserId=...order  by  id desc
```


这样就可以按照vm中的变量引用规则来遍历这个list了：

```
#foreach($m_gift in $m_gifts)
##显示数据
........
.........
#end
```

如果查询条件只有一个，那么可以直接这样写：（省略guzzBoundary及guzzAddLimit标签）
```
#guzzList({"business":"gift", "var": "m_gifts",  "limit" : "toUserId=$user.userId",  "pageNo" : 1, "pageSize" : 6,  "orderBy" :  "id desc" })
```

这里要注意几个问题：

(1) guzz标签中的business指的是针对的实体名称，也就是与数据库表映射的实体类名，并不是数据库表名。

(2) 如果这里不需要指定要查询的条数，要全部显示，pageSize属性设置成一个非常大的值。

(3) 如果查询条件较多，可以继续增加guzzAddLimit标签以设置多个查询条件，但条件之间为“与”的关系，不支持“或”的关系。比如：
```

#guzzBoundary()
	#guzzAddLimit(true, ["status=1"])
	#guzzAddLimit(true,["public=1"])
	#guzzAddLimit(true,["anonymous=1"])
	#guzzList({"business":"gift", "var": "m_gifts",  "limit" : "toUserId=$user.id",  "pageNo" : 1, "pageSize" : 6,  "orderBy" :  "id desc" })
#end

```

(4) guzz标签中可以引用vm中的变量，比如上述例子中就使用了$user.userId这个变量


## 详细介绍下这里所使用到的guzz标签 ##

**#guzzList()** 根据limit字段中设置的条件查询结果，并返回一个list。“business”：“gift”指出该查询针对的实体是gift，"var":"m\_gifts"用来设置返回的list所在变量，"limit":"toUserId=$user.userId"用来设置条件,pageNo为页号，pageSize为每页显示条数，"orderBy":"id desc"为按id倒序排列。

**#guzzAddLimit()** 用来添加查询条件的，在这里我们设置要查询对象的status不等于5。

**#guzzBoundary()** 用于划定条件添加范围。放在 list/get/page等数据读取标签外部，以#end结束。

在 **#guzzBoundary** 与#end之间加入 **guzzAddLimit** 标签，表示将guzzAddLimit设置的条件附加到随后出现的guzzList等查询请求中。


**#guzzGet** 用来查询数据库中的单条记录，写法如下：

```
#guzzGet({"business":"giftInfo", "var": "m_gift_info",  "limit" : "id=$infoId", "orderBy" :  "id desc" })
```

其中business指定要查询的实体对象为giftInfo。var表示查询后的结果用变量m\_gift\_info来存储引用，limit表示查询条件是id=$infoId, orderBy表示查询结果按id倒序后取第一个。



**更完整的例子：**

```
<!--查出所有符合limit条件礼物记录的前六条记录-->
#guzzBoundary()
	#guzzAddLimit(true, ["status=1"])
	#guzzAddLimit(true,["public=1"])
	#guzzAddLimit(true,["anonymous=1"])
	#guzzList({"business":"gift", "var": "m_gifts",  "limit" : "toUserId=$user.id",  "pageNo" : 1, "pageSize" : 6,  "orderBy" :  "id desc" })
#end
            
    #if($m_gifts.size()>0)  
		  <div class="sns_liwu clearfix">
		    <ul>
           <!--对找到的m_gifts进行遍历显示-->
             #foreach($m_gift in $m_gifts)
              <li>
              #set($infoId=$m_gift.giftInfoId)
             <!--根据礼物信息ID查询礼物信息表，得到此礼物信息-->
              #guzzGet({"business":"giftInfo", "var": "m_gift_info", "limit" : "id=$infoId", "orderBy" :  "id desc" })
			  <a href="" ><img src="$m_gift_info.imageUrl" border="0" width="50" height="50" alt="$m_gift_info.name" /></a><br /> 
              <a href="">$m_gift_info.name</a>
               </li>
              #end
			 </ul>
		  </div>
      #else
           	还没有收到礼物
      #end

```


## 更多示例 ##

> 分页查询（查询到的结果为 PageFlip 对象）：
```
#guzzBoundary()
	#guzzAddLimit(true, ["averageScore>0"])
	#guzzPage({"business":"filmInfo", "var": "films",  "limit" : "status=1", "pageNo" : 0, "pageSize" : 10,  "orderBy" :  "averageScore desc" })<!-- 好评电影 -->
#end
```


utf8encoding:
```
<td><a target="_blank" href="$!{mdomain}/xxxxx.do?ut=1&user=#utf8encoding(${user.nickname})">检索</a></td>
```











