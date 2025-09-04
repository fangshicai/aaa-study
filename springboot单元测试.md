IDEA快速创建测试类快捷键ctrl+shift+t

添加依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```



# controller层接口测试

使用`MockMvc`模拟http请求，

- 添加`@AutoConfigureMockMvc`可注入MockMvc，MockMvc用于请求构造
- 添加`@ActiveProfiles("develop")`表示激活`application-develop.yml`文件
- 添加`SpringBootTest`，集成测试 会加载全部依赖 启动慢。一般的`controller`层推荐使用`@WebMvcTest`进行测试。

## post请求

传body参数代码如下

```java
// 模拟http请求
@AutoConfigureMockMvc
// 激活的文件yml文件
@ActiveProfiles("develop")
// 集成测试 会加载全部依赖 启动慢
@SpringBootTest
// 单元测试 QueueManager 不会执行@PostConstruct
//@WebMvcTest(ReceiveDataServerController.class)
class ReceiveDataServerControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    void receiveData() throws Exception {
        // 构造结构体
        String body = "{\n" +
                "    \"body\":\"xxxx\"\n" +
                "}";
        // 请求构造
        MvcResult mvcResult =
                mockMvc.perform(
                                MockMvcRequestBuilders.post("/data/receiveData") // post请求
                                        .contentType(MediaType.APPLICATION_JSON).content(body)) // body参数
                        .andExpect(status().isOk()) // 添加断言
                        .andReturn();// 返回结果集

        // 获取utf-8字符返回结果
        String contentAsString = mvcResult.getResponse().getContentAsString(StandardCharsets.UTF_8);
        System.out.println(contentAsString);
    }
}
```



## get请求

```java
// 模拟http请求
@AutoConfigureMockMvc
// 激活的文件yml文件
@ActiveProfiles("develop")
// 集成测试 会加载全部依赖 启动慢
@SpringBootTest
@Slf4j
class DataControllerTest {

    @Autowired
    MockMvc mockMvc;
    
    /**
     * 多线程模拟多用户请求
     * @throws Exception
     */
    @Test
    void queryData() throws Exception {
        ExecutorService executorService = Executors.newFixedThreadPool(32);
        CountDownLatch countDownLatch = new CountDownLatch(100000);
        // 请求构造表单
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        long endTime = System.currentTimeMillis();
        long start = endTime - 1000 * 60 * 60 * 24;
        params.add("deviceId", "19624819469315072300");
        params.add("startTime", String.valueOf(start));
        params.add("endTime", String.valueOf(endTime));

        for (int i = 0; i < 100000; i++) {
            executorService.execute(() -> {
                try {
                    MvcResult mvcResult =
                            mockMvc.perform(
                                            MockMvcRequestBuilders.get("/data/query") // get请求
                                                    .params(params)
                                    )
                                    .andExpect(status().isOk()) // 添加断言
                                    .andReturn();// 返回结果集
                } catch (Exception e) {
                    throw new RuntimeException(e);
                } finally {
                    countDownLatch.countDown();
                }
            });
        }
        countDownLatch.await();
        log.info("耗时：{}",System.currentTimeMillis() - endTime);
    }

}
```





# service层测试

- 添加`SpringBootTest`，集成测试 会加载全部依赖 启动慢。





hey -c 100 -m POST -H "Content-Type: application/json"  -D body.json  -n 10000 http://47.113.114.234:39002/data/receiveData



hey -c 100 -m POST   -n 10000 http://47.113.114.234:39002/testNoParam
