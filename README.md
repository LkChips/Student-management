# 学生管理系统（Student Management System）

基于 **Servlet + JSP + MySQL** 的经典 JavaWeb 学生管理系统，采用传统 MVC 四层架构，无需 Spring 等重型框架，适合课程设计与 JavaWeb 学习。

## 一、技术栈

| 类别 | 技术 | 版本 |
|------|------|------|
| 语言 | Java | 1.8 |
| Web 容器 | Tomcat（tomcat7-maven-plugin 内嵌运行） | 2.2 |
| 视图 | JSP + JSTL | 2.1 / 1.2 |
| 数据库 | MySQL | 5.x / 8.x 均可 |
| 数据库驱动 | mysql-connector-java | 5.1.49 |
| 连接池 | C3P0 | 0.9.5.5 |
| 构建工具 | Maven | packaging = war |

## 二、功能模块

### 管理员端
- **控制台首页**：快捷操作与功能导航
- **课程管理**：课程的增、删、改、查，设置课程容量、学分、授课教师、学期
- **学生管理**：学生账号的增、删、改、查，重置学生密码（重置为 `123456`）
- **成绩管理**：为已选课学生录入/修改成绩，查看成绩列表
- **修改密码**：管理员自助修改密码

### 学生端
- **控制台首页**：个人快捷入口
- **选课管理**：在选课时间窗口内选课、退课（课程容量与时间窗口双重校验）
- **成绩查询**：查询本人各课程成绩
- **修改密码**：学生自助修改密码

### 通用
- 登录 / 退出登录，基于 Session 的认证
- 基于 Filter 的统一权限拦截（管理员、学生角色分离）
- 统一请求编码（UTF-8）

## 三、项目结构

```
Student management/
├── pom.xml                          # Maven 构建配置（依赖、tomcat7 插件）
├── sql/
│   ├── schema.sql                   # 建库建表脚本
│   ├── seed.sql                     # 初始测试数据（账户、课程、选课时间窗口）
│   └── reset_password.sql           # 密码重置脚本（忘记密码时使用）
└── src/main/
    ├── java/com/studentmgmt/
    │   ├── filter/
    │   │   ├── EncodingFilter.java  # 全局 UTF-8 编码过滤器
    │   │   └── AuthFilter.java      # 登录认证 + 角色授权过滤器
    │   ├── servlet/
    │   │   ├── LoginServlet.java        # POST /login
    │   │   ├── LogoutServlet.java       # GET  /logout
    │   │   ├── admin/
    │   │   │   ├── CourseServlet.java       # /admin/course/*
    │   │   │   ├── StudentManageServlet.java# /admin/student/*
    │   │   │   ├── GradeServlet.java        # /admin/grade/*
    │   │   │   └── AdminProfileServlet.java # /admin/profile/*
    │   │   └── student/
    │   │       ├── EnrollmentServlet.java   # /student/enrollment/*
    │   │       ├── GradeQueryServlet.java   # /student/grade/*
    │   │       └── StudentProfileServlet.java # /student/profile/*
    │   ├── service/                 # 业务逻辑层
    │   │   ├── UserService.java
    │   │   ├── CourseService.java
    │   │   ├── EnrollmentService.java
    │   │   └── GradeService.java
    │   ├── dao/                     # 数据访问层（JDBC + C3P0）
    │   │   ├── UserDao.java
    │   │   ├── CourseDao.java
    │   │   ├── EnrollmentDao.java
    │   │   └── GradeDao.java
    │   ├── model/                   # 实体类
    │   │   ├── User.java
    │   │   ├── Course.java
    │   │   ├── Enrollment.java
    │   │   └── Grade.java
    │   └── util/
    │       ├── DBUtil.java          # C3P0 连接池工具（读取 c3p0-config.xml）
    │       └── PasswordUtil.java    # 密码哈希工具（SHA-256 加盐）
    ├── resources/
    │   └── c3p0-config.xml          # 数据库连接池配置
    └── webapp/
        ├── WEB-INF/web.xml          # Servlet/Filter 注册与欢迎页
        ├── index.jsp                # 入口页（跳转登录）
        ├── login.jsp                # 登录页
        ├── admin/                   # 管理员端页面（dashboard、course、student、grade、profile）
        ├── student/                 # 学生端页面（dashboard、enrollment、grade、profile）
        └── static/
            ├── css/common.css       # 全局样式
            └── js/common.js         # 全局脚本
```

**请求流转**：`Filter（编码 → 认证/授权） → Servlet（接收/校验/分发） → Service（业务规则） → DAO（SQL） → MySQL`

## 四、数据库设计

数据库名：`student_mgmt`（utf8mb4）

| 表名 | 说明 | 关键约束 |
|------|------|----------|
| `t_user` | 用户/登录账户（管理员与学生共用） | `username` 唯一；`student_no` 唯一；`role`：0=学生，1=管理员；`status`：1=启用，0=禁用 |
| `t_course` | 课程信息 | `course_no` 唯一；`max_students` 选课容量 |
| `t_enrollment` | 选课记录 | `(student_id, course_id)` 唯一，防止重复选课；外键级联删除 |
| `t_grade` | 成绩记录 | `(student_id, course_id)` 唯一；`score` 为 NULL 表示未录入 |
| `t_config` | 系统配置 | 存放选课时间窗口 `enroll_start` / `enroll_end` |

初始化配置项：

```sql
enroll_start = 2026-06-01 00:00:00   -- 选课开始时间
enroll_end   = 2026-12-31 23:59:59   -- 选课结束时间
```

如需调整选课时间窗口，直接修改 `t_config` 表对应记录即可。

## 五、密码加密说明

密码采用 **SHA-256 加盐哈希** 存储，盐为用户名：

```
stored_hash = SHA-256( username + ":" + password )
```

- 存储与校验逻辑集中在 `com.studentmgmt.util.PasswordUtil`
- 数据库中不保存明文密码
- 管理员重置学生密码时，学生密码被重置为 `123456`（同样经过加盐哈希）
- 忘记所有密码时，可执行 `sql/reset_password.sql` 一键重置（脚本内哈希值已按加盐算法生成）

## 六、快速开始

### 1. 环境准备

- JDK 1.8+
- Maven 3.x
- MySQL 5.7+（本机默认连接：`localhost:3306`，账号 `root`，密码 `1234`）

### 2. 初始化数据库

依次执行：

```bash
mysql -u root -p < sql/schema.sql
mysql -u root -p < sql/seed.sql
```

> 如果数据库连接信息与上述不同，请修改 `src/main/resources/c3p0-config.xml` 中的 `jdbcUrl`、`user`、`password`。

### 3. 编译并启动

```bash
mvn compile
mvn tomcat7:run
```

启动后访问：

```
http://localhost:8080/student-management
```

### 4. 默认测试账户

| 用户名 | 密码 | 角色 |
|--------|------|------|
| `admin` | `admin123` | 管理员 |
| `zhangsan` | `123456` | 学生（学号 2024001） |
| `lisi` | `123456` | 学生（学号 2024002） |
| `wangwu` | `123456` | 学生（学号 2024003） |

初始课程数据（Java Programming、Database Systems 等 5 门课）见 `sql/seed.sql`。

## 七、常见问题（FAQ）

**1. 启动报错 `Memory database file ... tomcat-users.xml cannot be read`**

`target/tomcat` 目录残留损坏文件所致。解决：停止所有 Java 进程，删除整个 `target` 目录，重新执行 `mvn compile` 和 `mvn tomcat7:run`。

**2. 登录成功但功能页面 404 / 无响应**

确认 `web.xml` 中 Servlet 映射正确，且访问路径包含上下文前缀 `/student-management`。

**3. 页面中文乱码**

确认所有请求经过 `EncodingFilter`，且 MySQL 连接串带 `characterEncoding=utf8`，数据库使用 `utf8mb4`。

**4. 选课失败提示不在选课时间**

修改 `t_config` 表中 `enroll_start` / `enroll_end` 的值，覆盖当前时间。

**5. 密码忘记**

执行 `sql/reset_password.sql` 重置全部账户为默认密码（见第五节）。
