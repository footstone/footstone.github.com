---
layout:post
title:java.util.ConcurrentModificationException（iterator并发问题）
---
###java.util.ConcurrentModificationException（iterator并发问题）

项目中某次功能上线，刷新权限缓存数据时出现故障，导致用户不能正常访问系统页面，影响很大。由于一直以来没有在生产环境上做过该缓存的刷新，该bug隐藏较深，这里做个记录。

####现象

框架平台通过在过滤器中对所有请求URL做权限校验，来确保操作员只能访问自己权限内的资源。这部分权限数据设计为JVM缓存存在（CacheFactory.getAll(SecAllAccessCacheImpl.class)）。刷新缓存通过人工点击管理页面上的缓存刷新按钮实现，点击后触发该cache的refresh方法，从而实现刷新。某次生产环境刷新该cache后，导致所有操作员不能都访问菜单。

####异常

```
java.util.ConcurrentModificationException
 at java.util.HashMap$HashIterator.nextEntry(HashMap.java:977)
 at java.util.HashMap$KeyIterator.next(HashMap.java:1012)
 at java.util.AbstractCollection.toArray(AbstractCollection.java:171)
 at org.apache.commons.collections.collection.AbstractCollectionDecorator.toArray(AbstractCollectionDecorator.java:145)
 ......
```

####代码片段:

```
//LoginFilter.urlControl(…)
if (IS_INIT_NEW_URL_FUNCTION_MAP .equals(Boolean.FALSE)) {
	synchronized (IS_INIT_NEW_URL_FUNCTION_MAP ) {
		if (IS_INIT_NEW_URL_FUNCTION_MAP .equals(Boolean.FALSE)) {
			IS_INIT_NEW_URL_FUNCTION_MAP = null ;//help gc
			newUrlFunctionMap = getUseNewUrlFunctionMap( (String[]) 			CacheFactory.getAll(SecAllAccessCacheImpl.class ).keySet().toArray(new String[0]));
......
IS_INIT_NEW_URL_FUNCTION_MAP = Boolean.TRUE;
}
......
```

```
//AbstractCache.refresh()
public void refresh() throws Exception {
	//先加载数据，然后在锁定，然后在更新map数据
	HashMap map = getData();
	//如果getData数据抛出异常，那么继续使用原来的cache，同时也没有锁
	//以前的话，如果出现异常那么线程永远处于等待中
	LOCK = Boolean.TRUE;
	cache.clear();
	cache.putAll(map);
	COUNT = new Long(COUNT.longValue() + 1);
	LOCK = Boolean.FALSE;
	FIRST_INIT = Boolean.TRUE;
	map.clear();
	map = null;
	......
}
```

```
//AbstractCache.getAll()
public HashMap getAll() throws Exception {
	if ( FIRST_INIT.equals(Boolean.FALSE)) {
		refresh();
	}
	return cache;
}
```

####分析

注意3点：

1.	IS_INIT_NEW_URL_FUNCTION_MAP 为static全局变量，且在进入方法体内即立刻被置为null，在初始化数据完成后再置为true。
2.	AbstractCache中的cache为对象的全局变量，加载（refresh）和获取（getAll）均是对这一个变量操作，且该变量没有加锁。
3.	CacheFactory.getAll (SecAllAccessCacheImpl.class ) 所获得数据最终指向AbstractCache中的cache引用。

系统管理员刷新缓存时，cache重新加载，同时有操作员线上访问系统，经过过滤器，即触发如下代码：
CacheFactory.getAll(SecAllAccessCacheImpl.class).keySet().toArray(new String[0]))

于是存在并发问题。cache在被写入的同时，又被读取，且通过toArray方式生成数组。AbstractCollection.toArray(T[] a)实现如下：

```
public T[] toArray(T[] a) {
	// Estimate size of array; be prepared to see more or fewer elements
	int size = size();
	T[] r = a. length &gt;= size ? a :
	(T[])java.lang.reflect.Array
	.newInstance(a.getClass().getComponentType(), size);
	Iterator it = iterator();
	for (int i = 0; i &lt; r.length; i++) {
		if (! it.hasNext()) { // fewer elements than expected
			if (a != r)
				return Arrays.copyOf(r, i);
			r[i] = null; // null-terminate
			return r;
		}
		r[i] = (T)it.next();
	}
	return it.hasNext() ? finishToArray(r, it) : r;
}
```

```
//HashMap$KeyIterator.nextEntry()
final Entry<K,V> nextEntry() {
	if (modCount != expectedModCount)
		throw new ConcurrentModificationException();
	}
```

可见其通过iterator来遍历集合中的元素生成array。而原对象变化时，iterator不会同步该变化。网上有如下描述：
> Iterator被创建之后会建立一个指向原来对象的单链索引表，当原来的对象数量发生变化时，这个索引表的内容不会同步改变，所以当索引指针往后移动的时候就找不到要迭代的对象，所以按照 fail-fast 原则Iterator 会马上抛出java.util.ConcurrentModificationException异常。

产生该异常之后，IS_INIT_NEW_URL_FUNCTION_MAP = Boolean.TRUE;被跳过，导致该全局标志位null，所以后续所有请求经过该filter时，都会被nullpointer出来，导致故障。

####解决

目前解决方案是删除IS_INIT_NEW_URL_FUNCTION_MAP = null ;该解决方案在该场景下，仍然会抛出ConcurrentModificationException，但不会引起IS_INIT_NEW_URL_FUNCTION_MAP 为null，从而不会导致用户访问异常。

进一步解决方案（待验证）：
*对cache增加同步控制；
*CacheFactory.getAll(SecAllAccessCacheImpl.class )，获取到值后即赋值给一个局部变量。如下：

```
Map tmpMap = CacheFactory.getAll(SecAllAccessCacheImpl.class );
newUrlFunctionMap = getUseNewUrlFunctionMap( (String[])
tmpMap.keySet().toArray(new String[0]));
```

####总结

通过iterator访问集合元素时，不能同时对该集合元素有增删操作，否则会ConcurrentModificationException。
关于fail-fast原则的解释，参见[这里](http://geeklu.com/2010/07/fail-fast/)。