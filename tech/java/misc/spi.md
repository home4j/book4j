# SPI

Java提供了一套原生的SPI（Service Provider Interface）扩展机制，一些库，如Apache日志组件就是使用了SPI来指定具体的日志实现类。下面通过一个Java示例来展示具体SPI的使用。

## Demo

完整的说明可以参考 [官方示例文档](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html) ，此处仅写重点。

### 服务接口&实现

[Dictionary接口](https://github.com/joshuazhan/arsenal4j/blob/master/java/demo/src/main/java/me/joshua/arsenal4j/java/demo/spi/Dictionary.java)，字典服务，用于查单词的解释。

```java
public interface Dictionary {
    public String getDefinition(String word);
}
```

[ExtendedDictionary](https://github.com/joshuazhan/arsenal4j/blob/master/java/demo/src/main/java/me/joshua/arsenal4j/java/demo/spi/ExtendedDictionary.java)

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

另一个实现类（[GeneralDictionary](https://github.com/joshuazhan/arsenal4j/blob/master/java/demo/src/main/java/me/joshua/arsenal4j/java/demo/spi/GeneralDictionary.java)）类似，仅提供了不同的内容，不多做罗列。

### 服务加载

#### 配置

SPI的配置是classpath中```META-INF/services```目录下的与接口同名的文件中，示例的 [配置文件](https://github.com/joshuazhan/arsenal4j/blob/master/java/demo/src/main/resources/META-INF/services/me.joshua.arsenal4j.java.demo.spi.Dictionary) 完整路径为```META-INF/services/me.joshua.arsenal4j.java.demo.extension.simple.SimpleExt```。

配置内容则是具体实现类的全类名：
```
me.joshua.arsenal4j.java.demo.spi.ExtendedDictionary
me.joshua.arsenal4j.java.demo.spi.GeneralDictionary
```

#### 服务类加载和使用

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
		// 4. 单例，便于获取使用
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

1. 服务加载
  java.util.ServiceLoader<Dictionary>
1. 