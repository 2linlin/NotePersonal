---
title: 实例-Fastjson定制反序列化Timestamp
date: 2020-03-31 00:00:00
categories:
- 实例
- 轮子
tags:
top:
---

最近开发中遇到了一个问题，从第三方接口接入数据时，接口中的一个字段是Timestamp类型，但是第三方给了1899年时间的脏数据，导致后续处理时产生Bug。所以需要在fastjson进行反序列化时，对这个字段专门进行处理，凡是不符合规则的时间json字符串，都改为"1970-01-01 08:00:00.001"。

具体的处理方法是，实现`com.alibaba.fastjson.parser.deserializer.ObjectDeserializer`接口，然后在对应的实体里，加上注解`@JSONField(deserializeUsing = YourImplDeserializer.class)`即可。这样，在反序列化时，就可以按我们定制的方式处理了。

以下是Demo：

第一步，实现接口

```java
public class Test {
    public static void main(String[] args) {
        test();
    }

    private static void test() {
        Student stu = JSONObject.parseObject("{\"name\":\"张三\", \"bothday\":\"2000-01-01 00:00:01\"}", Student.class);
        System.out.println(JSON.toJSONString(stu));
    }
	
    // 实现接口，完成定制
    public static class TimestampDeserializer implements ObjectDeserializer {

        @Override
        public Timestamp deserialze(DefaultJSONParser parser, Type type, Object fieldName) {
            // System.out.println("11111111111111111" + type.getTypeName());
            // System.out.println("22222222222222222" + fieldName.toString());

            if ("java.sql.Timestamp".equals(type.getTypeName())) {
                // System.out.println(parser.getLexer().stringVal());
                SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
                // sdf.setTimeZone(TimeZone.getTimeZone("UTC"));

                Date date = new Date(0);

                try {
                    // System.out.println(date.toString());
                    date = sdf.parse(parser.getLexer().stringVal());    // 北京时间，多了8小时
                    // System.out.println(date.toString());
                    // System.out.println(new Date(946656000000L));
                } catch (ParseException e) {
                    e.printStackTrace();
                }

                // 时间为2000-01-01 08:00:00 CST之前，全部设为Null
                if (date.getTime() < 946656000000L) {
                    return null;
                }

                return new Timestamp(date.getTime());
            }
            return null;
        }

        @Override
        public int getFastMatchToken() {
            return 0;
        }
    }
}
```

第二步，加上注解

```java
public class Student {
    private String name;
	
    // 就是这个注解
    @JSONField(deserializeUsing = Test.TimestampDeserializer.class)
    private Timestamp bothday;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Timestamp getBothday() {
        return bothday;
    }

    public void setBothday(Timestamp bothday) {
        this.bothday = bothday;
    }
}
```

