## 스프링부트 스타터

우선 build.gradle에 아래와 같이 의존성을 추가한다.
```gradle
dependencies {
    compile "org.springframework.boot:spring-boot-starter-batch" // 의존성 추가
}
```

Application 또는 Configuration 클래스에 `@EnableBatchProcessing` 선언하도록 한다.
```java
@SpringBootApplication
@EnableBatchProcessing // 추가
public class SampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(SampleApplication.class, args);
    }

}
```

이제 사용하고자 하는 Job을 정의한 후 Application을 실행하도록 한다.
```bash
java -jar sample-batch.jar
```

Parameter를 전달할수도 있다.
```bash
java -jar sample-batch.jar param1=value1 param2=value2...
```

## In-Memory Repository 사용

SpringBatch 실행정보는 JobRepository를 통해 Database에 기록된다.

하지만 이렇게 Database를 통해 관리될 필요가 없는경우 In-Memory Repository를 사용하면된다.
[In-Memory Repository](https://docs.spring.io/spring-batch/reference/html/configureJob.html#inMemoryRepository)

여기서는 DefaultBatchConfigurer에서 createJobRepository()를 오버라이드하는 방법을 이용했다.
```java
@Configuration
public class SampleBatchConfig extends DefaultBatchConfigurer {

    @Override
    protected JobRepository createJobRepository() throws Exception {
        MapJobRepositoryFactoryBean factory = new MapJobRepositoryFactoryBean();
        factory.afterPropertiesSet();
        return factory.getObject();
    }

}
```

## 실행할 Job을 명시적으로 지정하기

기본적으로 SpringBoot를 사용하면 내부에 선언된 모든 Job을 실행시키게 된다.

하지만 명시적으로 특정 Job을 실행하기를 원한다면 application.properties에 아래와 같은 내용을 추가하도록 한다.
```properties
spring.batch.job.names=job1,job2,job3...
```

아래와 같이 런타임시에 지정하는 방법도 있다.
```bash
java -Dspring.batch.job.names=job1,job2,job3 -jar sample-batch.jar
```

무조건 명시적으로 Job을 지정하게끔 만들고 싶으면 어떻게 할까?

우선 application.properties에 아래 항목을 추가한다.
```properties
spring.batch.job.enabled=false
```

그러면 BatchAutoConfiguration에서는 JobLauncherCommandLineRunner빈을 자동으로 생성하지 않게된다.

이제 우리가 원하는대로 JobLauncherCommandLineRunner빈을 직접 생성하면 된다. (그리고 jobNames이 없으면 실행되지 않도록 처리한다.)
```java
@Configuration
@EnableConfigurationProperties(BatchProperties.class)
public class SampleBatchConfig {

    @Autowired
    private BatchProperties properties;

    @Bean
    public JobLauncherCommandLineRunner jobLauncherCommandLineRunner(JobLauncher jobLauncher, JobExplorer jobExplorer) {
        JobLauncherCommandLineRunner runner = new JobLauncherCommandLineRunner(jobLauncher, jobExplorer);
        String jobNames = this.properties.getJob().getNames();
        if (!StringUtils.hasText(jobNames)) {
            throw new IllegalArgumentException("Job names could not empty!"); // jobNames가 비어있으면 에러!
        }
        runner.setJobNames(jobNames);
        return runner;
    }
}

```