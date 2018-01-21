layout: android
title: android greendao3.0 多表关联踩坑实践
date: 2018-01-21 12:30:32
tags: android 
---
#### 前言
之前用过数据库框架：realm、kjdb，今天准备实践学习一下greendao 3.0。
greendao 3.0之前的版本有很大的不同，主要是增加了annotation注解，然后表之间和对象之间的关系也通过注解而变得更加灵活方便了。以前用过旧版本的都知道，对于多表多对象之间的关联，要写的代码不少。

我在学习greendao 3.0的时候，有一个感触，网上的文章很多，但是千篇一律，大多都是翻译官方文档而来，举得例子可谓是“无一例外”。网上搜罗了半天，收藏几篇比较好的帖子。
[史上最高效的ORM方案——GreenDao3.0详解](http://www.open-open.com/lib/view/open1480319783903.html)
[[Android ORM——初识greenDAO 3及使用greenDAO 3前应该掌握的一些知识点（一）](http://blog.csdn.net/crazymo_/article/details/54629916)](http://blog.csdn.net/crazymo_/article/details/54629916)

特别是关于3.0对象多表多对象关联的博客更是没有，所以打算自己实践学习然后总结分享一番。

#### 踩坑
主要踩了两个坑：
- greendao的关联关系是通过**主外键（对象之间关联的id）**来构建的。realm是直接通过对象关系来自动构建的。
- 如果属性是List<> xx,  greendao不会自动调用设置xx的值，只有**手动调用getXX的时候获取**.我在打印log的时候被坑惨了，无论怎么样都是为null.

#### 发现
- 首先bean类，会自动生成一些方法，比如get set 构造方法 getSession等
- 如果是list或者数组类型的属性XX，只有getXX方法，没有setXX方法
- 如果你要打印bean类的toString方法，这里**要调用getXXX方法，而不是直接打印对象**（因为没有赋值,而是在getXX的时候才赋值的）
```
    @Override
    public String toString() {
        return "Person{" +
               "students=" + getStudents() +  //这里不是直接students
                ", id=" + id +
                ", cardId=" + cardId +
                ", age=" + age +
                ", name='" + name + '\'' +
                ", average=" + average +
                ", cid=" + cid +
                ", cls=" + cls +
                ", fid=" + fid +
                ", friends=" + getFriends() +
                '}';
    }
```
- greendao支持 自身对自身的关系关联，比如

![](http://upload-images.jianshu.io/upload_images/1311457-73bf8704cedd9252.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

---

## 正文： 多表映射关联

#### 一对一：比如一个人有一个头
如果是按照以往的对象关系数据库
```
#person类中

   @Id(autoincrement = true)
    private Long id;
    private String name;

//    private Long hid;
    @ToOne
    private Head head;
```
```
  Head head = new Head();
        head.setId(13l);
        head.setName("head");

        Person p = new Person();
        p.setId(null);
        p.setHead(head);//直接设置对象
        p.setName("jafir");
```
直接写对象，然后setHead(head) 就搞定了。但是在greendao中一切对象关联关系都是通过主外键来实现的。应该改为如下：
```
#person类中
  @Id(autoincrement = true)
    private Long id;
    private String name;

    private Long hid;//这是与头关联的外键
    @ToOne(joinProperty = "hid") //这个是注解绑定 hid就是上面一行的hid
    private Head head;//对象，但是不需要setHead
```

build之后，就有setHid的方法了，我们的对象关联不采用setHead,而是setHid
```
PersonDao personDao = GreendaoHelper.getDaoSession().getPersonDao();
        HeadDao headDao = GreendaoHelper.getDaoSession().getHeadDao();

        Head head = new Head();
        head.setId(13l);//这里的head id和person里的hid一样
        head.setName("head");

        Person p = new Person();
        p.setId(null);
        p.setHid(13l);//这里的hid是head的id,就是这样通过id构建起关联的
        p.setName("jafir");

        headDao.insert(head);
        personDao.insert(p);

        List<Person> persons = personDao.queryBuilder().build().list();
        for (Person person : persons) {
            Log.d("debug","person:"+person.toString());
        }
```
注意：person的toString()方法系统生成的需要修改一下：
```
 @Override
    public String toString() {
        return "Person{" +
                "head=" + getHead() +//这里需要改为getHead
                ", name='" + name + '\'' +
                ", id=" + id +
                '}';
    }
```
###### tips:
- bean的id最好用Long类型而不是long
```
 @Id(autoincrement = true)
private Long id;
Person p = new Person();
        p.setId(null);
        p.setHid(14l);
        p.setName("jafir");
```
因为，如果**Long**,我们设置了id是自增长,我们可以```setId(null)```,便是自增长。如果是**long**类型，你设置```setId(null)```，就报空指针。

#### 一对多：比如一个老师有多个学生
```
#teacher类中
 @ToMany(referencedJoinProperty = "tid")//指定与之关联的其他类的id
 private List<Student> studnets;
```
```
#student类中
  @Id
  private Long id;
  private Long tid;//这个就是外键 就是person的id
```
关系描述：
多个学生都有同一个老师，所以每个学生的tid，就应该是同样的，并且tid 就是老师的id ,这样就构成了1对多的关系

![](http://upload-images.jianshu.io/upload_images/1311457-ce1187ae365babec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

注意：学生的自增长id ,跟与老师关联的tid是不一样的，两码事

#### 一对多：比如一个人有一群朋友（朋友也是person）
按照我们上面的思路，那么person类里面就应该有一个外键指向自身的主键id
```
#person类中
 private Long id;//自身id
 private Long fid;//外键关联id
 @ToMany(referencedJoinProperty ="fid" )//指定与之关联的其他类的id
 private List<Person> friends;
```

![](http://upload-images.jianshu.io/upload_images/1311457-3c4924653fee9baa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
如果一个人的id是1，他有3个朋友，那么friends里面person的fid都是1，这样这个人调用getFriends就能拿到自己的3个朋友。主要就是通过设置id来构建关联关系的。


#### 多对多：
多对多的话就比较复杂，不是两个表或者两个对象直接关联，而是要通过一个“第三者”


![](http://upload-images.jianshu.io/upload_images/1311457-76ff459f2380e8d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```
#person类中
    @Id(autoincrement = true)
    private Long id;
    private String name;
   // 对多，@JoinEntity注解：entity 中间表；sourceProperty 实体属性；targetProperty 外链实体属性
    @ToMany
    @JoinEntity(
            entity = JoinStudentToPerson.class,
            sourceProperty = "pid",
            targetProperty = "sid"
    )
    private List<Student> students;
```
```
//中间表   “第三者”
@Entity
public class JoinStudentToPerson {
   @Id(autoincrement = true)
    private Long id;
    //和person关联的id
    private Long pid;
    //和student关联的id
    private Long sid;
}
```
```
@Entity
public class Student {
    @Id
    private Long id;
    private String name;
    // 对多，@JoinEntity注解：entity 中间表；sourceProperty 实体属性；targetProperty 外链实体属性
    @ToMany
    @JoinEntity(
            entity = JoinStudentToPerson.class,
            sourceProperty = "sid",
            targetProperty = "pid"
    )
    private List<Person> persons;
}
```
然后测试代码
```
  private void test() {

        PersonDao personDao = GreendaoHelper.getDaoSession().getPersonDao();
        HeadDao headDao = GreendaoHelper.getDaoSession().getHeadDao();
        StudentDao studentDao = GreendaoHelper.getDaoSession().getStudentDao();
        JoinStudentToPersonDao spDao = GreendaoHelper.getDaoSession().getJoinStudentToPersonDao();

//        Head head = new Head();
//        head.setId(14l);
//        head.setName("head");

        Person p1 = new Person();
        p1.setId(1l);
        p1.setName("jafir1");
        Person p2 = new Person();
        p2.setId(2l);
        p2.setName("jafir2");
        Person p3 = new Person();
        p3.setId(3l);
        p3.setName("jafir3");


        Student stu1 = new Student();
        stu1.setId(1l);
        stu1.setName("stu1");
        Student stu2 = new Student();
        stu2.setId(2l);
        stu2.setName("stu2");
        Student stu3 = new Student();
        stu3.setId(3l);
        stu3.setName("stu3");


        // 模拟 多对多关系
        // 假如 p1有3个：stu1\stu2\stu3
        //  stu1 stu2 stu3 都有2个 :p1\p2

        //p1有stu1 stu2 stu3     那么反过来stu123都有p1
        JoinStudentToPerson sp1 = new JoinStudentToPerson();
        sp1.setPid(1l);
        sp1.setSid(1l);
        JoinStudentToPerson sp2 = new JoinStudentToPerson();
        sp2.setPid(1l);
        sp2.setSid(2l);
        JoinStudentToPerson sp3 = new JoinStudentToPerson();
        sp3.setPid(1l);
        sp3.setSid(3l);

        //p2有stu1 stu2 stu3     那么反过来stu123都有p2
        JoinStudentToPerson sp4 = new JoinStudentToPerson();
        sp4.setPid(2l);
        sp4.setSid(1l);
        JoinStudentToPerson sp5 = new JoinStudentToPerson();
        sp5.setPid(2l);
        sp5.setSid(2l);
        JoinStudentToPerson sp6 = new JoinStudentToPerson();
        sp6.setPid(2l);
        sp6.setSid(3l);

        spDao.insert(sp1);
        spDao.insert(sp2);
        spDao.insert(sp3);
        spDao.insert(sp4);
        spDao.insert(sp5);
        spDao.insert(sp6);

        personDao.insert(p1);
        personDao.insert(p2);
        personDao.insert(p3);

        studentDao.insert(stu1);
        studentDao.insert(stu2);
        studentDao.insert(stu3);

//        headDao.insert(head);
//        personDao.insert(p1);

        List<Person> persons = personDao.queryBuilder().build().list();

        for (Person person : persons) {
            Log.d("debug","person:"+person.toString());
        }
    }
```
注意：在person和student的toString里面不能都写getStudents getPerson，不然会你调我，我调你，然后死循环，出现log打印Stack Overflow。只能单独打印测试结果
```
//打印person的测试结果
Person{
students=[
Student{name='stu1', id=1}, 
Student{name='stu2', id=2},
 Student{name='stu3', id=3}], 
head=null, hid=null, name='jafir1', id=1}

Person{
students=[
Student{name='stu1', id=1}, 
Student{name='stu2', id=2}, 
Student{name='stu3', id=3}],
 head=null, hid=null, name='jafir2', id=2}

Person{
students=[], 
head=null, hid=null, name='jafir3', id=3}
```
students的测试结果就不列出了。
总之实现起来就是这样。

#### 后记
在探索3.0版本的时候，确实碰了不少壁，踩了很多坑，遇到了很多疑惑，但是通过自己不断地摸索探究测试，最终还是找到了解决的方法。希望对大家有用，于是分享出来，欢迎大家指正。

这个还要再说一下自己的看法：
对于greendao或者realm,个人觉得，realm确实更高级一点，是直接**对象关联**的，用起来也更方便一点，而且有很好的中文文档。greendao呢，其实还算是比较原始的数据库框架，但是它最大的优点就是**效率高**。

所以，如果你的数据量大要求效率，你应该使用greendao,不然简小的数据库还是建议使用realm。

