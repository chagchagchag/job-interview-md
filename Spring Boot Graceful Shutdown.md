## Spring Boot 에서의 Graceful Shutdown

Spring Boot 2.3 부터는 graceful shutdown 옵션을 직접 지정할 수 있다.<br>

요즘 자꾸 말이 꼬여서 설명이 잘 안될 것 같지만 일딴 스타트.<br>

<br>



## SIGINT, SIGTERM, SIGKILL

UNIX 로부터 출발한 os 들에서는 SIGINT, SIGTERM, SIGKILL 과 같은 신호(인터럽트)들을 제공한다. 이 3개의 신호들 중 Graceful Shutdown 설정시 인식하는 신호는 SIGINT, SIGTERM 이다. SIGINT는 콘솔에서 표준 입력으로 Ctrl + C 를 눌렀을 때 발생하는 인터럽트이다. SIGINT, SIGTERM 은 강제종료가 아니고 Graceful Shutdown 적용이 가능하다. SIGKILL 신호는 강제종료이기에 Graceful Shutdown 적용이 불가능하다.

- SIGINT : 2, interrupt program
- SIGKILL : 9, kill program
- SIGTERM : 15, software termination signal

> 참고: [리눅스 시그널 명령어(프로세스 종료) - kill](https://veneas.tistory.com/entry/Linux-%EB%A6%AC%EB%88%85%EC%8A%A4-%EC%8B%9C%EA%B7%B8%EB%84%90-%EB%AA%85%EB%A0%B9%EC%96%B4%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EC%A2%85%EB%A3%8C-kill)

<BR>



## graceful shutdown

graceful shutdown 기능은 스프링 부트 2.3 버전부터 제공하는 기능이다. graceful shutdown 이라는 것은 SIGINT, SIGTERM 과 같은 프로세스 종료 시그널을 받았을 때, 애플리케이션 서버는 이때부터 새로운 요청은 더 이상 받지 않고, 이미 받은 요청에 대한 처리가 끝날 때 까지 기다렸다가 처리가 끝나면 애플리케이션 서버를 종료시키는 것을 의미한다.

<br>



## 스프링 부트 2.3 +

스프링 부트 2.3 버전 이상부터는 아래와 같이 application.yml 을 작성하면, SIGINT, SIGTERM 신호를 받았을 때 더 이상 새로운 요청을 받지 않고, 기존에 처리하던 요청의 처리를 모두 마무리 한 후 서버 애플리케이션의 종료작업을 마무리한다.

```yaml
server:
	shutdown: graceful
```

<br>



그런데 만약 끝없이 종료되지 않는 로직(무한루프, 특정 네트워크 오류로 인한 무한 대기상태)이 있을 경우 영원히 종료되지 않는 문제가 발생할 수 있다. 이 경우 아래와 같이 timeout 설정을 추가해주어서 일정 시간이 지나더라도 종료가 되지 않을 경우 무시하고 종료하도록 할 수 있다.

```yaml
spring:
	lifecycle:
		timeout-per-shutdown-phase: "10s"
```

<br>



예를 들어서 /long-time-job 에서 10 초가 걸리는 작업을 실행중이라고 해보자. 그리고 SIGINT 요청이 발생했다. 서버는 종료 작업을 시작한다. graceful shutdown 이 적용되어 있기에 먼저 들어온 요청은 계속해서 처리 중이다. (직접 눈으로 확인할 경우, Ctrl + C 를 누르기 전 브라우저로 /long-time-job 에 접속한 후 Ctrl + C 로 종료해서 동작을 확인해보자)<br>

그리고 새로 들어온 요청은 거절된다.<br>

<br>



## 종료 로직 커스터마이징 

Spring Boot 2.3 이상의 버전에서도 단순히 server.shutdown = graceful 로 지정하는 것 외에도, SIGINT 발생한 시각을 db에 기록해두는 것이 필요할 수 있다. 또는 인스턴스의 Graceful Shutdown 시점에 대한 요청 정보 들을 RabbitMQ, 카프카 등의 메시지 큐에 전송하도록 장애 대응방법을 마련해야 할 수도 있다. 예를 들어 시점에 전달받은 Request URI 와 파라미터를 기록해둬야 할 수 있다. 이런 처리는 단순히 server.shutdown = graceful 로 지정한다고 해서 해결되지는 않는다.<br>

이렇게 커스텀한 설정을 해야 할 경우 아래와 같은 코드를 통해 서버 애플리케이션의 종료 시점에 수행할 작업들을 기술한다.<br>

```java
public class AppInitConfig{
    // … 
    @EventListener(ContextClosedEvent.class)
    @Transactional
    public void onContextStoppedEvent(){
        ApplicationRunningEventEntity entity = runningEventFactory.newEntity();
        applicationRunningHistory.save(entity);
        rabbitProducer.sendUnsavedData();
    }
    // … 
}
```

<br>



## graceful shutdown 수동 작성 (스프링 부트 2.2 버전 이하)

그런데 만약 Spring Boot 2.3 이하 버전인 2.2 버전 아래로는 아래와 같이 애플리케이션 종료시점에 대한 처리를 따로 해줘야 한다.<br>

> 참고한 자료는 [https://bkjeon1614.tistory.com/729](https://bkjeon1614.tistory.com/729) 이다.

일일이 설명을 적기에는 시간이 좀 걸려서 예제 코드와 깃헙 URL 만 남겨두었다. 톰캣을 수동으로 정지시키는 코드이다.<br>

깃헙 URL : [github.com/kopring/example-code/demo_graceful_shutdown_2.2](https://github.com/chagchagchag/kotlin-study-archive/tree/main/kopring/example-code/demo_graceful_shutdown_2.2)<br>

LongRunningController

```kotlin
package com.example.demo_graceful_shutdown.controller

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController

@RestController
class LongRunningJobsController {

    @GetMapping("/long-running-job")
    fun getLongRunningJob() : String{
        Thread.sleep(15000)
        return "실행 완료~!!"
    }

}
```

<br>



TomcatConfig.kt

```kotlin
package com.example.demo_graceful_shutdown.config

import com.example.demo_graceful_shutdown.shutdown.GracefulShutdownTomcatConnector
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class TomcatConfig (
    val gracefulShutdownTomcatConnector: GracefulShutdownTomcatConnector,
){

    @Bean
    fun webserverFactory(): ConfigurableServletWebServerFactory {
        val factory: TomcatServletWebServerFactory = TomcatServletWebServerFactory()
        factory.addConnectorCustomizers(gracefulShutdownTomcatConnector)
        return factory
    }

}
```

<br>



GracefulShutdownTomecatConnector

```kotlin
package com.example.demo_graceful_shutdown.shutdown

import org.apache.catalina.connector.Connector
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer
import org.springframework.stereotype.Component

@Component
class GracefulShutdownTomcatConnector (
) : TomcatConnectorCustomizer {

    var connector : Connector? = null

    override fun customize(connector: Connector?) {
        this.connector = connector
    }

}
```

<br>



GracefulShutdownComponent

```kotlin
package com.example.demo_graceful_shutdown.shutdown

import com.example.demo_graceful_shutdown.rabbitmq.RabbitProducer
import org.apache.tomcat.util.threads.ThreadPoolExecutor
import org.slf4j.Logger
import org.slf4j.LoggerFactory
import org.springframework.context.event.ContextClosedEvent
import org.springframework.context.event.EventListener
import org.springframework.stereotype.Component
//import java.util.concurrent.ThreadPoolExecutor
import java.util.concurrent.TimeUnit

@Component
class GracefulShutdownComponent (
    val rabbitProducer: RabbitProducer,
    val gracefulShutdownTomcatConnector: GracefulShutdownTomcatConnector,
){

    private val logger: Logger = LoggerFactory.getLogger(javaClass)

    @EventListener(ContextClosedEvent::class)
    fun onClose(){
        rabbitProducer.push("GRACEFUL_SHUTDOWN_TOPIC", "2023.11.30.10:00:00")
        gracefulShutdownTomcatConnector.connector?.let{
            it.pause()

            val threadPoolExecutor = it.protocolHandler.executor as ThreadPoolExecutor

            threadPoolExecutor.shutdown()

            try{
                threadPoolExecutor.awaitTermination(20, TimeUnit.SECONDS)
                logger.info("Stop Completed")
            } catch (e: InterruptedException){
                Thread.currentThread().interrupt()
                e.printStackTrace()
                logger.error("Web Application Graceful Shutdown FAILED... ")
            }
        }
    }
}
```

<br>



RabbitProducer.kt

```kotlin
package com.example.demo_graceful_shutdown.rabbitmq

import org.slf4j.Logger
import org.slf4j.LoggerFactory
import org.springframework.stereotype.Component

@Component
class RabbitProducer {

    private val logger : Logger = LoggerFactory.getLogger(javaClass)

    fun push(topic: String, data: String){
        logger.info("$topic 에 데이터($data) 전송 완료")
    }
}
```

<br>

