# Wisdom World

## 后端

### 1. 项目搭建

#### 1.1 基本流程

> 不使用SpringBoot初始化器

1. 创建简单的Maven项目，作为本项目的**父项目**

2. 本项目的父项目继承SpringBoot父项目

3. 在父项目中进行**依赖管理**

   - 只需要写SpringBoot父项目中没有的依赖

     > 如果覆盖了依赖，还不写版本号，就会出错

   - 自己添加的依赖记得写上版本号

4. 加入插件，用于构建项目

   ```xml
   <build>
     <plugins>
       <plugin>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-maven-plugin</artifactId>
       </plugin>
     </plugins>
   </build>
   ```

5. 声明父项目特征

   - 打包方式设置为pom
   - 删除src文件夹

6. 创建子Module

7. 在子Module中添加需要用到的依赖

8. 在子Module中创建application.yml和主启动类

   > 这里关于`@MapperScan`的配置是另创建了一个类叫`MyBatisPlusConfig`，然后用`@Configuration`和`@MapperScan`修饰并指定mapper路径；
   >
   > 然后顺便把分页也配置了

9. 创建`WebMvcConfig`配置类，顺便把cors配置了

### 2. 首页Idea列表

#### 2.1 技巧

- 创建合适的VO(View Object)，减少Dao对象与业务逻辑的耦合
- 记得考虑<u>判断是否为空</u>的问题

### 3. 最热标签

#### 3.1 技巧

- groupby的使用

### 4. 统一异常处理

#### 4.1 技巧

- 异常处理切面可以直接使用`@RestControllerAdvice`注解

#### 4.2 注意事项

- 这里能处理的异常是controller相关的异常，
  其他地方的异常，比如过滤器中的异常，是管不到的

### 5. 最热idea与最新idea

#### 5.1 DAO层

- `LambdaQueryWrapper`的`select`方法
- `LambdaQueryWrapper`的`last`方法

### 6. idea归档

#### 6.1 DAO层

- 从时间戳中提取年月日的方法：

  - `from_unixtime(bigint, varchar)`函数

    > 第一个参数必须是<u>`bigint`类型的**时间戳**</u>，且**<u>单位是秒</u>**
    >
    > 第二个参数是MySQL日期格式字符串

- `group by`中就可以使用`select`中的别名了

- DO对象：从数据库中查询出来，但不是持久化的对象

  > 一般是在SQL层对数据库中的数据进行加工得到的

### 7. 登录认证功能

#### 7.1 思想总结

- 登录的步骤可以先写好

  ```java
  //1.检查参数是否合法
  //2.根据用户名和密码去user表中查询是否存在
  //3.如果不存在，则登录失败
  //4.如果存在，则使用JWT生成Token：userId expiration，返回给前端
  //5.再将Token存入Redis中一份，相当于双重保险，防止未登录者伪造Token
  ```

- 在Redis中也存储其实是**双重保险**

  > 但在项目中我其实没这么做

#### 7.2 技巧

- `RedisTemplate`的简单使用
- fastjson的`parseObject`方法
- `BeanUtils.copyProperties(source, target)`的使用

#### 7.3 注意点

- SpringBoot配置的Cors和SpringSecurity的拦截可能造成冲突，解决方法：

  在SecurityConfig的配置中加上`cors()`

  ```java
  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http
      //关闭csrf（后面具体讲）
      .csrf().disable()
      //不通过Session获取SecurityContext
      //前后端分离项目不要用Session了
      .sessionManagement()
      .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
      .and()
      .cors()
      .and()
      .authorizeRequests()
      // 对于登录、注册接口，只允许匿名访问，也就是认证之后(携带了token)就访问不了了
      .antMatchers("/login").anonymous()
      .antMatchers("/register").anonymous()
      // 部分功能需要鉴权认证
      .antMatchers("/users/currentUser").authenticated()
      .antMatchers("/logout").authenticated()
      .antMatchers("/comments/create/change").authenticated()
      .antMatchers("/ideas/publish").authenticated();
    //其他接口任意访问
    //添加过滤器
    http.addFilterBefore(jwtAuthenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
    //配置异常处理器
    http.exceptionHandling()
      .authenticationEntryPoint(authenticationEntryPoint);
  }
  ```

- SpringSecurity会将logout自动重定向到login?logout

### 8. 注册

#### 8.1 注意

- 记得添加事务，如果注册过程中有异常，数据库里不能添加数据

### 9. idea详情

#### 9.1 心得

- `Mapper`需要用的时候肯定要创建，没得说
- `Service`在调用`Mapper`的时候，要考虑到二者之间有无业务逻辑上的联系，不要在`Service`中随意调用各种`Mapper`（不然的话直接写一个service不得了），有业务逻辑关联的时候再直接调，不然的话最好直接新建一个`Service`，然后在一个`Service`中调用另一个`Service`

#### 9.2 技巧

- 访问文章详情接口代表文章被阅读了一次，不仅需要读取文章主要内容，还需要增加文章阅读量，即更新表，但更新表的操作需要加锁，耗费时间，如果与访问文章详情放在同一接口下，会增加不必要的接口耗时，而且更新操作涉及事务，还可能影响到访问文章，所以选择多线程操作

- 多线程(线程池)的操作可以新建一个`ThreadService`来负责，然后新建一个`ThreadPoolConfig`配置类

  ```java
  @Configuration
  @EnableAsync
  public class ThreadPoolConfig {
      @Bean
      public Executor asyncServiceExecutor() { //与注解属性值匹配
          ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
          // 设置核心线程数
          executor.setCorePoolSize(5);
          // 设置最大线程数
          executor.setMaxPoolSize(20);
          //配置队列大小
          executor.setQueueCapacity(Integer.MAX_VALUE);
          // 设置线程活跃时间（秒）
          executor.setKeepAliveSeconds(60);
          // 设置默认线程名称
          executor.setThreadNamePrefix("wisdom world thread service");
          // 等待所有任务结束后再关闭线程池
          executor.setWaitForTasksToCompleteOnShutdown(true);
          //执行初始化
          executor.initialize();
          return executor;
      }
  }
  ```

  ```java
  public interface ThreadService {
  
  
      /**
       * idea阅读量++
       * @param idea
       */
      @Async("asyncServiceExecutor")
      void increaseIdeaViewCounts(Idea idea);
  }
  ```

  ```java
  @Service
  public class ThreadServiceImpl implements ThreadService {
      @Autowired
      IdeaService ideaService;
  
      @Override
      @Async("asyncServiceExecutor") //与config中的方法名匹配
      public void increaseIdeaViewCounts(Idea idea) {
          idea.setViewCounts(idea.getViewCounts() + 1);
          ideaService.updateById(idea);
      }
  }
  ```

- 使用MyBatisPlus调用update时，为了保证最小更新，可以让对象的属性为`null`

  > 一般来说是新建一个对象

  > 所以说和数据库映射的一定要用包装类型

- update时，尽量保证乐观锁

### 10. 评论功能

#### 10.1 技巧

- 为了防止`Long`型id过长，前端接收后端传来的id时发生精度损失，可以这样：

  - 在bean copy时将VO中的id改成`String`类型

  - 在VO中的`Long`型id字段上，加注解：

    ```java
    @JsonSerialize(using = ToStringSerializer.class)
    private Long id;
    ```

### 11. 写文章

#### 11.1 相关知识

- 在使用实体类对象插入后，MybatisPlus会**将生成的id回写**到实体类中
- 如果接收前端JSON参数的对象中，有一个字段是对象引用，则要求JSON中也有嵌套对象，否则可能导致JSON parse error

#### 11.2 技巧

- 要返回JSON时如果不想创建对象，可以利用Map

### 12. 图片上传

#### 12.1 注意点

- 上传的图片名要具有唯一性，所以要使用UUID



### 13. 归档查询

#### 13.1 DAO层技巧

- 当可能筛选条件多且不一定都要生效时，可以考虑使用**动态SQL**
- 自定义的查询怎么和分页结合？需要自己在SQL语句中写吗？不需要
  - 让`Mapper`方法返回`Page<Pojo>`，且方法的参数中传入`Page<Pojo>`类型参数即可
- `limit`是有可能改变结果顺序的



### 14 idea树查询

#### 14.1 需求分析

- 查出父idea
- 查出查出兄弟idea
- 查出子idea

或许可以返回一个ideaTreeVo，包含上述三个idea的相关信息



### 15 用户idea管理

#### 15.1 需求分析

- 用户的每个idea作为核心，树中包含其所有祖先，和所有后代
- 如果后代中包括用户的另一个idea，则合并







## 前端

### 1. 回到顶部组件

Vue提供了`transition`的封装组件，在下列情形中，可以给**任何元素和组件**添加 进入/离开 过渡

- 条件渲染、条件展示
- 动态组件
- 组件根节点

其实ElementUI中也有现成的组件

### 2. 首页-idea列表

#### 2.1 组件使用

- 布局组件`el-container`
- element ui 提供的分页加载

#### 2.2 技巧

- `v-bind`的进阶使用

  `v-bind={xxx, yyy, zzz}`相当于`:xxx="xxx" :yyy="yyy" :zzz="zzz"`

- `window.setTimeout(function, delay)`延迟异步任务

### 3. 统一请求处理

#### 3.1 技巧

- axios可以有catch，还可以有finnally

### 4. 主页完善

#### 4.1 技巧

- 避免router-link错误匹配上级router-view？
  - 在router/index.js中定义好**父子关系**

### 5. 请求响应拦截器

> 自己想加的功能，待实现



### 6. Idea详情

#### 6.1 注意点

- 路由的匹配会遵循书写的先后顺序，有冲突时要注意



### 7. 注册与登录

#### 7.1 技巧

- 前端如何存储登录状态？
  - 存储token
  - **集中式**状态 (**数据**) 管理：Vuex
- 什么时候需要自己用`promise`？想要实现异步等待，可以将异步操作封装到`promise`中，然后在`then`中进行等待后的操作
- 如果某个数据依赖另一个数据（比如vuex中的state），想让被依赖数据的改变影响到它，可以使用计算属性

### 8. Idea归档

#### 8.1 技巧

- 如果想<u>让子组件随着某个数据的变化而重新渲染</u>，可以考虑给子组件加一个key属性，让key属性也随着数据变化

### 9. 写Idea

#### 9.1 技巧

- query参数如果不是必须的，可以加一个问号，例如：`/write/:id?`





## 新增的功能

### 0 发布和编辑idea





### 1 查看Idea树

#### 1.1 后端

- DAO
  - 表加字段，父id字段
  - 实体类加属性，父id
- Service
  - 请求获取这个idea所在的idea树
    1. 找到祖宗
    2. 祖宗递归找后代
  - 返回值包含所有idea的：
    - id
    - title

> 不要忘记JSON序列化

#### 1.2 前端



### 2 延伸idea

#### 2.1 后端

待完善的功能：

- 添加idea时的parentId处理
- 延伸idea时parentId处理
- 编辑idea时parentId处理

#### 2.2 前端

- 未登录而去延伸的处理

- 延伸的显示
  - 可以考虑使用query参数

- Vue**插值语法不换行**的问题：设置css属性

  ```vue
  {
  	white-space: pre-wrap;
  }
  ```

  



### 3 收藏idea





### 4 展示属于自己的idea森林





### 5 收藏的idea碰撞





### 6 阅读量刷新bug







### 7 收藏功能

- 新建一个收藏表，插入与删除操作较多
- MyBatisPlus时间的自动填充，要注意类型相同
- 加一个联合索引防止重复收藏



### 8 idea的排序、热门的评估





### 9 根据不同的时间显示不同的热门





### 10 UI界面错乱的问题





### 11 路由匹配相同但组件不刷新

- 在对应的`router-view`中加入`key`即可

```vue
<router-view class="me-container" :key="this.$route.path"/>
```



### 12 标签搜索功能







### 13 idea审核

- 可以利用数据库的逻辑删除字段，或者新增**未审核字段**，可别新建一张大表
