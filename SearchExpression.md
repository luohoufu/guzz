## SearchExpression： ##

> SearchExpress is used to combine query conditions, and set parameters/tips to build a complicated query. SearchExpression can be instanced by its own static methods.

### Create SearchExpression by domain class: ###

```
SearchExpression se = SearchExpression.forClass(Article.class) ;

SearchExpression se = SearchExpression.forClass(Article.class, pageNo, pageSize) ;

```

> This example builds a SearchExpression to query "pageSize" Articles in page "pageNo". The first line doesn't pass any parameters, indicating to read the first 20 records in default.

> -**Warning:** The default pageNo(1) and pageSize(20) is stored in SearchExpression's static variables. This is a temporary solution. Don't modify the value, as in the future, we would move it to the configuration file, and drop the two variables.

> If you need to read all data, not just the first 20 records, build the SearchExpression this way:

```
SearchExpression se = SearchExpression.forLoadAll(Article.class) ;
```

### Create SearchExpression by businessName: ###

> This is similar to the above. You can pass a domain class's business name or its full qualified class name instead of the class.

```
SearchExpression se = SearchExpression.forBusiness("blogArticle", pageNo, pageSize) ;
```

> After building a SearchExpression, we need to set tips/parameters and add conditions for query. The conditions are built from SearchTerm.

### SearchExpression API： ###

> -**setOrderBy：**
> set the order by. Fields in the orderBy should be java property names. Separate multiple fields by comma, for example:id asc, createdTime desc

> -**setCountSelectPhrase:**
> Set count function for executing count operations in TranSession API. The default function is "count(`*`)", and can be set to be any database function returning a number, such as count(id), sum(score), and so on.

> -**setSkipCount：**
> Set the first count of records to skip when performing a list or page query. For example, you set skip count to 5, and query 20 records of page 1, the records actually returned would be from position 6 to 25 in database. The default value is zero, never skip.

> -**setPageFlipClass：**
> Set your PageFlip class to store the result in a page query. This method only affects the current SearchExpression. The flip class set must derived from org.guzz.dao.PageFlip.

> -**setCondition：**
> Set the query conditions. This should be the sql fragment after "where"(exclude the "where" keyword). You can set the condition if handy methods cann't help.

> -**setLoadRecords:**
> Set whether or not to load records in a page query. Set this to false, if you would load records from somewhere else, such as cache.

> -**setComputeRecordNumber：**
> Set whether or not to count records number in a page query. Set this to false, if you would read total count from somewhere else, such as cache.

### Build Query Conditions from SearchTerm: ###

> SearchTerm can be instanced directly, or by static methods in org.guzz.orm.se.Terms. The later one is recommended.

> SearchTerm can be considered as a concatenation of sql fragments, without any object oriented stuffs. Treat it as append() method of StringBuffer.

> Some SearchTerm accept property name as parameter(s). Remember that the property name passed to SearchTerm must be the java domain class's field name; it would never be the column name in the database.

> SearchTerm is stored under org.guzz.orm.se package.

> -**Primary SearchTerms：**

| **SearchTerm**  | **Function**  | **Construct Method** | **Construct Parameters** |
|:----------------|:--------------|:---------------------|:-------------------------|
| AndTerm         | Concatenate two and conditions. | public AndTerm(SearchTerm leftTerm, SearchTerm rightTerm) | two conditions to do and. |
| CompareTerm     | Compare a property to a value. | public CompareTerm(String propName, String operator ,Object propValue) | construct to: propName + operator + propValue. The available operators are defined as static variables in CompareTerm. |
| ConcatTerm      | Concatenate   | public ConcatTerm(SearchTerm leftTerm, SearchTerm rightTerm) | concatenate the two conditions together, add a whitespace between them. |
| InTerm          | "in" keyword of sql |public InTerm(String propName, List/int[.md](.md) values) | propName in values       |
| OrderByTerm     | order by statement | public OrderByTerm(String orderBy) | Separate multiple orderBys by comma.eg:id asc, title desc  |
| OrTerm          | "or" keyword  |public OrTerm (SearchTerm leftTerm, SearchTerm rightTerm) | leftTerm or rightTerm    |
| PropsSelectTerm | Specify the properties to load, rather than load all. | Use SearchExpression.setSelectedProps(String selectedProps) |eg: "id, name"            |
| StringCompareTerm | compare two strings. | public StringCompareTerm(String propName, String operator, String propValue, boolean ignoreCase) | support "like" keyword, and ignore case(add low(xxx) function in query). |
| WhereTerm       | add where statement. | public WhereTerm(SearchTerm condition) | set where statement. private API. |

> SearchTerm is not recommended to construct directly. Use the static methods in Terms and SearchExpression is the first choice.

> For example, to add a condition "userId=7 and status='deleted'", you can write:
```
 se.and(Terms.eq("userId", 7)) ;
 se.and(Terms.eq("status", "deleted")) ;
```

## How to write very complicated queries? ##

> For complicated queries, guzz suggests that you store them in guzz.xml, and perform queries by id as ibatis did. We believe that complicated sqls are better to be separated for dbas later.

> For weird but stable sqls, you can build them into CompiledSQLs, and use CompiledSQL's API.

**sqls in guzz.xml：**

> For guzz.xml in [TutorialGuzzXml](TutorialGuzzXml.md), the following code demonstrates how to execute the query named listCommentsByName.

```
	HashMap params = new HashMap() ;
	params.put("userName", "lily") ;
	
	List<Comment> comments = session.list("listCommentsByName", params, 1, 100) ;
```

**build CompiledSQLs in the java code：**

> First, fetch TransactionManager, then get CompiledSQLBuilder from it, and then, call buildCompiledSQL.

```
 	private CompiledSQL deleteFriendActionsSQL ;
 
 	//build and compile sqls on startup, and cache it.
 	public void startup() {
		//delete a friend's friendActions when un-followed.
		this.deleteFriendActionsSQL = tm.getCompiledSQLBuilder().buildCompiledSQL(FriendsAction.class, 
				"delete from @@" + FriendsAction.class.getName() + " where @copiedFrom = :copiedFrom and @owner = :owner"
				);
				
		//bind the paramName with its java property, to tell guzz the data type of that parameter.
		this.deleteFriendActionsSQL.addParamPropMapping("copiedFrom", "copiedFrom") ;
		this.deleteFriendActionsSQL.addParamPropMapping("owner", "owner") ;
	}
	
	.....
	
	//bind parameters, and execute the sql.
	public void runDeleteXXX(String owner， String starName){
		BindedCompiledSQL bsql = this.deleteFriendActionsSQL.bind("copiedFrom", starName).bind("owner", owner) ;
		//use owner to shadow the table.
		bsql.setTableCondition(owner) ;
				
		WriteTranSession write = tm.openRWTran(true) ;
		
		try{
			//perform deleting.
			int rowsDeleted = write.executeUpdate(bsql) ;
			
			//dec the count by the count service.
			slowUpdateService.updateCount(UserActionsCount.class, owner, "friendsActionCount", owner, -rowsDeleted) ;
		}finally{
			write.close() ;
		}
 	}
 
```

