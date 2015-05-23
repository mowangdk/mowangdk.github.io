---
layout: post
title: hibernate
category: project
description: hibernate 学习笔记
---
## ==&equals
今天开始记录学习hibernate的笔记，首先补充一下以前的东西，==&equals  主要是因为hibernate的联合主键的问题，里面需要复写联合主键类的equals和hashcode方法
于是想到了这个首先==是比较内存中对象的东西，主要是看两个对象是否是同一个对象的引用，而equals是object类的一个方法，在其内部也是复写了这个方法，但是由于object
类是所有类的父类，所有继承他的类都存在equals方法，其中有的对象对这个类的方法进行了重写，加入了对内容的比较，比如说是String类
     public boolean equals(Object anObject) {
      if (this == anObject) {
              return true; }
      if (anObject instanceof String) {
                String anotherString = (String)anObject;
                int n = count;
                if (n == anotherString.count) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = offset;
                int j = anotherString.offset;
                while (n-- != 0) {
                if (v1[i++] != v2[j++])
                return false;
                }  return true;
                } } return false;
                }
他除了对两个对象进行了基本的比较外，在确认为true的时候，进行了进一步的比较。接下来就是hashcode与equals的比较。


##equals和hashcode的解析

覆盖equals时看起来简单，但是如果覆盖不当就会导致错误，并且后果相当严重，effective java中写到，避免这个错误最好的方法就是不复写这个方法
类的每个实例只于自身相等，并且满足一下条件
 类的每个实例本质上都是唯一的。对于代表活动实体而不是值的类来说却是如此，例如Thread。Object提供的equals实现对于这些类来说正是正确的行为。
 不关心类是否提供了“逻辑相等”的测试功能。假如Random覆盖了equals，以检查两个Random实例是否产生相同的随机数序列，但是设计者并不认为客户需要或者期望这样的功能。在这样的情况下，从Object继承得到的equals实现已经足够了。
 超类已经覆盖了equals，从超类继承过来的行为对于子类也是合适的。大多数的Set实现都从AbstractSet继承equals实现，List实现从AbstractList继承equals实现，Map实现从AbstractMap继承equals实现。
 类是私有的或者是包级私有的，可以确定它的equals方法永远不会被调用。在这种情况下，无疑是应该覆盖equals方法的，以防止它被意外调用：
在覆盖equals方法的时候，必须遵守以下约定
 自反性。对于任何非null的引用值x，x.equals(x)必须返回true。
 对称性。对于任何非null的引用值x和y，当且仅当y.equals(x)返回true时，x.equals(y)必须返回true。
 传递性。对于任何非null的引用值x、y和z，如果x.equals(y)返回true，并且y.equals(z)也返回true，那么x.equals(z)也必须返回true。
 一致性。对于任何非null的引用值x和y，只要equals的比较操作在对象中所用的信息没有被修改，多次调用该x.equals(y)就会一直地返回true，或者一致地返回false。
 对于任何非null的引用值x，x.equals(null)必须返回false。
根据以上要求，得出诀窍
 使用==符号检查“参数是否为这个对象的引用”。如果是，则返回true。这只不过是一种性能优化，如果比较操作有可能很昂贵，就值得这么做。
 使用instanceof操作符检查“参数是否为正确的类型”。如果不是，则返回false。一般来说，所谓“正确的类型”是指equals方法所在的那个类。
 把参数转换成正确的类型。因为转换之前进行过instanceof测试，所以确保会成功。
 对于该类中的每个“关键”域，检查参数中的域是否与该对象中对应的域相匹配。如果这些测试全部成功，则返回true;否则返回false。
 当编写完成了equals方法之后，检查“对称性”、“传递性”、“一致性”。
 注意：
 <ul>
 <li>覆盖equals时总要覆盖hashCode 《Effective Java》作者说的</li>
 <li>不要企图让equals方法过于只能。</li>
 <li>不要将equals声明中的Object对象替换为其他的类型（因为这样我们并没有覆盖Object中的equals方法哦）</li>
 <li>每个覆盖equals 的方法都要覆盖hashcode方法，如果不这样做，会导致该类无法结合所有积蓄散列的集合一起正常运作，包括hashmap hashset hashtable</li>
 <li>在应用程序的执行期间，只要对象的equals方法的比较操作所用到的信息没有被修改，那么对这同一个对象调用多次，hashCode方法都必须始终如一地返回同一个整数。在同一个应用程序的多次执行过程中，每次执行所返回的整数可以不一致。</li>
 <li>如果两个对象根据equals()方法比较是相等的，那么调用这两个对象中任意一个对象的hashCode方法都必须产生同样的整数结果。</li>
 </ul>
 •如果两个对象根据equals()方法比较是不相等的，那么调用这两个对象中任意一个对象的hashCode方法，则不一定要产生相同的整数结果。但是程序员应该知道，给不相等的对象产生截然不同的整数结果，有可能提高散列表的性能。
hashcode方法一般用户不会去调用，比如在hashmap中，由于key是不可以重复的，他在判断key是不是重复的时候就判断了hashcode这个方法，而且也用到了equals方法。这里不可以重复是说equals和hashcode只要有一个不等就可以了！所以简单来讲，hashcode相当于是一个对象的编码，
就好像文件中的md5，他和equals不同就在于他返回的是int型搜索的，比较起来不直观。我们一般在覆盖equals的同时也要覆盖hashcode，
让他们的逻辑一致。举个例子，还是刚刚的例子，如果姓名和性别相等就算2个对象相等的话，那么hashcode的方法也要返回姓名的hashcode值加上性别的hashcode值，这样从逻辑上，他们就一致了。


##使用annotation来将id生成策略变成uuid，

首先在对应的实体类的表上面@GenericGenerator(name="****" , strategy="uuid")，然后在getId方法上写上GeneratedValue（generator="******"）,指定id的自定义生成策略


##ORmapping映射一对一之双向关联，
<p>使用的是annotation，但是始终有一个地方不太理解，就是mappedBy这个注解，因为双向关联，所以两个表中如果不设置mappedby的话就是每个表都有一个外键，这样的确会照成数据冗余
而且插入的时候可能还会报错，但是如果加上这个注解，就会变成一个外键，我生成了建表语句，观察了一下，这个建的表不是跟单项关联一样吗？？？于是在网上找了找答案：</p>
        *加了mappedBy的话，只在Person里面加了外键。我们在IdCard类里有一个Person属性，当get或load一个IdCard的时候，hibernate看到了你在这个OneToOne里面加了一个mappedBy，
        *所以会去Person类对应的表里去找一个外键与你要get的IdCard的主键相同的记录，放到IdCard的person属性中。这样也就能根据IdCard来找到Peorson了，也就实现了所谓的双向关联。
还是没太看懂......可能是数据库学的不太好吧。姑且先记上，回头有时间再说，
补充：之前的视频没有看完,所以留下了这个疑问,其实视频里也没讲太清楚,即单向关联和双向关联的映射在数据库中建的表是一致的,两者的区别即为建的类不相同，两个实体类互相关联，能够互相找到
对方。但是还是有疑问,希望以后做实验的时候在说吧
##关于hibernate4.3.8 的JoinColumn,MappedBy,JoinTable
今天真的不太顺利，出现了很多bug,都是因为版本的问题，首先第一个疑问，发生在一对多/多对一双向关联的时候,如果不加上JoinColumn的时候就会出现建立一个新表的状况，因为一对多是多对多的
特殊情况，这个我是理解的,但是为什么加了以恶搞JoinColumn就会不建表了？这个annotation给我的印象就是改变新建的表名而已，没有其他的了。好吧，这是第一个，还有就是MappedBy,昨天一对一
的时候，的确如此，但是今天一对多的时候就给我报错，上网查过之后说是不能够跟JoinColumn一起连用.....说好的双向关联必加MappedBy呢.....也是醉了，然后既然这样，那么JoinColumn的name
只能是默认的，原本的功能也失去了，第三个JoinTable 多对多，更改创建表的名字。。。。。呵呵


##java.lang.IllegalArgumentException: node to traverse cannot be null!
出现这个错误多半是hql语句出错，仔细检查便可

##mappedBy与JoinColumn
这两个东西一直困扰了我好久，什么时候设置，应该怎么设置，设置这个的用途，今天找到了一个不错的东西，于是将他搬到这里
<h3>mappedBy</h3>
  <ul>
        <li>只有one2one,one2many,many2many才有mappedBy属性,many2one不存在</li>
        <li>mappedBy标签一定是定义在被拥有方的,他指向拥有方</li>
        <li>mappedBy含义,应该说，拥有方能够自动维护跟被拥有方的关系,当然 如果从被拥有方通过手工强行来维护拥有方的关系也是可以做到的</li>
        <li>mappedBy跟JoinColumn/Jointable总是处于互斥的一方,可以理解正是由于拥有方的关联被拥有方的字段的存在，拥有方才拥有了被拥有方,mappedBy这方定义JoinColumn/Jointable总是失败的</li>
  </ul>
例子：
*@OneToOne(cascade = CascadeTye.ALL,optional = true)
* public IDCard getIdCard(){
*    return idCard;
* }
*@OneToOne(cascade = CascadeType.ALL,mappedBy = "idCard",optional = false)
* public Person getPerson(){
*    return person;
* }
<p>多了一个mappedBy这个方法，他表示什么呢？它表示当前所在表和Person的关系是定义在Person里面的idCard这个成员上面的，他表示此表是一对一关系中的从表，也就是关系是在person表中维护的，这是最重要的。Person表是关系的维护者，有主导权，它有个外键指向IDCard。
   我们也可以让主导权在IDCard上面，也就是让他产生一个指向Person的外键，这也是可以的，但是最好让Person来维护整个关系，这样更符合我们的思维。
   我们也可以看到在Person里面的IDCard是注释optional=true,也就是说一个人是可以没有身份证的，但是一个身份证是不可以没有人的，所以在IDCard里面注释Person的时候，optional=false了，这样就可以防止一个空的身份证记录进数据库。</p>


另一个
  ** 1.person与address的一对一单向关系：

   在address中没有特殊的注解。

   在Person中对应到数据库里面就有一个指向Address的外键.

   我们也可以增加注释指定外键的列的名字,如下:
   @OneToOne(cascade=CascadeType.ALL,optional=true)
   @JoinColumn(name="addressID")//注释本表中指向另一个表的外键。
       public Address getAddress() {
           return address;
       }
   如果我们不加的话,也是可以通过的,在JBOSS里面,它会自动帮你生成你指向这个类的类名加上下划线再加上id的列,也就是默认列名是:address_id.
   如果是主键相关联的话,那么可以运用如下注释
   @OneToOne(cascade={CascadeType.ALL})
      @PrimaryKeyJoinColumn
      public Address getAddress( ) {
         return homeAddress;
      }
   它表示两张表的关联是根据两张表的主键的

   —————————————————————————————————————————————————————————————————————

   2.person和phone的一对多单向关系：

   phone中没有特别的注释。

   person中：

   @OneToMany(cascade=CascadeType.ALL)
   @JoinColumn(name="personID")//注释的是另一个表指向本表的外键。
   public List<Phone> getPhones() {
         return phones;
       }


   我们可以在Person类里面发现@JoinColumn(name="personID")
   它代表是一对多,一是指类本身,多是指这个成员,也就是一个类可以对应多个成员.
   在一对多里面,无论是单向还是双向,映射关系的维护端都是在多的那一方,也就是Phone那里,因为要在数据库面表现的话,也只有让Phone起一个指向Person的外键,不可能在Person里面指向Phone,这一点和一对一不一样,一对一可以在任意一方起一个外键指向对方.可是一对多却不行了.

   在这里@JoinColumn这个注释指的却是在Phone里面的外键的列的名字,

   它并不像在一对一里面的注释指的是自己表里面的外键列名.这一点要特别注意一下.


   如果是一对多的双向关系,那么这个注释就要应用到多的那边去了,虽然注释还在Person类里面,但是它起的效果却是在Phone里面起一个叫personID的外键, 因为多的那边要有外键指向少的这边.
   如果你不加 @JoinColumn(name="personID")这个注释的话,那么JBOSS就会自动帮你生成一张中间表,

   它负现Person和Phone表之间的联系.它将会做如下事情:

   CREATE TABLE PERSON_PHONE
   (
       PERSON_id INT,
    PHONE_id INT
   );
   ALTER TABLE PERSON_PHONE ADD CONSTRAINT person_phone_unique
      UNIQUE (PHONE_id);

   ALTER TABLE PERSON_PHONE ADD CONSTRAINT personREFphone
      FOREIGN KEY (PERSON_id) REFERENCES PERSON (id);

   ALTER TABLE PERSON_PHONE ADD CONSTRAINT personREFphone2
      FOREIGN KEY (PHONE_id) REFERENCES PHONE (id);

   所以我们最好还是指定一下,以让程序产生更加确定的行为，不过一般是推荐另外生成一个中间表好一些，因为这样的话，对原来两张表的结构不对造成任何影响。在遗留系统的时候很多用，有些时候，一些表都是以前就建好了的，要改表的结构是不太可能的，所以这个时候中间的表就显得很重要了，它可以在不侵入原来表的情况下构建出一种更清淅更易管理的关系。

   所以一对多的单向关联，我们还是推荐使用一张中间表来建立关系。

   ---------------------------------------------------------------------------------------------------------------------------------------------

   3.person和country的多对一单向关系：

   country中无特别的注解。

   而person注解如下：

   @ManyToOne
   @JoinColumn(name="countryID")
       public Country getCountry() {
           return country;
       }
   在多对一的关系里面，关系的维护端都是在多的那一面，多的一面为主控方，拥有指向对方的外键。
   因为主控端是Person .外键也是建在Person上面，因为它是多的一面。当然我们在这里也可以省掉@JoinColumn，那样的话会怎么样呢，会不会像一对多单向一样生成中间的表呢？事实是不会的，在这里如果我们去掉@JoinColumn的话，那么一样会在Person表里面生成一列指向

   <h5>Country的外键，这一点和一对多的单向是不一样，在一对多的单向里面，如果我们不在Person 里面加上@JoinColumn这个注释，那么JBOSS将会为我们生成一个中间的表，这个表会有一个列指向Person主键，一个列指向Phone主键。所以说为了程序有一定的行为，有些东西我们还是不要省的好。
   其实多对一单向是有点向一对一单向的，在主控端里面，也就是从Person的角度来看，也就是对应了一个Country而已，只不过这个Country是很多Person所共用的，而一对一却没有这一点限制。</h5>

   －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－

   4.person和project的多对多单向关系：

   project没有特殊的注解。

   person：

   @ManyToMany
       public List<Project> getProjects() {
           return projects;
       }
   它需要设置中间表来维护关系，在数据库上跟多对多双向，只不过在编程的逻辑中不一样而已。

   //类似这个：@JoinTable(name = "PersonANDFlight", joinColumns = {@JoinColumn(name = "personID")},
   //inverseJoinColumns = {@JoinColumn(name = "flightID")})
   其实这个声明不是必要的,当我们不用@JoinTable来声明的时候,JBOSS也会为我们自动生成一个连接用的表,

   表名默认是主控端的表名加上下划线"_"再加上反转端的表名.

   类似

   @ManyToMany(cascade = CascadeType.ALL)
       @JoinTable(name = "PersonANDFlight",
   　　　　　　　　joinColumns = {@JoinColumn(name = "personID")},
   　　　　　　　　inverseJoinColumns = {@JoinColumn(name = "flightID")})
       public List<Flight> getFlights() {
           return flights;
       }

   －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
   在单向关系中没有mappedBy,主控方相当于拥有指向另一方的外键的一方。
   1.一对一和多对一的@JoinColumn注解的都是在“主控方”，都是本表指向外表的外键名称。
   2.一对多的@JoinColumn注解在“被控方”，即一的一方，指的是外表中指向本表的外键名称。
   3.多对多中，joinColumns写的都是本表在中间表的外键名称，
     inverseJoinColumns写的是另一个表在中间表的外键名称。
##org.hibernate.AnnotationException: No identifier specified for entity: relationMapping.extendrelationship.mapping_joined.Person
在搞继承映射的时候出现了这个异常，查了一下，说是加了Entity的实体类必须要有主键，否则就会出现这个错误


##关于javaEE6.0
一句话,hibernate4.3.8与javaEE6.0不匹配,所以之前的一些未得到解决的错误可能都来自于此,主要的组件可能是jpa的版本不同而导致的吧,之前load方法的问题也解决了
##关于程序自动建表的问题
自动建表确实有一点问题,不过确实是十分的方便,今天做学生，课程，分数的练习，由于自动建表的语句出现了错误，所以我打算自己建表，但是建完之后出现了错误，语句不能执行的错误，仔细检查后发现，自动建表id类型
为integer类型，可是我自己建的表却是int类型，我想可能是这方面的问题，于是我将表全部删掉重新建了一遍，终于好使了

##关于hibernate4.3.8的悲观锁问题
按照视频上来说,在load方法的里面添加第三个参数LockMode就可以建立悲观锁,但是当我试验的时候却发现这个方法已经过时了，这个就比较坑了。。而且官方的文档上还没有给出说明,暂且记住吧


##createQuery与createSQLquery的区别
前者用的hql语句进行查询，后者可以用sql语句查询
前者以hibernate生成的Bean为对象装入list返回
后者则是以对象数组进行存储
所以使用createSQLQuery有时候也想以hibernate生成的Bean为对象装入list返回，就不是很方便
突然发现createSQLQuery有这样一个方法可以直接转换对象
Query query = session.createSQLQuery(sql).addEntity(XXXXXXX.class);
XXXXXXX 代表以hibernate生成的Bean的对象，也就是数据表映射出的Bean。



##iterator与list
利用createQuery取得的结果可以有两种，1.使用List遍历.2.使用iterator遍历。简单说一下这两个的区别.记的好像看过。巩固一下。List为取出所有对象,但是iterator是取出所有的id然后取出的时候来进行id的查询



##关于细节问题
其一：createQuery使用的是hql，其中from..指定的是表名，但是如果给@Entity的name属性赋值的时候那个form后面的实体类名就会变成name后面的属性值
其二：使用.List方法的时候要求实体类必须得有默认的构造方法,今天为了方便就是写了一个带参数的构造方法，结果就报错了