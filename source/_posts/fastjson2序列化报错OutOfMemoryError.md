---
title: fastjson2序列化报错OutOfMemoryError
date: 2023-06-30 14:09:00
tags: 
- java
- fastjson2
categories:
- Java
---

报错log如下，这里用的是阿里的com.alibaba.fastjson2 **2.0.9版本**，该版限制了最大可以序列化大小是64M，超过了就报错OutOfMemoryError
~~~
Exception in thread "pool-4-thread-1" java.lang.OutOfMemoryError
	at com.alibaba.fastjson2.JSONWriterUTF16.writeNameRaw(JSONWriterUTF16.java:561)
	at com.alibaba.fastjson2.writer.FieldWriterImpl.writeFieldName(FieldWriterImpl.java:143)
	at com.alibaba.fastjson2.writer.ObjectWriter_3.write(Unknown Source)
	at com.alibaba.fastjson2.writer.ObjectWriterImplList.write(ObjectWriterImplList.java:278)
	at com.alibaba.fastjson2.JSON.toJSONString(JSON.java:1757)
	at com.liness.common.redis.configure.FastJson2JsonRedisSerializer.serialize(FastJson2JsonRedisSerializer.java:35)
	at org.springframework.data.redis.core.AbstractOperations.rawValue(AbstractOperations.java:128)
	at org.springframework.data.redis.core.DefaultValueOperations.set(DefaultValueOperations.java:323)
	at com.liness.common.redis.service.RedisService.setCacheObject(RedisService.java:71)
	at com.liness.business.web.controller.tools.DownLoadFileToolsController.lambda$exportPayDetail$2(DownLoadFileToolsController.java:184)
	at com.alibaba.ttl.TtlRunnable.run(TtlRunnable.java:59)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
~~~
要想解决，只能升级jar的版本，参考[https://github.com/alibaba/fastjson2/issues/841](https://github.com/alibaba/fastjson2/issues/841).
我这里是升级到**2.0.16**(考虑到升级版本和现有代码的兼容性，没有用最新的)，在序列化时增加一个features(features可以传多个的，根据自己需要传)：JSONWriter.Feature.LargeObject
~~~java
JSON.toJSONString(t, JSONWriter.Feature.WriteClassName, JSONWriter.Feature.LargeObject).getBytes(DEFAULT_CHARSET);
~~~
这样最大序列化对象即可以支持1G了。见以下源码：
~~~java
protected JSONWriter(Context context, Charset charset) {
        this.context = context;
        this.charset = charset;
        this.utf8 = charset == StandardCharsets.UTF_8;
        this.utf16 = charset == StandardCharsets.UTF_16;

        quote = (context.features & Feature.UseSingleQuotes.mask) == 0 ? '"' : '\'';

        // 64M or 1G
        maxArraySize = (context.features & LargeObject.mask) != 0 ? 1073741824 : 67108864;
    }
~~~
