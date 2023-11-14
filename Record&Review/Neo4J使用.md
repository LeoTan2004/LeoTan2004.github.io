# Neo4J使用

## 介绍

Neo4J是一个用Java写的图形数据库，有社区版和专业版，社区版开源[neo4j/neo4j: Graphs for Everyone (github.com)](https://github.com/neo4j/neo4j)

> 这里的图形数据库中的图形并不指的是图片，而是数学中的一些图论只是，一般用来表示关联。常用来做知识图谱和相关性分析，也可以用来做推荐算法

## 安装

下载地址：[Neo4j Desktop Download | Free Graph Database Download](https://neo4j.com/download/)

安装的话就正常安装

> 这里需要讲一下window和desktop的区别。他们都是运行在window上的，但是window版本只包括一个服务，当然也会提供web的管理界面，而desktop更像是一个管理的继承工具，有点像JetBrain的ToolBox工具一样，能过提供一些方便的管理工具，但是不建议在生产环境中使用。

## 启动

在安装目录下的的bin目录中有个neo4j.bat，这个就是启动脚本，但是直接点击是无法启动的，他是需要参数的。一般使用`\neo4j.bat console`即可启动。然后他会给出一个本地的地址，一般是`http://localhost:7474/`，进入地址之后登录，初始默认账号是`neo4j`，密码也是`neo4j`

## 基础介绍

### 模型

**Neo4J**里面主要有3大模型：结构、属性、集合。

1. 属性（能作为参数输入和返回）

   | 类型                   | 说明                                                         |
   | ---------------------- | ------------------------------------------------------------ |
   | Number                 | 数字类型，既可以表示整数，也可表示小数                       |
   | String                 | 字符串类型                                                   |
   | Boolean                | 布尔类型                                                     |
   | Temporal types         | 时间类型，可以包括：*Date*, *Time*, *LocalTime*, *DateTime*, *LocalDateTime* and *Duration* |
   | The spatial type Point | 地理坐标数据                                                 |

2. 结构（只能被返回）

   - 节点：包括Id，标签（Label），和属性
   - 关系：包含Id，类型（Type），属性，始末节点的Id
   - 路径：节点和关系的序列，用于描述图形中的一个通路或者路径

3. 集合（不能被当做属性存储，能作为参数输入和返回）

### 命名规范

- 能包含非英文字符（中文，希腊字符等）
- 不能数字开头
- 不能超出65534个字符
- 区分大小写
- 节点标签建议首字母大写（**VehicleOwner**）
- 关系类型建议全大写（**OWNS_VEHICLE**）

### 常规表达式写法

> 可参考[Expressions - Cypher Manual (neo4j.com)](https://neo4j.com/docs/cypher-manual/3.5/syntax/expressions/)

总体下来感觉和MYSQL差不多，值得注意的是它支持动态属性，例如`obj['prop']`，等效于`obj.prop`。同时参数需要加上`$`符号

### 语法

> w3school上面有教程[Neo4j - CQL简介_w3cschool](https://www.w3cschool.cn/neo4j/neo4j_cql_introduction.html)
>
> 但是这个里面的建立索引的语句已经过时，其他还没验证。也可以尝试看官方文档
>
> [Introduction - Cypher Manual (neo4j.com)](https://neo4j.com/docs/cypher-manual/current/introduction/)

## SpringData中的Neo4J

### Maven依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-neo4j</artifactId>
</dependency>
```

也可以使用stater快速生成。在**SpringData**中含有**Neo4J**的扩展选项。

### 配置

```yaml
spring:
	neo4j:
		uri: bolt://localhost:7687
		authentication:
			username: neo4j
			password: pwd
```

也可以使用配置类配置

```java
@Bean
Configuration cypherDslConfiguration() {
	return Configuration.newConfig()
                .withDialect(Dialect.NEO4J_5).build();
}
```

### 节点映射

> 可参考[Metadata-based Mapping :: Spring Data Neo4j](https://docs.spring.io/spring-data/neo4j/reference/object-mapping/metadata-based-mapping.html)

```java
import java.util.ArrayList;
import java.util.List;

import org.springframework.data.neo4j.core.schema.Id;
import org.springframework.data.neo4j.core.schema.Node;
import org.springframework.data.neo4j.core.schema.Property;
import org.springframework.data.neo4j.core.schema.Relationship;
import org.springframework.data.neo4j.core.schema.Relationship.Direction;

@Node("Movie") // 标记实体是Neo4J的实体映射对象
public class MovieEntity {

	@Id // 必须要有，可以配置生成唯一id,见@GeneratedValue，一般不允许改变
	private final String title;

	@Property("tagline") // 对象的属性标签，可以省略
	private final String description;

	@Relationship(type = "ACTED_IN", direction = Direction.INCOMING) // 关系声明，type是对于关系自定义的标签，注意命名规范
	private List<Roles> actorsAndRoles;

	@Relationship(type = "DIRECTED", direction = Direction.INCOMING)
	private List<PersonEntity> directors = new ArrayList<>();

	public MovieEntity(String title, String description) { 
		this.title = title;
		this.description = description;
	}

	// Getters omitted for brevity
}
```

### 关系映射

```java
@RelationshipProperties
public class Roles {

	@RelationshipId
	private Long id;

	private final List<String> roles;

	@TargetNode
	private final PersonEntity person;

	public Roles(PersonEntity person, List<String> roles) {
		this.person = person;
		this.roles = roles;
	}

	public List<String> getRoles() {
		return roles;
	}
}
```

```java
@Relationship(type = "ACTED_IN", direction = Direction.INCOMING)
private List<Roles> actorsAndRoles;
```

> 用例：
>
> [Metadata-based Mapping :: Spring Data Neo4j](https://docs.spring.io/spring-data/neo4j/reference/object-mapping/metadata-based-mapping.html#mapping.annotations.example)

### 映射类（接口）

映射接口一般是继承`Repository<T,ID>`接口的，然后通过Spring代理注入使用

SpringData提供了一些已经实现好的接口，常用的比如Crud接口

```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {

  <S extends T> S save(S entity);

  Optional<T> findById(ID primaryKey);

  Iterable<T> findAll();

  long count();

  void delete(T entity);

  boolean existsById(ID primaryKey);

  // … more functionality omitted.
}
```

当然对于查询结果也可以分页使用

```java
public interface PagingAndSortingRepository<T, ID>  {

  Iterable<T> findAll(Sort sort);

  Page<T> findAll(Pageable pageable);
}
```

```java
PagingAndSortingRepository<User, Long> repository = // … get access to a bean
Page<User> users = repository.findAll(PageRequest.of(1, 20));
```

我们也可以自己定义一些接口并且自己实现

```java
interface UserRepository extends CrudRepository<User, Long> {
    @Query("MATCH (n:User {lastname: $lastname}) RETURN n")
  long countByLastname(@Param("lastname")String lastname);
}
```

如果想要Spring代理注入的话，需要在配置类上加上注解`@EnableNeo4JRepositories`，然后就可以在下面的类中注入使用了

```java
class SomeClient {

  private final UserRepository repository;

  SomeClient(UserRepository repository) {
    this.repository = repository;
  }

  void doSomething() {
    List<User> users = repository.findByLastname("Matthews");
  }
}
```

也可以通过java代码的方式自定义存放过程

Example1:

```java
interface HumanRepository {
  void someHumanMethod(User user);
}

class HumanRepositoryImpl implements HumanRepository {

  public void someHumanMethod(User user) {
    // Your custom implementation
  }
}

interface ContactRepository {

  void someContactMethod(User user);

  User anotherContactMethod(User user);
}

class ContactRepositoryImpl implements ContactRepository {

  public void someContactMethod(User user) {
    // Your custom implementation
  }

  public User anotherContactMethod(User user) {
    // Your custom implementation
  }
}
```

Examp

```java
interface CustomizedSave<T> {
  <S extends T> S save(S entity);
}

class CustomizedSaveImpl<T> implements CustomizedSave<T> {

  public <S extends T> S save(S entity) {
    // Your custom implementation
  }
}
```

