<font size=3>
[官方地址](http://greenrobot.org/greendao/)

[官方github](https://github.com/greenrobot/greenDAO)
### GreenDao的作用
    通过greendao我们可以更快速的的操作数据库，我们可以使用简单的面向对象的API来存储、删除、更新、查询JAVA对象
### Greendao优缺点
    1、高性能。
    2、易于使用的强大API，涵盖关系和连接；
    3、最小的内存消耗;
    4、依赖库大小（<100KB）以保持较低的构建时间并避免65k方法限制;
    5、数据库加密：greenDAO支持SQLCipher，以确保用户的数据安全;
### Greendao配置
1.导入Gradle插件
    
```
在 Project的build.gradle 文件中添加:
buildscript {
    repositories {
        jcenter()
        mavenCentral() // add repository
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.1.2'
        classpath 'org.greenrobot:greendao-gradle-plugin:3.2.2' // add plugin
    }
}
```
2.添加依赖

```
在 Moudle:app的  build.gradle 文件中添加:
apply plugin: 'com.android.application'
apply plugin: 'org.greenrobot.greendao' // apply plugin
 
dependencies {
    implementation 'org.greenrobot:greendao:3.2.2' // add library
}
```
3.配置数据库信息

```
greendao {
    schemaVersion 1 //数据库版本号
    daoPackage 'com.aserbao.aserbaosandroid.functions.database.greenDao.db'
// 设置DaoMaster、DaoSession、Dao 包名，当不配置时默认在 module的build/generated/source/greendao
    targetGenDir 'src.main.java'//设置DaoMaster、DaoSession、Dao目录,请注意，这里路径用.不要用/
    generateTests false //设置为true以自动生成单元测试。
    targetGenDirTests 'src/main/java' //应存储生成的单元测试的基本目录。默认为 src / androidTest / java。
}
```
###### 配置完成，在Android Studio中使用Build> Make Project，重写build项目，GreenDao集成完成！


### Greendao使用
1. 初始化
    
```
 /**
     * 初始化GreenDao,直接在Application中进行初始化操作
     */
    private void initGreenDao() {
        DaoMaster.DevOpenHelper helper = new DaoMaster.DevOpenHelper(this, "aserbao.db");
        SQLiteDatabase db = helper.getWritableDatabase();
        DaoMaster daoMaster = new DaoMaster(db);
        daoSession = daoMaster.newSession();
    }
    
    private DaoSession daoSession;
    public DaoSession getDaoSession() {
        return daoSession;
    }
```
2、创建存储对象实体类

```
@Entity
public class Student {
    @Id(autoincrement = true)
    Long id;
    @Unique
    int studentNo;//学号
    int age; //年龄
    String telPhone;//手机号
    String sex; //性别
    String name;//姓名
    String address;//家庭住址
    String schoolName;//学校名字
    String grade;//几年级
    ……getter and setter and constructor method……
    }
```
3、增删改查的使用
    https://www.jianshu.com/p/53083f782ea2

4、混淆配置


```

-keepclassmembers class * extends org.greenrobot.greendao.AbstractDao {
public static java.lang.String TABLENAME;
}
-keep class **$Properties
 
# If you do not use SQLCipher:
-dontwarn org.greenrobot.greendao.database.**
# If you do not use Rx:
-dontwarn rx.**
```


### Greendao注解
 1、@Entity
 
 用于标记实体类，表示Greendao为其创建数据库表，生成响应的***dao。
 
```
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface Entity {

    /**
     *表示实体类在数据裤中常见的表名称，默认情况下数实体类的英文大写，英文单词以下划线分隔
     */
    String nameInDb() default "";

    /**
     *表示创建一个索引列，
     */
    Index[] indexes() default {};

    /**
     * 是否在数据库中创建表，默认为true
     */
    boolean createInDb() default true;

    /**
     * 
     */
    String schema() default "default";

    /**
     * 标记一个实体是否处于活动状态，活动实体有 update、delete、refresh 方法。默认为 false
     */
    boolean active() default false;

    /**
     * 是否产生所有参数构造器。默认为 true。无参构造器必定产生
     */
    boolean generateConstructors() default true;

    /**
     * 是否生成get和set方法，默认是true
     */
    boolean generateGettersSetters() default true;

    /**
     * 为这个实体类额外创建Dao的工具类
     */
    Class protobuf() default void.class;

}

```
2、@Id：用于标记字段，表示是否为表的ID，ID的字段应该定义为long或Long型。

```
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.FIELD)
public @interface Id {
    /**
     * 是否自增 默认是false
     */
    boolean autoincrement() default false;
}
```
3、@NotNull：用于标记字段是否是非空字段

4、@Unique ：用于标记字段是否唯一

5、@Generated：编译后自动生成的构造函数、方法等的注释，提示构造函数、方法等不能被修改

6、@Transient：标识 该字段不存入数据库中

7、 @ToOne：用于标记字段，字段类型应该是被@Entity 标记的类，对应一对一关系。
```
例如
@Entity
public class Student {
 
    @Id(autoincrement = true)
    private Long id;
    private String name;
    private Long tId;
    //  joinProperty 为外键
    @ToOne(joinProperty = "tId") 
    private Teacher teacher;
}
```
8、ToMany ：用于标记字段，字段类型应该是被@Entity 标记的类的列表，对应一对多关系或者对多关系。


```
一对多示例
@Entity
public class Teacher {
 
 
    @Id(autoincrement = true)
    private Long id;
    private String name;
    // referencedJoinProperty标记在Student表内的与Teacher关联的外键
    @ToMany(referencedJoinProperty = "tId")
    private List<Student> studnets; // not persisted
}

多对多示例
学生表
@Entity
public class Student {

    @Id(autoincrement = true)
    @NotNull
    private Long mId;//表格唯一ID

    @Unique
    private String identityID;//身份证

    private int mAge;//年龄

    @ToMany
    @JoinEntity(
            entity = Student_Course.class,
            sourceProperty = "studentId",
            targetProperty = "courseId"
    )
    private List<Course> list;

}
课程表
@Entity
public class Course {

    @Id(autoincrement = true)
    @NotNull
    private Long mId;//表格唯一ID

    private String course;//课程

}
关系表
@Entity
public class Student_Course {

    @Id(autoincrement = true)
    @NotNull
    private Long mId;//表格唯一ID

    private Long studentId;//学生ID

    private Long courseId;//课程ID
}

```

9、@OrderBy：用于标记字段，指集合按照某字段倒序排序（正弦或者倒叙）

```
@Entity
public class StudentClass {

    @Id(autoincrement = true)
    @NotNull
    private Long mId;

    private String name;//班级名称

    @ToMany(referencedJoinProperty="classId")//classId为学生表中的classId字段，和班级表中的mId对应
    @OrderBy(value = "mAge desc")// Student集合按照的mAge的倒序排序
    List<Student> list;


}
```
### Greendao 3 gradle插件方式和Generator方式代码生成的不同

1、gradle插件方式 是我们先定义实体，然后生成相应的DaoSession、DaoMaster、**Dao
   Generator方式 生成实体、 DaoSession、DaoMaster、**Dao

2、gradle插件不支持schemas
```
官方说明
In most cases, we recommend to use the new Gradle plugin instead. However,
some advanced features are currently only supported by the generator.
For example multiple schemas.
```
3、Greendao 3.2.2支持RxJava


```
例
notesQuery = daoSession.getNoteDao().queryBuilder().orderAsc(NoteDao.Properties.Text).rx();
        notesQuery.list()
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<List<Note>>() {
                    @Override
                    public void call(List<Note> notes) {
                        notesAdapter.setNotes(notes);
                    }
                });
```


schemas以我的理解：不同的数据库，就像是服务端D版R版数据库。利用schemas创建不同的数据库

## 目前 Greendao 2 与Greendao 3的其他不同目前还没有发现，网上查阅资料也没找到其他不同。仅此愚见，大家多多指教

</font>
