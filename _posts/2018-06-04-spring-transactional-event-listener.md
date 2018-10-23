---
layout: post
title:  "理解TransactionalEventListener"
date:   2018-06-04 18:00:00 +0800
categories: 后端开发
---

## 使用EventListener

Spring提供的事件监听机制,使产生事件的代码和处理事件的代码进行了解耦,例如用户注册后需要进行给用户发送邮件等一系列耗时操作,这些操作就没有必要和用户注册的代码耦合在一起,可以通过发布用户注册事件将这两个操作解耦,对于耗时操作可以同时使用`@Async`使事件处理器异步执行.代码如下:

```java
@Service
public class CustomerService {
    private final CustomerRepository customerRepository;
    private final ApplicationEventPublisher applicationEventPublisher;

    public CustomerService(CustomerRepository customerRepository, ApplicationEventPublisher applicationEventPublisher) {
        this.customerRepository = customerRepository;
        this.applicationEventPublisher = applicationEventPublisher;
    }

    @Transactional
    public Customer createCustomer(String name, String email) {
        final Customer newCustomer = customerRepository.save(new Customer(name, email));
        //用户注册成功后,发布事件
        final CustomerCreatedEvent event = new CustomerCreatedEvent(newCustomer);
        applicationEventPublisher.publishEvent(event);
        return newCustomer;
    }
}
```

```java
@Component
public class CustomerCreatedEventListener {
    private static final Logger LOGGER = LoggerFactory.getLogger(CustomerCreatedEventListener.class);
    private final EmailService emailService;
    public CustomerCreatedEventListener(EmailService emailService) {
        this.emailService = emailService;
    }

    @Async//这个注解使事件监听器异步执行
    @EventListener//被@EventListener注解修饰的方法为事件监听器
    public void processCustomerCreatedEvent(CustomerCreatedEvent event) {
        LOGGER.info("Event received: " + event);
        emailService.sendEmail(event.getCustomer());
    }
}
```

## 使用TransactionalEventListener

然而上面的代码是存在问题的,`createCustomer`方法有`@Transactional`注解,表示这个方法在事务内,发布事件的时候事务还没有提交,此时就发送邮件显然不妥,所以这个事件必须要在事务提交后才可以执行,现在`@TransactionalEventListener`就派上了用场,这个注解本身就是一个`@EventListener`,但是额外提供了对事务的支持,默认的事务阶段是事务提交之后.

```java
//TransactionalEventListener注解的源码片段
TransactionPhase phase() default TransactionPhase.AFTER_COMMIT;
```

参考:

1. <https://dzone.com/articles/transaction-synchronization-and-spring-application>
2. <https://docs.spring.io/spring/docs/4.3.12.RELEASE/spring-framework-reference/htmlsingle/#transaction-event>