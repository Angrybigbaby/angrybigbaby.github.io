---
layout:     post
title:      "一个具备可行性的开发规范长什么样子"
subtitle:   "面向后端"
date:       2025-05-15
author:     "Bigbaby"
tags:
    - 后端
    - Java
---

> 最近由于供应商拉跨，不得不亲自手搓一份开发规范出来。在撰写的过程中产生了一些思考，也通过这件事对整套开发流程做了一个系统的梳理。尽管很多公司没有规范，或者干脆沿用阿里的规范，但我想说的是，不论从代码质量还是团队管理，形成一套自己的规范还是很重要的。细看阿里的JAva开发规范不难发现，很多内容不见得适配公司的实际情况。因此要把这件事做到极致，就必须形成自己的规范。下述有部分内容脱胎于阿里的规范，但也修改和增加了很多适配公司的内容。
>
> 开发规范是确保代码质量、团队协作效率和项目可维护性的基石。本文基于实际项目经验，总结出一套**具备可行性的后端开发规范**，涵盖命名、代码格式、设计、接口、数据库等核心领域，并提供具体实施建议，帮助团队快速落地。

## 一、命名规范

### 1.1 强制性规范
1. **类名与接口名**  
   - 使用 `UpperCamelCase`（大驼峰），如 `UserService`、`OrderRepository`。
   - 反例：`user_service`、`orderrepository`。

2. **方法名、参数名、变量名**  
   - 使用 `lowerCamelCase`（小驼峰），如 `getUserById`、`calculateTotalPrice`。
   - 反例：`get_user_id`、`calculate_total_price`。

3. **常量命名**  
   - 全大写 + 下划线，如 `MAX_RETRY_TIMES`、`DEFAULT_TIMEOUT`。
   - 反例：`maxRetryTimes`、`defaultTimeout`。

4. **包名**  
   - 全小写，避免使用数字开头，如 `com.example.user.service`。
   - 反例：`com.example.UserService`、`com.example.user1`。

5. **枚举类名**  
   - 以 `Enum` 结尾，如 `StatusEnum`、`RoleEnum`。

### 1.2 推荐规范
- **实现类**  
  以 `Impl` 作为后缀，如 `UserServiceImpl`。
- **异常类**  
  以 `Exception` 结尾，如 `InvalidInputException`。
- **测试类**  
  以 `Test` 结尾，如 `UserServiceTest`。

---

## 二、代码格式规范

### 2.1 强制性规范
1. **缩进与空格**  
   - 使用 **4个空格缩进**，禁止使用 `Tab`。
   - 运算符两侧必须有空格，如 `int a = b + c;`。
   - 左大括号前加空格，如 `if (condition) {`。

2. **行长度限制**  
   - 单行字符数不超过 **120**，超出需换行。  
   - 示例：
     ```java
     List<User> users = userRepository.findByCriteria(
         "active", 
         "2025-05-15", 
         100
     );
     ```

3. **注释格式**  
   - 类、方法、接口必须使用 **Javadoc** 注释，格式如下：
     ```java
     /**
      * 用户服务类，提供用户相关的业务操作
      * 
      * @author 张三
      * @date 2025-05-15
      */
     public class UserService {
         /**
          * 根据用户ID获取用户详细信息
          * 
          * @param userId 用户唯一标识符
          * @return 用户实体对象
          */
         public User getUserById(Long userId) {
             return userRepository.findById(userId);
         }
     }
     ```

### 2.2 推荐规范
- **避免魔法值**  
  将字符串或数字常量提取为静态常量，如：
  ```java
  private static final String ACTION_1 = "action1";
  ```

- **右括号对齐**  
  右括号位于行首，提升代码可读性：
  ```java
  if (condition) {
      doSomething();
  } else {
      doOtherThing();
  }
  ```

---

## 三、设计规范

### 3.1 架构分层
- **单体应用分层**  
  ```text
  controller → service → repository → entity
  ```
  - **Controller层**：处理 HTTP 请求与响应。
  - **Service层**：实现业务逻辑。
  - **Repository层**：负责数据访问（DAO）。
  - **Entity层**：与数据库表映射的实体类。

- **微服务分层**  
  ```text
  api → apps → commons → domain
  ```
  - **api层**：定义接口和远程调用。
  - **apps层**：应用逻辑实现。
  - **commons层**：公共工具类和配置。
  - **domain层**：领域模型和业务逻辑。

### 3.2 类设计
1. **禁止默认包**  
   所有类必须归属自定义包，如 `com.example.user`。

2. **final关键字**  
   - 不可变类使用 `final`，如 `final class StringUtils`。
   - 关键方法使用 `final` 防止覆写，如 `final void validate()`。

3. **内部类**  
   - 静态内部类优先于非静态内部类，避免隐式持有外部类引用。
   - 示例：
     ```java
     public class Outer {
         // 静态内部类
         public static class StaticInner {
         }
     }
     ```

---

## 四、接口规范

### 4.1 RESTful API 设计
1. **URI 命名**  
   - 使用复数名词，如 `/users`、`/orders`。
   - 版本号置于 URI 第二部分，如 `/api/v1/users`。

2. **HTTP 方法**  
   | 方法 | 用途 |
   |------|------|
   | GET  | 获取资源 |
   | POST | 创建资源 |
   | PUT  | 更新资源 |
   | DELETE | 删除资源 |

3. **错误响应格式**  
   ```json
   {
     "code": "A0101",
     "message": "用户未登录"
   }
   ```
   - `code`：错误码，便于快速定位问题。
   - `message`：用户友好提示，避免暴露敏感信息。

### 4.2 安全性要求
1. **HTTPS 协议**  
   所有接口必须通过 HTTPS 传输，防止数据泄露。

2. **身份验证**  
   使用 JWT 或 OAuth 2.0 实现鉴权，避免硬编码凭证。

3. **频率控制**  
   限制接口请求频率，防止滥用和 DDoS 攻击。

---

## 五、数据库规范

### 5.1 表设计
1. **表名与字段名**  
   - 使用小写字母和下划线，如 `user_info`、`created_at`。
   - 反例：`UserInfo`、`created_at`。

2. **索引规范**  
   - 主键索引：`pk_user_id`。
   - 唯一索引：`uk_email`。
   - 普通索引：`idx_username`。

3. **SQL 编写**  
   - 避免 `SELECT *`，明确列出所需字段。
   - 示例：
     ```sql
     SELECT id, name, email FROM users WHERE status = 'active';
     ```

### 5.2 数据治理
1. **生命周期管理**  
   - ODS 层保留 3 年，DWD/DWS 层长期存储。
   - 定期清理历史数据，按年份分区。

2. **数据权限**  
   - ODS 层：仅 ETL 任务可写。
   - DWD/DWS 层：按业务线分配权限。
   - ADS 层：开放查询权限，禁止直接修改。

---

## 六、版本控制与提交规范

### 6.1 分支管理
- **分支类型**  
  | 分支类型 | 用途 | 生命周期 |
  |----------|------|----------|
  | `master` | 生产代码 | 永久 |
  | `dev` | 开发分支 | 永久 |
  | `feature-xxx` | 功能开发 | 合并后删除 |
  | `bugfix-xxx` | 修复紧急问题 | 合并后删除 |

### 6.2 提交规范
1. **Commit 信息格式**  
   ```text
   feat: 新增用户登录接口
   fix: 修复订单状态更新异常
   docs: 更新 README 文档
   ```

2. **提交内容要求**  
   - 每次提交必须是一个独立功能单元。
   - 禁止提交未完成代码或无意义修改（如 `fix: update`）。

3. **Tag 管理**  
   - 使用语义化版本号：`v1.2.3`。
   - 发布时打 Tag，并保留历史记录。

---

## 七、实施建议

### 7.1 工具支持
1. **IDE 配置**  
   - IntelliJ IDEA：启用 **Google Java Code Style**。
   - WebStorm：集成 **ESLint** 和 **Prettier** 自动格式化。

2. **代码检查**  
   - 使用 **SonarQube** 扫描代码质量问题。
   - 定期运行单元测试，覆盖率不低于 80%。

### 7.2 团队协作
- **代码审查**  
  - 所有 PR 必须通过至少一位同事的审核。
  - 重点检查命名、注释和边界条件处理。

- **文档同步**  
  - 使用 `README.md` 描述项目结构和启动步骤。
  - `CHANGELOG.md` 记录版本更新内容，如：
    ```markdown
    ## v1.2.0 - 2025-05-15
    ### Added
    - 新增用户权限模块
    ### Fixed
    - 修复登录接口的 SQL 注入漏洞
    ```

---

## 八、总结

一套**具备可行性的开发规范**需要满足以下条件：
1. **明确性**：规则清晰，避免歧义。
2. **可操作性**：通过工具（如 IDE 插件、CI/CD 流程）自动化执行。
3. **灵活性**：根据团队规模和项目需求调整细节。
4. **持续优化**：定期回顾规范，结合新技术和团队反馈迭代改进。

通过以上规范，团队可以显著提升代码质量、降低维护成本，并为项目长期发展奠定基础。规范不是束缚，而是团队协作的共同语言。只有在实践中不断验证和调整，才能真正发挥规范的价值。

**参考**  
- 《阿里 Java 开发手册（黄山版）》  
- 《Google Java Style Guide》  
- 《Spring Boot 最佳实践》
