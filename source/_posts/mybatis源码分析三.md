---
layout: '[post]'
title: mybatis源码分析三
date: 2019-10-30 00:57:38
tags: mybatis
categories: mybatis
---
# mybatis源码解析 --- mybatis由浅入深
<!--more-->
##  mybatis的本质

### 1、执行过程
*   从上一篇已经学会了如何使用mybatis,从代码中可以看出mybatis执行步骤主要分为以下几步：
    ```
        private static void readXmlMethod(){
            try{
                // 获取全局配置路径
                String resource = "mybatis-config.xml";
                // 使用Resources.getResourceAsStream加载全局配置文件
                InputStream inputStream = Resources.getResourceAsStream(resource);
                // 调用SqlSessionFactoryBuilder()方法构建读取到的数据源信息
                SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
                // 调用SqlSessionFactory.openSession()方法获取到SqlSession对象
                SqlSession session = sqlSessionFactory.openSession();
                // 加载UserMapper到session Mapper中
                UserMapper userMapper = session.getMapper(UserMapper.class);
                // 调用方法返回User对象
                User user = userMapper.selectUser(1);
                System.out.println(user);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    ```
    ```
        1、获取SqlSessionFactory:
            通过new SqlSessionFactoryBuilder().build(inputStream)的方式获取SqlSessionFactory 
        2、获取SqlSession
            调用SqlSessionFactory.openSession()方法获取到SqlSession
        3、获取对应接口类的Mapper
            session.getMapper(UserMapper.class)
        4、实现Mapper类的crud方法
            userMapper.selectUser(1);
    ```
### 2、mybatis是如何获取数据源
通过Debug模式来查看mybatis是如何获取数据源
![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(1).png)
1.  由上图可以看出SqlSessionFactory加载了配置文件里的数据源信息
    是如何获取到数据源信息的？带着疑问进一步debug
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(2).png)
    ```
        org.apache.ibatis.session.SqlSessionFactoryBuilder#build(java.io.InputStream, java.lang.String, java.util.Properties)
    ```
    ```
        SqlSessionFactoryBuilder的build方法中使用了XMLConfigBuilder来加载配置文件信息
        public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
            try {
              XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
              return build(parser.parse());
            } catch (Exception e) {
              throw ExceptionFactory.wrapException("Error building SqlSession.", e);
            } finally {
              ErrorContext.instance().reset();
              try {
                inputStream.close();
              } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
              }
            }
        }
    ```
    ```
        org.apache.ibatis.builder.xml.XMLConfigBuilder
    ```
    parser.parse()返回全局配置类org.apache.ibatis.session.Configuration 
    ```
        public Configuration parse() {
           if (parsed) {
             throw new BuilderException("Each XMLConfigBuilder can only be used once.");
           }
           parsed = true;
           parseConfiguration(parser.evalNode("/configuration"));
           return configuration;
         }
         
         private void parseConfiguration(XNode root) {
             try {
               //issue #117 read properties first
               propertiesElement(root.evalNode("properties"));
               Properties settings = settingsAsProperties(root.evalNode("settings"));
               loadCustomVfs(settings);
               loadCustomLogImpl(settings);
               typeAliasesElement(root.evalNode("typeAliases"));
               pluginElement(root.evalNode("plugins"));
               objectFactoryElement(root.evalNode("objectFactory"));
               objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
               reflectorFactoryElement(root.evalNode("reflectorFactory"));
               settingsElement(settings);
               // read it after objectFactory and objectWrapperFactory issue #631
               environmentsElement(root.evalNode("environments"));
               databaseIdProviderElement(root.evalNode("databaseIdProvider"));
               typeHandlerElement(root.evalNode("typeHandlers"));
               mapperElement(root.evalNode("mappers"));
             } catch (Exception e) {
               throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
             }
           }
    ```
2.  可以看的出parseConfiguration(XNode root)解析xml对应xml配置文件的节点名称
    ```
        org.apache.ibatis.session.Configuration
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(3).png)
    查看root参数:其实就是加载了配置文件mybatis-config.xml
    ```
            <configuration>
            <properties>
            <property name="driver" value="com.mysql.jdbc.Driver"/>
            <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useUnicode=true&characterEncoding=utf8&useSSL=false"/>
            <property name="username" value="root"/>
            <property name="password" value="root"/>
            </properties>
            <environments default="development">
            <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
            <property name="driver" value="${driver}"/>
            <property name="url" value="${url}"/>
            <property name="username" value="${username}"/>
            <property name="password" value="${password}"/>
            </dataSource>
            </environment>
            </environments>
            <mappers>
            <mapper resource="UserMapper.xml"/>
            </mappers>
            </configuration>
    
        ```
    查看配置文件xml节点可知,需要查看的是:
    environmentsElement(root.evalNode("environments"));这个方法
    ```
        private void environmentsElement(XNode context) throws Exception {
            if (context != null) {
              if (environment == null) {
                environment = context.getStringAttribute("default");
              }
              for (XNode child : context.getChildren()) {
                String id = child.getStringAttribute("id");
                if (isSpecifiedEnvironment(id)) {
                  TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                  DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                  DataSource dataSource = dsFactory.getDataSource();
                  Environment.Builder environmentBuilder = new Environment.Builder(id)
                      .transactionFactory(txFactory)
                      .dataSource(dataSource);
                  configuration.setEnvironment(environmentBuilder.build());
                }
              }
            }
        }
        查看conttext参数可以得到：其实就是加载了mybatis-config.xml中的environments节点
        <environments default="development">
        <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
        <property name="driver" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useUnicode=true&characterEncoding=utf8&useSSL=false"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
        </dataSource>
        </environment>
        </environments>

    ```
3.  由dataSourceElement(child.evalNode("dataSource"));来解析xml配置文件中的dataSource节点
    得到javax.sql.DataSource类，保存到org.apache.ibatis.mapping.Environment中
    ```
        public final class Environment {
          private final String id;
          private final TransactionFactory transactionFactory;
          private final DataSource dataSource;
          .....
        }
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(4).png)
    ```
        private void environmentsElement(XNode context) throws Exception {
            if (context != null) {
              if (environment == null) {
                environment = context.getStringAttribute("default");
              }
              for (XNode child : context.getChildren()) {
                String id = child.getStringAttribute("id");
                if (isSpecifiedEnvironment(id)) {
                  TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                  DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                  DataSource dataSource = dsFactory.getDataSource();
                  Environment.Builder environmentBuilder = new Environment.Builder(id)
                      .transactionFactory(txFactory)
                      .dataSource(dataSource);
                  configuration.setEnvironment(environmentBuilder.build());
                }
              }
            }
        }
        查看context可得：
        <dataSource type="POOLED">
        <property name="driver" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useUnicode=true&characterEncoding=utf8&useSSL=false"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
        </dataSource>

    ```
    ```
        (这是上面的代码。只是单独拿出来说明)
        获取到的数据源信息保存到全局配置信息类org.apache.ibatis.session.Configuration
        configuration.setEnvironment(environmentBuilder.build());
    ```
4.  自此,可以得获取数据源的基本步骤：
    ```
       >   org.apache.ibatis.session.SqlSessionFactoryBuilder#build
       >   org.apache.ibatis.builder.xml.XMLConfigBuilder#parse
       >   org.apache.ibatis.builder.xml.XMLConfigBuilder#parseConfiguration
       >   org.apache.ibatis.builder.xml.XMLConfigBuilder#environmentsElement
       >   org.apache.ibatis.builder.xml.XMLConfigBuilder#dataSourceElement
       >   org.apache.ibatis.session.Configuration#setEnvironment
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(5).png)

### 3、mybatis是如何获取执行语句
还是通过Debug模式来查看mybatis是如何获取执行语句
1.  解析UserMapper.xml
    根据前面获取数据源的思路，需要得知是由哪个类来解析UserMapper.xml的
    ```
        <?xml version="1.0" encoding="UTF-8" ?>
                <!DOCTYPE mapper
                        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
                        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
        <mapper namespace="com.github.sdcxy.mybatis.mapper.UserMapper">
        <select id="selectUser" resultType="com.github.sdcxy.entity.User">
            select * from sys_user where id = #{id}
          </select>
        </mapper>
    ```
    由于在配置文件中配置了UserMapper.xml的信息
    ```
        <mappers>
            <mapper resource="UserMapper.xml"/>
        </mappers>
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(6)-解析mappers.png)
2.  由org.apache.ibatis.builder.xml.XMLConfigBuilder#mapperElement来加载mappers节点
    ```
        org.apache.ibatis.builder.xml.XMLConfigBuilder#mapperElement
    ```
    ```
        private void mapperElement(XNode parent) throws Exception {
            if (parent != null) {
              for (XNode child : parent.getChildren()) {
                if ("package".equals(child.getName())) {
                  String mapperPackage = child.getStringAttribute("name");
                  configuration.addMappers(mapperPackage);
                } else {
                  String resource = child.getStringAttribute("resource");
                  String url = child.getStringAttribute("url");
                  String mapperClass = child.getStringAttribute("class");
                  if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                    mapperParser.parse();
                  } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                    mapperParser.parse();
                  } else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                  } else {
                    throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                  }
                }
              }
            }
        }
        
        查看context可得：
        <mappers>
            <mapper resource="UserMapper.xml"/>
        </mappers>
        
    ```
    ```
        根据源码或者官方文档可以得知加载mappers标签有4种方法
        分别为：package,name,reousrce,class
        其中package的优先级最高(代码由上至下原理)
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(7)-解析mappers4种方式.png)
3.  通过resources方式解析mybatis-config.xml中映射的UserMapper.xml
    因为配置文件使用的是resources的方式,所以只看resources这部分源码,
    其实这跟前面加载数据源的方式类型，也是通过读取流的方式将文件内容加载到内存中。
    ```
        if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
        }
    ```
    ```
        org.apache.ibatis.builder.xml.XMLMapperBuilder#parse
    ```
    可以看到对应的“/mapper”就是对应配置里的mapper标签
    ```
        public void parse() {
            if (!configuration.isResourceLoaded(resource)) {
              configurationElement(parser.evalNode("/mapper"));
              configuration.addLoadedResource(resource);
              bindMapperForNamespace();
            }
        
            parsePendingResultMaps();
            parsePendingCacheRefs();
            parsePendingStatements();
        }
    ```
    ```
        org.apache.ibatis.builder.xml.XMLMapperBuilder#configurationElement
    ```
    ```
        private void configurationElement(XNode context) {
            try {
              String namespace = context.getStringAttribute("namespace");
              if (namespace == null || namespace.equals("")) {
                throw new BuilderException("Mapper's namespace cannot be empty");
              }
              builderAssistant.setCurrentNamespace(namespace);
              cacheRefElement(context.evalNode("cache-ref"));
              cacheElement(context.evalNode("cache"));
              parameterMapElement(context.evalNodes("/mapper/parameterMap"));
              resultMapElements(context.evalNodes("/mapper/resultMap"));
              sqlElement(context.evalNodes("/mapper/sql"));
              buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
            } catch (Exception e) {
              throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
            }
        }
        查看context参数可得到：这跟Mapper.xml配置是一致的。
        <mapper namespace="com.github.sdcxy.mybatis.mapper.UserMapper">
        <select resultType="com.github.sdcxy.entity.User" id="selectUser">
            select * from sys_user where id = #{id}
          </select>
        </mapper>

    ```
4.  crud的解析方法：
    buildStatementFromContext(context.evalNodes("select|insert|update|delete"))
    ```
        org.apache.ibatis.builder.xml.XMLMapperBuilder#buildStatementFromContext(java.util.List<org.apache.ibatis.parsing.XNode>)
    ```
    可以得知mybatis其实使用一个list集合来保存这些crud则行语句
    ```
         <select resultType="com.github.sdcxy.entity.User" id="selectUser">
             select * from sys_user where id = #{id}
         </select>
    ```
    org.apache.ibatis.builder.xml.XMLStatementBuilder#parseStatementNode
    这个方法主要将select|insert|update|delete标签对应的属性进行赋值，
    最后保存到org.apache.ibatis.mapping.MappedStatement类中
    ```
        org.apache.ibatis.builder.MapperBuilderAssistant#addMappedStatement(java.lang.String, org.apache.ibatis.mapping.SqlSource, org.apache.ibatis.mapping.StatementType, org.apache.ibatis.mapping.SqlCommandType, java.lang.Integer, java.lang.Integer, java.lang.String, java.lang.Class<?>, java.lang.String, java.lang.Class<?>, org.apache.ibatis.mapping.ResultSetType, boolean, boolean, boolean, org.apache.ibatis.executor.keygen.KeyGenerator, java.lang.String, java.lang.String, java.lang.String, org.apache.ibatis.scripting.LanguageDriver, java.lang.String)
    ```
    ```
        public void parseStatementNode() {
            String id = context.getStringAttribute("id");
            String databaseId = context.getStringAttribute("databaseId");
        
            if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
              return;
            }
        
            String nodeName = context.getNode().getNodeName();
            SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
            boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
            boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
            boolean useCache = context.getBooleanAttribute("useCache", isSelect);
            boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);
        
            // Include Fragments before parsing
            XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
            includeParser.applyIncludes(context.getNode());
        
            String parameterType = context.getStringAttribute("parameterType");
            Class<?> parameterTypeClass = resolveClass(parameterType);
        
            String lang = context.getStringAttribute("lang");
            LanguageDriver langDriver = getLanguageDriver(lang);
        
            // Parse selectKey after includes and remove them.
            processSelectKeyNodes(id, parameterTypeClass, langDriver);
        
            // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
            KeyGenerator keyGenerator;
            String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
            keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
            if (configuration.hasKeyGenerator(keyStatementId)) {
              keyGenerator = configuration.getKeyGenerator(keyStatementId);
            } else {
              keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                  configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
                  ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
            }
        
            SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
            StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
            Integer fetchSize = context.getIntAttribute("fetchSize");
            Integer timeout = context.getIntAttribute("timeout");
            String parameterMap = context.getStringAttribute("parameterMap");
            String resultType = context.getStringAttribute("resultType");
            Class<?> resultTypeClass = resolveClass(resultType);
            String resultMap = context.getStringAttribute("resultMap");
            String resultSetType = context.getStringAttribute("resultSetType");
            ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
            if (resultSetTypeEnum == null) {
              resultSetTypeEnum = configuration.getDefaultResultSetType();
            }
            String keyProperty = context.getStringAttribute("keyProperty");
            String keyColumn = context.getStringAttribute("keyColumn");
            String resultSets = context.getStringAttribute("resultSets");
        
            builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
                fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
                resultSetTypeEnum, flushCache, useCache, resultOrdered,
                keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
        }
    ```
    ```
        public MappedStatement addMappedStatement(
              String id,
              SqlSource sqlSource,
              StatementType statementType,
              SqlCommandType sqlCommandType,
              Integer fetchSize,
              Integer timeout,
              String parameterMap,
              Class<?> parameterType,
              String resultMap,
              Class<?> resultType,
              ResultSetType resultSetType,
              boolean flushCache,
              boolean useCache,
              boolean resultOrdered,
              KeyGenerator keyGenerator,
              String keyProperty,
              String keyColumn,
              String databaseId,
              LanguageDriver lang,
              String resultSets) {
        
            if (unresolvedCacheRef) {
              throw new IncompleteElementException("Cache-ref not yet resolved");
            }
        
            id = applyCurrentNamespace(id, false);
            boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        
            MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
                .resource(resource)
                .fetchSize(fetchSize)
                .timeout(timeout)
                .statementType(statementType)
                .keyGenerator(keyGenerator)
                .keyProperty(keyProperty)
                .keyColumn(keyColumn)
                .databaseId(databaseId)
                .lang(lang)
                .resultOrdered(resultOrdered)
                .resultSets(resultSets)
                .resultMaps(getStatementResultMaps(resultMap, resultType, id))
                .resultSetType(resultSetType)
                .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
                .useCache(valueOrDefault(useCache, isSelect))
                .cache(currentCache);
        
            ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
            if (statementParameterMap != null) {
              statementBuilder.parameterMap(statementParameterMap);
            }
        
            MappedStatement statement = statementBuilder.build();
            configuration.addMappedStatement(statement);
            return statement;
        }
    ```
    ```
        上面这个方法中可以看到全局配置类Configuration有一个方法来保存MappedStatement
        configuration.addMappedStatement(statement);
    ```
5.  自此,可以得获取数据库执行语句的基本步骤：
    ```
       >   org.apache.ibatis.session.SqlSessionFactoryBuilder#build
       >   org.apache.ibatis.builder.xml.XMLConfigBuilder#parse
       >   org.apache.ibatis.builder.xml.XMLConfigBuilder#parseConfiguration
       >   org.apache.ibatis.builder.xml.XMLConfigBuilder#mapperElement
       >   org.apache.ibatis.builder.xml.XMLMapperBuilder#parse
       >   org.apache.ibatis.builder.xml.XMLMapperBuilder#configurationElement
       >   org.apache.ibatis.builder.xml.XMLMapperBuilder#buildStatementFromContext(java.util.List<org.apache.ibatis.parsing.XNode>)
       >   org.apache.ibatis.builder.xml.XMLStatementBuilder#parseStatementNode
       >   org.apache.ibatis.mapping.MappedStatement
       >   org.apache.ibatis.session.Configuration#addMappedStatement
       ```
       ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(8)-获取sql.png)
       
### 4、mybatis是如何操作数据库语句
1.  调用sqlSessionFactory.openSession()方法
    ```
        org.apache.ibatis.session.SqlSessionFactory#openSession()
        
        public SqlSession openSession() {
            return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
        }
    ```
2.  然后调用openSessionFromDataSource()方法
    ```
        org.apache.ibatis.session.defaults.DefaultSqlSessionFactory#openSessionFromDataSource
        
        private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
            Transaction tx = null;
            try {
              final Environment environment = configuration.getEnvironment();
              final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
              tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
              final Executor executor = configuration.newExecutor(tx, execType);
              return new DefaultSqlSession(configuration, executor, autoCommit);
            } catch (Exception e) {
              closeTransaction(tx); // may have fetched a connection so lets call close()
              throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
            } finally {
              ErrorContext.instance().reset();
            }
        }
    ```
    ```
        final Environment environment = configuration.getEnvironment();
        这里可以看到这一行代码，这个其实就是前面获取数据源之后保存的，
        这个时候需要执行数据库语句和链接数据库，所以从全局配置类取出。
    ```
   ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(9)-获取配置类的enviroment.png)
3.  调用mybatis的执行器
    ```
        从上面代码可以看到创建一个mybatis执行器
        final Executor executor = configuration.newExecutor(tx, execType);
    ```
    ```
        mybatis有3中执行引擎 分别为
        SIMPLE(简单), REUSE(复用), BATCH(批量)
        public enum ExecutorType {
          SIMPLE, REUSE, BATCH
        }
        默认为 SIMPLE(简单)
        org.apache.ibatis.executor.SimpleExecutor
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(10)-mybatis执行器.png)
 
4.  使用DefaultSqlSession类来接收SqlSession
    ```
        org.apache.ibatis.session.defaults.DefaultSqlSession
    ```
5.  使用session.getMapper(UserMapper.class)方法来获取UserMapper
    ```
        org.apache.ibatis.session.Configuration#getMapper     
    ```
    ```
        实例化一个MapperProxy代理类来接收UserMapper.class
        org.apache.ibatis.binding.MapperProxyFactory#newInstance(org.apache.ibatis.session.SqlSession)
        public T newInstance(SqlSession sqlSession) {
            final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
            return newInstance(mapperProxy);
        }
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(11)-MapperProxy.png)
6.  调用UserMapper中的 selectUser()方法来查询数据

    ```
        调用MapperProxy.invoke方法
        org.apache.ibatis.binding.MapperProxy#invoke
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            try {
              if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
              } else if (method.isDefault()) {
                return invokeDefaultMethod(proxy, method, args);
              }
            } catch (Throwable t) {
              throw ExceptionUtil.unwrapThrowable(t);
            }
            final MapperMethod mapperMethod = cachedMapperMethod(method);
            return mapperMethod.execute(sqlSession, args);
        }
        从代码可以看出，要获取到实例MapperMethod类
        然后执行执行mapperMethod.execute(sqlSession, args)方法。
        
        org.apache.ibatis.binding.MapperMethod类
        public class MapperMethod {
        
          private final SqlCommand command;
          private final MethodSignature method;
          
          public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
             this.command = new SqlCommand(config, mapperInterface, method);
             this.method = new MethodSignature(config, mapperInterface, method);
          }
          .....
        }
    ```
    ```
        实例化SqlCommand()保存到org.apache.ibatis.binding.MapperMethod#MapperMethod中
        
        org.apache.ibatis.binding.MapperMethod.SqlCommand#SqlCommand
        
        public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
              final String methodName = method.getName();
              final Class<?> declaringClass = method.getDeclaringClass();
              MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
                  configuration);
              if (ms == null) {
                if (method.getAnnotation(Flush.class) != null) {
                  name = null;
                  type = SqlCommandType.FLUSH;
                } else {
                  throw new BindingException("Invalid bound statement (not found): "
                      + mapperInterface.getName() + "." + methodName);
                }
              } else {
                name = ms.getId();
                type = ms.getSqlCommandType();
                if (type == SqlCommandType.UNKNOWN) {
                  throw new BindingException("Unknown execution method for: " + name);
                }
              }
        }
    ```
    ```
        实例化MethodSignature类保存到MapperMethod类中
        org.apache.ibatis.binding.MapperMethod.MethodSignature
       
        public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
              Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
              if (resolvedReturnType instanceof Class<?>) {
                this.returnType = (Class<?>) resolvedReturnType;
              } else if (resolvedReturnType instanceof ParameterizedType) {
                this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
              } else {
                this.returnType = method.getReturnType();
              }
              this.returnsVoid = void.class.equals(this.returnType);
              this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
              this.returnsCursor = Cursor.class.equals(this.returnType);
              this.returnsOptional = Optional.class.equals(this.returnType);
              this.mapKey = getMapKey(method);
              this.returnsMap = this.mapKey != null;
              this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
              this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
              this.paramNameResolver = new ParamNameResolver(configuration, method);
         }
            得到UserMapper.class接口类中定义的selectUser方法
            public abstract com.github.sdcxy.entity.User com.github.sdcxy.mybatis.mapper.UserMapper.selectUser(int)
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(12)-获取Mapper接口类中的方法.png)
    
    执行org.apache.ibatis.binding.MapperMethod#execute方法
    ```
        public Object execute(SqlSession sqlSession, Object[] args) {
          Object result;
          switch (command.getType()) {
            case INSERT: {
              Object param = method.convertArgsToSqlCommandParam(args);
              result = rowCountResult(sqlSession.insert(command.getName(), param));
              break;
            }
            case UPDATE: {
              Object param = method.convertArgsToSqlCommandParam(args);
              result = rowCountResult(sqlSession.update(command.getName(), param));
              break;
            }
            case DELETE: {
              Object param = method.convertArgsToSqlCommandParam(args);
              result = rowCountResult(sqlSession.delete(command.getName(), param));
              break;
            }
            case SELECT:
              if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
              } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
              } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
              } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
              } else {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
                if (method.returnsOptional()
                    && (result == null || !method.getReturnType().equals(result.getClass()))) {
                  result = Optional.ofNullable(result);
                }
              }
              break;
            case FLUSH:
              result = sqlSession.flushStatements();
              break;
            default:
              throw new BindingException("Unknown execution method for: " + command.getName());
          }
          if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
            throw new BindingException("Mapper method '" + command.getName()
                + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
          }
          return result;
        }
    ```
    从源码可以看出判断调用的是那种CRUD方法.
    然后调用sqlSession.selectOne(command.getName(), param)方法。
    ```
        case SELECT:
            if (method.returnsVoid() && method.hasResultHandler()) {
              executeWithResultHandler(sqlSession, args);
              result = null;
            } else if (method.returnsMany()) {
              result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
              result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
              result = executeForCursor(sqlSession, args);
            } else {
              Object param = method.convertArgsToSqlCommandParam(args);
              result = sqlSession.selectOne(command.getName(), param);
              if (method.returnsOptional()
                  && (result == null || !method.getReturnType().equals(result.getClass()))) {
                result = Optional.ofNullable(result);
              }
            }
            break;
           
    ```
    调用selectList方法.
    从源码可以看出获取之前保存在Configuration中的MappedStatement
    然后使用mybatis的执行器调用query方法       
    ```
        org.apache.ibatis.session.defaults.DefaultSqlSession#selectOne(java.lang.String, java.lang.Object)
        org.apache.ibatis.session.defaults.DefaultSqlSession#selectList(java.lang.String, java.lang.Object)
        
        public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
            try {
              MappedStatement ms = configuration.getMappedStatement(statement);
              return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
            } catch (Exception e) {
              throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
            } finally {
              ErrorContext.instance().reset();
            }
        }
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(13)-调用selectList方法.png)
    ```
        org.apache.ibatis.executor.CachingExecutor#query(org.apache.ibatis.mapping.MappedStatement, java.lang.Object, org.apache.ibatis.session.RowBounds, org.apache.ibatis.session.ResultHandler)
        执行query方法中可以看到获取绑定的sql语句,并且创建缓存
        public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
            BoundSql boundSql = ms.getBoundSql(parameterObject);
            CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
            return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        }
        
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(14)-获取绑定的SQL语句.png)
    ```
        获取到selectList的返回值
        User(id=1, username=小明, age=18, telephone=13800138000, remark=测试数据)
    ```
    ![logo](mybatis源码分析三/mybatis源码解析-mybatis由浅入深-获取SqlSessionFactory(15)-获取返回值.png)
    由此可得知，maybait操作数据库的步骤:
    ```
        >   org.apache.ibatis.session.SqlSessionFactory#openSession()
        >   org.apache.ibatis.session.defaults.DefaultSqlSessionFactory#openSessionFromDataSource
        >   org.apache.ibatis.session.defaults.DefaultSqlSession
        >   org.apache.ibatis.session.Configuration#getMapper
        >   org.apache.ibatis.binding.MapperProxy#invoke
        >   org.apache.ibatis.binding.MapperMethod#MapperMethod
        >   org.apache.ibatis.binding.MapperMethod#execute
        >   org.apache.ibatis.session.SqlSession#selectOne(java.lang.String, java.lang.Object)
        >   org.apache.ibatis.session.defaults.DefaultSqlSession#selectList(java.lang.String, java.lang.Object)
        >   org.apache.ibatis.executor.Executor#query(org.apache.ibatis.mapping.MappedStatement, java.lang.Object, org.apache.ibatis.session.RowBounds, org.apache.ibatis.session.ResultHandler)
    ```

### 5、源码地址：
*   源码地址：https://github.com/sdcxy/parse_source_code
    

