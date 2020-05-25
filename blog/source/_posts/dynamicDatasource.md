---
title: 动态切库的实现
date: 2020-05-25 16:00:31
categories:
    - 动态切库
tags:
    - 多数据源
    - 动态切库
    - jta-atomikos
    - abstractRoutingDataSource
---
# 基于AbstractRoutingDataSource实现动态数据源
#### 实现思路1：
- 维护一个dataSource,里面放置多个数据源
- abstractRoutingDataSource以下简称RoutingDataSource
- RoutingDataSource中getConnection方法被重写，由getDetermineDataSource决定
- 归根结底，由lookUpKey的值，决定获取怎么样的数据源，从而获取连接
- 实现：AOP切面动态的判断规则，修改lookUpKey的值，实现多数据源切换
- 注：事务可能不支持
- 支持分布式事务，需要jta-atomikors
- jta-atomikosDemo地址：https://github.com/zzzyyyzhuyu/jta-atomikos-demo
----------------------------------------------------------------------
#### RouteingDataSource部分源码
```````````````````````````````````````````````````````````````````````
package org.springframework.jdbc.datasource.lookup;

public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
    
    //存放数据源集合
	private Map<Object, Object> targetDataSources;
    
    //存放默认数据源
	private Object defaultTargetDataSource;

	private boolean lenientFallback = true;

	private DataSourceLookup dataSourceLookup = new JndiDataSourceLookup();

	private Map<Object, DataSource> resolvedDataSources;

	private DataSource resolvedDefaultDataSource;

    //设置数据源列表 （key-value,通过key取value）
	public void setTargetDataSources(Map<Object, Object> targetDataSources) {
		this.targetDataSources = targetDataSources;
	}
    //设置默认数据源
	public void setDefaultTargetDataSource(Object defaultTargetDataSource) {
		this.defaultTargetDataSource = defaultTargetDataSource;
	}


	/**
	 * Set the DataSourceLookup implementation to use for resolving data source
	 * name Strings in the {@link #setTargetDataSources targetDataSources} map.
	 * <p>Default is a {@link JndiDataSourceLookup}, allowing the JNDI names
	 * of application server DataSources to be specified directly.
	 */
	public void setDataSourceLookup(DataSourceLookup dataSourceLookup) {
		this.dataSourceLookup = (dataSourceLookup != null ? dataSourceLookup : new JndiDataSourceLookup());
	}

    //数据源初始化之后，初始化相关信息
	@Override
	public void afterPropertiesSet() {
		if (this.targetDataSources == null) {
			throw new IllegalArgumentException("Property 'targetDataSources' is required");
		}
		this.resolvedDataSources = new HashMap<Object, DataSource>(this.targetDataSources.size());
		for (Map.Entry<Object, Object> entry : this.targetDataSources.entrySet()) {
			Object lookupKey = resolveSpecifiedLookupKey(entry.getKey());
			DataSource dataSource = resolveSpecifiedDataSource(entry.getValue());
			this.resolvedDataSources.put(lookupKey, dataSource);
		}
		if (this.defaultTargetDataSource != null) {
			this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);
		}
	}

    //获取连接方法，由determineTargetDataSource方法获取数据源
	@Override
	public Connection getConnection() throws SQLException {
		return determineTargetDataSource().getConnection();
	}
    
    //根据当前的lookedUpKey来获取当前数据源
	protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
		Object lookupKey = determineCurrentLookupKey();
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {//若数据源为空，则取默认数据源
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}

	//需要重点实现的方法，获取当前lookUpkey
	protected abstract Object determineCurrentLookupKey();
```````````````````````````````````````````````````````````````````````

#### 实现思路2：
- 自己维护多个DataSource
- [详情可参考](https://github.com/xkcoding/spring-boot-demo/tree/master/spring-boot-demo-dynamic-datasource)
```
public class DynamicDataSource extends HikariDataSource {
    @Override
    public Connection getConnection() throws SQLException {
        // 获取当前数据源 id
        Long id = DatasourceConfigContextHolder.getCurrentDatasourceConfig();
        // 根据当前id获取数据源
        HikariDataSource datasource = DatasourceHolder.INSTANCE.getDatasource(id);

        if (null == datasource) {
            datasource = initDatasource(id);
        }

        return datasource.getConnection();
    }

    /**
     * 初始化数据源
     * @param id 数据源id
     * @return 数据源
     */
    private HikariDataSource initDatasource(Long id) {
        HikariDataSource dataSource = new HikariDataSource();

        // 判断是否是默认数据源
        if (DatasourceHolder.DEFAULT_ID.equals(id)) {
            // 默认数据源根据 application.yml 配置的生成
            DataSourceProperties properties = SpringUtil.getBean(DataSourceProperties.class);
            dataSource.setJdbcUrl(properties.getUrl());
            dataSource.setUsername(properties.getUsername());
            dataSource.setPassword(properties.getPassword());
            dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        } else {
            // 不是默认数据源，通过缓存获取对应id的数据源的配置
            DatasourceConfig datasourceConfig = DatasourceConfigCache.INSTANCE.getConfig(id);

            if (datasourceConfig == null) {
                throw new RuntimeException("无此数据源");
            }

            dataSource.setJdbcUrl(datasourceConfig.buildJdbcUrl());
            dataSource.setUsername(datasourceConfig.getUsername());
            dataSource.setPassword(datasourceConfig.getPassword());
            dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        }
        // 将创建的数据源添加到数据源管理器中，绑定当前线程
        DatasourceHolder.INSTANCE.addDatasource(id, dataSource);
        return dataSource;
    }
}
```
