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