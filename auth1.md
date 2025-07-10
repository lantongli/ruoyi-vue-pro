# 芋道(yudao)项目权限管理设计调研报告

## 项目概述

芋道(yudao)项目是一个基于Spring Boot 2.7.18的企业级脚手架项目，采用模块化架构设计，具备完善的权限管理体系。项目版本为2.6.0-jdk8-SNAPSHOT，支持多租户、多层级权限控制。

## 一、权限设计层级分析

### 1.1 四层权限架构设计

芋道项目的权限管理设计了**四个层级**：

#### 第一层：应用层权限(Application Level)
- **功能菜单权限**：基于菜单树结构的功能模块访问控制
- **API接口权限**：基于注解的后端接口访问控制
- **前端按钮权限**：基于权限标识的UI组件显示控制

#### 第二层：数据范围权限(Data Scope Level)  
基于`DataScopeEnum`枚举定义了5种数据权限范围：
```java
ALL(1),              // 全部数据权限
DEPT_CUSTOM(2),      // 指定部门数据权限
DEPT_ONLY(3),        // 部门数据权限
DEPT_AND_CHILD(4),   // 部门及以下数据权限
SELF(5)              // 仅本人数据权限
```

#### 第三层：租户层权限(Tenant Level)
- **租户隔离**：基于tenant_id实现数据完全隔离
- **租户套餐**：通过TenantPackage控制租户可使用的功能模块
- **跨层级隔离**：DB、Redis、Web、Security、Job、MQ、Async全链路租户隔离

#### 第四层：字段级权限(Field Level)
- **敏感字段控制**：可配置字段级别的访问权限
- **数据脱敏**：支持敏感数据的脱敏处理

### 1.2 权限设计核心组件

```java
// 权限服务核心接口
public interface PermissionService {
    // 用户-角色关联管理
    void assignUserRole(Long userId, Set<Long> roleIds);
    
    // 角色-菜单关联管理  
    void assignRoleMenu(Long roleId, Set<Long> menuIds);
    
    // 角色-数据权限管理
    void assignRoleDataScope(Long roleId, Integer dataScope, Set<Long> dataScopeDeptIds);
    
    // 权限校验
    boolean hasAnyPermissions(Long userId, String... permissions);
    boolean hasAnyRoles(Long userId, String... roles);
}
```

## 二、角色与租户的区别分析

### 2.1 角色(Role)设计特点

#### 核心属性
```java
public class RoleDO extends TenantBaseDO {
    private Long id;           // 角色ID
    private String name;       // 角色名称
    private String code;       // 角色标识
    private Integer type;      // 角色类型(系统内置/自定义)
    private Integer dataScope; // 数据权限范围
    private Set<Long> dataScopeDeptIds; // 指定部门权限
}
```

#### 功能特点
- **功能导向**：主要控制用户能访问哪些功能模块和API接口
- **数据权限**：通过dataScope控制用户能看到哪些数据范围
- **租户内权限**：角色权限范围仅在当前租户内有效
- **多角色支持**：一个用户可以拥有多个角色，权限采用并集方式

### 2.2 租户(Tenant)设计特点

#### 核心属性
```java
public class TenantDO extends BaseDO {
    private Long id;              // 租户编号
    private String name;          // 租户名称
    private Long contactUserId;   // 联系人用户编号
    private Integer status;       // 租户状态
    private Long packageId;       // 租户套餐编号
    private LocalDateTime expireTime; // 过期时间
    private Integer accountCount; // 账号数量限制
}
```

#### 功能特点
- **数据隔离**：实现完全的数据隔离，不同租户间数据完全独立
- **功能套餐**：通过TenantPackage控制租户可使用的功能模块
- **资源限制**：控制租户的账号数量、过期时间等资源配额
- **全链路隔离**：从数据库到缓存、从消息队列到异步任务的全方位隔离

### 2.3 角色与租户的关系

```
租户(Tenant) 1:N 用户(User) N:M 角色(Role) N:M 菜单权限(Menu)
     |                |              |
     |                |              +-- 数据权限(DataScope)
     |                |
     |                +-- 部门(Dept)
     |
     +-- 租户套餐(TenantPackage) N:M 菜单权限(Menu)
```

**区别总结**：
- **租户**是**物理隔离**，不同租户间数据完全隔离
- **角色**是**逻辑权限**，在租户内部进行功能和数据权限控制
- **租户**决定"能使用哪些功能模块"，**角色**决定"能访问哪些具体功能和数据"

## 三、实际开发中的权限设定指南

### 3.1 租户层权限设定

#### 创建租户套餐
```java
// 1. 定义租户套餐，控制租户可使用的功能模块
TenantPackageDO packageDO = TenantPackageDO.builder()
    .name("标准版套餐")
    .status(CommonStatusEnum.ENABLE.getStatus())
    .menuIds(Set.of(1L, 2L, 3L)) // 指定可使用的菜单权限
    .build();
```

#### 创建租户
```java
// 2. 创建租户，绑定套餐
TenantDO tenantDO = TenantDO.builder()
    .name("XX公司")
    .packageId(packageDO.getId())
    .accountCount(100) // 限制账号数量
    .expireTime(LocalDateTime.now().plusYears(1)) // 设置过期时间
    .build();
```

### 3.2 角色层权限设定

#### 创建角色
```java
// 3. 在租户内创建角色
RoleDO roleDO = new RoleDO();
roleDO.setName("销售经理");
roleDO.setCode("SALES_MANAGER");
roleDO.setType(RoleTypeEnum.CUSTOM.getType());
roleDO.setDataScope(DataScopeEnum.DEPT_AND_CHILD.getScope()); // 可查看本部门及下级部门数据
roleDO.setTenantId(tenantId);
```

#### 分配菜单权限
```java
// 4. 为角色分配菜单权限
Set<Long> menuIds = Set.of(100L, 101L, 102L); // 销售相关菜单
permissionService.assignRoleMenu(roleDO.getId(), menuIds);
```

#### 分配数据权限
```java
// 5. 设置数据权限范围
permissionService.assignRoleDataScope(
    roleDO.getId(), 
    DataScopeEnum.DEPT_CUSTOM.getScope(), 
    Set.of(10L, 11L) // 指定可访问的部门ID
);
```

### 3.3 用户层权限设定

#### 创建用户并分配角色
```java
// 6. 创建用户
AdminUserDO userDO = AdminUserDO.builder()
    .username("zhangsan")
    .nickname("张三")
    .deptId(10L) // 所属部门
    .tenantId(tenantId) // 所属租户
    .build();

// 7. 为用户分配角色
Set<Long> roleIds = Set.of(roleDO.getId());
permissionService.assignUserRole(userDO.getId(), roleIds);
```

### 3.4 数据权限自动控制

#### 启用数据权限
```java
// 8. 配置数据权限规则(框架自动处理)
@Configuration
public class DataPermissionConfiguration {
    
    @Bean
    public DeptDataPermissionRule deptDataPermissionRule(PermissionCommonApi permissionApi) {
        DeptDataPermissionRule rule = new DeptDataPermissionRule(permissionApi);
        // 为需要数据权限控制的实体添加部门字段映射
        rule.addDeptColumn(OrderDO.class); // 订单表按部门过滤
        rule.addUserColumn(OrderDO.class);  // 订单表按用户过滤
        return rule;
    }
}
```

#### 使用数据权限注解
```java
// 9. 在业务方法上使用数据权限
@RestController
public class OrderController {
    
    @DataPermission(enable = true) // 启用数据权限
    @GetMapping("/list")
    public CommonResult<List<OrderRespVO>> getOrderList() {
        // 框架会自动在SQL中添加数据权限WHERE条件
        // 如：WHERE dept_id IN (10, 11) OR user_id = 1001
        return orderService.getOrderList();
    }
}
```

### 3.5 API权限控制

#### 使用Spring Security注解
```java
@RestController  
public class UserController {
    
    // 方式1：基于权限标识控制
    @PreAuthorize("@ss.hasPermission('system:user:create')")
    @PostMapping("/create")
    public CommonResult<Long> createUser() {
        // 只有拥有'system:user:create'权限的用户才能访问
    }
    
    // 方式2：基于角色控制
    @PreAuthorize("@ss.hasRole('ADMIN')")
    @DeleteMapping("/delete")
    public CommonResult<Boolean> deleteUser() {
        // 只有拥有'ADMIN'角色的用户才能访问
    }
}
```

### 3.6 前端权限控制

#### Vue组件权限控制
```javascript
// 10. 前端权限指令使用
<template>
  <div>
    <!-- 基于权限显示按钮 -->
    <el-button v-hasPermi="['system:user:create']">新增用户</el-button>
    
    <!-- 基于角色显示内容 -->
    <div v-hasRole="['ADMIN', 'SUPER_ADMIN']">
      管理员专用内容
    </div>
  </div>
</template>
```

### 3.7 开发最佳实践

#### 权限设计原则
1. **最小权限原则**：默认无权限，按需授权
2. **职责分离**：不同角色有明确的职责边界
3. **租户隔离**：确保租户间数据完全隔离
4. **权限继承**：合理利用部门层级和角色继承关系

#### 常见开发场景
```java
// 场景1：多租户SaaS应用
// 租户A的管理员只能管理租户A的用户，无法看到租户B的数据

// 场景2：企业内部OA系统  
// 销售经理可以查看自己部门及下级部门的销售数据
// 普通销售只能查看自己的销售数据

// 场景3：数据权限动态控制
@DataPermission(enable = true)
public List<OrderDO> getOrderList() {
    // 系统会根据当前用户的数据权限自动添加WHERE条件
    // 管理员：无WHERE条件(查看全部)
    // 部门经理：WHERE dept_id IN (10,11,12) 
    // 普通员工：WHERE user_id = 1001
    return orderMapper.selectList();
}
```

## 四、技术架构总结

### 4.1 核心技术栈
- **权限框架**：Spring Security + 自定义权限注解
- **数据权限**：MyBatis-Plus + JSqlParser SQL解析
- **多租户**：基于tenant_id的行级数据隔离
- **缓存策略**：Redis缓存权限信息，提升性能

### 4.2 权限流转链路
```
用户登录 → 加载用户角色 → 获取角色权限 → 缓存权限信息 → 
请求拦截 → 权限校验 → SQL权限过滤 → 返回结果
```

芋道项目通过这套完善的权限管理体系，能够满足从简单的功能权限控制到复杂的多租户、多层级数据权限控制的各种需求，为企业级应用提供了安全可靠的权限解决方案。