# SPI

Java提供了一套原生的SPI（Service Provider Interface）扩展机制，Apache commons-logging就可以用SPI来配置日志实现类，下面是示例。

## Demo

完整的说明可以参考 [官方示例文档](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html) ，此处仅写重点。

### 服务接口&实现

[Dictionary接口](https://github.com/home4j/arsenal4j/tree/master/java/demo/src/test/java/me/joshua/arsenal4j/java/demo/spi/Dictionary.java)，字典服务，用于查单词的解释。

```java
public interface Dictionary {
    public String getDefinition(String word);
}
```

[ExtendedDictionary](https://github.com/home4j/arsenal4j/tree/master/java/demo/src/test/java/me/joshua/arsenal4j/java/demo/spi/ExtendedDictionary.java)

```java
public class ExtendedDictionary implements Dictionary {

	private SortedMap<String, String> map;

	public ExtendedDictionary() {
		map = new TreeMap<String, String>();
		map.put("xml", "a document standard ...");
		map.put("REST", "an architecture ...");
	}

	@Override
	public String getDefinition(String word) {
		return map.get(word);
	}
}
```

另一个实现类（[GeneralDictionary](https://github.com/home4j/arsenal4j/blob/master/java/demo/src/test/java/me/joshua/arsenal4j/java/demo/spi/GeneralDictionary.java)）类似，不多做罗列。

### 服务加载

#### 配置

SPI的配置是```META-INF/services```目录下与接口同名的文件，示例 [配置文件](https://github.com/home4j/arsenal4j/blob/master/java/demo/src/test/resources/META-INF/services/me.joshua.arsenal4j.java.demo.spi.Dictionary) 路径为```META-INF/services/me.joshua.arsenal4j.java.demo.extension.simple.SimpleExt```。

内容是具体实现类的全类名：
```
me.joshua.arsenal4j.java.demo.spi.ExtendedDictionary
me.joshua.arsenal4j.java.demo.spi.GeneralDictionary
```

#### 加载和使用

```java
public class DictionaryService {

	private static DictionaryService service;
	private ThreadLocal<ServiceLoader<Dictionary>> localLoader;

	private DictionaryService() {
		// 2. ServiceLoader实例线程不安全
		localLoader = new ThreadLocal<ServiceLoader<Dictionary>>() {
			@Override
			protected ServiceLoader<Dictionary> initialValue() {
				// 1. 加载服务
				return ServiceLoader.load(Dictionary.class);
			}
		};
	}

	public static synchronized DictionaryService getInstance() {
		if (service == null) {
			service = new DictionaryService();
		}
		return service;
	}

	public String getDefinition(String word) {
		String definition = null;

		try {
			// 3. 遍历并调用服务
			Iterator<Dictionary> dictionaries = localLoader.get().iterator();
			while (definition == null && dictionaries.hasNext()) {
				Dictionary d = dictionaries.next();
				definition = d.getDefinition(word);
			}
		} catch (ServiceConfigurationError serviceError) {
			definition = null;
			serviceError.printStackTrace();

		}
		return definition;
	}
}
```

几个注意点：

1. 服务加载<br/>
  通过```java.util.ServiceLoader.load(Class<T>)```来完成，扫描classpath下所有与接口同名的配置文件，解析实现类并初始化。
2. 线程不安全<br/>
  `Instances of this class are not safe for use by multiple concurrent threads. `<br/>
  就如其注释所说，`ServiceLoader`是线程不安全的，所以结合`ThreadLocal`避免并发问题。
3. 遍历调用<br/>
  只能通过遍历来获取服务实现，有一定性能损耗，对开发不友好。

# LightExt

因SPI在使用时有所不便，所以考虑对其进行改进，而 [Dubbo](http://dubbo.io/) 提供了一个很好的方案，基于它的扩展加载机制，实现了一个精简的服务扩展框架，即LightExt。

LightExt解决了SPI的两个问题：

1. 线程安全
2. 根据名字获取扩展实例，避免遍历所有实例

## Demo

#### 配置

LightExt的配置目录为```META-INF/lightext```（和SPI不同），目录下与接口同名的文件，示例 [配置文件](https://github.com/home4j/lightext/tree/master/src/test/resources/META-INF/lightext/io.github.home4j.lightext.simple.SimpleExt) 路径为```META-INF/lightext/io.github.home4j.lightext.simple.SimpleExt```。

```
# Comment 1
impl1=io.github.home4j.lightext.simple.SimpleExtImpl1#Hello World

impl2 = io.github.home4j.lightext.simple.SimpleExtImpl2  # Comment 2
   impl3=io.github.home4j.lightext.simple.SimpleExtImpl3 # with head space
```

1. 配置是KV格式，key为扩展实现的名字，value为实现类
2. 注释使用“#”开头，在一行中“#”及其后面的内容都将被忽略
3. 空行和首尾的空白字符都会被忽略

#### 加载和使用

```java
// 根据接口获取加载器（线程安全的，可供多线程复用）
LightExtLoader<SimpleExt> simpleLoader = LightExtLoader.getExtLoader(SimpleExt.class);

// 获取对应名称的扩展实现
SimpleExt impl1 = simpleLoader.getExtension("impl1");

// 泛型接口，可直接指定类型
SimpleExtImpl2 impl2 = simpleLoader.getExtension("impl2");

// 获取默认扩展实现
SimpleExt defaultExt = simpleLoader.getDefaultExtension();
```

默认的扩展实现需要用注解配置，未配置直接获取会抛```IllegalStateException```。

```java
@Default("impl1")
public interface SimpleExt {
	String echo(String s);
}
```

更详细的接口说明可参考 [项目文档](https://github.com/home4j/lightext)。

#### 实现

```java
public <S extends T> S getExtension(String name) {
    if (StringUtils.isBlank(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }

    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());

        // 确保拿到的是cachedInstances中的Holder实例
        holder = cachedInstances.get(name);
    }

    Object instance = holder.get();
    if (instance == null) {
        // 基于Holder，实现细粒度的同步，避免使用Loader类或Loader实例这种粗粒度的锁
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建扩展实例
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (S) instance;
}
```
