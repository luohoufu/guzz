## Guzz JPA Annotation ##

> The Annotation of guzz is a combination of JPA specifications and some extended definitions to support guzz's special features like hibernate did.

> Guzz's annotation is only designed and should be used to replace the "hbm.xml", so it is a limited sub-collection of the JPA specifications without supports between classes(one-one, one-many, many-to, many-many) and JPA persist api.

> The extended annotations are similar to hibernate's, including the names and usages, except for the package name.

> The JPA specifications part is a sub-collection of JPA 1.0, without support between classes(one-one, one-many, many-to, many-many) and JPA persist api.

## How to annotate a domain class? ##

> Most of the time, to annotate a domain class in guzz, you have to add two annotations for that class: @javax.persistence.Entity and @org.guzz.annotations.Entity.

> The first one is a force demand of JPA specifications, while the later one is used to define the "business name" of the class for later using in your code.

> -**A typical annotated domain class would be something like:**

```
package org.guzz.test;

import java.util.Date;

import javax.persistence.Column;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.TableGenerator;

import org.guzz.annotations.Table;


@javax.persistence.Entity
@org.guzz.annotations.Entity(businessName = "comment")
@Table(name="TB_COMMENT", shadow = CommentShadowView.class)
@TableGenerator(
		name = "commentGen",
		table="tb_id",
		catalog="somelog",
		schema="some_schema",
		pkColumnName="pk",
		pkColumnValue="2",
		valueColumnName="id_count",
		initialValue=100,
		allocationSize=20
		/*
		 * create table tb_id(pk int(11) primary key, id_count int(11) default 0)
		 * insert into tb_id(pk, id_count) values(2, 100)
		 */
)
public class Comment {

	@javax.persistence.Id
	@GeneratedValue(generator="commentGen", strategy=GenerationType.TABLE)
	private int id ;
	
	private int userId ;
	
	private String userName ;
	
	//@javax.persistence.Basic(fetch=FetchType.LAZY)
	@Column(name="DESCRIPTION")
	private String content ;
	
	private Date createdTime ;

	public int getId() {
		return id;
	}

	public void setId(int id) {
		this.id = id;
	}

	//other gets/sets....	
}
```

> -**Note that:** Every annotated class must declare a primary key through @javax.persistence.Id. Guzz use that to decide the mapped properties should be on field or on property based on where the primary key annotation is.

> Read more of JPA Annotations Guzz supported, and guzz's extended annotations in [Guzz JPA Annotation Reference](AppendJPAAnnotation.md).

## How to add annotated domain classes to a system? ##

> We already know that, to add a hbm.xml to a system, you should add declarations in guzz.xml like

```
<business name="user" dbgroup="default" class="org.guzz.test.User" interpret="" file="classpath:org/guzz/test/User.hbm.xml" />
<business name="book" class="org.guzz.test.Book" file="classpath:org/guzz/test/Book.hbm.xml" />
```

> In the same principle, to add a annotated class, you should add declarations in guzz.xml with a new tag: a-business

> "a-business" has three attributes, and each tag declares one domain class. The three attributes are:

| **Attribute Name** | **Required** | **Notes** |
|:-------------------|:-------------|:----------|
| class              | Required     | the full qualified class name of the annotated domain class |
| name               | Optional     | business name. Override the businessName annotated in the domain class. |
| dbgroup            | Optional     | database group to store this object. Override the dbGroup annotated in the domain class. |

**A typical sample:**
```
        <a-business name="user" dbgroup="default" class="org.guzz.test.User"/>
	<a-business name="book" class="org.guzz.test.Book" />
	<a-business name="userInfo" dbgroup="oracle" class="org.guzz.test.UserInfo" />
	<a-business name="userInfo2" dbgroup="default" class="org.guzz.test.UserInfoH2" />
	<a-business name="comment" dbgroup="default" class="org.guzz.test.Comment"/>
	<a-business name="cargo" class="org.guzz.test.shop.Cargo" />
	<a-business name="sp" class="org.guzz.test.shop.SpecialProperty" />
```

> The source code of the annotated classes can be found in guzz's test source code.


## Samples of Annotating Primary Generators ##

**native(auto-increment in mysql and H2):**
```
@javax.persistence.Entity
@org.guzz.annotations.Entity(businessName="user")
@Table(name="tb_user")
public class User implements Serializable {
	
	private int id ;
	...
	
	@javax.persistence.Id
	@GeneratedValue(strategy=GenerationType.AUTO)
	public int getId() {
		return id;
	}
	...
```

**assigned:**
```
@javax.persistence.Entity
@Table(name="tb_user")
public class User implements Serializable {
	
	private int id ;
	...
	
	@javax.persistence.Id
	@GenericGenerator(name = "assignedGen", strategy = "assigned")
	@GeneratedValue(generator = "assignedGen")
	public int getId() {
		return id;
	}
	...
```


**sequence:**
```
@javax.persistence.Entity
@Table(name="tb_user")
public class User implements Serializable {
	
	private int id ;
	...
	
	@javax.persistence.Id
	@Column(name="pk")
	@GeneratedValue(generator="userIdGen")
	@GenericGenerator(name="userIdGen", strategy="sequence", parameters={@Parameter(name="sequence", value="seq_user_id")}
	)
	public id getId() {
		return id;
	}
	...
```


**random:**
```
@javax.persistence.Entity
@Table(name="tb_invitation_key")
public class InvitationKey implements Serializable {
	
	private String id ;
	...
	
	@javax.persistence.Id
	@GeneratedValue(generator="randomGen")
	@GenericGenerator(name = "randomGen", strategy = "random", parameters={@Parameter(name = "length", value = "32")})
	public String getId() {
		return id;
	}
	...
```

**uuid:**
```
@javax.persistence.Entity
@Table(name="tb_invitation_key")
public class InvitationKey implements Serializable {
	
	private String id ;
	...
	
	@javax.persistence.Id
	@GeneratedValue(generator="uuidGen")
	@GenericGenerator(name = "uuidGen", strategy = "uuid")
	public String getId() {
		return id;
	}
	...
```

Other Identify Generators and their parameters: [TutorialHbmXml](TutorialHbmXml.md)

Feel free to supply us more samples in the comment below.