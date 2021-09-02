## tomcat：

@SpringBootApplication
@EnableAspectJAutoProxy
public class Application extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
                 // 这里可以加一些初始化操作
        return application.sources(Application.class);
    }
}

pom文件：
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-tomcat</artifactId>
  <scope>provided</scope>
</dependency>
https://fulln.github.io/2018/09/12/springboot/


## jar：

@Slf4j
@SpringBootApplication(
        scanBasePackages = ***
        exclude = ***.class
)
@EnableDiscoveryClient
@EnableFeignClients(basePackages = ***)
@EnableCircuitBreaker
public class Application implements ApplicationRunner {
    public static void main(final String[] args) {
        SpringApplication.run(Application.class, args);
    }
}


<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>

基于jar的Spring Boot工程

来自 <https://blog.csdn.net/u012965203/article/details/92833005> 
![image](https://user-images.githubusercontent.com/79843418/131820964-f86b8ed6-0253-46f9-8f47-4cd03d7555e8.png)
