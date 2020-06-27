# flowable-ui
flowable集成

Flowable官方提供的五个war包

| starter            | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| `flowable-modeler` | 让具有建模权限的用户可以创建流程模型、表单、选择表与应用定义。 |
| `flowable-idm`     | 身份管理应用。为所有Flowable UI应用提供单点登录认证功能，并且为拥有IDM管理员权限的用户提供了管理用户、组与权限的功能 |
| `flowable-task`    | 运行时任务应用。提供了启动流程实例、编辑任务表单、完成任务，以及查询流程实例与任务的功能。 |
| `flowable-admin`   | 管理应用。让具有管理员权限的用户可以查询BPMN、DMN、Form及Content引擎，并提供了许多选项用于修改流程实例、任务、作业等。管理应用通过REST API连接至引擎，并与Flowable Task应用及Flowable REST应用一同部署。 |
| `flowable-rest`    | Flowable页面包含的常用REST API                               |

以官方提供的war包为基准，集成以上四个默认页面对应的REST接口。

### 开始集成

#### 后端集成

- maven

~~~xml
<dependency>
  <groupId>org.flowable</groupId>
  <artifactId>flowable-spring-boot-starter-rest</artifactId>
  <version>${flowable.version}</version>
</dependency>
<!-- flowable UI集成 -->
<dependency>
  <groupId>org.flowable</groupId>
  <artifactId>flowable-ui-modeler-conf</artifactId>
  <version>${flowable.version}</version>
</dependency>
<dependency>
  <groupId>org.flowable</groupId>
  <artifactId>flowable-ui-task-conf</artifactId>
  <version>${flowable.version}</version>
</dependency>
<dependency>
  <groupId>org.flowable</groupId>
  <artifactId>flowable-ui-admin-conf</artifactId>
  <version>${flowable.version}</version>
</dependency>
<dependency>
  <groupId>org.flowable</groupId>
  <artifactId>flowable-ui-idm-conf</artifactId>
  <version>${flowable.version}</version>
</dependency>
~~~

flowable.version 采用6.4.1

**注：** 

> flowable-ui-xxx-conf: UI独立于业务外的配置
> flowable-ui-xxx-logic: UI的业务逻辑
> flowable-ui-xxx-rest: 提供常用的操作REST API

以上就是集成Flowable UI的基础包了。

由于flowable-ui-xxx-conf中包中，每个中都有对应自己的如`ApplicationConfiguration`等配置类，所以如果用默认配置会出现

~~~
conflicts with existing, non-compatible bean definition of same name and class
~~~

类似的错误信息，提示在Spring容器中存在两个相同名称的Bean。

所以不能使用默认的配置类，只能自己去编写配置类，完成flowable-ui-xxx-conf的配置。

- AppDispatcherServletConfiguration

```java
@Configuration
@ComponentScan(value = {
        "org.flowable.ui.admin.rest",
        "org.flowable.ui.task.rest.runtime",
        "org.flowable.ui.idm.rest.app",
        "org.flowable.ui.common.rest.exception",
        "org.flowable.ui.modeler.rest.app",
        "org.flowable.ui.common.rest"},
        excludeFilters = {
                @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, value = RemoteAccountResource.class),
                @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, value = StencilSetResource.class),
                @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, value = EditorUsersResource.class),
                @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, value = EditorGroupsResource.class)
        })
@EnableAsync
public class AppDispatcherServletConfiguration implements WebMvcRegistrations
// 下面类内容省略，可参考org.flowable.ui.xxx.servlet.AppDispatcherServletConfiguration类
```

<font color="red">**注:**</font> 通过`StencilSetResource`类汉化流程设计器(其实没必要重写类，只需要将stencilset目录下的stencilset_bpmn.json文件内容汉化即可)

- ApplicationConfiguration

~~~java
@Configuration
@EnableConfigurationProperties({FlowableIdmAppProperties.class, FlowableModelerAppProperties.class, FlowableAdminAppProperties.class})
@ComponentScan(
    basePackages = {
        "org.flowable.ui.admin.repository",
        "org.flowable.ui.admin.service",
        "org.flowable.ui.task.model.component",
        "org.flowable.ui.task.service.runtime",
        "org.flowable.ui.task.service.debugger",
        "org.flowable.ui.idm.conf",
        "org.flowable.ui.idm.security",
        "org.flowable.ui.idm.service",
        "org.flowable.ui.modeler.repository",
        "org.flowable.ui.modeler.service",
        "org.flowable.ui.common.filter",
        "org.flowable.ui.common.service",
        "org.flowable.ui.common.repository",
        "org.flowable.ui.common.security",
        "org.flowable.ui.common.tenant"
    },
    excludeFilters = {
        @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, value = org.flowable.ui.idm.conf.ApplicationConfiguration.class)
    }
)
~~~

<font color="red">**注:**</font> 通过@ComponentScan注解，扫描需要的包及其接口。

- DatabaseConfiguration

~~~java
/**
 * 重写flowable-ui-xxx-conf 中的 DatabaseConfiguration 类,
 * 包括:flowable-ui-modeler-conf和flowable-ui-admin-conf 的DatabaseConfiguration
 */
@Configuration
@EnableTransactionManagement
public class DatabaseConfiguration {
    private static final Logger LOGGER = LoggerFactory.getLogger(DatabaseConfiguration.class);
    @Bean
    public Liquibase modelerLiquibase(DataSource dataSource) {
        Liquibase liquibase = null;
        try {
            DatabaseConnection connection = new JdbcConnection(dataSource.getConnection());
            Database database = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(connection);
            database.setDatabaseChangeLogTableName("ACT_DE_" + database.getDatabaseChangeLogTableName());
            database.setDatabaseChangeLogLockTableName("ACT_DE_" + database.getDatabaseChangeLogLockTableName());
            liquibase = new Liquibase("META-INF/liquibase/flowable-modeler-app-db-changelog.xml", new ClassLoaderResourceAccessor(), database);
            liquibase.update("flowable");
            return liquibase;
        } catch (Exception e) {
            throw new InternalServerErrorException("Error creating liquibase database", e);
        } finally {
            closeDatabase(liquibase);
        }
    }
    @Bean
    public Liquibase adminLiquibase(DataSource dataSource) {
        LOGGER.debug("Configuring Liquibase");
        Liquibase liquibase = null;
        try {
            DatabaseConnection connection = new JdbcConnection(dataSource.getConnection());
            Database database = DatabaseFactory.getInstance().findCorrectDatabaseImplementation(connection);
            database.setDatabaseChangeLogTableName("ACT_ADM_" + database.getDatabaseChangeLogTableName());
            database.setDatabaseChangeLogLockTableName("ACT_ADM_" + database.getDatabaseChangeLogLockTableName());
            liquibase = new Liquibase("META-INF/liquibase/flowable-admin-app-db-changelog.xml", new ClassLoaderResourceAccessor(), database);
            liquibase.update("flowable");
            return liquibase;
        } catch (Exception e) {
            throw new InternalServerErrorException("Error creating liquibase database");
        } finally {
            closeDatabase(liquibase);
        }
    }
    private void closeDatabase(Liquibase liquibase) {
        if (liquibase != null) {
            Database database = liquibase.getDatabase();
            if (database != null) {
                try {
                    database.close();
                } catch (DatabaseException e) {
                    LOGGER.warn("Error closing database", e);
                }
            }
        }
    }
}
~~~

<font color="red">**注:**</font> 创建Modeler的模型存储表和Admin首页的服务配置表。

- TaskUserCacheImpl

~~~java
/**
 * 重写 {@link org.flowable.ui.task.service.idm.UserCacheImpl} 类,避免启动时和 {@link org.flowable.ui.idm.service.UserCacheImpl} 命名冲突
 * Cache containing User objects to prevent too much DB-traffic (users exist separately from the Flowable tables, they need to be fetched afterward one by one to join with those entities).
 * <p>
 *
 * @author Frederik Heremans
 * @author Joram Barrez
 * @author Filip Hrisafov
 */
@Service
public class TaskUserCacheImpl implements UserCache 
~~~

<font color="red">**注:**</font> 重写 {@link org.flowable.ui.task.service.idm.UserCacheImpl} 类,避免启动时和 {@link org.flowable.ui.idm.service.UserCacheImpl} 命名冲突

- org.flowable.ui.admin.service.engine.CmmnTaskService

~~~java
package org.flowable.ui.admin.service.engine;
/**
 * 覆盖jar包中的CmmnTaskService的定义，修改Bean的定义名称,避免和org.flowable.cmmn.api.CmmnTaskService命名冲突
 * Service for invoking Flowable REST services.
 */
@Service("adminCmmnTaskService")
public class CmmnTaskService 
~~~

flowable的jar包导致的冲突问题解决完毕之后，以下是application.yml中需要对Flowable做的配置。

~~~yml
spring:
  profiles:
    active: dev
  application:
    name: flowable
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
server:
  tomcat:
    uri-encoding: UTF-8
  port: 8080
  servlet:
    context-path: /${spring.application.name}
  compression:
    mime-types: application/json,application/xml,text/html,text/xml,text/plain
    enabled: true
    min-response-size: 4096

flowable:
  labelFontName: 宋体
  activityFontName: 宋体
  annotationFontName: 宋体
  rest:
    app:
      authentication-mode: verify-privilege
  process:
    definition-cache-limit: 1
  idm:
    app:
      admin:
        password: test
        user-id: admin
        first-name: admin
        last-name: admin
  common:
    app:
      role-prefix:
      idm-url: http://localhost:${server.port}/${spring.application.name}
      idm-admin:
        user: admin
        password: test
  xml:
    encoding: UTF-8
  modeler:
    app:
      rest-enabled: true
      deployment-api-url: http://localhost:${server.port}/${spring.application.name}/app-api
  rest-api-enabled: true
  admin:
    app:
      security:
        encryption:
          credentials-secret-spec: 9FGl73ngxcOoJvmL
          credentials-i-v-spec: j8kdO2hejA9lKmm6
      server-config:
        app:
          context-root: ${spring.application.name}
          password: test
          server-address: http://localhost
          user-name: admin
          port: ${server.port}
          rest-root: app-api
          name: Flowable App app
          description: Flowable App REST config
        process:
          context-root: ${spring.application.name}
          server-address: http://localhost
          password: test
          user-name: admin
          rest-root: process-api
          port: ${server.port}
          name: Flowable Process app
          description: Flowable Process REST config
        form:
          context-root: ${spring.application.name}
          server-address: http://localhost
          password: test
          user-name: admin
          port: ${server.port}
          rest-root: form-api
          name: Flowable Form app
          description: Flowable Form REST config
        dmn:
          context-root: ${spring.application.name}
          server-address: http://localhost
          password: test
          user-name: admin
          port: ${server.port}
          rest-root: dmn-api
          name: Flowable DMN app
          description: Flowable DMN REST config
        cmmn:
          context-root: ${spring.application.name}
          password: test
          server-address: http://localhost
          user-name: admin
          port: ${server.port}
          rest-root: cmmn-api
          name: Flowable CMMN app
          description: Flowable CMMN REST config
        content:
          context-root: ${spring.application.name}
          server-address: http://localhost
          password: test
          user-name: admin
          rest-root: content-api
          port: ${server.port}
          name: Flowable Content app
          description: Flowable Content REST config
  database-schema-update: true
management:
  endpoint:
    health:
      roles: access-admin
      show-details: when_authorized
  endpoints:
    jmx:
      unique-names: true
# MyBatis配置比较重要，手动去扫描Flowable默认的Mapper.xml，以及设置字段类型
mybatis:
  mapper-locations:
    - classpath:/META-INF/admin-mybatis-mappings/*.xml
    - classpath:/META-INF/modeler-mybatis-mappings/*.xml
  configuration-properties:
    prefix:
    boolValue: TRUE
    blobType: BLOB
~~~

以下是application-dev.yml文件的数据源配置

~~~yml
spring:
  datasource:
    druid:
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://192.168.209.128:3306/flowableui?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true&useSSL=false
      password: 123456
      username: root

~~~
至此后端集成完毕。

#### 静态资源集成

一开始准备将官方提供的页面集成到后端工程来，但是由于modeler页面是直接放到static目录下的，而其他三个页面只能自己建目录存放（idm, admin, task），导致访问modeler时是以后端项目请求路径为根访问路径的，在modeler请求后端接口时，请求路径没有问题；可访问idm、admin、task时，则必须加上前缀路径，导致请求后端接口时，不管将前端请求后端的路径改为相对路径还是绝对路径，最终请求路径都存在问题。
通过继承WebMvcConfigurationSupport（springboot2.x后用此类） 类后重写addResourceHandlers实现静态文件映射，而不影响后端接口。

~~~java
    //静态资源配置
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/p/**")
                .addResourceLocations("classpath:/templates/page/");
        registry.addResourceHandler("/a/**")
                .addResourceLocations("classpath:/templates/assets/");
        registry.addResourceHandler("/**")
                .addResourceLocations("classpath:/static/");
        registry.addResourceHandler("/idm/**")
                .addResourceLocations("classpath:/idm/");
        registry.addResourceHandler("/admin/**")
                .addResourceLocations("classpath:/admin/");
        registry.addResourceHandler("/task/**")
                .addResourceLocations("classpath:/task/");
    }
~~~

然后将admin、idm、task的静态资源放到resources下，modeler页面是直接放到static目录下

#### Flowable与SpringBoot版本坑

`flowable-ui-modeler-conf`中`AppDispatcherServletConfiguration`配置类，实现了`WebMvcRegistrations`接口，SpringBoot1.x中，WebMvcRegistrations在`spring-boot-autoconfigure`中的web包下，而在SpringBoot2.x中，WebMvcRegistrations被移到web.servlet下了，所以在使用默认类时，SpringBoot版本必须是2.x。

#### Flowable官方包冲突

由于是根据flowable-modeler、flowable-idm、flowable-task、flowable-admin来集成的，所以后端基础包也是需要这对应的四个，故直接集成了flowable-ui-modeler-conf、flowable-ui-idm-conf、flowable-ui-task-conf、flowable-ui-admin-conf，但是由于这四个包中存在各自的自动配置类，而且在modeler和admin中，自动配置类又和他自己的业务逻辑有关联，而且存在多个类命名相同的Spring Bean，导致Spring上下文由于出现相同名称的Bean而初始化失败，但是默认配置有些又不能不用，最终的解决办法就是去除原来所有的默认配置，将冲突的通过@ComponentScan的excludeFilters属性移除掉，然后自己去自定义配置。

#### 登录权限问题

flowable-ui-modeler-conf、flowable-ui-idm-conf、flowable-ui-task-conf、flowable-ui-admin-conf这四个包中都存在SecurityConfiguration配置类，在集成的过程中，modeler-conf其实我们希望的是以idm的登录为主，其他模块自己的安全校验配置不需要，但是实际上由于其他模块存在安全配置以及默认的拦截器，导致idm自己的安全验证不能通过。

解决方式是我们排除掉认证，然后自定义获取用户接口，内置用户

启动类排除认证

~~~java
@SpringBootApplication(exclude = {SecurityAutoConfiguration.class})
~~~

获取用户接口、内置用户

~~~java
@RestController
@RequestMapping("/login")
public class FlowableController {

    /**
     * 获取默认的管理员信息
     * @return
     */
    @RequestMapping(value = "/rest/account", method = RequestMethod.GET, produces = "application/json")
    public UserRepresentation getAccount() {
        //内置用户
        User user = new UserEntityImpl();
        user.setId("admin");
        SecurityUtils.assumeUser(user);
        //获取用户信息
        UserRepresentation userRepresentation = new UserRepresentation();
        userRepresentation.setId("admin");
        userRepresentation.setEmail("admin@flowable.org");
        userRepresentation.setFullName("Administrator");
        userRepresentation.setFirstName("Administrator");
        List<String> privileges = new ArrayList<String>();
        privileges.add(DefaultPrivileges.ACCESS_MODELER);
        privileges.add(DefaultPrivileges.ACCESS_IDM);
        privileges.add(DefaultPrivileges.ACCESS_ADMIN);
        privileges.add(DefaultPrivileges.ACCESS_TASK);
        privileges.add(DefaultPrivileges.ACCESS_REST_API);
        userRepresentation.setPrivileges(privileges);
        return userRepresentation;
    }
}
~~~

修改static目录下（原modeler模块）scripts/configuration/url-config.js，getAccountUrl的url改为刚才修改的

~~~js
    getAccountUrl: function () {
        return FLOWABLE.CONFIG.contextRoot + '/login/rest/account';
    },
~~~

修改admin目录下scripts/app.js，scripts/services.js 中  /app/rest/account 改为/login/rest/account


### 页面集成


之前在resources目录下新建idm、task、admin目录，将flowable-idm.war、flowable-task.war、flowable-admin.war中的flowable-xxx/WEB-INF/classes/static下的内容分别复制到对应目录下

统一请求路径修改：

在static/scripts/app-cfg.js文件中

~~~js
'use strict';
var FLOWABLE = FLOWABLE || {};
var pathname = window.location.pathname.replace(/^(\/[^\/]*)(\/.*)?$/, '$1').replace(/\/$/, '');
FLOWABLE.CONFIG = {
	'onPremise' : true,
	'contextRoot' : pathname,
	'webContextRoot' : pathname,
	'datesLocalization' : false
};
~~~

在resources/idm/scripts/app-cfg.js文件中，修改webContextRoot

```js
'use strict';
var FLOWABLE = FLOWABLE || {};
var pathname = window.location.pathname.replace(/^(\/[^\/]*)(\/.*)?$/, '$1').replace(/\/$/, '');
FLOWABLE.CONFIG = {
	'onPremise' : true,
	'contextRoot' : pathname,
	'webContextRoot' : pathname + "/idm",
	'datesLocalization' : false
};
```

在resources/task/scripts/app-cfg.js文件中，修改webContextRoot

```js
'use strict';
var FLOWABLE = FLOWABLE || {};
var pathname = window.location.pathname.replace(/^(\/[^\/]*)(\/.*)?$/, '$1').replace(/\/$/, '');
FLOWABLE.CONFIG = {
	'onPremise' : true,
	'contextRoot' : pathname,
	'webContextRoot' : pathname + "/task",
	'datesLocalization' : false
};
```

admin和其他的不一样，采取的方式是在resources/admin/scripts/config.js文件中，增加上面类似配置。

~~~js
var pathname = window.location.pathname.replace(/^(\/[^\/]*)(\/.*)?$/, '$1').replace(/\/$/, '');
FlowableAdmin.webContextRoot = pathname;
~~~

然后访问时增加相应的静态文件路
至此前后端基本集成完毕。

执行后端SpringBoot启动类，启动后端服务，初始化会新建需要的表。


此时通过以下四个链接，即可访问Flowable官方提供的四个操作页面，modelers账号是我们内置的账号,其余模块需要用户信息，访问时需要放开SecurityAutoConfiguration.class

| 模块             | 地址                                                    |
| ---------------- | ------------------------------------------------------- |
| flowable-modeler | http://127.0.0.1:8080/flowable/                         |
| flowable-task    | http://127.0.0.1:8080/flowable/task/workflow/index.html |
| flowable-idm     | http://127.0.0.1:8080/flowable/idm/index.html           |
| flowable-admin   | http://127.0.0.1:8080/flowable/admin/index.html         |