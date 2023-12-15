---
title: 'Jetpack 使用总结 - Room'
author: litao
date: 2021-03-19 17:13:00 +0800
categories: [Android,Jetpack]
tags: [Android,Jetpack]





---

## 一、数据库（database）



### 1. 数据库类的定义及注意点

```kotlin
@Database(
  entities = [User::class], 
  version = 3,
  exportSchema = true,
  views = [UserShadow::class],
  autoMigrations = [
        AutoMigration(from = 1, to = 2),
        AutoMigration(from = 2, to = 3, spec = AutoMigration2_3::class),
    ]
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
  
  
  
  	internal class AutoMigration2_3 : AutoMigrationSpec {
        override fun onPostMigrate(db: SupportSQLiteDatabase) {
            super.onPostMigrate(db)
   
        }
    }
}

```

- `entities`: 指定数据库的实体类，使用 `@Entity`所标记的类
- `autoMigrations`: 指定某些版本是否允许自动升级，如果满足条件可以进行自动升级，但需要保证`exportSchema =true ` 此处默认值就是true，通常不要去更改，如果自动迁移后我们还想做一些其他事情可以指定一个`spec`在其中实现我们额外的逻辑。
- views: 指定一张虚拟表，使用`@DatabaseView`进行注解，可使用当前已有的一张表或多张表组合起来，提供一个指定结构的查询

### 2. 数据库实例的创建

```kotlin
        Room.databaseBuilder(appContext, AppDatabase::class.java, DATABASE_NAME)
                .addCallback(object : Callback() {
                    override fun onCreate(db: SupportSQLiteDatabase) {
                        super.onCreate(db)
                       
                    }

                    override fun onOpen(db: SupportSQLiteDatabase) {
                        super.onOpen(db)                
                    }
                })
                .addMigrations(
                    MIGRATION_1_2,
                    MIGRATION_2_3
                )
                .allowMainThreadQueries()
                .build()
```

设置项中基本覆盖到了日常使用的所有场景，需注意几点

- `allowMainThreadQueries `:并不是一劳永逸的，应该尽量避免设置，在初期就做限制，因为早期同步异步并不能明显的看到性能问题，后期逐步积累，等出现性能问题时可能相关逻辑已经累计到一个难以轻易修改的地步了。
- `setQueryExecutor`、`setTransactionExecutor`：设置后可能会对数据库操作有所提升，但大部分情况都是微乎其微的，应该按需添加，如果没有密集的操作、没有出现明显的性能问题并不适合去添加。使用是也要充分考虑异步导致的各种问题，很容易发生死锁。

- `addMigrations`：数据库升级时先看是否满足自动迁移条件，满足的话直接在头部声明为自动迁移即可。

## 二、数据实体(entities)

### 1. 实体的定义及注意点

与大多数据库一致，我们可以指定主键，指定符合主键，指定外键，设置索引等操作。

```kotlin
@Fts4
@Entity(primaryKeys = ["firstName", "lastName"])
data class User(
    @PrimaryKey val id: Int,

    val firstName: String?,
    val lastName: String?,
  	@Ignore val picture: Bitmap? //某些字段不需要加入到数据库中时使用@Ignore进行标记
  	
)
```

注意点

- `@Ignore`；如果实体存在父类，又希望忽略父类中的字段，那么可以在`@Entity`注解中加入`ignoredColumns`来指明需要忽略的字段
- `indices`：我们可以为实体添加单列索引、多列索引、唯一索引等，但需要结合查询模式和频率进行选择，因为过度使用可能会起到副作用，因为插入、更新、删除时都需要更新索引。建立索引后引进行测试确保对查询性能确实有所改善。索引的创建有些极端情况可能会影响到应用到启动，这种情况下也可以放到后台异步去创建索引

- `@Fts4`: 指定全文搜索时可以直接在当前表操作，也可以创建虚拟表与实体表进行关联。尽量替换掉之前使用like来进行全文查找的逻辑。

### 2. 复杂实体的引用

有时我们的实体内可能不光都是一些room支持的基本类型，可能包含一些其他类型

```kotlin
//Data类型系统内部是无法直接进行序列化的
@Entity
data class User(private val birthday: Date?)


//我们可以创建一个类型转换器，使用@TypeConverter注解来告诉room这种类型改如何序列化，
class Converters {
  @TypeConverter
  fun fromTimestamp(value: Long?): Date? {
    return value?.let { Date(it) }
  }

  @TypeConverter
  fun dateToTimestamp(date: Date?): Long? {
    return date?.time?.toLong()
  }
}

//然后使用@@TypeConverters将转换器添加到room中
//这里需要注意，@TypeConverters也可以作用于@Entity,@Dao上，来进一步控制作用域
@Database(entities = [User::class], version = 1)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
  abstract fun userDao(): UserDao
}


```





## 三、数据访问对象(DAOs)

```kotlin
@Dao
interface UserDao {
    @Insert
    fun insertAll(vararg users: User)

    @Delete
    fun delete(user: User)

    @Query("SELECT * FROM user")
    fun getAll(): List<User>
}
```

### 1.  多重映射

```kotlin
//2.4版本以上直接编写多重映射的sql
@Query(
    "SELECT * FROM user" +
    "JOIN book ON user.id = book.user_id"
)
fun loadUserAndBookNames(): Map<User, List<Book>>

//使用group by 子句
@Query(
    "SELECT * FROM user" +
    "JOIN book ON user.id = book.user_id" +
    "GROUP BY user.name WHERE COUNT(book.id) >= 3"
)
fun loadUserAndBookNames(): Map<User, List<Book>>

//如果不需要返回整个对象，可以使用@MapInfo注解
@MapInfo(keyColumn = "userName", valueColumn = "bookName")
@Query(
    "SELECT user.name AS username, book.name AS bookname FROM user" +
    "JOIN book ON user.id = book.user_id"
)
fun loadUserAndBookNames(): Map<String, List<String>>
```

### 2.  定义实体关系型查询

创建嵌套对象

```kotlin
data class Address(
    val street: String?,
    val state: String?,
    val city: String?,
    @ColumnInfo(name = "post_code") val postCode: Int
)

@Entity
data class User(
    @PrimaryKey val id: Int,
    val firstName: String?,
  	//存在嵌套对象时，可能存在命名冲突，可以使用prefix添加前缀来解决
    @Embedded val address: Address?
)
```

一对一关系

```kotlin
//需建立关系的实体
@Entity
data class User(
    @PrimaryKey val userId: Long,
    val name: String,
    val age: Int
)

@Entity
data class Library(
    @PrimaryKey val libraryId: Long,
    val userOwnerId: Long
)

//创建中间数据类建立关系，子实体添加@Relation注解，写入父子（parentColumn，entityColumn）建立关系的字段名称
data class UserAndLibrary(
    @Embedded val user: User,
    @Relation(
         parentColumn = "userId",
         entityColumn = "userOwnerId"
    )
    val library: Library
)

//进行查询，底层是两次操作，添加@Transaction保证原子性
@Transaction
@Query("SELECT * FROM User")
fun getUsersAndLibraries(): List<UserAndLibrary>
```

一对多关系

```kotlin
//与一对一本质上是一样的，只是返回多个对象
data class UserAndLibrary(
    @Embedded val user: User,
    @Relation(
         parentColumn = "userId",
         entityColumn = "userOwnerId"
    )
    val library: List<Library>
)
```

多对多关系

```kotlin
// 多对多关系中子实体一般不保存父实体引用，所以需要创建一个关联实体
@Entity
data class Playlist(
    @PrimaryKey val playlistId: Long,
    val playlistName: String
)

@Entity
data class Song(
    @PrimaryKey val songId: Long,
    val songName: String,
    val artist: String
)

@Entity(primaryKeys = ["playlistId", "songId"])
data class PlaylistSongCrossRef(
    val playlistId: Long,
    val songId: Long
)


//这种多对多的关系查询取决于我们想要查询怎么样的关系
data class PlaylistWithSongs(
    @Embedded val playlist: Playlist,
    @Relation(
         parentColumn = "playlistId",
         entityColumn = "songId",
         associateBy = Junction(PlaylistSongCrossRef::class)
    )
    val songs: List<Song>
)

data class SongWithPlaylists(
    @Embedded val song: Song,
    @Relation(
         parentColumn = "songId",
         entityColumn = "playlistId",
         associateBy = Junction(PlaylistSongCrossRef::class)
    )
    val playlists: List<Playlist>
)

//查询与其他关系查询一致
@Transaction
@Query("SELECT * FROM Playlist")
fun getPlaylistsWithSongs(): List<PlaylistWithSongs>

@Transaction
@Query("SELECT * FROM Song")
fun getSongsWithPlaylists(): List<SongWithPlaylists>

```



### 3. 相关注意点


- `Dao`可以定义为接口或抽象类

- `Dao`的行为可以使用注解直接指定（通常用在无条件的CRUD，但需要注意冲突行为的处理在`onConflict`中指定），也可以直接编写sql语句

- 如果我们只想查询一部分，可以新建子类，直接返回指定字段

  ```kotlin
  data class NameTuple(
      @ColumnInfo(name = "first_name") val firstName: String?,
      @ColumnInfo(name = "last_name") val lastName: String?
  )
  
  @Query("SELECT first_name, last_name FROM user")
  fun loadFullName(): List<NameTuple>
  ```

- 在建立查询关系是尽量考虑关系嵌套不要太深，处理大量数据时可能会带来明显的性能问题，通常情况尽量控制在1层，不要为了方便随便嵌套。

- 通常我们结合Kotlin的 Coroutines  或 返回可观察到Flow能够更高效并清晰的处理逻辑。

- 返回数据为Flow时可以在收集数据时指定`distinctUntilChanged`运算符，避免room在查询时可能发出的一些重复值，达到一个类似`StateFlow`的效果。

  
