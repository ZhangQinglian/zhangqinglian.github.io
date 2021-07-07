---
title: Android官方架构组件介绍之Room[翻译]
date: 2017-05-22 17:52:27
tags: android
cover: http://cdn.zqlxtt.cn/final-architecture.png
---
## 持久库Room

Room在SQLite上提供了一个抽象层，以便在利用SQLite的全部功能的同时使流畅的数据库访问。

需要处理一些重要的结构化数据的App通常会从本地的持久数据中受益匪浅。最常见的就是使用本地缓存，这样的话下次如果设备无法联网用户也能浏览本地数据并进行更改。等下次联网后再和服务器进行同步。

Android的Framework为了支持处理原始SQL而提供了SQLite这一强大的API，当时SQLite的API还是相对比较低级，在使用的时候需要花费大量的经历：

- 没有对原始SQL语句的编译时验证，随着数据库表格的更改，你需要更新相关SQL操作，而这个过程可能耗时且容易出错。
- 你需要使用大量的样板代码在SQL查询和Java数据对象之间进行转换。

`Room`在为SQL提供抽象层的同时也会考虑到上述的问题。
<!-- more -->
下面是Room中三个主要组件：

- **Database：**此组件用于创建数据库的持有者，同时在类层级上使用注解来定义一系列的`Entity`，这些Entity对应着数据库中的表格。Database类中的方法则用来获取对应的DAO列表。Database是App层与底层SQLite之间的连接点。
在应用中要使用此组件的话需要继承`RoomDatabase`。然后通过`Room.databaseBuilder()`或者`Room.inMemoryDatabaseBuilder().`获得该类的实例。（讲到这里其实读者可以发现，这不就是GreenDao吗？😂）。

- **Entity：**此组件的一个实例表示数据库的一行数据，对于每个Entity类来说，都会有对应的`table`被创建。想要这些Entity被创建，就需要写在上面Database的注解参数`entities`列表中。默认Entity中的所有字段都会拿来创建表，除非在该字段上加上`@Ignore`注解。

> **注意：**Entity默认都只有空的构造方法（如果DAO类可以访问每个持久化字段），或者构造方法的参数与Entity中的字段的类型和名字相匹配。Room可以使用全字段构造方法，也可以使用部分字段构造方法。

- **DAO：**这个组件用来表示具有`Data Access Object(DAO)`功能的类或接口。DAO类是Room的重要组件，负责定义访问数据库的方法。继承`RoomDatabase`的类必须包含一个0参数且返回DAO类的方法。当在编译期生成代码的时候，Room会创建实现此DAO的类。

> **注意：**通过使用DAO类而不是传统的查询接口来访问数据库，可以做到数据库组件的分离。同时DAO可以在测试APP时支持Mock数据。

<!--more-->

下面是其三者和数据库的关系图：

![room architecture](http://backup.flutter-dev.cn/room_architecture.png)

下面看一下简单的实例，其包含一个Entity，一个Dao以及一个Database。

User.java

```java
@Entity
public class User {
    @PrimaryKey
    private int uid;

    @ColumnInfo(name = "first_name")
    private String firstName;

    @ColumnInfo(name = "last_name")
    private String lastName;

    // Getters and setters are ignored for brevity,
    // but they're required for Room to work.
}
```

UserDao.java

```java
@Dao
public interface UserDao {
    @Query("SELECT * FROM user")
    List<User> getAll();

    @Query("SELECT * FROM user WHERE uid IN (:userIds)")
    List<User> loadAllByIds(int[] userIds);

    @Query("SELECT * FROM user WHERE first_name LIKE :first AND "
           + "last_name LIKE :last LIMIT 1")
    User findByName(String first, String last);

    @Insert
    void insertAll(User... users);

    @Delete
    void delete(User user);
}
```

AppDatabase.java

```java
@Database(entities = {User.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```

当创建完这些文件后，你就可以使用下面的方法来获得被创建的AppDatabase实例：

```java
AppDatabase db = Room.databaseBuilder(getApplicationContext(),
        AppDatabase.class, "database-name").build();
```

> **注意：**实例化AppDatabase对象时，应遵循单例设计模式，因为每个数据库实例都相当昂贵，而且很少需要访问多个实例。


## Entity

当一个类被添加了`@Entity`注解并且在Database的`@entities`被引用，Room就会为其创建对应的数据库。

默认情况Room会为Entity的每个字段创建对应的数据库列，如果某个字段不想被创建的话可以使用`@Ignore`注解：

```java
@Entity
class User {
    @PrimaryKey
    public int id;

    public String firstName;
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

为了Room可以访问到Entity的字段，你可以将这些字段声明为`public`，或者可以给这些字段提供`setter`和`getter`方法。如果使用setter和getter的话，需要注意命名规则。具体参照`Java Beans`。

### Primary key

每个Entity至少定义一个主键，即使你的Entity只有一个字段也是如此。定义主键使用`@PrimaryKey`。如果你想让Room给你的Entity自动生成ID的话，可以使用@Primary的`autoGenerate`属性。如果Entity具有复合主键的话，可以使用@Entity的primaryKeys属性，参照下方代码：

```java
@Entity(primaryKeys = {"firstName", "lastName"})
class User {
    public String firstName;
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

默认情况Room使用Entity的类名来作为数据库的表名。如果想自定义表名，可以使用@Entity的`tableName`属性,如下：

```java
@Entity(tableName = "users")
class User {
    ...
}
```

> **注意：**SQLite中的表名是大小写不敏感的。

与上面的tableName类似，Room使用Entity的字段名来作为对应的列名，如果想要自定义类名，可以使用`@ColumnInfo`注解的`name`属性，如下：

```java
@Entity(tableName = "users")
class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

### 索引及唯一性

在适当的字段上添加索引可以加快数据库的访问速度，要在Entity上添加索引可以使用@Entity的`indices`属性，可以添加索引或组合索引：

```java
@Entity(indices = {@Index("firstName"), @Index("last_name", "address")})
class User {
    @PrimaryKey
    public int id;

    public String firstName;
    public String address;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```
有些情况下，数据库中的某个字段或字段组合必须是唯一的，可以通过将@Index的属性`unique`设置为ture来实现这一唯一性。以下代码用于放置User表中出现姓名组合相同的数据。

```java
@Entity(indices = {@Index(value = {"first_name", "last_name"},
        unique = true)})
class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;

    @Ignore
    Bitmap picture;
}
```

### 表间关系

由于SQLite是关系型数据库，所以你可以指定对象之间的关系，但在Room中这是命令禁止的。

虽然在Room中的Entity不能有直接的引用关系，但Room任然支持在Entity间定义`Foreign Key`。

例如有个另一个Entity叫做`Book`，你可以使用`@ForeignKey`来定义它和User之间的关系，如下：

```java
@Entity(foreignKeys = @ForeignKey(entity = User.class,
                                  parentColumns = "id",
                                  childColumns = "user_id"))
class Book {
    @PrimaryKey
    public int bookId;

    public String title;

    @ColumnInfo(name = "user_id")
    public int userId;
}
```

外键是十分强大的，它允许你指定引用实体发生更新是发生的行为，比如，当需要删除一个用户的时候删除其下所有的图书，只需要为Book的@ForeignKey的属性`onDelete`设置为`CASCADE`。

> **注意：**SQLite在处理`@Insert(onConflict=REPLACE)`的时候，其实是进行了`REMOVE`和`REPLACE`两个操作，而不是单单的`UPDATE`。此时这里的REMOVE操作可能会影响到对应的外键，


### 嵌套对象

有时你需要在数据库逻辑中表达一个实体或者Java类，你可以使用`@Embedded`注解来实现。具体看例子。

例如上面的User实体有一个`Address`类型的字段，Address包含了`street,city,state`和`postCode`这几个字段。当生成表格时，Address中的字段将被分别定义为User表中的列名。如下：

```java
class Address {
    public String street;
    public String state;
    public String city;

    @ColumnInfo(name = "post_code")
    public int postCode;
}

@Entity
class User {
    @PrimaryKey
    public int id;

    public String firstName;

    @Embedded
    public Address address;
}
```

这是User表包含以下字段：`id, firstName, street, state, city`和`post_code`。

> **注意：**以上是可以多重嵌套的。

如果User中嵌套的A和B中存在相同字段，可以使用@Embedded的`prefix`属性，Room会在生成table的时候将prefix的值加在列名前。

## Data Access Objects (DAOs)

Room中的主要组件就是`Dao`，DAO以简洁的方式抽象访问数据库。

### Intert

当你创建了一个DAO的方法并加上`@Insert`注解，Room就会生成一个这个方法是实现，用于完成此次插入操作：

```java
@Dao
public interface MyDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    public void insertUsers(User... users);

    @Insert
    public void insertBothUsers(User user1, User user2);

    @Insert
    public void insertUsersAndFriends(User user, List<User> friends);
}
```
如果插入方法只接受一个参数的话，表示仅仅插入一条数据，这是这个方法可以返回一个long型值，为新行的id。如果参数为数组或集合，则需要返回对应的`long[]`或者`List<Long>`。

### Update

Update是一个用于更新批量数据的实用方法，它通过主键来匹配需要更改数据库数据：

```java
@Dao
public interface MyDao {
    @Update
    public void updateUsers(User... users);
}
```
此方法可以返回一个int型数据，表示此次修改影响到的行数。

### DELETE

Delete用于批量删除数据库中的数据，它也是通过主键来匹配需要删除的数据：

```java
@Dao
public interface MyDao {
    @Delete
    public void deleteUsers(User... users);
}
```
此方法可以返回一个int型数据，表示此次删除的行数。

### QUERY

`@Query`是DAO中的一个重要注解，它允许你对数据库进行读写操作。每一个@Query方法都会在编译期做校验，所以如果query存在问题的话，你的App编译将无法通过。

Room同时也会校验query的返回值，如果返回结果和查询语句中的结果不匹配，Room将会以一下两种方式提醒你：

- 如果有部分字段匹配的话会给出警告。
- 如果没有字段匹配，则给出错误提示。

#### 简单的查询

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM user")
    public User[] loadAllUsers();
}
```
这是一个加载所有用户的查询，写法比较简单。在编译期，Room知道需要查询User的所有列的值。如果查询语句包含语法错误或者没有user这个表，则Room会在编译时期报错并给出错误信息。

#### 查询的参数传递
大部分情况，你需要给查询语句传递特定的参数，比如查询特定年龄段的User，如下：

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge")
    public User[] loadAllUsersOlderThan(int minAge);
}
```
在编译器处理这个查询操作的时候，Room会将参数minAge与`:minAge`进行绑定。如果此时无法匹配，则会出现编译错误。

当然也可以传递多个参数，如下：

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
    public User[] loadAllUsersBetweenAges(int minAge, int maxAge);

    @Query("SELECT * FROM user WHERE first_name LIKE :search "
           + "OR last_name LIKE :search")
    public List<User> findUserWithName(String search);
}
```

#### 返回所有列的子集

通常你需要的只是Entity的一部分字段，例如你的UI只需要先死User的姓名，而不是所有信息。这是为了保证UI的更新速度，你会选择只查询姓名这个两个数据。

只要可以将查询的结果集映射到返回对象的字段，你就可以返回任何对象，如下：

```java
public class NameTuple {
    @ColumnInfo(name="first_name")
    public String firstName;

    @ColumnInfo(name="last_name")
    public String lastName;
}
```
现在你可以在DAO中使用NameTuple了。

```java
@Dao
public interface MyDao {
    @Query("SELECT first_name, last_name FROM user")
    public List<NameTuple> loadFullName();
}
```
Room能够返回的`first_name`和`last_name`能够映射到NameTuple，所以Room会生成相应的赋值代码。如果返回字段太多或者字段不存在于NameTuple中，则会发生编译出错。

> **注意：**这里的NameTuple也可以使用@Embedded注解。

#### 将集合作为参数传递

有些情况当你查询时需要传递较多的变量，例如想要查询某一地区集合下的所有用户，这个集合可能包含几十个地区，如果用上述简单的参数传递恐怕够呛，现在看看怎么用集合传递：

```java
@Dao
public interface MyDao {
    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    public List<NameTuple> loadUsersFromRegions(List<String> regions);
}
```
Room可以判断你传递的是集合，并在SQL语句中将你的参数进行展开并填充。

#### 可监听的查询

在进行查询的时候，你希望UI会在查询结束后自动更新UI，为了满足这一点，这里可以使用前面讲到的`LiveData`对你的查询返回值进行封装。如下：

```java
@Dao
public interface MyDao {
    @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
    public LiveData<List<User>> loadUsersFromRegionsSync(List<String> regions);
}
```

如果你比较熟悉RxJava，那么很高兴告诉你，Room同样支持返回ExJava2中的`Publisher`和`Flowable`对象，如下：

```java
@Dao
public interface MyDao {
    @Query("SELECT * from user where id = :id LIMIT 1")
    public Flowable<User> loadUserById(int id);
}
```

#### 直接返回Cursor
如果你的App中有部分逻辑需要直接用Cursor的话，可以将DAO的返回值设置为Curso，如下：

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM user WHERE age > :minAge LIMIT 5")
    public Cursor loadRawUsersOlderThan(int minAge);
}
```

> **注意：**Room很不推荐使用以上Cursor的方法，应为你并不知道Cursor有无数据或者包含哪些列。

#### 多表联查

Room支持多表联查，如果返回数据是可监听的，那么Room会监听所有查询中涉及到的表并及时更新数据。
下面这个例子是通过内联查询某个名字下借阅的图书：

```java
@Dao
public interface MyDao {
    @Query("SELECT * FROM book "
           + "INNER JOIN loan ON loan.book_id = book.id "
           + "INNER JOIN user ON user.id = loan.user_id "
           + "WHERE user.name LIKE :userName")
   public List<Book> findBooksBorrowedByNameSync(String userName);
}
```

你也可以通过查询返回纯java对象，如下：

```java
@Dao
public interface MyDao {
   @Query("SELECT user.name AS userName, pet.name AS petName "
          + "FROM user, pet "
          + "WHERE user.id = pet.user_id")
   public LiveData<List<UserPet>> loadUserAndPetNames();

   // You can also define this class in a separate file, as long as you add the
   // "public" access modifier.
   static class UserPet {
       public String userName;
       public String petName;
   }
}
```

## 类型转换

Room中的类型转换支持你将某个类的值存储到某一列中，为此Room提供了`TypeConverter`这个类用于将自定义类转换成Room所支持的类型。

例如我们想要将`Date`对象进行存储，我们可以这么写：

```java
public class Converters {
    @TypeConverter
    public static Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public static Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}
```

这样定义完以后，下次Room遇到Date，就能将其转换成Room所支持的Long了。

下面看看AppDatabase要怎么写：

```java
@Database(entities = {User.java}, version = 1)
@TypeConverters({Converter.class})
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```
在AppDatabase上添加`TypeConverters注解`,并将Converter作为其参数。

接着User实体：

```java
@Entity
public class User {
    ...
    private Date birthday;
}
```

然后是DAO：

```java
@Dao
public interface UserDao {
    ...
    @Query("SELECT * FROM user WHERE birthday BETWEEN :from AND :to")
    List<User> findUsersBornBetweenDates(Date from, Date to);
}
```

这里你可以对@TypeConverter做一些范围限制，比如限制只能在某个Entity，某个DAO或某个DAO方法中使用。详细说明可见@TypeConverter文档。


## 数据库迭代升级

当你的App迭代升级的时候，也需要给你的Entity做迭代升级，为此你将修改Entity的代码。当你的用户升级到最新的App版本的时候，你可不希望他们丢失老版本的所有数据，尤其是在没有服务器备份的情况下。

Room支持通过写`Migration`类来保留用户数据。每个Migration都需要指定上一个版本和现在的版本，在App运行的时候，Room会运行每一个Migration的`migrate`方法，并使用正确顺序将数据库升级到最新版本。

> **注意：**如果你不提供Migration的话，Room会重建数据库而不是升级数据库，这样的后果就是用户数据会全部都是。


```java
Room.databaseBuilder(getApplicationContext(), MyDb.class, "database-name")
        .addMigrations(MIGRATION_1_2, MIGRATION_2_3).build();

static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("CREATE TABLE `Fruit` (`id` INTEGER, "
                + "`name` TEXT, PRIMARY KEY(`id`))");
    }
};

static final Migration MIGRATION_2_3 = new Migration(2, 3) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        database.execSQL("ALTER TABLE Book "
                + " ADD COLUMN pub_year INTEGER");
    }
};
```

> **注意：**为了使迁移逻辑保持正常运行，请使用完整的查询语句，即使用硬编码（对这里推荐硬编码）。而不是用一些字符串引用。

一旦升级工作完成，Room会进行schema的验证，如验证有误，则会抛出异常。

### 测试升级
Migration并不是简单的数据库写入操作，一旦升级失败，会对App致命的Crash。为了保证应用的稳定性，应该事先测试Migration，Room提供了一套测试框架，下面我们来简单学习下。

#### 导出Schema文件
Room需要将你数据库的Schema已Json格式的文件导出，为了导出Schema，需要在`build.gradle`中做如下配置：

```groovy
android {
    ...
    defaultConfig {
        ...
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = ["room.schemaLocation":
                             "$projectDir/schemas".toString()]
            }
        }
    }
}
```
你需要将导出的Json文件保存起来，以便Room通过schema文件创建老版数据库进行升级测试。

为了进行升级测试，需要将`android.arch.persistence.room:testing`添加到你的测试依赖当中，然后添加如下配置：

```groovy
android {
    ...
    sourceSets {
        androidTest.assets.srcDirs += files("$projectDir/schemas".toString())
    }
}
```
测试框架提供了名为`MigrationTestHelper`的类，它可以读取schema文件，这也是一个遵循`Junit4`测试原则的类。具体测试代码如下：

```java
@RunWith(AndroidJUnit4.class)
public class MigrationTest {
    private static final String TEST_DB = "migration-test";

    @Rule
    public MigrationTestHelper helper;

    public MigrationTest() {
        helper = new MigrationTestHelper(InstrumentationRegistry.getContext(),
                MigrationDb.class.getCanonicalName(),
                new FrameworkSQLiteOpenHelperFactory());
    }

    @Test
    public void migrate1To2() throws IOException {
        SupportSQLiteDatabase db = helper.createDatabase(TEST_DB, 1);

        // db has schema version 1. insert some data using SQL queries.
        // You cannot use DAO classes because they expect the latest schema.
        db.execSQL(...);

        // Prepare for the next version.
        db.close();

        // Re-open the database with version 2 and provide
        // MIGRATION_1_2 as the migration process.
        db = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2);

        // MigrationTestHelper automatically verifies the schema changes,
        // but you need to validate that the data was migrated properly.
    }
}

```

## 测试数据库

当你的应用程序运行测试时，如果你没有测试数据库本身，则不需要创建完整的数据库。Room允许你轻松地模拟测试中的数据访问层。这个过程是可能的，因为您的DAO不会泄露您的数据库的任何细节。测试其余的应用程序时，应该创建DAO类的模拟或假的实例。

这里推荐在Android设备上编写JUnit测试，因为这些测试并不需要UI的支持，所以这些测试会比UI测试速度更快。

测试代码如下：

```java
@RunWith(AndroidJUnit4.class)
public class SimpleEntityReadWriteTest {
    private UserDao mUserDao;
    private TestDatabase mDb;

    @Before
    public void createDb() {
        Context context = InstrumentationRegistry.getTargetContext();
        //将数据库建在内存中，可以让你的测试整体更加一体化，更密闭。
        mDb = Room.inMemoryDatabaseBuilder(context, TestDatabase.class).build();
        mUserDao = mDb.getUserDao();
    }

    @After
    public void closeDb() throws IOException {
        mDb.close();
    }

    @Test
    public void writeUserAndReadInList() throws Exception {
        User user = TestUtil.createUser(3);
        user.setName("george");
        mUserDao.insert(user);
        List<User> byName = mUserDao.findUsersByName("george");
        assertThat(byName.get(0), equalTo(user));
    }
}
```

## 补充：禁止Entity之间的相互引用

将数据库中的关系映射到相应的对象模型是一个常见的做法，在服务器端可以很好地运行，在访问它们时，它们可以很方便地加载字段。

然而，在客户端，延迟加载是不可行的，因为它可能发生在UI线程上，并且在UI线程中查询磁盘上的信息会产生显着的性能问题。UI线程有大约16ms的时间来计算和绘制Activity的更新的布局，所以即使一个查询只需要5 ms，你的应用程序仍然可能耗尽用于绘制的时间，引起明显的卡顿。更糟糕的是，如果并行运行单独的事务，或者设备忙于其他磁盘重的任务，则查询可能需要更多时间才能完成。但是，如果不使用延迟加载，则应用程序将获取比其需要的更多数据，从而产生内存消耗问题。

ORM通常将此决定留给开发人员，以便他们可以为应用程序的用例做最好的事情。不幸的是，开发人员不会在他们的应用程序和UI之间共享模型。UI随着时间的推移而变化，难以预料和调试的问题会不断出现。

例如，使用加载Book对象列表的UI为例，每本书都有一个Author对象。你可能最初设计你的查询时使用延迟加载，以便Book的实例使用getAuthor()方法来返回作者。一段时间后，你意识到需要在应用中显示作者姓名。你可以轻松添加方法调用，如以下代码片段所示：

```java
authorNameTextView.setText(user.getAuthor().getName());
```
就这么一个简单的操作，导致了在主线程中访问数据库。如果Author用引用了另一张表，那情况可能更糟糕。如果需求变化，这个界面不在需要作者姓名，那么你的代码可能会做无畏的延迟加载。

基于以上原因，Room禁止Entity之间的引用，如果需要加载相关数据，可以使用显示的方法去加载。