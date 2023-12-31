### 合格的UT的要求：
- 测试的是一个代码单元内部的逻辑，而不是各模块之间的交互
- 无依赖，不需要实际运行环境就可以测试代码
- 运行效率高，可以随时执行
### 如何设计适合测试的接口：
- 单元测试测试的对象是类，测试类的功能在各种情况下是否符合预期，并不是测试实现。所以只需要测试能够跟其他类交互的public方法就可以了，好处是如果重构代码之后仍然能够复用UT
- static方法最好是完全不依赖任何第三方服务，自己实现业务逻辑。如果依赖第三方，使用reference传入，而不是写死在class或者方法中
- 扩展开放，修改闭合，不对老代码入侵，避免UT重复修改
- 类的抽象、方法的提取，使代码更加精简
- 依赖写死在方法中不利于测试
### 测试方法的类型
- 有参无反馈：验证调用链，以某个对象为标志，检测调用次数或者调用参数。
- 有参有反馈：验证输入输出
- 无参有反馈：验证输出
- 无参无反馈：验证调用链
### 各层分别做UT
- controller:
- service: 对自己应该存在的业务场景进行UT,涉及到的dao层与中间件需要mock
- dao/中间件交互： 涉及的第三方api调用通过mock实现
### 用法
注意 当springboot版本高了之后，用法会不一样 4.x 5.x
#### 断言
- junit的断言
```
import static org.junit.jupiter.api.Assertions.*; 5.x
import static org.junit.Assert.*; 4.x

assertThrows
...
```
- hamcrest断言
![[Pasted image 20230823161904.png]]
```
import static org.hamcrest.MatcherAssert.*;  
import static org.hamcrest.CoreMatchers.*;
```
第一个参数是实际值，第二个参数是Matcher，代表是否匹配
#### 参数化测试
很多时候，函数有很多输入输出，为了达到测试的覆盖率，需要很多样例
覆盖要求：
- 代码覆盖
- 分支覆盖
- 判断覆盖
参数化测试可以帮助我们进行多个输入输出的组织
junit5的参数化测试https://blog.csdn.net/u013019701/article/details/103330908
```
@ParameterizedTest  
@MethodSource("data")  
void testParamterized(Integer[] src,ArrayList<Integer> expected){  
    assertThat(Lists.newArrayList(src),equalTo(expected));  
}  
  
static List<Arguments> data(){  
    List<Arguments> data = new ArrayList<>();  
    data.add(Arguments.of(new Integer[0],new ArrayList<Integer>()));  
    List<Integer> rs = new ArrayList<>();  
    rs.add(1);  
    rs.add(2);  
    data.add(Arguments.of(new Integer[]{1,2},rs));  
    return data;  
}
```
![[Pasted image 20230824141000.png]]
#### Mock
模拟对象行为，按照期望运行，而要进行测试的方法调用的就是这个模拟对象的假方法
用途：
- 测试时无法使用真实环境
- 调用接口未完成/比较复杂
- 真实对象很难创建
- 真实对象的有些行为很难触发

简单使用，直接Mockito.mock去mock一个对象，这里面的方法都是空方法
**注意，mock对象不会触发类的构造方法与构造方法块，但是静态代码块会触发**
```
TestDao mock = Mockito.mock(TestDao.class);  
System.out.println(mock.testBool());   //false
System.out.println(mock.testObject(new Object())); //null
```

注解使用j5
```
@TestInstance(TestInstance.Lifecycle.PER_CLASS)  
class ListsTest {  
  
    @Mock  
    private TestDao testDao;  
  
    @InjectMocks  
    private TestService testService;  
  
    @BeforeAll  
    public void init(){  
        MockitoAnnotations.initMocks(ListsTest.this);  
    }  
  
    @Test  
    void testAnnotationMock(){  
        System.out.println(testService.testB());  
    }
}
```
- 操作mock返回的对象 让其有特定的行为
1. 有返回值的方法
```
@Test  
void testMock(){  
    Object o = new Object();  
    Object p = new Object();  
    LogEntity logEntity = new LogEntity();  
	 // 当调用testDao.testObject方法，且参数是o时，按照调用次数返回结果
	Mockito.when(testDao.testObject(o)).thenReturn(false).thenReturn(true);     // 注意会覆盖掉
    Mockito.when(testDao.testObject(o)).thenReturn(true);  
    // 当调用testDao.testObject方法，且对象为o时，会根据Answer回调返回结果
    Mockito.doAnswer(invocation -> {  
        Object argument = invocation.getArgument(0);  
        if (argument instanceof LogEntity)return true;  
        else return false;  
    }).when(testDao).testObject(o);  
  
    Mockito.when(testDao.testBool()).thenThrow(RuntimeException.class);  
    Mockito.when(testDao.testBool()).thenAnswer(invocation -> false);  
      
}
```
- 无返回值方法
```
String s1 = "1234";  
  
Mockito.doNothing().when(testDao).send(s1);  
  
testDao.send(s1);  
  
Mockito.doThrow(RuntimeException.class).when(testDao).send(ArgumentMatchers.anyString());  
  
assertThrows(RuntimeException.class,()->{  
    testDao.send("22");  
});
```
- 参数匹配器
用于匹配参数类型，如果一个参数使用了则都需要使用
import org.mockito.ArgumentMatchers
![[Pasted image 20230824101802.png]]
如何自定义参数匹配器：
![[Pasted image 20230824102031.png]]
```
Mockito.when(testDao.testObject(ArgumentMatchers.argThat(o->{  
    return o instanceof LogEntity;  
}))).thenThrow(RuntimeException.class);  
Mockito.when(testDao.testObject(ArgumentMatchers.argThat(Objects::isNull))).thenReturn(true);  
  
LogEntity logEntity = new LogEntity();  
assertThrows(RuntimeException.class,()->{  
   testDao.testObject(logEntity);  
});  
  
assertEquals(true,testDao.testObject(null));
```
- Mockito.verify
可以用于验证某个方法的调用次数
![[Pasted image 20230824103304.png]]
#### PowerMock
用于模拟私有、final方法以及final类
#### JMock
http://jmockit.cn/index.htm
![[Pasted image 20230824104220.png]]
![[Pasted image 20230824104424.png]]
![[Pasted image 20230824104510.png]]
