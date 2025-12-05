# MyBatis-Plus 面试复习指南

> 基于 restaurant-management-system 项目实战分析
>
> **项目发现**：当前项目使用的是原生 MyBatis + PageHelper，本指南将展示如何使用 MyBatis-Plus 改进现有代码

---

## 目录
1. [MyBatis-Plus 核心概念](#1-mybatis-plus-核心概念)
2. [与原生 MyBatis 对比](#2-与原生-mybatis-对比)
3. [核心注解详解](#3-核心注解详解)
4. [CRUD 接口](#4-crud-接口)
5. [条件构造器](#5-条件构造器)
6. [分页插件](#6-分页插件)
7. [代码生成器](#7-代码生成器)
8. [高级特性](#8-高级特性)
9. [性能优化](#9-性能优化)
10. [常见面试题](#10-常见面试题)

---

## 1. MyBatis-Plus 核心概念

### 1.1 什么是 MyBatis-Plus？

MyBatis-Plus（简称 MP）是一个 MyBatis 的增强工具，在 MyBatis 的基础上只做增强不做改变，为简化开发、提高效率而生。

**三大特性**：
- **无侵入**：只做增强不做改变，引入它不会对现有工程产生影响
- **损耗小**：启动即会自动注入基本 CRUD，性能基本无损耗，直接面向对象操作
- **强大的 CRUD 操作**：内置通用 Mapper、通用 Service，仅仅通过少量配置即可实现单表大部分 CRUD 操作

### 1.2 为什么要用 MyBatis-Plus？

**原生 MyBatis 的痛点**（在你的项目中）：
```java
// EmployeeMapper.java - 每个简单的CRUD都需要写SQL
@Select("select * from employee where username = #{username}")
Employee getByUsername(String username);

@Insert("insert into employee(name, username, password, phone, sex, id_number, create_time, update_time, create_user, update_user,status) " +
        "values (#{name}, #{username}, #{password}, #{phone}, #{sex}, #{idNumber}, #{createTime}, #{updateTime}, #{createUser}, #{updateUser}, #{status})")
@AutoFill(value = OperationType.INSERT)
void insert(Employee employee);

@Select("select * from employee where id = #{id}")
Employee getById(Long id);
```

**使用 MyBatis-Plus 后**：
```java
// EmployeeMapper.java - 继承BaseMapper即可，无需写SQL
public interface EmployeeMapper extends BaseMapper<Employee> {
    // 自动拥有 selectById、insert、updateById、deleteById 等方法
    // 只需要写业务特定的方法
}

// Service层使用
Employee employee = employeeMapper.selectById(id);
employeeMapper.insert(employee);
employeeMapper.updateById(employee);
```

---

## 2. 与原生 MyBatis 对比

### 2.1 依赖配置对比

#### 你的项目（原生 MyBatis）
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>

<!-- 分页需要额外引入PageHelper -->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.3.0</version>
</dependency>
```

#### MyBatis-Plus（一个依赖搞定）
```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
```

### 2.2 实体类配置对比

#### 你的项目（原生 MyBatis）
```java
// Employee.java - 无特殊注解
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Employee implements Serializable {
    private Long id;
    private String username;
    private String name;
    private String password;
    private String phone;
    private String sex;
    private String idNumber;
    private Integer status;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
    private Long createUser;
    private Long updateUser;
}
```

#### MyBatis-Plus 版本
```java
@Data
@TableName("employee")  // 指定表名（如果类名与表名一致可省略）
public class Employee implements Serializable {

    @TableId(value = "id", type = IdType.AUTO)  // 主键策略
    private Long id;

    private String username;
    private String name;
    private String password;
    private String phone;
    private String sex;

    @TableField("id_number")  // 字段映射（驼峰可省略）
    private String idNumber;

    private Integer status;

    @TableField(fill = FieldFill.INSERT)  // 自动填充
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableField(fill = FieldFill.INSERT)
    private Long createUser;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Long updateUser;
}
```

### 2.3 Mapper 接口对比

#### 你的项目（原生 MyBatis）
```java
@Mapper
public interface EmployeeMapper {

    @Select("select * from employee where username = #{username}")
    Employee getByUsername(String username);

    @Insert("insert into employee(...) values (...)")
    @AutoFill(value = OperationType.INSERT)
    void insert(Employee employee);

    Page<Employee> pageQuery(EmployeePageQueryDTO employeePageQueryDTO);

    @AutoFill(value = OperationType.UPDATE)
    void update(Employee employee);

    @Select("select * from employee where id = #{id}")
    Employee getById(Long id);
}
```

#### MyBatis-Plus 版本
```java
@Mapper
public interface EmployeeMapper extends BaseMapper<Employee> {
    // 已自动拥有以下方法（无需编写）：
    // - insert(Employee entity)
    // - deleteById(Serializable id)
    // - updateById(Employee entity)
    // - selectById(Serializable id)
    // - selectList(Wrapper<Employee> wrapper)
    // - selectPage(Page<Employee> page, Wrapper<Employee> wrapper)
    // ... 还有更多

    // 只需要写复杂的自定义查询
    @Select("select * from employee where username = #{username}")
    Employee getByUsername(String username);
}
```

### 2.4 分页功能对比

#### 你的项目（PageHelper）
```java
// CategoryServiceImpl.java
public PageResult pageQuery(CategoryPageQueryDTO categoryPageQueryDTO) {
    // 使用PageHelper分页
    PageHelper.startPage(categoryPageQueryDTO.getPage(),
                        categoryPageQueryDTO.getPageSize());

    // 下一条SQL自动加入LIMIT
    Page<Category> page = categoryMapper.pageQuery(categoryPageQueryDTO);

    return new PageResult(page.getTotal(), page.getResult());
}
```

```xml
<!-- CateGoryMapper.xml -->
<select id="pageQuery" resultType="com.sky.entity.Category">
    select * from category
    <where>
        <if test="name != null and name != ''">
            and name like concat('%',#{name},'%')
        </if>
        <if test="type != null">
            and type = #{type}
        </if>
    </where>
    order by sort asc, create_time desc
</select>
```

#### MyBatis-Plus 分页
```java
// CategoryServiceImpl.java
public PageResult pageQuery(CategoryPageQueryDTO dto) {
    // 创建分页对象
    Page<Category> page = new Page<>(dto.getPage(), dto.getPageSize());

    // 构造查询条件
    LambdaQueryWrapper<Category> wrapper = new LambdaQueryWrapper<>();
    wrapper.like(StringUtils.isNotBlank(dto.getName()),
                Category::getName, dto.getName())
           .eq(dto.getType() != null, Category::getType, dto.getType())
           .orderByAsc(Category::getSort)
           .orderByDesc(Category::getCreateTime);

    // 执行分页查询
    categoryMapper.selectPage(page, wrapper);

    return new PageResult(page.getTotal(), page.getRecords());
}
```

**无需 XML 配置！**

### 2.5 动态 SQL 对比

#### 你的项目（MyBatis XML）
```xml
<!-- DishMapper.xml -->
<select id="pageQuery" resultType="com.sky.vo.DishVO">
    select d.*, c.name as categoryName
    from dish d
    left outer join category c on d.category_id = c.id
    <where>
        <if test="name != null">
            and d.name like concat('%',#{name},'%')
        </if>
        <if test="categoryId != null">
            and d.category_id = #{categoryId}
        </if>
        <if test="status != null">
            and d.status = #{status}
        </if>
    </where>
    order by d.create_time desc
</select>
```

#### MyBatis-Plus（条件构造器）
```java
// Java代码实现，无需XML
LambdaQueryWrapper<Dish> wrapper = new LambdaQueryWrapper<>();
wrapper.like(StringUtils.isNotBlank(dto.getName()),
            Dish::getName, dto.getName())
       .eq(dto.getCategoryId() != null,
            Dish::getCategoryId, dto.getCategoryId())
       .eq(dto.getStatus() != null,
            Dish::getStatus, dto.getStatus())
       .orderByDesc(Dish::getCreateTime);

// 关联查询需要自定义（或使用selectJoinPage等扩展方法）
Page<DishVO> page = new Page<>(pageNum, pageSize);
List<DishVO> list = dishMapper.selectDishVOPage(page, wrapper);
```

### 2.6 批量操作对比

#### 你的项目（XML foreach）
```xml
<!-- DishFlavorMapper.xml -->
<insert id="insertBatch">
    insert into dish_flavor (dish_id, name, value)
    values
    <foreach collection="dishFlavors" item="dishFlavor" separator=",">
        (#{dishFlavor.dishId}, #{dishFlavor.name}, #{dishFlavor.value})
    </foreach>
</insert>

<delete id="batchDeleteById">
    delete from dish where id in
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</delete>
```

#### MyBatis-Plus
```java
// 批量插入
dishFlavorMapper.insertBatch(dishFlavors);  // 继承BaseMapper自带

// 批量删除
dishMapper.deleteBatchIds(ids);  // 自带方法

// 或使用Service层的批量方法
dishService.saveBatch(dishList);  // 批量保存
dishService.removeByIds(ids);     // 批量删除
```

---

## 3. 核心注解详解

### 3.1 @TableName

**作用**：指定实体类对应的数据库表名

```java
@TableName("employee")  // 如果类名与表名一致，可省略
public class Employee { }

@TableName(value = "t_user", schema = "mybatis")
public class User { }

// 排除非表字段
@TableName(excludeProperty = {"field1", "field2"})
public class User { }
```

**面试要点**：
- 默认使用类名转下划线作为表名（UserInfo → user_info）
- 可通过全局配置统一添加表名前缀

### 3.2 @TableId

**作用**：标识主键字段及生成策略

```java
public class Employee {
    @TableId(value = "id", type = IdType.AUTO)
    private Long id;
}
```

**主键策略（IdType）**：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| `AUTO` | 数据库自增 | MySQL、PostgreSQL |
| `NONE` | 无状态（跟随全局） | 由全局配置决定 |
| `INPUT` | 手动输入 | 自己生成主键值 |
| `ASSIGN_ID` | 雪花算法（默认） | 分布式系统 |
| `ASSIGN_UUID` | UUID | 需要全局唯一字符串ID |

**面试高频问题**：
- **Q: 雪花算法的优缺点？**
  - 优点：趋势递增、高性能、分布式唯一
  - 缺点：依赖机器时间、ID较长

- **Q: 在你的项目中应该用哪种策略？**
  - 单体应用：`AUTO`（数据库自增）
  - 分布式系统：`ASSIGN_ID`（雪花算法）

### 3.3 @TableField

**作用**：配置字段属性

```java
public class Employee {
    // 字段映射（驼峰自动映射可省略）
    @TableField("id_number")
    private String idNumber;

    // 自动填充
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    // 排除非表字段
    @TableField(exist = false)
    private String tempField;

    // 查询时不返回该字段
    @TableField(select = false)
    private String password;

    // 条件构造器忽略该字段
    @TableField(condition = SqlCondition.LIKE)
    private String name;
}
```

**自动填充策略（FieldFill）**：
- `DEFAULT`：默认不处理
- `INSERT`：插入时填充
- `UPDATE`：更新时填充
- `INSERT_UPDATE`：插入和更新时填充

### 3.4 自动填充实现

#### 你的项目（自定义 AOP 切面）
```java
// AutoFillAspect.java
@Aspect
@Component
public class AutoFillAspect {
    @Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")
    public void autoFillPointCut() {}

    @Before("autoFillPointCut()")
    public void autoFill(JoinPoint joinPoint) throws Exception {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        AutoFill autoFill = signature.getMethod().getAnnotation(AutoFill.class);

        Object[] args = joinPoint.getArgs();
        Object entity = args[0];

        LocalDateTime now = LocalDateTime.now();
        Long currentId = BaseContext.getCurrentId();

        if (autoFill.value() == OperationType.INSERT) {
            // 使用反射设置createTime、createUser、updateTime、updateUser
        } else if (autoFill.value() == OperationType.UPDATE) {
            // 使用反射设置updateTime、updateUser
        }
    }
}
```

#### MyBatis-Plus（实现 MetaObjectHandler）
```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // 插入时自动填充
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "createUser", Long.class, getCurrentUserId());
        this.strictInsertFill(metaObject, "updateUser", Long.class, getCurrentUserId());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        // 更新时自动填充
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictUpdateFill(metaObject, "updateUser", Long.class, getCurrentUserId());
    }

    private Long getCurrentUserId() {
        // 从上下文获取当前用户ID（类似你项目中的BaseContext）
        return BaseContext.getCurrentId();
    }
}
```

**关键区别**：
- 你的项目：自定义注解 + AOP 切面 + 反射
- MyBatis-Plus：实现接口 + 注解配置，更简洁

### 3.5 @TableLogic

**作用**：逻辑删除标记

```java
public class Employee {
    @TableLogic(value = "0", delval = "1")  // 0=未删除，1=已删除
    private Integer deleted;
}
```

**效果**：
```java
// 删除操作变为更新
employeeMapper.deleteById(1L);
// 实际执行：UPDATE employee SET deleted=1 WHERE id=1 AND deleted=0

// 查询自动过滤已删除数据
employeeMapper.selectList(null);
// 实际执行：SELECT * FROM employee WHERE deleted=0
```

**你的项目对比**：
```java
// EmployeeMapper.java - 需要手动处理
@Update("update employee set status = #{status} where id = #{id}")
void updateStatus(Long id, Integer status);
```

### 3.6 @Version

**作用**：乐观锁标记

```java
public class Employee {
    @Version
    private Integer version;
}
```

**配置拦截器**：
```java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        return interceptor;
    }
}
```

**使用效果**：
```java
// 查询时自动带出version
Employee employee = employeeMapper.selectById(1L); // version=1

// 更新时自动校验version
employee.setName("新名字");
employeeMapper.updateById(employee);
// SQL: UPDATE employee SET name='新名字', version=2 WHERE id=1 AND version=1

// 如果version不匹配，更新失败
```

---

## 4. CRUD 接口

### 4.1 BaseMapper 接口

**继承 BaseMapper 后自动拥有的方法**：

```java
public interface EmployeeMapper extends BaseMapper<Employee> {
    // 已自动拥有以下方法，无需编写
}
```

#### 插入操作
```java
// 插入一条记录
int insert(Employee entity);
```

#### 删除操作
```java
// 根据ID删除
int deleteById(Serializable id);

// 根据条件删除
int delete(Wrapper<Employee> wrapper);

// 批量删除
int deleteBatchIds(Collection<? extends Serializable> idList);

// 根据Map条件删除
int deleteByMap(Map<String, Object> columnMap);
```

#### 更新操作
```java
// 根据ID更新（null字段不更新）
int updateById(Employee entity);

// 根据条件更新
int update(Employee entity, Wrapper<Employee> updateWrapper);
```

#### 查询操作
```java
// 根据ID查询
Employee selectById(Serializable id);

// 根据ID批量查询
List<Employee> selectBatchIds(Collection<? extends Serializable> idList);

// 根据Map条件查询
List<Employee> selectByMap(Map<String, Object> columnMap);

// 根据条件查询一条记录
Employee selectOne(Wrapper<Employee> queryWrapper);

// 根据条件查询总记录数
Long selectCount(Wrapper<Employee> queryWrapper);

// 根据条件查询所有记录
List<Employee> selectList(Wrapper<Employee> queryWrapper);

// 根据条件查询所有记录（返回Map）
List<Map<String, Object>> selectMaps(Wrapper<Employee> queryWrapper);

// 根据条件分页查询
IPage<Employee> selectPage(IPage<Employee> page, Wrapper<Employee> queryWrapper);
```

### 4.2 IService 接口

**Service 层继承 IService**：

```java
public interface EmployeeService extends IService<Employee> {
    // 已自动拥有大量方法
}

@Service
public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee>
    implements EmployeeService {
    // 实现类继承ServiceImpl
}
```

#### 保存操作
```java
// 保存单条
boolean save(Employee entity);

// 批量保存
boolean saveBatch(Collection<Employee> entityList);
boolean saveBatch(Collection<Employee> entityList, int batchSize);

// 保存或更新（根据ID是否存在）
boolean saveOrUpdate(Employee entity);
boolean saveOrUpdateBatch(Collection<Employee> entityList);
```

#### 删除操作
```java
// 根据ID删除
boolean removeById(Serializable id);

// 根据条件删除
boolean remove(Wrapper<Employee> queryWrapper);

// 批量删除
boolean removeByIds(Collection<? extends Serializable> idList);

// 根据Map删除
boolean removeByMap(Map<String, Object> columnMap);
```

#### 更新操作
```java
// 根据ID更新
boolean updateById(Employee entity);

// 根据条件更新
boolean update(Wrapper<Employee> updateWrapper);
boolean update(Employee entity, Wrapper<Employee> updateWrapper);

// 批量更新
boolean updateBatchById(Collection<Employee> entityList);
```

#### 查询操作
```java
// 根据ID查询
Employee getById(Serializable id);

// 根据条件查询一条
Employee getOne(Wrapper<Employee> queryWrapper);

// 查询所有
List<Employee> list();
List<Employee> list(Wrapper<Employee> queryWrapper);

// 查询总数
long count();
long count(Wrapper<Employee> queryWrapper);

// 分页查询
IPage<Employee> page(IPage<Employee> page);
IPage<Employee> page(IPage<Employee> page, Wrapper<Employee> queryWrapper);
```

### 4.3 与你的项目对比

#### 你的项目（原生 MyBatis）
```java
// EmployeeServiceImpl.java
@Service
public class EmployeeServiceImpl implements EmployeeService {
    @Autowired
    private EmployeeMapper employeeMapper;

    public void save(EmployeeDTO employeeDTO) {
        Employee employee = new Employee();
        BeanUtils.copyProperties(employeeDTO, employee);
        employee.setStatus(StatusConstant.ENABLE);
        employee.setPassword(DigestUtils.md5DigestAsHex(
            PasswordConstant.DEFAULT_PASSWORD.getBytes()));
        employeeMapper.insert(employee);
    }

    public Employee getById(Long id) {
        return employeeMapper.getById(id);
    }

    public void update(EmployeeDTO employeeDTO) {
        Employee employee = new Employee();
        BeanUtils.copyProperties(employeeDTO, employee);
        employeeMapper.update(employee);
    }
}
```

#### MyBatis-Plus 版本
```java
@Service
public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee>
    implements EmployeeService {

    public void save(EmployeeDTO employeeDTO) {
        Employee employee = new Employee();
        BeanUtils.copyProperties(employeeDTO, employee);
        employee.setStatus(StatusConstant.ENABLE);
        employee.setPassword(DigestUtils.md5DigestAsHex(
            PasswordConstant.DEFAULT_PASSWORD.getBytes()));
        this.save(employee);  // 继承自ServiceImpl
    }

    public Employee getById(Long id) {
        return this.getById(id);  // 继承自ServiceImpl
    }

    public void update(EmployeeDTO employeeDTO) {
        Employee employee = new Employee();
        BeanUtils.copyProperties(employeeDTO, employee);
        this.updateById(employee);  // 继承自ServiceImpl
    }
}
```

**优势**：
- 无需注入 Mapper
- 自动拥有批量操作方法
- 链式调用更优雅

---

## 5. 条件构造器

### 5.1 Wrapper 继承体系

```
AbstractWrapper
    ├── QueryWrapper<T>         // 查询条件构造器
    ├── UpdateWrapper<T>        // 更新条件构造器
    ├── LambdaQueryWrapper<T>   // Lambda查询（推荐）
    └── LambdaUpdateWrapper<T>  // Lambda更新（推荐）
```

### 5.2 QueryWrapper vs LambdaQueryWrapper

#### 你的项目（XML 动态 SQL）
```xml
<!-- DishMapper.xml -->
<select id="pageQuery" resultType="com.sky.vo.DishVO">
    select d.*, c.name as categoryName
    from dish d
    left outer join category c on d.category_id = c.id
    <where>
        <if test="name != null">
            and d.name like concat('%',#{name},'%')
        </if>
        <if test="categoryId != null">
            and d.category_id = #{categoryId}
        </if>
        <if test="status != null">
            and d.status = #{status}
        </if>
    </where>
    order by d.create_time desc
</select>
```

#### QueryWrapper（字符串方式）
```java
QueryWrapper<Dish> wrapper = new QueryWrapper<>();
wrapper.like(dto.getName() != null, "name", dto.getName())
       .eq(dto.getCategoryId() != null, "category_id", dto.getCategoryId())
       .eq(dto.getStatus() != null, "status", dto.getStatus())
       .orderByDesc("create_time");

List<Dish> list = dishMapper.selectList(wrapper);
```

**缺点**：字段名是字符串，容易写错，重构不友好

#### LambdaQueryWrapper（Lambda方式，推荐）
```java
LambdaQueryWrapper<Dish> wrapper = new LambdaQueryWrapper<>();
wrapper.like(dto.getName() != null, Dish::getName, dto.getName())
       .eq(dto.getCategoryId() != null, Dish::getCategoryId, dto.getCategoryId())
       .eq(dto.getStatus() != null, Dish::getStatus, dto.getStatus())
       .orderByDesc(Dish::getCreateTime);

List<Dish> list = dishMapper.selectList(wrapper);
```

**优点**：
- 类型安全，编译期检查
- 重构友好，IDE可以自动重命名
- 避免字段名拼写错误

### 5.3 常用条件方法

#### 比较操作
```java
LambdaQueryWrapper<Employee> wrapper = new LambdaQueryWrapper<>();

// 等于：=
wrapper.eq(Employee::getStatus, 1);
// SQL: status = 1

// 不等于：!=
wrapper.ne(Employee::getStatus, 0);
// SQL: status != 0

// 大于：>
wrapper.gt(Employee::getAge, 18);
// SQL: age > 18

// 大于等于：>=
wrapper.ge(Employee::getAge, 18);
// SQL: age >= 18

// 小于：<
wrapper.lt(Employee::getAge, 60);
// SQL: age < 60

// 小于等于：<=
wrapper.le(Employee::getAge, 60);
// SQL: age <= 60

// 区间：BETWEEN
wrapper.between(Employee::getAge, 18, 60);
// SQL: age BETWEEN 18 AND 60

// 不在区间：NOT BETWEEN
wrapper.notBetween(Employee::getAge, 0, 18);
// SQL: age NOT BETWEEN 0 AND 18
```

#### 模糊查询
```java
// LIKE '%值%'
wrapper.like(Employee::getName, "张");
// SQL: name LIKE '%张%'

// NOT LIKE '%值%'
wrapper.notLike(Employee::getName, "王");
// SQL: name NOT LIKE '%王%'

// LIKE '%值'
wrapper.likeLeft(Employee::getName, "三");
// SQL: name LIKE '%三'

// LIKE '值%'
wrapper.likeRight(Employee::getName, "张");
// SQL: name LIKE '张%'
```

**你的项目对比**：
```xml
<!-- 原生MyBatis -->
<if test="name != null">
    and name like concat('%',#{name},'%')
</if>
```

#### 空值判断
```java
// IS NULL
wrapper.isNull(Employee::getPhone);
// SQL: phone IS NULL

// IS NOT NULL
wrapper.isNotNull(Employee::getEmail);
// SQL: email IS NOT NULL
```

#### IN 查询
```java
// IN
wrapper.in(Employee::getId, Arrays.asList(1, 2, 3));
// SQL: id IN (1, 2, 3)

// NOT IN
wrapper.notIn(Employee::getStatus, Arrays.asList(0, 2));
// SQL: status NOT IN (0, 2)
```

**你的项目对比**：
```xml
<!-- DishMapper.xml -->
<delete id="batchDeleteById">
    delete from dish where id in
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</delete>
```

#### 子查询
```java
// IN 子查询
wrapper.inSql(Employee::getDeptId,
    "SELECT id FROM department WHERE status = 1");
// SQL: dept_id IN (SELECT id FROM department WHERE status = 1)

// EXISTS 子查询
wrapper.exists("SELECT 1 FROM department WHERE department.id = employee.dept_id");
```

#### 排序
```java
// 升序
wrapper.orderByAsc(Employee::getAge);
// SQL: ORDER BY age ASC

// 降序
wrapper.orderByDesc(Employee::getCreateTime);
// SQL: ORDER BY create_time DESC

// 多字段排序
wrapper.orderByAsc(Employee::getSort)
       .orderByDesc(Employee::getCreateTime);
// SQL: ORDER BY sort ASC, create_time DESC
```

**你的项目对比**：
```xml
<!-- CateGoryMapper.xml -->
<select id="pageQuery" resultType="com.sky.entity.Category">
    select * from category
    <where>...</where>
    order by sort asc, create_time desc
</select>
```

#### 逻辑组合
```java
// OR
wrapper.eq(Employee::getName, "张三")
       .or()
       .eq(Employee::getAge, 25);
// SQL: (name = '张三' OR age = 25)

// AND 嵌套
wrapper.eq(Employee::getStatus, 1)
       .and(w -> w.gt(Employee::getAge, 18)
                  .lt(Employee::getAge, 60));
// SQL: status = 1 AND (age > 18 AND age < 60)

// OR 嵌套
wrapper.eq(Employee::getStatus, 1)
       .or(w -> w.eq(Employee::getName, "张三")
                 .eq(Employee::getAge, 25));
// SQL: status = 1 OR (name = '张三' AND age = 25)
```

#### 条件判断
```java
// condition 为 true 时才拼接条件
wrapper.like(StringUtils.isNotBlank(name), Employee::getName, name)
       .eq(status != null, Employee::getStatus, status)
       .between(minAge != null && maxAge != null,
                Employee::getAge, minAge, maxAge);
```

**你的项目对比**：
```xml
<!-- 原生MyBatis需要用<if>标签 -->
<if test="name != null and name != ''">
    and name like concat('%',#{name},'%')
</if>
<if test="status != null">
    and status = #{status}
</if>
```

### 5.4 字段选择
```java
// 查询指定字段
wrapper.select(Employee::getId, Employee::getName, Employee::getAge);
// SQL: SELECT id, name, age FROM employee

// 排除某些字段
wrapper.select(Employee.class, info ->
    !info.getColumn().equals("password") &&
    !info.getColumn().equals("deleted"));
// SQL: SELECT id, name, ... (排除password和deleted)
```

### 5.5 分组和聚合
```java
// GROUP BY
wrapper.groupBy(Employee::getDeptId);
// SQL: GROUP BY dept_id

// HAVING
wrapper.groupBy(Employee::getDeptId)
       .having("COUNT(1) > 5");
// SQL: GROUP BY dept_id HAVING COUNT(1) > 5
```

**你的项目对比**：
```xml
<!-- OrderDetailMapper.xml -->
<select id="getTop10" resultType="com.sky.dto.SalesTop10ReportDTO">
    select od.name as name, sum(od.number) as number
    from order_detail od
    left join orders o on od.order_id = o.id
    <where>...</where>
    group by od.name
    order by number desc
    limit 0,10
</select>
```

### 5.6 UpdateWrapper 使用

#### 你的项目（原生 MyBatis）
```xml
<!-- EmployeeMapper.xml -->
<update id="update" parameterType="Employee">
    update employee
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="username != null">username = #{username},</if>
        <if test="password != null">password = #{password},</if>
        <if test="phone != null">phone = #{phone},</if>
        <if test="sex != null">sex = #{sex},</if>
        <if test="idNumber != null">id_number = #{idNumber},</if>
        <if test="updateTime != null">update_time = #{updateTime},</if>
        <if test="updateUser != null">update_user = #{updateUser}</if>
    </set>
    where id = #{id}
</update>
```

#### UpdateWrapper 方式
```java
// 方式1：只更新指定字段
LambdaUpdateWrapper<Employee> wrapper = new LambdaUpdateWrapper<>();
wrapper.set(Employee::getStatus, 1)
       .set(Employee::getUpdateTime, LocalDateTime.now())
       .eq(Employee::getId, id);

employeeMapper.update(null, wrapper);
// SQL: UPDATE employee SET status = 1, update_time = '...' WHERE id = ?

// 方式2：根据条件批量更新
wrapper.set(Employee::getStatus, 0)
       .in(Employee::getId, Arrays.asList(1, 2, 3));

employeeMapper.update(null, wrapper);
// SQL: UPDATE employee SET status = 0 WHERE id IN (1, 2, 3)

// 方式3：实体+条件更新
Employee employee = new Employee();
employee.setStatus(1);

LambdaUpdateWrapper<Employee> wrapper = new LambdaUpdateWrapper<>();
wrapper.eq(Employee::getDeptId, 10);

employeeMapper.update(employee, wrapper);
// SQL: UPDATE employee SET status = 1 WHERE dept_id = 10
```

### 5.7 链式调用

```java
// Query链式调用
List<Employee> list = new LambdaQueryChainWrapper<>(employeeMapper)
    .like(Employee::getName, "张")
    .ge(Employee::getAge, 18)
    .orderByDesc(Employee::getCreateTime)
    .list();

// Update链式调用
boolean success = new LambdaUpdateChainWrapper<>(employeeMapper)
    .set(Employee::getStatus, 1)
    .eq(Employee::getId, id)
    .update();
```

---

## 6. 分页插件

### 6.1 分页配置

#### 你的项目（PageHelper）
```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.3.0</version>
</dependency>
```

```java
// 使用方式
PageHelper.startPage(pageNum, pageSize);
Page<Employee> page = employeeMapper.pageQuery(dto);
```

#### MyBatis-Plus 分页
```java
// 配置拦截器
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        PaginationInnerInterceptor paginationInterceptor =
            new PaginationInnerInterceptor(DbType.MYSQL);

        // 单页分页条数限制（防止恶意查询）
        paginationInterceptor.setMaxLimit(500L);

        // 溢出总页数后是否进行处理（默认不处理）
        paginationInterceptor.setOverflow(false);

        interceptor.addInnerInterceptor(paginationInterceptor);

        return interceptor;
    }
}
```

### 6.2 基本分页使用

```java
// Mapper接口（继承BaseMapper即可）
public interface EmployeeMapper extends BaseMapper<Employee> {
}

// Service层使用
public PageResult pageQuery(EmployeePageQueryDTO dto) {
    // 1. 构造分页对象
    Page<Employee> page = new Page<>(dto.getPage(), dto.getPageSize());

    // 2. 构造查询条件
    LambdaQueryWrapper<Employee> wrapper = new LambdaQueryWrapper<>();
    wrapper.like(StringUtils.isNotBlank(dto.getName()),
                Employee::getName, dto.getName())
           .orderByDesc(Employee::getCreateTime);

    // 3. 执行分页查询
    employeeMapper.selectPage(page, wrapper);

    // 4. 返回结果
    return new PageResult(page.getTotal(), page.getRecords());
}
```

### 6.3 IPage 接口

```java
// 分页对象常用方法
IPage<Employee> page = new Page<>(current, size);

// 获取总记录数
long total = page.getTotal();

// 获取当前页数据
List<Employee> records = page.getRecords();

// 获取总页数
long pages = page.getPages();

// 获取当前页码
long current = page.getCurrent();

// 获取每页显示条数
long size = page.getSize();

// 是否有上一页
boolean hasPrevious = page.hasPrevious();

// 是否有下一页
boolean hasNext = page.hasNext();
```

### 6.4 自定义分页查询

#### 你的项目中的复杂分页查询
```java
// DishMapper.java
Page<DishVO> pageQuery(DishPageQueryDTO dishPageQueryDTO);
```

```xml
<!-- DishMapper.xml -->
<select id="pageQuery" resultType="com.sky.vo.DishVO">
    select d.*, c.name as categoryName
    from dish d
    left outer join category c on d.category_id = c.id
    <where>...</where>
    order by d.create_time desc
</select>
```

```java
// DishServiceImpl.java
public PageResult pageQuery(DishPageQueryDTO dishPageQueryDTO) {
    PageHelper.startPage(dishPageQueryDTO.getPage(),
                        dishPageQueryDTO.getPageSize());
    Page<DishVO> page = dishMapper.pageQuery(dishPageQueryDTO);
    return new PageResult(page.getTotal(), page.getResult());
}
```

#### MyBatis-Plus 方式
```java
// Mapper接口
@Mapper
public interface DishMapper extends BaseMapper<Dish> {
    // 自定义分页查询（带关联查询）
    IPage<DishVO> selectDishVOPage(IPage<DishVO> page,
                                   @Param(Constants.WRAPPER) Wrapper<Dish> wrapper);
}
```

```xml
<!-- DishMapper.xml -->
<select id="selectDishVOPage" resultType="com.sky.vo.DishVO">
    select d.*, c.name as categoryName
    from dish d
    left outer join category c on d.category_id = c.id
    ${ew.customSqlSegment}  <!-- MyBatis-Plus会自动注入WHERE和ORDER BY -->
</select>
```

```java
// Service层使用
public PageResult pageQuery(DishPageQueryDTO dto) {
    Page<DishVO> page = new Page<>(dto.getPage(), dto.getPageSize());

    LambdaQueryWrapper<Dish> wrapper = new LambdaQueryWrapper<>();
    wrapper.like(StringUtils.isNotBlank(dto.getName()),
                Dish::getName, dto.getName())
           .eq(dto.getCategoryId() != null,
                Dish::getCategoryId, dto.getCategoryId())
           .orderByDesc(Dish::getCreateTime);

    dishMapper.selectDishVOPage(page, wrapper);

    return new PageResult(page.getTotal(), page.getRecords());
}
```

**${ew.customSqlSegment}** 会自动注入：
- WHERE 子句
- ORDER BY 子句
- 其他条件

### 6.5 不查询总记录数

```java
// 当不需要总记录数时，可以提高性能
Page<Employee> page = new Page<>(current, size, false); // false表示不查询total

employeeMapper.selectPage(page, wrapper);
```

---

## 7. 代码生成器

### 7.1 MyBatis-Plus Generator

**作用**：根据数据库表自动生成 Entity、Mapper、Service、Controller 代码

#### 依赖配置
```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.5</version>
</dependency>

<!-- 模板引擎（可选Freemarker或Velocity） -->
<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
    <version>2.3.32</version>
</dependency>
```

#### 代码生成器示例
```java
public class CodeGenerator {

    public static void main(String[] args) {
        // 数据源配置
        DataSourceConfig dataSourceConfig = new DataSourceConfig.Builder(
            "jdbc:mysql://localhost:3306/sky_take_out",
            "root",
            "password"
        ).build();

        // 全局配置
        GlobalConfig globalConfig = new GlobalConfig.Builder()
            .outputDir(System.getProperty("user.dir") + "/src/main/java")
            .author("YourName")
            .enableSwagger()  // 开启Swagger注解
            .disableOpenDir() // 禁止打开输出目录
            .build();

        // 包配置
        PackageConfig packageConfig = new PackageConfig.Builder()
            .parent("com.sky")
            .entity("entity")
            .mapper("mapper")
            .service("service")
            .serviceImpl("service.impl")
            .controller("controller")
            .build();

        // 策略配置
        StrategyConfig strategyConfig = new StrategyConfig.Builder()
            .addInclude("employee", "dish", "category")  // 指定表名
            .entityBuilder()
                .enableLombok()  // 启用Lombok
                .enableTableFieldAnnotation()  // 启用字段注解
                .logicDeleteColumnName("deleted")  // 逻辑删除字段
                .addTableFills(  // 自动填充字段
                    new Column("create_time", FieldFill.INSERT),
                    new Column("update_time", FieldFill.INSERT_UPDATE)
                )
            .mapperBuilder()
                .enableBaseResultMap()  // 启用BaseResultMap
                .enableBaseColumnList()  // 启用BaseColumnList
            .serviceBuilder()
                .formatServiceFileName("%sService")
                .formatServiceImplFileName("%sServiceImpl")
            .controllerBuilder()
                .enableRestStyle()  // 开启@RestController
            .build();

        // 执行生成
        AutoGenerator generator = new AutoGenerator(dataSourceConfig);
        generator.global(globalConfig);
        generator.packageInfo(packageConfig);
        generator.strategy(strategyConfig);
        generator.execute();
    }
}
```

#### 生成的代码示例

**Entity**：
```java
@Data
@TableName("employee")
public class Employee implements Serializable {

    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    @TableField("name")
    private String name;

    @TableField(value = "create_time", fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(value = "update_time", fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic
    @TableField("deleted")
    private Integer deleted;
}
```

**Mapper**：
```java
@Mapper
public interface EmployeeMapper extends BaseMapper<Employee> {
}
```

**Service**：
```java
public interface EmployeeService extends IService<Employee> {
}

@Service
public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee>
    implements EmployeeService {
}
```

**Controller**：
```java
@RestController
@RequestMapping("/employee")
public class EmployeeController {

    @Autowired
    private EmployeeService employeeService;

    @GetMapping("/{id}")
    public Employee getById(@PathVariable Long id) {
        return employeeService.getById(id);
    }
}
```

### 7.2 与你的项目对比

你的项目中手动编写了：
- 11个实体类
- 11个Mapper接口
- 11个XML映射文件
- 10个Service接口
- 10个Service实现类

**使用代码生成器后**：
- 运行一次生成器，所有代码自动生成
- 统一的代码风格
- 自动添加注解和注释
- 大幅提高开发效率

---

## 8. 高级特性

### 8.1 多数据源配置

#### 你的项目（单数据源）
```yaml
# application.yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/sky_take_out
    username: root
    password: root
```

#### MyBatis-Plus 多数据源
```xml
<!-- 引入多数据源依赖 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.6.1</version>
</dependency>
```

```yaml
spring:
  datasource:
    dynamic:
      primary: master  # 主数据源
      strict: false
      datasource:
        master:
          url: jdbc:mysql://localhost:3306/db1
          username: root
          password: root
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave_1:
          url: jdbc:mysql://localhost:3306/db2
          username: root
          password: root
          driver-class-name: com.mysql.cj.jdbc.Driver
```

```java
// 使用@DS注解切换数据源
@Service
@DS("master")  // 指定数据源
public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee>
    implements EmployeeService {

    @DS("slave_1")  // 方法级别切换数据源
    public List<Employee> listFromSlave() {
        return this.list();
    }
}
```

### 8.2 SQL 性能分析

```java
// 配置性能分析拦截器（仅开发环境使用）
@Configuration
@Profile({"dev", "test"})  // 只在dev和test环境生效
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // SQL性能分析（打印SQL执行时间）
        interceptor.addInnerInterceptor(new IllegalSQLInnerInterceptor());

        return interceptor;
    }
}
```

### 8.3 防止全表更新和删除

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    // 阻止恶意的全表更新删除
    interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());

    return interceptor;
}
```

**效果**：
```java
// 以下操作会被拦截并抛出异常
employeeMapper.delete(null);  // 全表删除
employeeMapper.update(new Employee(), null);  // 全表更新
```

### 8.4 数据权限插件

```java
// 实现数据权限处理器
@Component
public class MyDataPermissionHandler implements DataPermissionHandler {

    @Override
    public Expression getSqlSegment(Expression where, String mappedStatementId) {
        // 获取当前用户
        Long userId = getCurrentUserId();

        // 只能查看自己创建的数据
        return new EqualsTo(new Column("create_user"), new LongValue(userId));
    }
}
```

```java
// 配置数据权限拦截器
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    // 数据权限插件
    interceptor.addInnerInterceptor(new DataPermissionInterceptor(
        new MyDataPermissionHandler()
    ));

    return interceptor;
}
```

**效果**：
```java
// 自动在SQL中添加 WHERE create_user = 当前用户ID
List<Order> orders = orderMapper.selectList(null);
// SQL: SELECT * FROM orders WHERE create_user = 1
```

### 8.5 SQL 注入器

**作用**：扩展BaseMapper，添加自定义方法

```java
// 1. 自定义方法
public class DeleteAll extends AbstractMethod {
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass,
                                                 Class<?> modelClass,
                                                 TableInfo tableInfo) {
        String sql = "DELETE FROM " + tableInfo.getTableName();
        SqlSource sqlSource = languageDriver.createSqlSource(
            configuration, sql, modelClass);
        return this.addDeleteMappedStatement(mapperClass, "deleteAll",
                                            sqlSource);
    }
}

// 2. 自定义SQL注入器
public class MySqlInjector extends DefaultSqlInjector {
    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass,
                                              TableInfo tableInfo) {
        List<AbstractMethod> methodList = super.getMethodList(mapperClass, tableInfo);
        methodList.add(new DeleteAll());  // 添加自定义方法
        return methodList;
    }
}

// 3. 配置注入器
@Bean
public MySqlInjector mySqlInjector() {
    return new MySqlInjector();
}

// 4. 自定义BaseMapper
public interface MyBaseMapper<T> extends BaseMapper<T> {
    int deleteAll();  // 自定义方法
}

// 5. 使用
public interface EmployeeMapper extends MyBaseMapper<Employee> {
}

// 现在所有Mapper都拥有deleteAll方法
employeeMapper.deleteAll();
```

### 8.6 字段类型处理器

**作用**：处理Java类型与JDBC类型的转换

#### 你的项目中可能遇到的问题
```java
// 如果有JSON字段
public class Dish {
    private String flavors;  // 数据库中存储JSON字符串
}
```

#### MyBatis-Plus 类型处理器
```java
public class Dish {
    // 自动将JSON字符串转换为List对象
    @TableField(typeHandler = JacksonTypeHandler.class)
    private List<DishFlavor> flavors;
}
```

**全局配置**：
```yaml
mybatis-plus:
  configuration:
    default-enum-type-handler: org.apache.ibatis.type.EnumOrdinalTypeHandler
  type-handlers-package: com.sky.handler
```

---

## 9. 性能优化

### 9.1 批量操作优化

#### 你的项目（XML foreach）
```xml
<!-- DishFlavorMapper.xml -->
<insert id="insertBatch">
    insert into dish_flavor (dish_id, name, value)
    values
    <foreach collection="dishFlavors" item="dishFlavor" separator=",">
        (#{dishFlavor.dishId}, #{dishFlavor.name}, #{dishFlavor.value})
    </foreach>
</insert>
```

#### MyBatis-Plus 批量插入
```java
// 方式1：使用Service的saveBatch（推荐）
List<DishFlavor> flavors = ...;
dishFlavorService.saveBatch(flavors);  // 默认每批1000条

// 方式2：指定批次大小
dishFlavorService.saveBatch(flavors, 500);  // 每批500条

// 方式3：使用InsertBatchSomeColumn（需要SQL注入器）
dishFlavorMapper.insertBatchSomeColumn(flavors);
```

**性能对比**：
- 循环insert：1000条 ≈ 10秒
- XML foreach：1000条 ≈ 2秒
- saveBatch：1000条 ≈ 1秒

### 9.2 只查询需要的字段

```java
// 不好的做法：查询所有字段
List<Employee> list = employeeMapper.selectList(null);

// 好的做法：只查询需要的字段
LambdaQueryWrapper<Employee> wrapper = new LambdaQueryWrapper<>();
wrapper.select(Employee::getId, Employee::getName, Employee::getPhone);
List<Employee> list = employeeMapper.selectList(wrapper);
```

### 9.3 使用 exists 代替 in

```java
// 当子查询返回大量数据时，exists性能更好
wrapper.exists("SELECT 1 FROM department WHERE department.id = employee.dept_id");
```

### 9.4 避免 N+1 查询问题

#### 问题示例
```java
// 查询所有菜品
List<Dish> dishes = dishMapper.selectList(null);

// 为每个菜品查询口味（N+1问题）
for (Dish dish : dishes) {
    List<DishFlavor> flavors = dishFlavorMapper.selectList(
        new LambdaQueryWrapper<DishFlavor>()
            .eq(DishFlavor::getDishId, dish.getId())
    );
    dish.setFlavors(flavors);
}
```

#### 优化方案
```java
// 一次性查询所有菜品ID
List<Long> dishIds = dishes.stream()
    .map(Dish::getId)
    .collect(Collectors.toList());

// 一次性查询所有口味
List<DishFlavor> allFlavors = dishFlavorMapper.selectList(
    new LambdaQueryWrapper<DishFlavor>()
        .in(DishFlavor::getDishId, dishIds)
);

// 在内存中分组关联
Map<Long, List<DishFlavor>> flavorMap = allFlavors.stream()
    .collect(Collectors.groupingBy(DishFlavor::getDishId));

// 填充数据
dishes.forEach(dish ->
    dish.setFlavors(flavorMap.getOrDefault(dish.getId(), new ArrayList<>()))
);
```

### 9.5 使用缓存

```java
@Service
@CacheConfig(cacheNames = "employee")
public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee>
    implements EmployeeService {

    @Cacheable(key = "#id")
    public Employee getById(Long id) {
        return super.getById(id);
    }

    @CacheEvict(key = "#employee.id")
    public boolean updateById(Employee employee) {
        return super.updateById(employee);
    }
}
```

---

## 10. 常见面试题

### 10.1 基础概念题

#### Q1: MyBatis-Plus 与 MyBatis 的区别？

**答案**：
- **MyBatis**：半自动ORM框架，需要手写SQL
- **MyBatis-Plus**：在MyBatis基础上增强，提供CRUD接口，无需手写SQL

**主要增强**：
1. **通用CRUD**：BaseMapper、IService提供单表CRUD
2. **条件构造器**：LambdaQueryWrapper替代XML动态SQL
3. **分页插件**：内置分页，无需额外依赖
4. **代码生成器**：根据数据库表生成代码
5. **自动填充**：createTime、updateTime自动填充
6. **逻辑删除**：@TableLogic实现软删除
7. **乐观锁**：@Version实现乐观锁

**结合你的项目说明**：
- 你的项目使用PageHelper分页，MyBatis-Plus内置分页
- 你的项目用AOP实现字段填充，MyBatis-Plus用MetaObjectHandler
- 你的项目用XML写动态SQL，MyBatis-Plus用Wrapper

#### Q2: MyBatis-Plus 的核心注解有哪些？

| 注解 | 作用 | 示例 |
|------|------|------|
| @TableName | 指定表名 | @TableName("employee") |
| @TableId | 主键策略 | @TableId(type = IdType.AUTO) |
| @TableField | 字段配置 | @TableField(fill = FieldFill.INSERT) |
| @TableLogic | 逻辑删除 | @TableLogic(value = "0", delval = "1") |
| @Version | 乐观锁 | @Version |

#### Q3: 主键生成策略有哪些？

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| AUTO | 数据库自增 | 单体应用 + MySQL |
| ASSIGN_ID | 雪花算法（默认） | 分布式系统 |
| ASSIGN_UUID | UUID | 需要字符串ID |
| INPUT | 手动输入 | 自定义ID生成 |

**雪花算法特点**：
- 64位Long类型
- 高位：时间戳（41位）
- 中位：机器ID（10位）
- 低位：序列号（12位）
- 优点：趋势递增、高性能、分布式唯一
- 缺点：依赖系统时钟、ID较长

#### Q4: 如何实现字段自动填充？

```java
// 1. 实体类添加注解
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;

// 2. 实现MetaObjectHandler
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime",
            LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime",
            LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime",
            LocalDateTime.class, LocalDateTime.now());
    }
}
```

**与你的项目对比**：
- 你的项目：自定义@AutoFill注解 + AOP切面 + 反射
- MyBatis-Plus：实现接口 + 注解配置

### 10.2 条件构造器题

#### Q5: QueryWrapper 和 LambdaQueryWrapper 的区别？

```java
// QueryWrapper - 字符串方式（不推荐）
QueryWrapper<Employee> wrapper = new QueryWrapper<>();
wrapper.eq("status", 1)
       .like("name", "张")
       .orderByDesc("create_time");

// LambdaQueryWrapper - Lambda方式（推荐）
LambdaQueryWrapper<Employee> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(Employee::getStatus, 1)
       .like(Employee::getName, "张")
       .orderByDesc(Employee::getCreateTime);
```

**Lambda方式优点**：
1. **类型安全**：编译期检查，避免字段名写错
2. **重构友好**：IDE可以自动重命名
3. **代码提示**：IDE可以智能提示字段
4. **可读性强**：直接引用getter方法

#### Q6: 如何实现动态条件查询？

**你的项目（XML）**：
```xml
<where>
    <if test="name != null">
        and name like concat('%',#{name},'%')
    </if>
    <if test="status != null">
        and status = #{status}
    </if>
</where>
```

**MyBatis-Plus**：
```java
LambdaQueryWrapper<Employee> wrapper = new LambdaQueryWrapper<>();
wrapper.like(name != null, Employee::getName, name)
       .eq(status != null, Employee::getStatus, status);

// 或使用Optional
wrapper.like(StringUtils.isNotBlank(name), Employee::getName, name)
       .eq(Objects.nonNull(status), Employee::getStatus, status);
```

#### Q7: 如何实现复杂的OR查询？

```java
// 需求：(status = 1) OR (name = '张三' AND age = 25)
LambdaQueryWrapper<Employee> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(Employee::getStatus, 1)
       .or(w -> w.eq(Employee::getName, "张三")
                 .eq(Employee::getAge, 25));

// SQL: WHERE status = 1 OR (name = '张三' AND age = 25)
```

### 10.3 分页题

#### Q8: MyBatis-Plus 分页原理？

**答案**：
1. 配置分页拦截器（PaginationInnerInterceptor）
2. 拦截selectPage方法
3. 执行 COUNT 查询获取总记录数
4. 在原SQL后添加 LIMIT 子句
5. 封装分页结果返回

**关键点**：
- 物理分页，不是内存分页
- 自动执行两条SQL（COUNT + SELECT）
- 支持多数据库方言

#### Q9: 如何优化分页性能？

```java
// 1. 不查询总记录数（当不需要total时）
Page<Employee> page = new Page<>(current, size, false);

// 2. 只查询需要的字段
wrapper.select(Employee::getId, Employee::getName);

// 3. 使用索引字段排序
wrapper.orderByDesc(Employee::getId);  // ID是主键，有索引

// 4. 避免深分页（LIMIT 10000,10 性能差）
// 改用游标分页：WHERE id > lastId LIMIT 10
```

### 10.4 性能优化题

#### Q10: 如何批量插入10万条数据？

```java
// 方式1：使用saveBatch（推荐）
List<Employee> list = ...;  // 10万条
employeeService.saveBatch(list, 1000);  // 每批1000条

// 方式2：使用insertBatchSomeColumn（需要SQL注入器）
employeeMapper.insertBatchSomeColumn(list);

// 方式3：JDBC批处理（最快）
@Transactional(rollbackFor = Exception.class)
public void batchInsert(List<Employee> list) {
    SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
    EmployeeMapper mapper = sqlSession.getMapper(EmployeeMapper.class);
    for (Employee employee : list) {
        mapper.insert(employee);
    }
    sqlSession.commit();
    sqlSession.close();
}
```

**性能对比**：
- 循环insert：10万条 ≈ 17分钟
- saveBatch：10万条 ≈ 1分钟
- JDBC批处理：10万条 ≈ 20秒

#### Q11: N+1 查询问题如何解决？

**问题**：
```java
// 查询100个订单
List<Orders> orders = orderMapper.selectList(null);

// 为每个订单查询详情（N+1问题，执行101条SQL）
for (Orders order : orders) {
    List<OrderDetail> details = orderDetailMapper.selectList(
        new LambdaQueryWrapper<OrderDetail>()
            .eq(OrderDetail::getOrderId, order.getId())
    );
    order.setDetails(details);
}
```

**解决方案**：
```java
// 1. 一次性查询所有订单ID
List<Long> orderIds = orders.stream()
    .map(Orders::getId)
    .collect(Collectors.toList());

// 2. 一次性查询所有详情（只执行2条SQL）
List<OrderDetail> allDetails = orderDetailMapper.selectList(
    new LambdaQueryWrapper<OrderDetail>()
        .in(OrderDetail::getOrderId, orderIds)
);

// 3. 内存中分组
Map<Long, List<OrderDetail>> detailMap = allDetails.stream()
    .collect(Collectors.groupingBy(OrderDetail::getOrderId));

// 4. 填充数据
orders.forEach(order ->
    order.setDetails(detailMap.getOrDefault(order.getId(), new ArrayList<>()))
);
```

### 10.5 高级特性题

#### Q12: 如何实现逻辑删除？

```java
// 1. 实体类添加注解
@TableLogic(value = "0", delval = "1")
private Integer deleted;

// 2. 全局配置（可选）
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0

// 3. 使用效果
employeeMapper.deleteById(1L);
// SQL: UPDATE employee SET deleted=1 WHERE id=1 AND deleted=0

employeeMapper.selectList(null);
// SQL: SELECT * FROM employee WHERE deleted=0
```

**优点**：
- 数据可恢复
- 符合审计要求
- 对业务代码透明

**缺点**：
- 占用存储空间
- 查询性能略有下降
- 唯一索引需要特殊处理

#### Q13: 如何实现乐观锁？

```java
// 1. 实体类添加version字段
@Version
private Integer version;

// 2. 配置乐观锁拦截器
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    return interceptor;
}

// 3. 使用
Employee employee = employeeMapper.selectById(1L);  // version=1
employee.setName("新名字");
employeeMapper.updateById(employee);
// SQL: UPDATE employee SET name='新名字', version=2 WHERE id=1 AND version=1

// 如果version不匹配，更新失败，返回0
```

**适用场景**：
- 高并发修改场景
- 防止数据覆盖
- 避免使用悲观锁（SELECT FOR UPDATE）

#### Q14: 如何实现多租户？

```java
// 1. 实现租户处理器
@Component
public class MyTenantLineHandler implements TenantLineHandler {

    @Override
    public Expression getTenantId() {
        // 从上下文获取租户ID
        return new LongValue(getCurrentTenantId());
    }

    @Override
    public String getTenantIdColumn() {
        return "tenant_id";  // 租户ID字段名
    }

    @Override
    public boolean ignoreTable(String tableName) {
        // 排除不需要租户过滤的表
        return "employee".equalsIgnoreCase(tableName);
    }
}

// 2. 配置多租户拦截器
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    TenantLineInnerInterceptor tenantInterceptor =
        new TenantLineInnerInterceptor(new MyTenantLineHandler());
    interceptor.addInnerInterceptor(tenantInterceptor);

    return interceptor;
}

// 3. 效果
orderMapper.selectList(null);
// SQL: SELECT * FROM orders WHERE tenant_id = 1
```

### 10.6 项目实战题

#### Q15: 在你的项目中，如何从原生 MyBatis 迁移到 MyBatis-Plus？

**答案**（结合你的项目）：

**第一步：修改依赖**
```xml
<!-- 移除原有依赖 -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
</dependency>

<!-- 添加MyBatis-Plus -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
```

**第二步：修改实体类**
```java
// Employee.java
@Data
@TableName("employee")
public class Employee implements Serializable {

    @TableId(value = "id", type = IdType.AUTO)
    private Long id;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    // ... 其他字段
}
```

**第三步：修改 Mapper 接口**
```java
// EmployeeMapper.java
@Mapper
public interface EmployeeMapper extends BaseMapper<Employee> {
    // 移除简单的CRUD方法，保留复杂的自定义查询
    @Select("select * from employee where username = #{username}")
    Employee getByUsername(String username);
}
```

**第四步：修改 Service 层**
```java
// EmployeeServiceImpl.java
@Service
public class EmployeeServiceImpl extends ServiceImpl<EmployeeMapper, Employee>
    implements EmployeeService {

    // 简化CRUD操作
    public Employee getById(Long id) {
        return this.getById(id);  // 使用继承的方法
    }

    public void save(EmployeeDTO employeeDTO) {
        Employee employee = new Employee();
        BeanUtils.copyProperties(employeeDTO, employee);
        this.save(employee);  // 使用继承的方法
    }
}
```

**第五步：替换分页代码**
```java
// 原PageHelper方式
PageHelper.startPage(page, pageSize);
Page<Employee> page = employeeMapper.pageQuery(dto);

// MyBatis-Plus方式
Page<Employee> page = new Page<>(dto.getPage(), dto.getPageSize());
LambdaQueryWrapper<Employee> wrapper = new LambdaQueryWrapper<>();
wrapper.like(StringUtils.isNotBlank(dto.getName()),
            Employee::getName, dto.getName());
employeeMapper.selectPage(page, wrapper);
```

**第六步：实现自动填充**
```java
// 移除AutoFillAspect
// 实现MetaObjectHandler
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime",
            LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "createUser",
            Long.class, BaseContext.getCurrentId());
    }
    // ...
}
```

**第七步：配置拦截器**
```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        interceptor.addInnerInterceptor(
            new PaginationInnerInterceptor(DbType.MYSQL));

        // 防止全表更新删除
        interceptor.addInnerInterceptor(
            new BlockAttackInnerInterceptor());

        return interceptor;
    }
}
```

**迁移收益**：
- 代码量减少约30%
- XML文件减少
- 无需PageHelper依赖
- 统一的CRUD风格
- 更好的类型安全

#### Q16: 项目中如何处理复杂的关联查询？

**你的项目示例**：
```xml
<!-- DishMapper.xml -->
<select id="pageQuery" resultType="com.sky.vo.DishVO">
    select d.*, c.name as categoryName
    from dish d
    left outer join category c on d.category_id = c.id
    <where>...</where>
</select>
```

**MyBatis-Plus 方案**：
```java
// 方式1：保留XML（推荐）
@Mapper
public interface DishMapper extends BaseMapper<Dish> {
    IPage<DishVO> selectDishVOPage(IPage<DishVO> page,
                                   @Param(Constants.WRAPPER) Wrapper<Dish> wrapper);
}
```

```xml
<!-- DishMapper.xml -->
<select id="selectDishVOPage" resultType="com.sky.vo.DishVO">
    select d.*, c.name as categoryName
    from dish d
    left outer join category c on d.category_id = c.id
    ${ew.customSqlSegment}
</select>
```

```java
// 使用
Page<DishVO> page = new Page<>(pageNum, pageSize);
LambdaQueryWrapper<Dish> wrapper = new LambdaQueryWrapper<>();
wrapper.like(StringUtils.isNotBlank(name), Dish::getName, name)
       .orderByDesc(Dish::getCreateTime);

dishMapper.selectDishVOPage(page, wrapper);
```

**方式2：使用MyBatis-Plus Join扩展**
```xml
<dependency>
    <groupId>com.github.yulichang</groupId>
    <artifactId>mybatis-plus-join</artifactId>
    <version>1.4.6</version>
</dependency>
```

```java
MPJLambdaWrapper<Dish> wrapper = new MPJLambdaWrapper<>();
wrapper.selectAll(Dish.class)
       .select(Category::getName, "categoryName")
       .leftJoin(Category.class, Category::getId, Dish::getCategoryId)
       .like(StringUtils.isNotBlank(name), Dish::getName, name);

IPage<DishVO> page = dishMapper.selectJoinPage(
    new Page<>(pageNum, pageSize), DishVO.class, wrapper);
```

---

## 11. 总结

### 11.1 MyBatis-Plus 核心优势

| 特性 | 原生MyBatis | MyBatis-Plus |
|------|------------|--------------|
| **简单CRUD** | 需要手写SQL | 继承BaseMapper自动拥有 |
| **分页** | 需要PageHelper | 内置分页插件 |
| **动态SQL** | XML \<if\>标签 | LambdaQueryWrapper |
| **批量操作** | XML \<foreach\> | saveBatch自动批处理 |
| **字段填充** | 自定义AOP | MetaObjectHandler |
| **逻辑删除** | 手动处理 | @TableLogic自动处理 |
| **乐观锁** | 手动SQL | @Version自动处理 |
| **代码生成** | 手动编写 | 代码生成器 |

### 11.2 你的项目可以改进的地方

基于对你项目的分析，以下是可以用MyBatis-Plus改进的地方：

1. **简化Mapper接口**
   - 移除简单的@Select、@Insert、@Update
   - 继承BaseMapper获得CRUD方法

2. **统一分页方式**
   - 移除PageHelper依赖
   - 使用MyBatis-Plus内置分页

3. **简化动态SQL**
   - 大部分XML可以用LambdaQueryWrapper替代
   - 只保留复杂的关联查询XML

4. **自动填充优化**
   - 移除AutoFillAspect
   - 使用MetaObjectHandler更简洁

5. **批量操作优化**
   - 使用saveBatch替代XML foreach
   - 性能更好，代码更简洁

### 11.3 面试准备建议

1. **熟悉你的项目**
   - 理解现有MyBatis代码
   - 知道如何用MyBatis-Plus改进

2. **掌握核心特性**
   - 注解使用（@TableName、@TableId等）
   - 条件构造器（LambdaQueryWrapper）
   - 分页插件
   - 自动填充

3. **性能优化**
   - 批量操作
   - N+1问题
   - 只查询需要的字段

4. **高级特性**
   - 逻辑删除
   - 乐观锁
   - 多租户（了解即可）

5. **实战经验**
   - 结合你的项目说明
   - 对比原生MyBatis和MyBatis-Plus
   - 说明迁移方案

### 11.4 学习资源

- **官方文档**：https://baomidou.com/
- **GitHub**：https://github.com/baomidou/mybatis-plus
- **示例项目**：https://github.com/baomidou/mybatis-plus-samples

---

## 附录：你的项目代码位置

| 类型 | 路径 |
|------|------|
| **实体类** | `/sky-pojo/src/main/java/com/sky/entity/` |
| **Mapper接口** | `/sky-server/src/main/java/com/sky/mapper/` |
| **XML映射** | `/sky-server/src/main/resources/mapper/` |
| **Service** | `/sky-server/src/main/java/com/sky/service/` |
| **Service实现** | `/sky-server/src/main/java/com/sky/service/impl/` |
| **配置** | `/sky-server/src/main/resources/application.yml` |

**关键代码参考**：
- 分页：`CategoryServiceImpl.java:pageQuery()`
- 动态SQL：`DishMapper.xml`
- 批量插入：`DishFlavorMapper.xml:insertBatch()`
- 自动填充：`AutoFillAspect.java`

---

**祝你面试顺利！🎉**