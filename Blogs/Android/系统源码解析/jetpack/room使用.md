### 1. 特性

- **SQL语句高亮**
- **简单入门**
- **功能强大**
- **数据库监听**
- **支持Kotlin协程/RxJava/Guava**



### 2.使用

- **添加依赖**

  ```groovy
  dependencies {
    def room_version = "2.2.0-rc01"
  
    implementation "androidx.room:room-runtime:$room_version"
    annotationProcessor "androidx.room:room-compiler:$room_version" 
    // Kotlin 使用 kapt 替代 annotationProcessor
  
    // 可选 - Kotlin扩展和协程支持
    implementation "androidx.room:room-ktx:$room_version"
  
    // 可选 - RxJava 支持
    implementation "androidx.room:room-rxjava2:$room_version"
  
    // 可选 - Guava 支持, including Optional and ListenableFuture
    implementation "androidx.room:room-guava:$room_version"
  
    // 测试帮助
    testImplementation "androidx.room:room-testing:$room_version"
  }
  ```

  ```groovy
  android {
    ...
      defaultConfig {
        ...
          javaCompileOptions {
            annotationProcessorOptions {
              arguments = [
                "room.schemaLocation":"$projectDir/schemas".toString(),
                "room.incremental":"true",
                "room.expandProjection":"true"]
            }
          }
      }
  }
  ```

  Room在创建数据库对象时就会创建好所有已注册的数据表结构

  1. 创建数据库
  2. 创建操作接口（增删查改）
  3. 创建数据类（表字段）

- **创建数据库**

  ```kotlin
  @Database(entities = [Book::class], version = 1)
  abstract class SQLDatabase : RoomDatabase() {
      abstract fun book(): BookDao// 操作接口
  }
  
  ```

- **创建操作接口**

  ```kotlin
  // 创建增删查改的接口，只需要在抽象类或接口添加@Dao注解即可
  @Dao
  interface BookDao {
  
      @Query("select * from Book where")
      fun qeuryAll(): List<Book>
  
      /*
      * @Insert注解声明当前的方法为新增的方法，并且可以设置当新增冲突的时候处理的方法。
      */
      @Insert(onConflict = OnConflictStrategy.REPLACE)
      fun insert(vararg book: Book): List<Long>
  
      /*
      * @Delete注解声明当前的方法为删除的方法，
      */
      @Delete
      fun delete(book: Book): Int
      
      @Delete
      fun delete(book: List<Book>)
      
      /*
      * @Query注解不仅可以声明这是一个查询语句，也可以用来删除和修改，不可以用来新增。
      */
      @Query("delete from Book where Id = :id")
      fun deleteAccount(id : Int)
      
      /*
      * @Update注解声明当前方法是一个更新方法
      */
      @Update
      fun update(book: Book): Int
      
      @Query("update Book set price = :price where name = :name")
      fun updatePrice(name : String, price: Int)
      
      @Query("select * from Book where Id = :id")
      fun getBookViaId(id : Int): List<Book>
      
      /*
      * 配合LiveData 返回所有的书籍
      */
      @Query("SELECT * FROM Book")
      fun getAllBook(): LiveData<List<Book>>
      
      /*
      * 配合RxJava 通过Id查询单本书籍
      */
      @Query("SELECT * FROM Book where Id = :id")
      fun getAllBook(id: Int): Flowable<Book>
  
  }
  ```

- **创建数据类**

  ```kotlin
  /**
   * 用户表
   */
  @Entity(tableName = "user")
  data class User(
      @ColumnInfo(name = "user_account") val account: String // 账号
      , @ColumnInfo(name = "user_pwd") val pwd: String // 密码
      , @ColumnInfo(name = "user_name") val name: String
      , @Embedded val address: Address // 地址
      , @Ignore val state: Int // 状态只是临时用，所以不需要存储在数据库中
  ) {
      @PrimaryKey(autoGenerate = true)
      @ColumnInfo(name = "id")
      var id: Long = 0
  }
  
  /**
   * 地址
   */
  data class Address(
      val street:String,
      val state:String,
      val city:String,
      val postCode:String
  )
  ```

  

  ```kotlin
  /**
  * 书本信息表
  */
  @Entity(tableName = "Book")
  data class Book(
      @PrimaryKey(autoGenerate = true)
      @ColumnInfo(name = "Id")
      var number: Long = 0
      @ColumnInfo(name = "book_name")
      var title:String
      @ColumnInfo(name = "book_price")
      var price: Double = 0
      @Embedded val publisher : Publisher // 出版社
  )
  
  /**
  * 出版社
  */
  data class Publisher(
      val name: String,
      val street:String,
      val state:String,
      val city:String,
      val postCode:String
  )
  
  ```

  ```kotlin
  /**
   * 收藏的书
   */
  @Entity( tableName = "fav_book", foreignKeys = [ForeignKey(entity = Book::class, parentColumns = ["id"], childColumns = ["book_id"]), ForeignKey(entity = User::class, parentColumns = ["id"], childColumns = ["user_id"])], indices = [Index("book_id")]
  )
  data class FavouriteBook(
      @ColumnInfo(name = "book_id") val shoeId: Long // 外键 鞋子的id
      , @ColumnInfo(name = "user_id") val userId: Long // 外键 用户的id
      , @ColumnInfo(name = "fav_date") val date: Date // 创建日期
  
  ) {
      @PrimaryKey(autoGenerate = true)
      @ColumnInfo(name = "id")
      var id: Long = 0
  }
  ```

  **注解及说明**

  |     注解      | 说明                                                         |
  | :-----------: | :----------------------------------------------------------- |
  |   `@Entity`   | 声明这是一个表（实体），主要参数：`tableName`-表名、`foreignKeys`-外键、`indices`-索引。 |
  | `@ColumnInfo` | 主要用来修改在数据库中的字段名。                             |
  | `@PrimaryKey` | 声明该字段主键并可以声明是否自动创建。                       |
  |   `@Ignore`   | 声明某个字段只是临时用，不存储在数据库中。                   |
  |  `@Embedded`  | 用于嵌套，里面的字段同样会存储在数据库中。                   |

  

- **使用**

  ```kotlin
  val mDatabase = Room.databaseBuilder(this, SQLDatabase::class, "database_name").build()
  
  val book = Book("活着")
  db.book().insert(book)
  
  val books = db.user().qeuryAll()
  ```



### 3.注解

- **@Entry**

  修饰类作为数据表，名称不区分大小写

  ```java
  public @interface Entity {
      /**
       * 数据表名, 默认以类名为表名
       */
      String tableName() default "";
  
      /**
       * 索引 示例: @Entity(indices = {@Index("name"), @Index("last_name", "address")})
       */
      Index[] indices() default {};
  
      /**
       * 是否继承父类索引
       */
      boolean inheritSuperIndices() default false;
  
      /**
       * 联合主键
       */
      String[] primaryKeys() default {};
  
      /**
       * 外键数组
       */
      ForeignKey[] foreignKeys() default {};
  
      /**
       * 忽略字段数组
       */
      String[] ignoredColumns() default {};
  }
  
  ```

  >ROOM要求每个数据库序列化字段为public

- **@Index**

  ```java
  public @interface Index {
      /**
       * 指定索引的字段名称
       */
      String[] value();
  
      /**
       * 索引字段名称
       * index_${tableName}_${fieldName} 示例: index_Foo_bar_baz
       */
      String name() default "";
  
      /**
       * 唯一
       */
      boolean unique() default false;
  }
  
  ```

- **@Ignore**

  被该注解修饰的字段不会被加入表结构中

- **@Database**

  指定数据库初始化参数

  ```java
  public @interface Database {
      /**
       * 指定数据库初始化时创建数据表
       */
      Class<?>[] entities();
  
      /**
       * 指定数据库包含哪些视图
       */
      Class<?>[] views() default {};
  
      /**
       * 数据库当前版本号
       */
      int version();
  
      /**
       * 是否允许到处数据库概要, 默认为true. 要求配置gradle`room.schemaLocation`才有效
       */
      boolean exportSchema() default true;
  }
  
  ```

- **@PrimaryKey**

  每个数据库要求至少设置一个主键字段，即使只有一个字段的数据表

  ```java
  boolean autoGenerate() default false; // 主键自动增长。如果主键设置自动生成, 则要求必须为Long或者Int类型.
  ```

- **@ForeginKey**

  ```java
  public @interface ForeignKey {
    // 引用外键的表的实体
    Class entity();
    
    // 要引用的外键列
    String[] parentColumns();
    
    // 要关联的列
    String[] childColumns();
    
    // 当父类实体(关联的外键表)从数据库中删除时执行的操作
    @Action int onDelete() default NO_ACTION;
    
    // 当父类实体(关联的外键表)更新时执行的操作
    @Action int onUpdate() default NO_ACTION;
    
    // 在事务完成之前，是否应该推迟外键约束
    boolean deferred() default false;
    
    // 给onDelete，onUpdate定义的操作
    int NO_ACTION = 1; // 无动作
    int RESTRICT = 2; // 存在子健记录时禁止删除父键
    int SET_NULL = 3; // 子表删除会导致父键置为NULL 
    int SET_DEFAULT = 4; // 子表删除会导致父键置为默认值 
    int CASCADE = 5; // 父键删除时子表关键的记录全部删除
    
    @IntDef({NO_ACTION, RESTRICT, SET_NULL, SET_DEFAULT, CASCADE})
    @interface Action {
      }
  }
  
  ```

  示例

  ```java
  @Entity
  @ForeignKey(entity = Person::class, parentColumns = ["personId"],childColumns = ["bookId"], onDelete = ForeignKey.RESTRICT )
  data class Book(
      @PrimaryKey(autoGenerate = true)
      var bookId: Int = 0,
      @ColumnInfo(defaultValue = "12") var title: String = "冰火之歌"
  )
  ```

- **@ColumnInfo**

  修饰字段作为数据库中的列

  ```java
  public @interface ColumnInfo {
      /**
       * 列名, 默认为当前修饰的字段名
       */
      String name() default INHERIT_FIELD_NAME;
  
      /**
       * 指定当前字段属于Affinity类型, 一般不用
       */
      @SQLiteTypeAffinity int typeAffinity() default UNDEFINED;
    
    	// 以下类型
      int UNDEFINED = 1;
      int TEXT = 2;
      int INTEGER = 3;
      int REAL = 4; //
      int BLOB = 5;
  
      /**
       * 该字段为索引
       */
      boolean index() default false;
  
      /**
       * 指定构建数据表时排列 列的顺序
       */
      @Collate int collate() default UNSPECIFIED;
    
      int UNSPECIFIED = 1; // 默认值, 类似于BINARY
      int BINARY = 2; // 区分大小写
      int NOCASE = 3; // 不区分大小写
      int RTRIM = 4; // 区分大小写排列, 忽略尾部空格
  		@RequiresApi(21)
      int LOCALIZED = 5; // 按照当前系统默认的顺序
      @RequiresApi(21)
      int UNICODE = 6; // unicode顺序
  
      /**
       * 当前列的默认值, 这种默认值如果改变要求处理数据库迁移, 该参数支持SQL语句函数
       */
      String defaultValue() default VALUE_UNSPECIFIED;
  }
  
  ```

- **@Index**

  增加查询速度

  ```java
  // 需要被添加索引的字段名
  String[] value();
  
  // 索引名称
  String name() default "";
  
  // 表示字段在表中唯一不可重复
  boolean unique() default false;
  
  ```

- **@RawQuery**

  该注解用于修饰参数为`SupportSQLiteQuery`的`Dao`函数，用于原始查询（一般使用@Query）

  ```java
   interface RawDao {
      @RawQuery
      fun queryBook(query:SupportSQLiteQuery): Book
   }
  
  var book = rawDao.queryBook(SimpleSQLiteQuery("SELECT * FROM song ORDER BY name DESC"));
  // 或者返回可观察的对象Flow
  @RawQuery(observedEntities = [Book::class])
  fun query(query:SupportSQLiteQuery): Flow<MutableList<Book>>
  
  ```

- **@Embedded**

  如果数据表实体存在一个字段属于另外一个对象，该字段使用此注解即可让外部类包含该类所有的字段。外部类不要求为@Entry修饰的表结构

  ```java
  String prefix() default  ""; 
  // 前缀, 在数据表中的字段名都会被添加此前缀
  ```

- **@Relation**

  关联查询
  
  