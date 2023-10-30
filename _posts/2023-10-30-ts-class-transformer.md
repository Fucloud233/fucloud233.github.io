---
layout: post
title: "Typescript小记: class-transformer转换混合类型数组"
---

# 背景

<!-- 最近在写一个VSCode的插件，虽然说也能用Javascript写，但习惯了静态语言，用动态语言编程总感觉缺点什么。 
所以虽然没有学过Typescript，也还是硬着头皮上了。 -->

<!-- 
写程序的时候难免需要用一些结构体/类，此时js和ts的区别就体现出来了。
js存储对象用的是Object，但它没有固定的类型和字段名，用起来有点随性。
而ts则在js的基础上，引入了class这个关键字，换句话说，就是实现了类似C++中的功能。
这样，有类的限制，在编码时就会更规范，bug自然就会减少了。

后面，又涉及到这些对象的IO读写问题。
当时想着，ts和js差不多，那就用Json来存吧。
不过class自然不是Object，在序列化和反序列时难免有些问题，
特别时出现嵌套对象的情况。
几经搜索，发现Typescript本身并不能解决这种问题，
直到后面遇到`class-transformer`。 -->

最近笔者在用Typescript写VSCode的插件，
用Json文件存储设置信息`SettingsInfo`，
设置信息内部有一个数组，由字符串和`ProjectInfo`对象。
后者当然就是项目信息内容，而前者则是项目文件的路径。

```typescript
class SettingsInfo {
    linkedProjects: (String | ProjectInfo)[]
}
```

由于用Typescript写了类，所以就用`class-transformer`来序列化和反序列化，
但就是在处理这个数组的时候遇到了问题。

# Bug描述

一开始，笔者只关注`ProjectInfo`的问题，所以只注解了这个字段。

```typescript
class SettingsInfo {
    @Type(() => ProjectInfo)
    @Expose({name: FIELD_NAME})
    linkedProjects: (String | ProjectInfo)[]
｝
```

可等到关于`ProjectInfo`方面的代码写好之后，
实现`string`的序列化和反序列化时，问题就出现了。
每次在读写设置信息时，
就发现之前写入的字符串被后面的字符串覆盖了。

# Debug之路

不假思索，问题肯定时出现在读取的过程。可问题时为什么只读到了`ProjectInfo`对象呢？
肯定和`@Type(()=>ProjectInfo)`这条注解脱不了干系。
我们在定义这个注解的时候，只考虑了`ProjectInfo`这个类，
因此，当transformer遇到字符串时，
自然是不能识别的，
最终结果便读取不到字符串的内容了。

明确了Bug的原因，就需要想办法解决。
一开始以为这个库很智能，
应该能够通过下面的方法直接实现多类型的识别。

```typescript
@Type(() => ProjectInfo, String)
@Type(() => ProjectInfo | String)
```

当然最后肯定是没有通过的。没办法，只能去看官方文档了。
看了半天觉得最贴近的一条是这个[`Providing more than one type option`](https://github.com/typestack/class-transformer#providing-more-than-one-type-option)。
于是我照着他实现了一遍。

```typescript
class SettingsInfo {
    @Type(() => Object, {
        discriminator: {
        property: 'type',
        subTypes: [
            { value: String, name: 'string' },
            { value: ProjectInfo, name: 'projectInfo' }
        ],
    })
    @Expose({name: FIELD_NAME})
    linkedProjects: (String | ProjectInfo)[]
}
```

这个代码看起来没什么问题，但实际上还是运行不了。
原因很简单，如果`class-transformer`要判断转换的类型，
需要`property`这个字段来判断，
比如说，当字段为该字段为`projetInfo`时，就转换为`ProjectInfo`对象。
可是，字符串是基本数据类型，也就是不可能存在字段，所以此代码就不能运行。

# 解决方案

当我穷途末路时，直到我看到了这个[`advanced-usage`](https://github.com/typestack/class-transformer#advanced-usage)。
虽然说这个库没有提供满足我们需求的注解，
但是它提供了`@Transform`让我们自定义转换过程。

这个注解有提供了以下这些字段，
而笔者真正使用其实只有`value`这个字段，
也就是字段的值。

```typescript
@Transform(({ value, key, obj, type }) => value)
```

Argument    |Description
------------|-----------
value	    |The property value before the transformation.
key	        |The name of the transformed property.
obj         |The transformation source object.
type	    |The transformation type.
options	    |The options object passed to the transformation method.

通过一个循环读取列表，
遇到字符串就直接添加，
否则通过`plainToInstance`函数转换成`ProjectInfo`类。
通过这样的方法，我们既能读取到数组中的对象，
又能读取到字符串了。

```typescript
class SettingsInfo {
    @Transform(value => {
        let items: any[] = value['value'];
        return items.map((item, _index, _array) => {
            if(typeof item === 'string') {
                return item;
            }
            return plainToInstance(ProjectInfo, item);
        });
    })
    @Expose({name: FIELD_NAME})
    linkedProjects: (String | ProjectInfo)[]
}
```

不过Debug并没有到这里结束，
因为现在读取没有问题了，但是写入出问题了。
尝试了这个库提供的`@TransformClassToPlain`和`@TransformPlainToClass`注解，
但并不是很奏效。
又只能去看官方文档了，最终是在[官方实例](`https://github.com/typestack/class-transformer/blob/master/sample/sample5-custom-transformer/User.ts`)
中找到了解决方法。

原来如果是用`@transform`注解，在`class`和`plain`的转换过程中，
都是在使用该注解内定义的函数，显然这个函数不能应对正反两个过程。
库的开发者也意识到了这个问题，为这个注解提供了一个隐藏属性`toPlainOnly`/`toClassOnly`。

> 为什么叫隐藏属性呢？因为当时在官方文档内没找到这个属性

最终我们在添加这个属性，完成地修复了这个bug。

```typescript
class SettingsInfo {
    @Transform(value => {
        let items: any[] = value['value'];
        return items.map((item, _index, _array) => {
            if(typeof item === 'string') {
                return item;
            }
            return plainToInstance(ProjectInfo, item);
        });
    }, { toClassOnly: true })
    @Expose({name: FIELD_NAME})
    linkedProjects: (String | ProjectInfo)[]
}

```