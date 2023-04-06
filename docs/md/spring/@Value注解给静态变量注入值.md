## @Value注解不可以直接给静态化变量注入
```javascript
@Value("${login.name}")
private static String loginName;
```
在Java中，静态变量也称为类变量。也就是说，它们属于一个类，而不是一个特定的实例。因此，类初始化的时候也将初始化静态变量相反，类的实例 初始化的时候也将初始化 实例变量(非静态变量)。类的所有实例共享该类的静态变量。

@value 是在 bean实例化后，在属性填充过程中进行赋值的，static初始化要早于@value。

## 解决方案1:set注入
```javascript
/**文件存储目录*/
    public static String SAVE_PATH;

    //记得去掉static
    @Value("${local.file.temp.dir}")
    public void setSavePath(String savePath){
        SAVE_PATH = savePath;
    }

```
## 解决方法2:@PostConstruct注解
```javascript
    /**文件存储目录*/
    public static String SAVE_PATH;

    @Value("${local.file.temp.dir}")
    public String SAVE_PATH_TEMP;

    @PostConstruct
    private void init(){
        SAVE_PATH = SAVE_PATH_TEMP;
    }
```
@PostConstruct 是在 bean 初始化（initializeBean）过程中调用的，是在@value之后调用的，可以通过这种方式给静态变量赋值。

@PostConstruct作用是在bean初始化之前调用被注解注释的方法

## 解决方法3:实现InitializingBean接口
```javascript
public class IndexController implements InitializingBean {

    /**文件存储目录*/
    public static String SAVE_PATH;

    @Value("${local.file.temp.dir}")
    public String SAVE_PATH_TEMP;

    @Override
    public void afterPropertiesSet() throws Exception {
        SAVE_PATH = SAVE_PATH_TEMP;
    }
}
```
该接口与@PostConstruct注解作用一致