## EE的优势
    ·对性能的损耗很小
        EE的官网中描述了整体框架，可以看到EE在项目和ES中充当这翻译的作用，我们用MP的语法来操作ES，
        EE会将我们的语法翻译成RestHighLevelCLient的语法，针对查询条件时使用队列FIFO的方式来进行
        逐一转换，在现在市面上的机器中这种遍历的性能损耗可以忽略

        查询结果EE会帮我们自动转化为List<T>的形式，即使我们不用EE也会需要这种转换。
    
    ·拓展性好
        EE除了类似于MP的语法还支持原生语法和混合查询，对拓展性有更好的支撑

## EE的接入要求
    ·底层采用了RestHighLevelCLient,版本为7.14,对该版本兼容最好；7.X版本均支持。

## demo
### 依赖
```xml
        <dependency>
            <groupId>cn.easy-es</groupId>
            <artifactId>easy-es-boot-starter</artifactId>
            <version>${Latest version}</version>
        </dependency>

```
### 基础配置
```yaml
easy-es:
  enable: true #默认为true,若为false则认为不启用本框架
  address : 127.0.0.1:9200 # es的连接地址,必须含端口 若为集群,则可以用逗号隔开 例如:127.0.0.1:9200,127.0.0.2:9200
  username: elastic #若无 则可省略此行配置
  password: WG7WVmuNMtM4GwNYkyWH #若无 则可省略此行配置

```
### 扫描路径指定
```java
@SpringBootApplication
@EsMapperScan("com.xpc.easyes.sample.mapper")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```
### 实体类
```java
@Data
public class Document {
    /**
     * es中的唯一id
     */	
    private String id;
    /**
     * 文档标题
     */
    private String title;
    /**
     * 文档内容
     */
    private String content;
}
```
### Mapper
```java
public interface DocumentMapper extends BaseEsMapper<Document> {
}
```

### 基础操作
```text
    // 测试插入数据
    Document document = new Document();
    document.setTitle("老汉");
    document.setContent("推*技术过硬");
    int successCount = documentMapper.insert(document);
    System.out.println(successCount);

    // 测试查询
    String title = "老汉";
    LambdaEsQueryWrapper<Document> wrapper = new LambdaEsQueryWrapper<>();
    wrapper.eq(Document::getTitle,title);
    Document document = documentMapper.selectOne(wrapper);
    System.out.println(document);
    Assert.assertEquals(title,document.getTitle());
    
    // 测试更新 更新有两种情况 分别演示如下:
    // case1: 已知id, 根据id更新 (为了演示方便,此id是从上一步查询中复制过来的,实际业务可以自行查询)
    String id = "krkvN30BUP1SGucenZQ9";
    String title1 = "隔壁老王";
    Document document1 = new Document();
    document1.setId(id);
    document1.setTitle(title1);
    documentMapper.updateById(document1);

    // case2: id未知, 根据条件更新
    LambdaEsUpdateWrapper<Document> wrapper = new LambdaEsUpdateWrapper<>();
    wrapper.eq(Document::getTitle,title1);
    Document document2 = new Document();
    document2.setTitle("隔壁老李");
    document2.setContent("推*技术过软");
    documentMapper.update(document2,wrapper);
    
    // 测试删除数据 删除有两种情况:根据id删或根据条件删
    LambdaEsQueryWrapper<Document> wrapper = new LambdaEsQueryWrapper<>();
    String title = "隔壁老李";
    wrapper.eq(Document::getTitle,title);
    int successCount = documentMapper.delete(wrapper);
    System.out.println(successCount);
```
## 官网
https://www.easy-es.cn/pages/ec7460/
