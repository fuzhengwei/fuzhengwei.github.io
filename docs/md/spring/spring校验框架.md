#### 校验注解说明
##### @Validated
提供分组

可在类型、方法、方法参数上添加，不能用在成员属性上

不可嵌套验证

可配合@Valid进行嵌套验证

##### @Valid
无分组

可用在方法、构造函数、方法参数、成员属性上

用在方法入参上无法提供嵌套验证，用在成员属性上，可以提示框架进行验证

#### 依赖
```javascript
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```
#### 注解说明
|注解|作用|
| ----- | ----- |
|@Valid|被注释的元素是一个对象，需要检查此对象的所有字段值|
|@Null|被注释的元素必须为 null|
|@NotNull|被注释的元素必须不为 null|
|@AssertTrue|被注释的元素必须为 true|
|@AssertFalse|被注释的元素必须为 false|
|@Min(value)|被注释的元素必须是一个数字，其值必须大于等于指定的最小值|
|@Max(value)|被注释的元素必须是一个数字，其值必须小于等于指定的最大值|
|@DecimalMin(value)|被注释的元素必须是一个数字，其值必须大于等于指定的最小值|
|@DecimalMax(value)|被注释的元素必须是一个数字，其值必须小于等于指定的最大值|
|@Size(max, min)|被注释的元素的大小必须在指定的范围内|
|@Digits (integer, fraction)|被注释的元素必须是一个数字，其值必须在可接受的范围内|
|@Past|被注释的元素必须是一个过去的日期|
|@Future|被注释的元素必须是一个将来的日期|
|@Email|被注释的元素必须是电子邮箱地址|
|@Length(min=, max=)|被注释的字符串的大小必须在指定的范围内|
|@NotEmpty|被注释的字符串的必须非空|
|@Range(min=, max=)|被注释的元素必须在合适的范围内|
|@NotBlank|被注释的字符串的必须非空|



#### 分组使用说明
·需要在参数上添加@Valid注解表明参数需要被校验

·需要在方法上添加上@Validated(xxx.class)表明注解分组

·需要在分类接口上继承Default接口，表明不说明分组时的默认校验

```Plain Text
javax.validation.groups.Default
public interface VipUpdate extends Default
```