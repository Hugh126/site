---
title: jdbc处理大数据时的坑
date: 2023-12-20 19:44:09
tags: 
- jdbc
- 数据库
categories: Spring+
---
作为有点经验的java coder，使用jdbc或者mybatis的时候有没有发现一点问题？插入大量数据怎么这么慢！查询大量结果，怎么一下就OOM了！！
<!--more-->

## 先说结论 
1. jdbc默认只会一条条执行insert，即使你调用的是batch
2. jdbc默认一次性将全部结果拉取到客户端，数据量一大极容易OOM

### 解决方案：
1. url加上参数`rewriteBatchedStatements=true`,其作用是改写你的sql为多行insert/update
2. url加上参数`defaultfetchsize=-214783648`,并使用`StreamingResult`

### 实验准备
不动手试试，总是无法辨明真伪的。  
自行建一个有jdbc的项目，通过junit单元测试验证。
 - maven依赖:  
``` xml
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.47</version>
    </dependency>    

        <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
      <scope>test</scope>
    </dependency>
```
- jvm参数  
不设置最大内存的话，特别是大量拉取的时候对比不了效果。我设置的最大堆内存是5m。  
`-Xmx5m -Xms5m`  
为方便，顺便打印下相关参数：  
``` java
    @BeforeClass
    public static void beforeClass() throws Exception {
        RuntimeMXBean runtimeMxBean = ManagementFactory.getRuntimeMXBean();
        runtimeMxBean.getInputArguments().stream().forEach(System.out::println);
    }
```

## 批量插入
```
   /**
     * 插入本地10w条数据，实验结果：
     * 真批量：1s
     * 假批量：超过6min，差别远不止别人测试几十倍。这个差距应该是随着数据量不断放大的
     */
    @Test
    public void test2() {
        String url = "jdbc:mysql://localhost:3306/benchtest";
//        String url = "jdbc:mysql://localhost:3306/benchtest?rewriteBatchedStatements=true";
        Connection connection = null;
        try {
            long start = System.currentTimeMillis();
            connection = DriverManager.getConnection(url, username, password);
            PreparedStatement statement = connection.prepareStatement("INSERT INTO user (email, pass, name) VALUES (?, ?, ?)");
            for (int i = 0; i < 100000; i++) {
                statement.setString(1, "email"+i);
                statement.setString(2, "pass"+i);
                statement.setString(3, "name"+i);
                statement.addBatch();
            }
            int[] updateCounts = statement.executeBatch();
            connection.close();
            long end = System.currentTimeMillis();
            System.out.println("updateCounts=" + updateCounts.length + " cost=" + (end-start)/1000);
        } catch (SQLException throwables) {
            throwables.printStackTrace();
        }
    }
```

### 批量插入可能存在的问题
一般创建的连接都是自动commit的，无法针对异常rollback。也就是说，你批量插入一堆数据，保不齐什么时候异常了，只插入了一半。至于说插入了多少数据，那不知道（自动commit的时机应该是客户端根据缓存自行确定的）。

#### 测试实验
这里我准备插入50w，所以让它在45w的时候异常，测试一下实际插入情况。  
``` sql
# 清空表
truncate user;
# 构造1个唯一索引
create unique index user_email_uindex
	on user (email);
# 插入一条冲突数据（）
INSERT INTO benchtest.user (id, name, sno, email, pass, source) VALUES (1, DEFAULT, null, 'email450000', '', '');
```
实验结果： 表中存在356961条数据
> 说明：之前测试插入5w数据，想着4w5的时候异常，结果直接提交成功了。

#### 批量插入优化
既然知道问题是自动提交引发的，那么改成手动提交就可以了。  
``` java
   @Test
    public void test2() throws SQLException {
        String url = "jdbc:mysql://localhost:3306/benchtest?rewriteBatchedStatements=true";
        Connection connection = null;
        try {
            long start = System.currentTimeMillis();
            connection = DriverManager.getConnection(url, username, password);
            connection.setAutoCommit(false);
            System.out.println("AutoCommit=" + connection.getAutoCommit());
            PreparedStatement statement = connection.prepareStatement("INSERT INTO user (email, pass, name) VALUES (?, ?, ?)");
            for (int i = 0; i < 500000; i++) {
                statement.setString(1, "email"+i);
                statement.setString(2, "pass"+i);
                statement.setString(3, "name"+i);
                statement.addBatch();
            }
            int[] updateCounts = statement.executeBatch();
            connection.commit();
            long end = System.currentTimeMillis();
            System.out.println("updateCounts=" + updateCounts.length + " cost=" + (end-start)/1000);
        } catch (SQLException throwables) {
            throwables.printStackTrace();
            connection.rollback();
        }finally {
            if (connection != null) {
                connection.close();
            }
        }
    }
```


## 大结果集查询
这个主要是defaultfetchsize的影响，默认0，即一次性获取全部结果。  

具体参数可以参考[官网](https://dev.mysql.com/doc/connector-j/en/connector-j-connp-props-security.html#cj-conn-prop_allowMultiQueries)

### jdbc
``` java
/**
     * 客户端流式读取
     * 测试时候设置  -Xmx5m -Xms5m， 然后对比 enableStreamingResults 差异
     *
     * 结果： 如果不设置流式读取，会OOM
     *
     * [原理] 据说客户端和服务端两个buffer相互阻塞发力。无法验证，看看就行
     * https://zhuanlan.zhihu.com/p/47060032
     */
    @Test
    public void streamFetch() {
        Connection connection = null;
        try {
            String erpUrl = "jdbc:mysql://localhost:3306/erp";
            connection = DriverManager.getConnection(erpUrl, username, password);
            com.mysql.jdbc.PreparedStatement preparedStatement  = (com.mysql.jdbc.PreparedStatement)
                    connection.prepareStatement("select * from erp_member_tag");
            preparedStatement.enableStreamingResults();
            ResultSet rs = preparedStatement.executeQuery();
            int i=1;
            while (rs.next()) {
                if (i%10000==0) {
                    System.out.println((i++) + "--" +rs.getString("id") + "--" + rs.getString("member_id"));
                }
                i++;
            }
            System.err.println("end=" + i);
        }catch (SQLException throwables) {
            throwables.printStackTrace();
        } finally {
            try {
                connection.close();
            } catch (SQLException throwables) {
                throwables.printStackTrace();
            }
        }
    }
```

### jdbcTemplate
``` java
/**
     * [无论是设置setFetchSize(Integer.MIN_VALUE)或是URL中制定，就可以直接使用流式结果集， 很多blog中说的模拟两可]
     *
     * 如果使用RowMapper，则会将每一行读入内存，将其转换为对象，最终仍会OOM
     *
     * 解决办法： 使用RowCallbackHandler 自己处理结果集
     */
    @Test
    public void streamFetchByTemplate() {
//        String url = "jdbc:mysql://localhost:3306/erp";
        String url = "jdbc:mysql://localhost:3306/erp?defaultfetchsize=-214783648";
        DriverManagerDataSource dataSource = new DriverManagerDataSource(url, username, password);
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
//        jdbcTemplate.setFetchSize(Integer.MIN_VALUE);
        AtomicInteger i = new AtomicInteger();
        jdbcTemplate.query("select * from erp_member_tag", new RowCallbackHandler() {
            @Override
            public void processRow(ResultSet rs) throws SQLException {
                int x = i.getAndIncrement();
                if (x%10000==0) {
                    System.out.println(x + "--" +rs.getString("id") + "--" + rs.getString("member_id"));
                }
            }
        });
        System.err.println("end=" + i);
    }
```



## jdbc参数最佳实践
``` ini
// 使用utf-8编码，注意：这是java中的编码，在mysql中utf8mb4对应的也是这个
characterEncoding=utf8
// 不使用证书
useSSL=false
// 使用服务器端预处理语句
useServerPrepStmts=true
// 指定单个预处理语句缓存的大小限制，默认256
&prepStmtCacheSglLimit=10000000000
// 指定要使用的预定义配置:高性能
&useConfigs=maxPerformance
// 指定是否将批量插入语句重写为多行插入语句
&rewriteBatchedStatements=true
// 从服务器返回行数，默认0，即默认一次返回所有行
&defaultfetchsize=-214783648
```