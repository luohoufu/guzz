> TranSession is guzz's persist API divided into 2 sub interfaces, ReadonlyTranSession for read and WriteTranSession for write.

> A TranSession manages a transaction, all operations in a un-auto-commit TranSession acts in a single transaction cycle until you commit or rollback it.

## GuzzBaseDao: ##

> GuzzBaseDao is a handy base class for implementing DAOs of your system. It provides common useful methods, and is suggested to be used for all DAOs.

> -**WARNING: Without Spring Declaration transaction in scope, all write methods in GuzzBaseDao act in their own standalone transactions! They won't work together in one atom transaction.**

> If you had already configured Spring declaration transaction, all WRITE operations will be under the control of the standard Spring transaction; But all READ operations will still open their separate readonly slave-database connections to perform.

> Call GuzzBaseDao.getWriteTemplate() to get the current WriteTemplate for reading in the current Spring transaction or performing others WRITE operations not listed in GuzzBaseDao.

> WriteTemplate is similar to HibernateTemplate or JDBCTemplate.

## ReadonlyTranSession, API for read ##

> ReadonlyTranSession is and can be used only for querying data. Example:

```
TransactionManager tm = guzzContext.getTransactionManager() ;

ReadonlyTranSession session = getTransactionManager().openDelayReadTran() ;

try{
	SearchExpression se = SearchExpression.forClass(SystemLog.class) ;
	se.and(Terms.eq("categoryId", 18)) ;
	se.setOrderBy("importance desc, id asc") ;
	return session.list(se) ;
}finally{
	session.close() ;
}
```

> The example shows you how to fetch a ReadonlyTranSession.

> ReadonlyTranSession supports three query API: [SearchExpression](SearchExpression.md) the object-oriented query language, CompiledSQL for writing arbitrary sqls in code, and configured sqls looked up by ID in guzz.xml.

### SearchExpression: ###
> [SearchExpression](SearchExpression.md) is similar to criteria in hibernate, but less powerful.

> Query Expression is organized and combined by SearchTerm. [Read More](SearchExpression#SearchTerm.md).

### CompiledSQL： ###

> CompiledSQL is a complement to SearchExpression, as the later one cann't be so powerful enough.

> CompiledSQL is a compiled version of sql, including raw sql, parameters, orm, and so on.

> CompiledSQL is thread-safe, and better than normal sql for performance.

> Guzz asks every sql compiled to be a CompiledSQL to execute, and suggests you to cache it.

> The sql to be compiled should be a sql that can be executed directly in your database, except that it can own named parameters and property-column transformation. The rule of the transformation is the same of the sqls configured in guzz.xml(@@businessName used for table name, and @propertyName used for column name). For example:

```
TransactionManager tm = super.getTransactionManager() ;

String sql = "update @@" + SystemLog.class.getName() +  " set @importance = :level where @id = :id" ;
CompiledSQL cs = tm.getCompiledSQLBuilder().buildCompiledSQL(SystemLog.class, sql) ;

WriteTranSession session = tm.openRWTran(true) ;

try{
	session.executeUpdate(cs.bind("level", 3).bind("id", 5)) ;
}finally{
	session.close() ;
}
```

> This example calls CompiledSQLBuilder's buildCompiledSQL() to build a CompiledSQL. The first parameter is used to tell guzz which database group the sql should be executed in, the second is the sql statement.

> The WriteTranSession executes a update execution. If this is a (list) query sql, guzz will use the first parameter before to do ORM, the result returned would be List`<`SystemLog`>`.

> CompiledSQL must be binded with parameters or Loaders, and turned into a temporary object BindedCompiledSQL, before it can be handed to TranSession. The actual sql for one execution is represented as BindedCompiledSQL in fact.

> BindedCompiledSQL is **NOT** thread-safe, and designed to be used for only one time. The parameters and other tips set in a BindedCompiledSQL only affects its current execution.

> For example, sometimes we need to temporary alter a sql's returned value(changing the orm), then we can use BindedCompiledSQL's setRowDataLoader(RowDataLoader) to achieve this.

> guzz carries two RowDataLoaders in default, FirstColumnDataLoader for retrieving the first column of a ResultSet, and FormBeanRowDataLoader for mapping the ResultSet to any pojo class or a Map.

> Example:

```
TransactionManager tm = super.getTransactionManager() ;

String sql = "select count(*) as count, company from mycompany join xxx... where group by company having salary = :salary " ;

CompiledSQL cs = builder.buildCompiledSQL(Company.class, sql) ;

WriteTranSession session = tm.openRWTran(true) ;

try{
        //Map the ResultSet to HashMap, bind parameters, and execute the query.
	List<Map> result = session.list(cs.setRowDataLoader(FormBeanRowDataLoader.newInstanceForClass(HashMap.class)).bind("salary", 3000)) ;
}finally{
	session.close() ;
}
```

> In the example, we use FormBeanRowDataLoader to map the result to a java.util.HashMap, not the default Company.class.

> RowDataLoader is the interface to handle how to map a row of the java.sql.ResultSet in query to a java object instance. It is light-weight, and is recommended to use for any special OR mappings in your system.

### Query by Id configured in guzz.xml: ###

> Similar to ibatis. [Read more about how to write the xml](TutorialGuzzXml.md).

  1. Configure sqls and ORMs. The `<business>`s are default ORMs：

```
 	<sqlMap dbgroup="default">
		<select id="selectUser" orm="user">
			select * from @@user
				 where 
				 	@id = :id
		</select>
		
		<select id="selectUserByName" orm="user">
			select id, userName, vip, password from tb_user where userName = :userName
			
			<paramsMapping>
				<map paramName="userName" propName="userName" />
			</paramsMapping>
		</select>
	
		<update id="updateUserFavCount" orm="userObjectMap">
			update @@user set @favCount = favCount + 1
		</update>
	
		<select id="getCount" orm="user">
			select 30 as totalCount
		</select>
		
		<select id="selectUserByName2" orm="user" result-class="java.util.HashMap">
			select pk, userName, VIP_USER, MyPSW, FAV_COUNT from TB_USER where userName = :userName
		</select>
		
		<select id="selectUsers" orm="userObjectMap">
			select @id, @name, @vip, @favCount from @@user
		</select>
		
		<select id="listCommentsByName" orm="commentMap">
			select * from @@commentMap where @userName = :userName
			
			<paramsMapping>
				<map paramName="userName" propName="userName" />
			</paramsMapping>
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
	
	<sqlMap dbgroup="cargoDB">
		<select id="selectCrossSize" orm="cargo" result-class="org.guzz.orm.rdms.MyCrossStitch">
			select name, price, gridNum, size, brand from @@cargo where id > :id
		</select>
	</sqlMap> 
 
```

> 2. Query with id and parameters:

> example 1: listCommentsByName：
```

	HashMap params = new HashMap() ;
	params.put("userName", "lily") ;
		
	ReadonlyTranSession session = tm.openDelayReadTran() ;
	Guzz.setTableCondition(tableCondition) ;
	List<Comment> comments = session.list("listCommentsByName", params, 1, 20) ;

```

> example 2: getCount：
```

	ReadonlyTranSession session = tm.openDelayReadTran() ;
	int totalCount = (Integer) session.findCell00("getCount", null, "int") ;
	assertEquals(totalCount, 30) ;
	
```

> example 3: selectUserByName2：
```
	HashMap params = new HashMap() ;
	params.put("userName", "name 1") ;
	
	ReadonlyTranSession session = tm.openDelayReadTran() ;
	List users2 = session.list("selectUserByName2", params, 1, 10) ;
	
	assertNotNull(users2) ;
	assertEquals(users2.size(), 1) ;
	assertEquals(users2.get(0).getClass(), java.util.HashMap.class) ;
		
	//The Map's key is the property name in the User javabean
	java.util.Map u = (Map) users2.get(0) ;
	assertTrue(u.containsKey("favCount")) ;

```


### Query by wild-card sqls built in runtime: ###

> Build sqls in runtime and execute them.

```
 	ReadonlyTranSession session = tm.openDelayReadTran() ; 
	Guzz.setTableCondition("all") ;
	
	//list all cargoes priced above 100.00
	//concate a sql statement
	String sql="select c.* from (select @id, @name, @storeCount from tb_cargo_book where @price>=:param_price union all select @id, @name, @storeCount from tb_cargo_crossstitch where @price>=:param_price) as c ";
	
	//Compile the sql. The first parameter "Cargo.class" indicates the database group to run the sql and the default result-object mapping.
	CompiledSQL cs = tm.getCompiledSQLBuilder().buildCompiledSQL(Cargo.class, sql) ; 
	
	//Bind the sql parameter "param_price" with a strong type.
	cs.addParamPropMapping("param_price", "price") ;
	
	//List the first 1000 cargoes.
	List<Cargo> cargoes = session.list(cs.bind("param_price", 100.00), 1, 1000) ;	
	
	//Temporary change the result to HashMap -- set bsql
	List<HashMap<String, Object>> cargoes2 = session.list(cs.bind("param_price", 100.00).setResultClass(java.util.HashMap.class), 1, 1000) ;
	
	//Change the default result to HashMap -- set CompiledSQL
	cs.setResultClass(java.util.HashMap.class) ;	
	List<HashMap<String, Object>> cargoes3 = session.list(cs.bind("param_price", 100.00), 1, 1000) ;
	List<HashMap<String, Object>> cargoes4 = session.list(cs.bind("param_price", 100.00), 2, 20) ;
	
	session.close() ;
 
```


## WriteTranSession, API for write ##

> WriteTranSession is the API interface for performing updating operations the databases. In a master-slave database architecture, WriteTranSession operates the master database only.

```
TransactionManager tm = super.getTransactionManager() ;
WriteTranSession session = tm.openRWTran(true) ;
```

> To create a WriteTranSession, we need to pass a boolean parameter to indicate how to control the transaction. true: auto-commit; false:commit manually.

> If you pass false to create a un-auto-commit transaction, remember to call session.commit() to commit the transaction, or call session.rollback() to roll back it.

> WriteTranSession only allows querying a record by primary key. If you need to query data from the master database with more options, open a ReadonlyTranSession with no delay to do it.

## Batch: ##

> There are two ways to do batch updating in guzz, by object or by sql statement. In both ways, you need to open a WriteTranSession first.

> -**By object:**
```
		WriteTranSession session = tm.openRWTran(false) ;
		ObjectBatcher batcher = session.createObjectBatcher() ;
				
		for(int loop = 0 ; loop < 1000 ; loop++){
			User user = new User() ;
			user.setUserName("hello un " + i) ;
				
			batcher.insert(user) ;
		}

                batcher.executeUpdate() ;
		session.commit() ;

		session.close() ;
```

> -**By sql statement:**
```
CompiledSQL cs = tm.getCompiledSQLBuilder().buildCompiledSQL(User.class, "delete from @@" + User.class.getName() + " where @id = :id") ;

WriteTranSession session = tm.openRWTran(false) ;
SQLBatcher batcher = session.createCompiledSQLBatcher(cs) ;

for(int loop = 0 ; loop < 100 ; loop++){
	batcher.addNewBatchParams("id", user.getId()) ;
}
	
batcher.executeUpdate() ;
session.commit() ;

```

> After opening a WriteTranSession, create a Batcher, add objects or parameters to the batcher as its API required, then executeUpdate.

> -**NOTE:** A batcher can be used only for a single business domain's single operation in a single physical table. If you need to operate multiple tables at once, create multiple batcher for each table.

## Atom of Transaction: ##

> A WriteTranSession holds a transaction. If you need to handle a long transaction, call tm.openRWTran(false) to open a manual controlled (un-auto-commit) WriteTranSession.

> In a un-auto-commit WriteTranSession, any methods executed would be under only one transaction, no matter which database it actually operated. Even if you create a JDBCTemplate or a Batcher from it, the methods in the JDBCTemplate and Batcher would under the same transaction.

> When you call commit() or rollback(), the transaction would be committed or rolled-back fully.

> Guzz will take care of the distributed transaction among your configured multiple databases.
