---
layout: post
title: GC原理
date: 2017-07-20 
EntityManager使用方法
---

Session bean or MD bean对Entity bean的操作（包括所有的query, insert, update, delete操作）都是通过EntityManager实例来完成的。EntityManager是由EJB 容器自动地管理和配置的，不需要用户自己创建。

那么Session bean or MD bean如何获得EntityManager实例呢？？
非常简单，就是通过下列代码进行依赖注入：
Public class sessionbean1{
@PersistenceContext
EntityManager em;
。。。
}

注意：如果persistence.xml文件中配置了多个<persistence-unit>。那么在注入EntityManager对 象时必须指定持久化名称，通过@PersistenceContext注释的unitName属性进行指定，例：

@PersistenceContext(unitName="foshanshop")
EntityManager em;

如果只有一个<persistence-unit>，不需要明确指定。

请注意：Entity Bean被EntityManager管理时，EntityManager会跟踪他的状态改变，在任何决定更新实体Bean的时候便会把发生改变的值同步 到数据库中（跟hibernate一样）。但是如果entity Bean从EntityManager分离后，他是不受管理的，EntityManager无法跟踪他的任何状态改变。

EntityManager一些常用的API（包含query, insert, update, delete操作）
### 一.get entity —— find() or getReference() ###

Person person = em.find(Person.class,1);

当在数据库中没有找到记录时，getReference()和find()是有区别的，find()方法会返回null，而getReference() 方法会抛出javax.persistence.EntityNotFoundException例外，另外getReference()方法不保证 entity Bean已被初始化。如果传递进getReference()或find()方法的参数不是实体Bean，都会引发 IllegalArgumentException例外


### 二.insert —— persist() ###

Person person = new Person();
person.setName(name);
//把数据保存进数据库中
em.persist(person);

如果传递进persist()方法的参数不是实体Bean，会引发IllegalArgumentException


### 三.update —— 分2种情况 ###

情况1：当实体正在被容器管理时，你可以调用实体的set方法对数据进行修改，在容器决定flush时（这个由container自行判断），更新的数据 才会同步到数据库，而不是在调用了set方法对数据进行修改后马上同步到数据库。如果你希望修改后的数据马上同步到数据库，你可以调用 EntityManager.flush()方法。
public void updatePerson() {
try {
Person person = em.find(Person.class, 1);
person.setName("lihuoming"); //方法执行完后即可更新数据
} catch (Exception e) {
e.printStackTrace();
}
}

 情况2：在实体Bean已经脱离了EntityManager的管理时，你调用实体的set方法对数据进行修改是无法同步更改到数据库的。你必须调用 EntityManager.merge()方法。调用之后，在容器决定flush时（这个由container自行判断），更新的数据才会同步到数据 库。如果你希望修改后的数据马上同步到数据库，你可以调用EntityManager.flush()方法。
   
public boolean updatePerson(Person person) {
try {
em.merge(person);
} catch (Exception e) {
e.printStackTrace();
return false;
}
return true;
}

下面的代码会调用上面的方法。因为下面的第二行代码把实体Bean 返回到了客户端，这时的实体Bean已经脱离了容器的管理，在客户端对实体Bean进行修改，最后把他返回给EJB 容器进行更新操作：

PersonDAO persondao = (PersonDAO) ctx.lookup("PersonDAOBean/remote");
Person person = persondao.getPersonByID(1); //此时的person 已经脱离容器的管理
person.setName("张小艳");
persondao.updatePerson(person);

执行em.merge(person)方法时，容器的工作规则：
1>     如果此时容器中已经存在一个受容器管理的具有相同ID的person实例，容器将会把参数person的内容拷贝进这个受管理的实例，merge()方法 返回受管理的实例，但参数person仍然是分离的不受管理的。容器在决定Flush时把实例同步到数据库中。

2>容器中不存在具有相同ID的person实例。容器根据传进的person参数Copy出一个受容器管理的person实例，同时 merge()方法会返回出这个受管理的实例，但参数person仍然是分离的不受管理的。容器在决定Flush时把实例同步到数据库中。

如果传递进merge ()方法的参数不是实体Bean，会引发一个IllegalArgumentException。

### 四.Delete —— Remove() ###

Person person = em.find(Person.class, 2);
//如果级联关系cascade=CascadeType.ALL，在删除person 时候，也会把级联对象删除。
//把cascade属性设为cascade=CascadeType.REMOVE 有同样的效果。
em.remove (person);

如果传递进remove ()方法的参数不是实体Bean，会引发一个IllegalArgumentException

### 五.HPQL query —— createQuery() ###


除了使用find()或getReference()方法来获得Entity Bean之外，你还可以通过JPQL得到实体Bean。

要执行JPQL语句，你必须通过EntityManager的createQuery()或createNamedQuery()方法创建一个Query 对象

Query query = em.createQuery("select p from Person p where p. name=’黎明’");
List result = query.getResultList();
Iterator iterator = result.iterator();
while( iterator.hasNext() ){
//处理Person
}
…
// 执行更新语句
Query query = em.createQuery("update Person as p set p.name =?1 where p. personid=?2");
query.setParameter(1, “黎明”);
query.setParameter(2, new Integer(1) );
int result = query.executeUpdate(); //影响的记录数
…
// 执行更新语句
Query query = em.createQuery("delete from Person");
int result = query.executeUpdate(); //影响的记录数


#### 六.SQL query —— createNaiveQuery() ####
注意：该方法是针对SQL语句，而不是HPQL语句

//我们可以让EJB3 Persistence 运行环境将列值直接填充入一个Entity 的实例，
//并将实例作为结果返回.
Query query = em.createNativeQuery("select * from person", Person.class);
List result = query.getResultList();
if (result!=null){
Iterator iterator = result.iterator();
while( iterator.hasNext() ){
Person person= (Person)iterator.next();
… ..
}
}
…
// 直接通过SQL 执行更新语句
Query query = em.createNativeQuery("update person set age=age+2");
query.executeUpdate();

### 七.Refresh entity —— refresh() ###
　如果你怀疑当前被管理的实体已经不是数据库中最新的数据，你可以通过refresh()方法刷新实体，容器会把数据库中的新值重写进实体。这种情况一般发 生在你获取了实体之后，有人更新了数据库中的记录，这时你需要得到最新的数据。当然你再次调用find()或getReference()方法也可以得到 最新数据，但这种做法并不优雅。

Person person = em.find(Person.class, 2);
//如果此时person 对应的记录在数据库中已经发生了改变，
//可以通过refresh()方法得到最新数据。
em.refresh (person);



### 八.Check entity是否在EntityManager管理当中 —— contains() ###
contains()方法使用一个实体作为参数，如果这个实体对象当前正被持久化内容管理，返回值为true，否则为false。如果传递的参数不是实体 Bean，将会引发一个IllegalArgumentException.

Person person = em.find(Person.class, 2);
。。。
if (em.contains(person)){
//正在被持久化内容管理
}else{
//已经不受持久化内容管理
}

### 九.分离所有当前正在被管理的实体 —— clear() ###

在处理大量实体的时候，如果你不把已经处理过的实体从EntityManager中分离出来，将会消耗你大量的内存。调用EntityManager 的clear()方法后，所有正在被管理的实体将会从持久化内容中分离出来。有一点需要说明下，在事务没有提交前（事务默认在调用堆栈的最后提交，如：方 法的返回），如果调用clear()方法，之前对实体所作的任何改变将会掉失，所以建议你在调用clear()方法之前先调用flush()方法保存更 改。

### 十.将实体的改变立刻刷新到数据库中 —— flush() ###

当EntityManager对象在一个session bean 中使用时，它是和服务器的事务上下文绑定的。EntityManager在服务器的事务提交时提交并且同步它的内容。在一个session bean 中，服务器的事务默认地会在调用堆栈的最后提交（如：方法的返回）。

例子1：在方法返回时才提交事务
public void updatePerson(Person person) {
try {
Person person = em.find(Person.class, 2);
person.setName("lihuoming");
em.merge(person);
//后面还有众多修改操作
} catch (Exception e) {
e.printStackTrace();
}
//更新将会在这个方法的末尾被提交和刷新到数据库中
}

为了只在当事务提交时才将改变更新到数据库中，容器将所有数据库操作集中到一个批处理中，这样就减少了代价昂贵的与数据库的交互。当你调用 persist( ), merge( )或remove( )这些方法时，更新并不会立刻同步到数据库中，直到容器决定刷新到数据库中时才会执行，默认情况下，容器决定刷新是在“相关查询”执行前或事务提交时发 生，当然“相关查询”除find()和getreference()之外，这两个方法是不会引起容器触发刷新动作的，默认的刷新模式是可以改变的，具体请
考参下节。

如果你需要在事务提交之前将更新刷新到数据库中，你可以直接地调用EntityManager.flush()方法。这种情况下，你可以手工地来刷新数据 库以获得对数据库操作的最大控制。
public void updatePerson(Person person) {
try {
Person person = em.find(Person.class, 2);
person.setName("lihuoming");
em.merge(person);
em.flush();//手动将更新立刻刷新进数据库

//后面还有众多修改操作
} catch (Exception e) {
e.printStackTrace();
}
}

### 十一. 改变实体管理器的Flush模式 —— setFlushMode() ###
的Flush模式有2种类型：AUTO and COMMIT。AUTO为缺省模式。你可以改变他的值，如下：
entityManager.setFlushMode(FlushModeType.COMMIT);

FlushModeType.AUTO：刷新在查询语句执行前(除了find()和getreference()查询)或事务提交时才发生，使用场合：在 大量更新数据的过程中没有任何查询语句(除了find()和getreference()查询)的执行。

FlushModeType.COMMIT：刷新只有在事务提交时才发生，使用场合：在大量更新数据的过程中存在查询语句(除了find()和 getreference()查询)的执行。

其实上面两种模式最终反映的结果是：JDBC 驱动跟数据库交互的次数。JDBC 性能最大的增进是减少JDBC 驱动与数据库之间的网络通讯。FlushModeType.COMMIT模式使更新只在一次的网络交互中完成，而FlushModeType.AUTO 模式可能需要多次交互（触发了多少次Flush 就产生了多少次网络交互）
### 十二.  获取持久化实现者的引用 —— getDelegate() ###
用过getDelegate()方法，你可以获取EntityManager持久化实现者的引用，如Jboss EJB3的持久化产品采用Hibernate，可以通过getDelegate()方法获取对他的访问，如：
HibernateEntityManager manager = (HibernateEntityManager)em.getDelegate();

获得对Hibernate的引用后，可以直接面对Hibernate进行编码，不过这种方法并不可取，强烈建议不要使用。在Weblogic 中，你也可以通过此方法获取对Kodo 的访问。