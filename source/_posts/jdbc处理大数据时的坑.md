---
title: jdbc处理大数据时的坑
date: 2023-12-20 19:44:09
tags:
---
作为有点经验的java coder，使用jdbc或者mybatis的时候有没有发现一点问题？插入大量数据怎么这么慢！查询大量结果，怎么一下就OOM了！！
<!--more-->

> 先说结论：
1. jdbc默认只会一条条执行insert，即使你调用的是batch
2. jdbc默认一次性将全部结果拉取到客户端，数据量一大极容易OOM
> 解决方案：
1. url加上参数`rewriteBatchedStatements=true`
2. url加上参数`defaultfetchsize=-214783648`,并使用`StreamingResult`
下面列举下demo说明：  
> 依赖:  
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
> TODO: 测试下，应该不是在一个事务内，可能无法回滚

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