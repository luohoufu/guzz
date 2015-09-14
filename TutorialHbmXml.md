On mapping the ResultSet to java Object, guzz supports both hibernate and ibatis's models. You can define mappings in hbm.xml for each domain class, or in guzz.xml for each sql statement.

## hbm.xml mapping as hibernate ##

> Hibernate's xxx.hbm.xml configuration file is compatible to guzz. You can import or manage the hbm.xml file by hibernate's tools as your favorite.

> If you prefer hibernate's hbm.xml file, some features of guzz may tip you errors as they are not defined in hibernate's dtd definition. Don't worry, this won't affect guzz to interpret it.

## Declare hbm.xml in guzz ##

For each hbm.xml mapping file, insert a "business" in guzz.xml:

```
<business name="user" dbgroup="default" file="classpath:org/guzz/test/User.hbm.xml" />
```

One "business" element declares one business domain class, and owns 5 attributes:

| **attribute** | **Required** | **function** |
|:--------------|:-------------|:-------------|
| name          | Required     | business name |
| file          | Required     | location of hbm.xml |
| dbgroup       | Optional     | database group to store this domain object. The default value is "default". |
| class         | Optional     | domain class name to override the "class" definition in hbm.xml. |
| interpret     | Optional     | User defined Interpreter to translate custom search terms. |

**A typical example:**
```
<business name="user" dbgroup="mainDB" class="org.guzz.test.User" interpret="" file="classpath:org/guzz/test/User.hbm.xml" />
<business name="book" class="org.guzz.test.Book" file="classpath:org/guzz/test/Book.hbm.xml" />
```

You can find all detailed configurations in guzz's test source codes.

## Definition of guzz's own mapping ##

> Guzz's class-table mapping is similar to hibernate's, change the DTD header to

```
<!DOCTYPE guzz-mapping PUBLIC "-//GUZZ//GUZZ MAPPING DTD//EN" "http://www.guzz.org/dtd/guzz-mapping.dtd">
```

> if you prefer it( to get tips of guzz's features from IDE).

> The root element of the mapping xml can be `<guzz-mapping>` or `<hibernate-mapping>`(hibernate has more mature tools in IDE), as guzz parse the mapping xml from `<class>` element and ignore the root one.

> Unlike hibernate's mapping file, a hbm.xml file can only contain one class-table mapping in guzz.

> To active guzz's unique features, some new attributes can be used in guzz's hbm.xml.

> -**3 new attributes added to element class:**

> businessName：The default business name. Can be overrided in guzz.xml while declaring it. [guzz1.3.1](since.md)

> dbGroup：The default database group to store this business objects. Can be overrided in guzz.xml while declaring it. [guzz1.3.1](since.md)

> shadow：used to split tables and active Custom Table. We'll talk more in the later chapters.

> -**2 new attributes added to element property:**

> null：decide what value to return if the value is null in database.

> loader：specify the special loading strategy for this property. Through loader, you can load a property from file system, or from cache, or even from a third-part system. Read more: LazyLoad

> -**Support defining date format for attribute 'type':**

> The attribute "type" is used to define the data type for a property, eg:int, string, etc. For date and time, you can add a "|" and the date/time format(See also in java.util.DateFormat) after the type value.

> For example: type="datetime|YYYY-MM-dd HH:mm:SS", type="date|MM-dd".

> After telling guzz the date format, you can pass date parameter in String format in query. Guzz will do the type conversation for you. This feature is very useful if you need the end user to input the date, or pass a date parameter in guzz taglib.

> -**enum support:**

> Guzz supports storing enum in database both in ordinal and value style. For ordinal, the column type of the table should be int; for value, the column type should be something of varchar.

> Define a enum worked in ordinal style: type="enum.ordinal|enum class's full qualified class name". eg: type="enum.ordinal|com.company.EmployeeStatus".
> Define a enum worked in value style: type="enum.string|enum class's full qualified class name". eg: type="enum.string|com.company.EmployeeStatus".

> -**Id generators in guzz compared with hibernate:**

| **Generator** | **hibernate** | **guzz** | **Description for guzz** |
|:--------------|:--------------|:---------|:-------------------------|
| increment     |√              |×         |It generates identifiers of type long, short or int that are unique only when no other process is inserting data into the same table. It should not the used in the clustered environment.|
| identity      |√              |√         |It supports identity columns in DB2, MySQL, MS SQL Server, Sybase and HypersonicSQL. The returned identifier is of type long, short or int.|
| sequence      | √             | √        | The sequence generator uses a sequence in DB2, PostgreSQL, Oracle, SAP DB, McKoi or a generator in Interbase. The returned identifier is of type long, short or int |
| hilo          | √             | √        | The hilo generator uses a hi/lo algorithm to efficiently generate identifiers of type long, short or int, given a table and column (by default guzz\_unique\_key and next\_hi respectively) as a source of hi values. The hi/lo algorithm generates identifiers that are unique only for a particular database. Do not use this generator with connections enlisted with JTA or with a user-supplied connection. Parameters supported: table, column, db\_group, max\_lo |
| seqhilo       | √             | √        | The seqhilo generator uses a hi/lo algorithm to efficiently generate identifiers of type long, short or int, given a named database sequence. |
| uuid          | √             | √        | The uuid generator uses a 128-bit UUID algorithm to generate identifiers of type string, unique within a network (the IP address is used). The UUID is encoded as a string of hexadecimal digits of length 32. |
| guid          | √             | √        | It uses a database-generated GUID string on MS SQL Server and MySQL. |
| native        | √             | √        | It picks identity, sequence or hilo depending upon the capabilities of the underlying database. Take it as an alias of the real identity generator used. |
| assigned      | √             | √        | lets the application to assign an identifier to the object before save() is called. This is the default strategy if no `<generator>` element is specified. |
| select        | √             | ×        | retrieves a primary key assigned by a database trigger by selecting the row by some unique key and retrieving the primary key value. |
| foreign       | √             | ×        | uses the identifier of another associated object. Usually used in conjunction with a `<one-to-one>` primary key association. |
| silent        | ×             | √        | The silent generator does nothing. Usually used when you need to manage the id in the database, for example with a trigger. |
| random        | ×             | √        | A <tt>RandomIdGenerator</tt> that returns a string of length 32 (or the length you given in parameter:length). This string will consist of only a-z and 0-9, and is unable to predicate. <br><b>Note: the length maybe a little shorter than the given length.</b> <br>Mapping parameters supported: length <br>
<tr><td> hilo.multi    </td><td> ×             </td><td> √        </td><td> Works as hilo, but allows many Ids stored in a single table disguised by the primary key of each row. Mapping parameters supported: table, column, db_group, max_lo, pk_column_name, pk_column_value(required, and must be a positive integer)</td></tr></tbody></table>

> The parameters and their meanings in guzz is similar to hibernate, on one exception: the table name of hilo is "guzz\_unique\_key" in guzz, not "hibernate\_unique\_key".

> Besides, for generator hilo, sequence and seqhilo, a new parameter "db\_group" is introduced in guzz, to identify which database group should be used when creating a primary key. If you want to make sure that id won't duplicate even between db groups, you can specify all tables to create primary key from a same db\_group. This parameter is optional, guzz will create primary key for a table in its db group in default.

> -**Notes for id generator：**

  * 1. Oracle's native id generator is sequence generator, so for oracle, you have pass the sequence name, for example:
```
            <generator class="native">
        		<param name="sequence">seq_sometable_id</param>
        	</generator>
```

> Or, guzz will the default sequence "guzz\_sequence".

> All sequences should be created by yourself, guzz won't create anything. In oralce, you can execute:

```
CREATE SEQUENCE guzz_sequence INCREMENT BY 1 START WITH 1
```

> to create it.

  * 2. In Mysql and H2 database, native means "identify". param name="sequence" will be ignored.



## a sample of hbm.xml file ##

```

<?xml version="1.0"?>
<!DOCTYPE guzz-mapping PUBLIC "-//GUZZ//GUZZ MAPPING DTD//EN" "http://www.guzz.org/dtd/guzz-mapping.dtd">
<hibernate-mapping>
    <class name="org.guzz.service.core.impl.IncUpdateBusiness" table="tb_guzz_su" businessName="guzzSlowUpdate" dbGroup="logDB">
        <id name="id" type="bigint" column="gu_id">
        	<generator class="native">
        		<param name="sequence">seq_iub_id</param>
        	</generator>
        </id>
        <property name="dbGroup" type="string" column="gu_db_group" />
        <property name="tableName" type="string" column="gu_tab_name" />
        <property name="columnToUpdate" type="string" column="gu_inc_col" />
        <property name="pkColunName" type="string" column="gu_tab_pk_col" />
        <property name="pkValue" type="string" column="gu_tab_pk_val" />
        <property name="countToInc" type="int" column="gu_inc_count" />
    </class>
</hibernate-mapping>

```


## mapping as ibatis ##

> This is defined in guzz.xml. Sample:

```
	<sqlMap dbgroup="default">
	
		<update id="updateUserFavCount" orm="userObjectMap">
			update @@user set @favCount = favCount + 1
		</update>
		
		<update id="updateUserFavCount2" orm="user">
			update @@user set @favCount = favCount + 1
		</update>
		
		<select id="selectUsers" orm="userObjectMap">
			select @id, @name, @vip, @favCount from @@user
		</select>
		
		<select id="listCommentsByName" orm="commentMap" >
			select * from @@commentMap where @userName = :userName
		</select>
				
		<orm id="userObjectMap" class="org.guzz.test.UserModel">
			<result property="id" column="pk"/>
		    <result property="name" column="userName"/>
		    <result property="favCount" column="FAV_COUNT"/>
		    <result property="vip" column="VIP_USER"/>
		</orm>
		
		<orm id="commentMap" dbgroup="commentDB" class="org.guzz.test.Comment" table="TB_COMMENT" shadow="org.guzz.test.CommentShadowView">
			<result property="id" column="id" type="int"/>
		    <result property="userId" column="userId" type="int"/>
		    <result property="userName" column="userName" type="string" />
		    <result property="createdTime" column="createdTime" type="datetime" />
		</orm>
	</sqlMap>

```