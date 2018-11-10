---
layout: post
title: MapStruct 代替BeanUtil 和ModelMapper
categories: Java
description:
keywords: mapping
date: 2017-02-15
---
### Object mapping 的技术分类：
1. 运行期 反射调用set/get 或者是直接对成员变量赋值 。 该方式通过invoke执行赋值，实现时一般会采用beanutil, Javassist等开源库。这类的代表：Dozer,ModelMaper
2. 编译期 动态生成set/get代码的class文件 ，在运行时直接调用该class文件。该方式实际上扔会存在set/get代码，只是不需要自己写了。 这类的代表：MapStruct,Selma,Orika
### 主要框架性能对比：
每秒钟执行的object mapping越多越好。
![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/8b18840183c7caaf6f0847995a4f0c8a.png)
明显可以看出通过在运行期进行反射的方式执行，性能远不如编译器生成class的方式；

MapStruct 与 Selma的对比：https://java.libhunt.com/project/mapstruct/vs/selma
MapStruct 与 ModelMapper的对比：https://java.libhunt.com/project/mapstruct/vs/modelmapper

综合比较性能、问题排查、文档、成熟度、扩展性等因素来考虑，MapStruct 是一个不错的选择；

### Maven依赖：
```java
    <properties>
        <java.version>1.8</java.version>
        <org.mapstruct.version>1.1.0.Final</org.mapstruct.version>
    </properties>
```

```java
   <dependency>
       <groupId>org.mapstruct</groupId>
       <artifactId>mapstruct-jdk8</artifactId>
       <version>${org.mapstruct.version}</version>
   </dependency>
```

```java
    <plugin>
           <groupId>org.apache.maven.plugins</groupId>
           <artifactId>maven-compiler-plugin</artifactId>
           <version>3.5.1</version>
           <configuration>
               <source>${java.version}</source>
               <target>${java.version}</target>
               <encoding>${java.encoding}</encoding>
               <annotationProcessorPaths>
                   <path>
                       <groupId>org.mapstruct</groupId>
                       <artifactId>mapstruct-processor</artifactId>
                       <version>${org.mapstruct.version}</version>
                   </path>
               </annotationProcessorPaths>
           </configuration>
    </plugin>
```

### MapStruct的基本使用：
```java
@Mapper(componentModel = "spring")
public interface MonitorAppGroupIdcDTOMapper {

    MonitorAppGroupIdcDTOMapper MAPPER = Mappers.getMapper(MonitorAppGroupIdcDTOMapper.class);

    void mapping(MonitorAppGroupIdcDTO source, @MappingTarget MonitorAppGroupIdcDTO dest);
}
```

编译后生成的部分代码结构：

```java
    @Component
public class MonitorAppGroupIdcDTOMapperImpl implements MonitorAppGroupIdcDTOMapper {
    public MonitorAppGroupIdcDTOMapperImpl() {
    }

    public void mapping(MonitorAppGroupIdcDTO source, MonitorAppGroupIdcDTO dest) {
        if(source != null) {
            dest.setId(source.getId());
            dest.setGmtCreate(source.getGmtCreate());
            ...
        }
    }
}
```

可以看出MapStruct还需要依赖对象的get/set方法，有时候编写一堆的get/set方法看上去很不美观，期望能通过自动生成的方式插入get/set方法，其解决方案是使用lombok。mapstrcut官网也有二者结合的例子：https://github.com/mapstruct/mapstruct/issues/510。Lombok 带来的问题是，如果我们期望通过公有的get/set方法范围私有属性时，IDE会提示方法不存在，这个时候我们可以下载安装Intellij Idea中的"Lombok plugin"来解决，但是这种方案带来了一定的繁琐性。比较好的方式是，对于DO或者DTO中的属性，如果属性为私有属性，需要通过get/set方法来访问的，那么就手工生成get/set方法，如果属性本身为共有属性的，那么就可以借助Lombok来自动生成get/set方法了。

### MapStruct的属性与方法：
#### 1. @Mapper注解的componentModel属性
`componentModel`属性用于指定自动生成的接口实现类的组件类型。这个属性支持四个值：

* default: 这是默认的情况，mapstruct不使用任何组件类型, 可以通过Mappers.getMapper(Class)方式获取自动生成的实例对象。
* cdi: the generated mapper is an application-scoped CDI bean and can be retrieved via @Inject
* spring: 生成的实现类上面会自动添加一个@Component注解，可以通过Spring的 @Autowired方式进行注入
* jsr330: 生成的实现类上会添加@javax.inject.Named 和@Singleton注解，可以通过 @Inject注解获取。




