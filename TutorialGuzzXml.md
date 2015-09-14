## core configuration ##

> The guzz.xml is the core configuration file of guzz framework.

> It defines everything for guzz, including application configurations, database usages, ORM, fundamental services and so on.

**guzz.xml sample:**

```
<guzz-configs>
	
	<config-server>
		<server class="org.guzz.config.LocalFileConfigServer">
			<param name="resource" value="guzz_app.properties" />
		</server>
	</config-server>

        <dialect class="org.guzz.dialect.H2Dialect"></dialect>
	<dialect name="mysql5dialect" class="org.guzz.dialect.Mysql5Dialect" />
	<dialect name="oracle10gdialect" class="org.guzz.dialect.Oracle10gDialect" />
	
	<tran>
		<dbgroup name="default" masterDBConfigName="masterDB" />
		<dbgroup name="log" masterDBConfigName="masterDB" dialectName="mysql5dialect" />
		<dbgroup name="oracle" masterDBConfigName="oracleDB" dialectName="oracle10gdialect" />
	</tran>	
	
	<import resource="include/part2.xml" />

	<business name="user" dbgroup="default" class="org.guzz.test.User" interpret="" file="classpath:org/guzz/test/User.hbm.xml" />
	<business name="book" class="org.guzz.test.Book" file="classpath:org/guzz/test/Book.hbm.xml" />
	<business name="userInfo" dbgroup="oracle" class="org.guzz.test.UserInfo" file="classpath:org/guzz/test/UserInfo.hbm.xml" />
	<business name="userInfo2" dbgroup="default" file="classpath:org/guzz/test/UserInfoH2.hbm.xml" />
	<business name="comment" dbgroup="default" file="classpath:org/guzz/test/Comment.hbm.xml" />
	<business name="cargo" dbgroup="default" file="classpath:org/guzz/test/shop/Cargo.hbm.xml" />
	<business name="sp" dbgroup="default" file="classpath:org/guzz/test/shop/SpecialProperty.hbm.xml" />

	<service name="onlyForTest" configName="onlyForTestConfig" class="org.guzz.test.sample.SampleTestService" />
	<service name="onlyForTest2" configName="onlyForTest2Config" class="org.guzz.test.sample.SampleTestService2" />
	
	<sqlMap dbgroup="default">
		<select id="selectUser" orm="user" dbgroup="default">
			select * from @@user
			 where 
			 	@id = :id and @status = :checked
                        <paramsMapping>
				<map paramName="userName" propName="userName" />
				<map paramName="checked" type="int" />
			</paramsMapping>
		</select>
		
		<select id="selectUserByName" orm="user">
			select @id, @userName, @vip, @password from @@user where @userName = :userName
		</select>
	
		<update id="updateUserFavCount" orm="userObjectMap">
			update @@user set @favCount = favCount + 1
		</update>
		
		<select id="selectUsers" orm="userObjectMap">
			select @id, @name, @vip, @favCount from @@user
		</select>
		
		<select id="listCommentsByName" orm="commentMap">
			select-* from @@commentMap where @userName = :userName
		</select>
				
		<orm id="userObjectMap" class="org.guzz.test.UserModel">
			<result property="id" column="pk"/>
		    <result property="name" column="userName"/>
		    <result property="favCount" column="FAV_COUNT"/>
		    <result property="vip" column="VIP_USER"/>
		</orm>
		
		<orm id="commentMap" class="org.guzz.test.Comment" table="TB_COMMENT" shadow="org.guzz.test.CommentShadowView">
			<result property="id" column="id" type="int"/>
		    <result property="userId" column="userId" type="int"/>
		    <result property="userName" column="userName" type="string" />
		    <result property="createdTime" column="createdTime" type="datetime" />
		</orm>
	</sqlMap>

</guzz-configs>
```

Notes:

**config-server:**Designate the service of the configuratios management servers where guzz should read configurations from. The default implementation of the service is to read from a local properties file like this sample. guzz\_app.properties if the properties file located in the same folder of the guzz.xml.

**guzz\_app.properties:** A properties file stored with application's configurations. The file content is orgnized by group just like what mysql does. We will talk more in the next chapter. The example is for introducing parameters to the org.guzz.config.LocalFileConfigServer.

**dialect:** The dialect of database, just like what hibernate has done。Currently, guzz's dialect is used for maintaining primary key and physical page navigation. Unlike hibernate, you can define many dialects at once with different dialectNames for differenct database providers in a single system. The default dialect name is "default".

**tran:** To define the database groups. One transcation can own several dbgroups with each dbgroup designating a master and a slave database configuration group name, and dialect name (to identify the database provider. Default dialect name is "default"). So, guzz can query the database configurations from the configuration service with the master/salve configuration name. In the above example, we set up 3 dbgroups, the default H2 database group, the log database group in mysql, and the oracle database group. The connection parameters of the databases is stored in guzz\_app.properties.

**business:** To declare the domain object used in the system. One domain object corresponds one table in the database. The domain object must be given a short (nick) name (hereafter businessName), which will represent the class-name in guzz. Business can define the dbgroup attributes, marking the storage database of the object in the so many database groups, leaving blank means the "default" dbgroup. The "file" attribute means the location of hbm.xml file. The "interpret" attibute controls advanced taglib conditions, and is optional. Another optional attibute "class" is used to override the domain class defined in the hbm.xml.

**import:** Import a sub configuration file. The "resource" attribute designate the location of the sub-file related to this file. "import" will copy all contents of the sub-file to the position of "import", and then translate the configuration file as a whole. The mechanism is the same as "marco" in the Assemble Language, or inline function in c++.

**Service:** To declare Service for the application. "name" is the service name and the only flag you can get your service back in programming; "configName" is the name of the configuration group in "guzz\_app.properties". "class" is the full qulified class name of the service's client implementor. "class" must declare the interface of "org.guzz.Service".

**sqlMap:** Declare the sql and its OR mapping, like ibatis. The attribute "dbgroup" declares the database group where the sql should be executed. The default "dbgroup" is "default".

**sqlMap/select|update:** Define sql statements for query/update/delete, identified by the "id" attribute(the name to find it in programming). The "orm" attribute declares how to map the query result to result returned to user. The sql statements should be standard sqls that can be executed directly in database, besides a little short-hand tip: You can use @@businessName or @@full-class-name to replace the table name, and @property-name-in-java-pojo or @orm/result/property to replace the column name in database. The tip is used when you don't want to remember both the name in java and the database.

> When you are writing a sql with paraments, replace it with ":parameterName", for example: ":userName". In java, pass the parameters in a Map (eg:userName->value) when executed the sql. Guzz will bind the parameters for you in the PreparedStatement way.

> Inside guzz, when binding parameters, the data type is required. In default way, the data type is resolved by the JDBC driver, and its fine for most of the case. However, in some special features, for example in "Dynamic SQL" with unknown parameter, you have to designate data-type manually. To declare the data type  required of a parameter, you just have to map the parameter name with the property name of the domain object it queried. Guzz will use the property's data-type for the parameter. The mapping is declared by adding "paramsMapping/map" in "select/update" fragment. For example, sql named selectUser" in the above sample, guzz will understand the parameter "userName" is a string and "checked" is a integer. Even if you pass "checked" as a string "1", guzz will convert it into a integer 1 for you.

> The "orm" attribute of "select|update" can be the "orm" name defined in this file, or the businessName defined in "business" element(borrow the mapping definition in the hbm.xml or annotation). The result of the executed sql will be, of course, transformed to the result class defined in the "orm" used. However, you can specify the attribute of "result-class" to change the returned transformed bean object for the very sql. The result-class can be a javabean or a Map, for example: result-class="org.guzz.SomeBeanForm", or result-class="java.util.HashMap".

**orm:** Treat it as in ibatis, it defines how to map the queried ResultSet to java object. Unlike ibatis, you can specify the attribute of "dbgroup" to override the dbgroup inherited from the outer "sqlMap". The "orm" definition can be put in a "sqlMap" to function in that "sqlMap" or out of it to be a global one. The "orm" name must be unique from the businessName, because inside guzz, every business declares its businessName to be a global "orm". That is why you can use businessName in "orm" attribute above. The "orm" defined inside the "sqlMap" has a higher priority over global orms.

> If a sql choose a businessName as its orm, it will inherit all features in the hbm.xml definitions, including table shadowing, lazy load, and so on.

**orm/result:** Each "result" represents a field mapping of the ORM. "property" is the java property name, "column" is the column name in the table, and "type" is the data type of the property. The "type" attribute is optional, it can be detected by reflection automaticly.


## Special characters escaping in sqls ##

Some characters are retained by guzz which should be escaped while concating them to sql values.

The characters:
| character | escaped |
|:----------|:--------|
| :         | \:      |
| @         | \@      |
| '         | \'      |
| \         | \\      |
| "         | \"      |

For example:
```

select * from @@article where @title=:title and @createdTime > to_date('20110101 0\:0\:0', 'YYYYMMDD HH24\:MI\:SS')

```

In the above example, ":" in the time segments are escaped to "\:", or guzz will treat them as named parameters and cause errors.

**Caution:** If the parameter values are passed as named parameters, do not escape anything.


## Dialect and Database Providers Supported ##

| **Database Provider** | **Dialect** | **Notes** |
|:----------------------|:------------|:----------|
| Mysql 5.0+            | org.guzz.dialect.Mysql5Dialect | older versions may work too, but not tested. |
| Oracle 8i             | org.guzz.dialect.Oracle8iDialect | -         |
| Oracle 9i             | org.guzz.dialect.Oracle9iDialect | -         |
| Oracle 10g            | org.guzz.dialect.Oracle10gDialect | -         |
| H2 Database           | org.guzz.dialect.H2Dialect| -         |
| Microsoft SQL Server 2000+ | org.guzz.dialect.MSSQLDialect| Page Navigation is not supported. Not Tested! |

## Data Types Supported ##

| **Database Provider** | **data type in database** | **type value in guzz** | **Java type** | **Notes** |
|:----------------------|:--------------------------|:-----------------------|:--------------|:----------|
| All                   | int                       | int                    | int           | integer   |
| All                   | string                    | string                 | java.lang.String | String    |
| All                   | varchar                   | string                 | java.lang.String | String    |
| All                   | nvarchar                  | string                 | java.lang.String | String    |
| All                   | char                      | string                 | java.lang.String | String    |
| All                   | nchar                     | string                 | java.lang.String | String    |
| All                   | text                      | string                 | java.lang.String | String    |
| All                   | datetime                  | datetime               | java.util.Date or java.sql.Timestamp | Timestamp, including date and time |
| All                   | timestamp                 | datetime               | java.util.Date or java.sql.Timestamp  | Timestamp, including date and time |
| All                   | date                      | date                   | java.util.Date or java.sql.Date | date      |
| All                   | time                      | time                   | java.util.Date or java.sql.Time | time      |
| All                   | bool                      | boolean                | boolean       | boolean   |
| All                   | boolean                   | boolean                | boolean       | bolean    |
| All                   | bigint                    | bigint                 | long          | long      |
| All                   | long                      | bigint                 | long          | long      |
| All                   | double                    | double                 | double        | double    |
| All                   | money                     | decimal                | java.math.BigDecimal | currency  |
| All                   | decimal                   | decimal                | java.math.BigDecimal | currency  |
| All                   | float                     | float                  | float         | float     |
| All                   | short                     | short                  | short         | short     |
| All                   | smallint                  | short                  | short         | short     |
| All                   | tinyint                   | short                  | short         | short     |
| All                   | byte                      | byte                   | byte          | one byte  |
| All                   | bit                       | byte                   | byte          | one byte  |
| All                   | bytes                     | bytes                  | byte`[]`      | byte array |
| All                   | binary                    | bytes                  | byte`[]`      | byte array |
| All                   | varbinary                 | bytes                  | byte`[]`      | byte array |
| All                   | clob                      | clob                   | org.guzz.pojo.lob.TranClob | clob      |
| All                   | blob                      | blob                   | org.guzz.pojo.lob.TranBlob | blog      |
| All                   | int                       | enum.ordinal|enum.class.fullname | enum          | java enum. The type in database is int. enum.class.fullname is the full qulified class name of the enum class. |
| All                   | varchar                   | enum.string|enum.class.fullname | enum          | java enum. The type in database is varchar. enum.class.fullname is the full qulified class name of the enum class. |
| Microsoft SQL Server  | image                     | bytes                  | byte`[]`      | byte array |
| Microsoft SQL Server  | varbinary                 | bytes                  | byte`[]`      | byte array |
| Oracle8i+             | oracle long               | Oracle.Long            | java.lang.String | String    |
| Oracle8i+             | varchar2                  | string                 | java.lang.String | String    |
| Oracle8i+             | nclob                     | clob                   | org.guzz.pojo.lob.TranClob | clob      |
| Oracle8i+             | raw                       | bytes                  | byte`[]`      | byte array |