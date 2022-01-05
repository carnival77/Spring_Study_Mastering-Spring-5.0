# 의존 관계 주입 및 단위 테스트하기

## 의존 관계 주입

의존 관계 주입은 스프링 프레임워크의 가장 중요한 기능으로 느슨하게 연결된 애플리케이션을 쉽게 개발할 수 있다.

느슨하게 연결된 애플리케이션은 단위 테스트가 쉬우므로 유지 관리가 훨씬 쉽다.

스프링 프레임워크에서 의존 관계 주입은 **스프링 IoC 컨테이너**에서 구현된다.

## 학습 목표

1. [의존 관계란 무엇인가?](#학습목표1)
2. [의존 관계 주입(DI)이란 무엇인가?](#학습목표2)
3. [의존 관계 주입을 올바르게 사용하면 애플리케이션을 테스트할 수 있을까?](#학습목표3)
4. [스프링은 빈 팩토리와 ApplicationContext로 의존 관계 주입을 어떻게 구현할까?](#학습목표4)
5. [컴포넌트 스캔이란 무엇일까?](#학습목표5)
6. [자바와 XML 애플리케이션 컨텍스트의 차이점은 무엇인가?](#학습목표6)
7. [스프링 컨텍스트의 단위 테스트는 어떻게 작성할까?](#학습목표7)
8. [모킹은 단위 테스트를 어떻게 쉽게 처리할까?](#학습목표8)
9. [다른 빈 스코프는 무엇일까?](#학습목표9)
10. [CDI란 무엇이며, 스프링은 CDI를 어떻게 지원할까?](#학습목표10)

예제 프로젝트 구성
<img src="https://user-images.githubusercontent.com/52997401/147812548-d0ca301e-9efd-4e5b-a0dd-570fca7b41de.png">

#### 학습목표1

## 의존 관계란 무엇인가?

클래스의 의존 관계는 역할을 수행하기 위해 의존하는 다른 클래스다.

예시)

```java
public class BusinessServiceImpl{

  public long calculateSum(User user) {
    DataServiceImpl dataService = new DataServiceImpl();

    long sum = 0;
    for (Data data : dataService.retrieveData(user)) {
      sum += data.getValue();
    }

    return sum;

  }
}


```
BusinessServiceImpl는 DataServiceImpl 인스턴스를 생성해 데이터베이스에서 데이터를 가져온다.

**DataServiceImpl 는 BusinessServiceImpl 의 의존 관계다.**

## 의존 관계를 갖는 이유

SRP(Single Responsibility Principle) : 애플리케이션의 각 클래스와 컴포넌트가 명확하게 정의된 특정 책임을 갖도록 하는 것. 같은 클래스가 여러 기능을 갖게 하지 않는다.

![](https://media.vlpt.us/images/cocodori/post/b9c1d872-f964-4ebb-9538-54e5db139723/15545765386427_Screen%20Shot%202019-04-06%20at%205.29.27%20pm.png)

- UI 레이어 : View 로직(최종 사용자에게 데이터를 표시하는 방법)을 담당
- 비즈니스 레이어 : 비즈니스 로직을 담당
- 데이터 레이어 : 데이터베이스와의 상호 작용을 담당
- 통합 레이어 : 웹 서비스나 다른 연결 메커니즘으로 다른 애플리케이션과 통신

		비즈니스 레이어 클래스는 데이터 레이어의 클래스와 소통한다. 
		즉, 비즈니스 레이어 클래스는 데이터 레이어의 클래스에 종속된다. 
		데이터 레이어 클래스는 비즈니스 레이어의 클래스에 의존 관계다.

## 의존 관계 주입의 개념

### 느슨한 결합 : 각 구성 요소와 클래스가 서로 느슨하게 연결되게 함으로써, 다른 클래스와 구성 요소에 영향을 주지 않고 클래스 및 구성 요소를 변경할 수 있게 한다.

ex) 예제 : 단단히 결합된 코드를 느슨한 결합으로 바꾸기.

**단단한 결합**
```java
DataServiceImpl dataService = new DataServiceImpl();
```
BusinessServiceImpl는 DataServiceImpl 인스턴스를 생성해 데이터베이스에서 데이터를 가져온다.
-> BusinessServiceImpl는 DataServiceImpl 인스턴스를 생성해, 두 클래스 사이의 단단한 결합이 생성된다.

### 단단한 결합을 제거하고 코드를 느슨하게 결합하기

1. DataServiceImpl 의 인터페이스 작성

	BusinessServiceImpl에서 직접 DataServiceImpl 클래스를 사용하기보다는 DataService 인터페이스를 사용한다.
```java
public interface DataService {
  List<Data> retrieveData(User user);
}

public class DataServiceImpl implements DataService {
  public List<Data> retrieveData(User user) {
    return Arrays.asList(new Data(10), new Data(20));
  }
}
```
인터페이스를 사용하도록 BusinessServiceImpl의 코드를 업데이트한다.
```java
public class BusinessServiceImpl {

  public long calculateSum(User user) {
    DataService dataService =  new  DataServiceImpl();

    long sum = 0;
    for (Data data : dataService.retrieveData(user)) {
      sum += data.getValue();
    }

    return sum;

  }
}
```
현재 DataService 인터페이스를 사용하고 있지만 BusinessServiceImpl은 DataServiceImpl 인스턴스를 생성함으로써 여전히 밀접하게 결합되어 있다.

-> 로직을 이동해서 다른 곳에 DataServiceImpl 을 생성하고, BusinessServiceImpl에서 사용 가능하게끔 한다.

2. BusinessServiceImpl 외부로 의존 관계 생성 이동하기

DataServiceImpl 의존 관계를 BusinessServiceImpl 외부에서 생성한 뒤, 그것을 BusinessServiceImpl 이 사용할 수 있게 해서 문제를 해결해야 한다.

먼저 BusinessServiceImpl 에 DataService 를 인수로 허용하는 새 생성자를 추가한다.

```java
public class BusinessServiceImpl {
  private DataService dataService;
  public BusinessServiceImpl(DataService dataService){
    this.dataService = dataService;
  }
  public long calculateSum(User user) {
    long sum = 0;
    for (Data data : dataService.retrieveData(user)) {
      sum += data.getValue();
    }
    return sum;
  }
}
```

BusinessServiceImpl 은 더 이상 DataServiceImpl 과 단단하게 연결되어 있지 않다.
DataService 를 구현한 후 BusinessServiceImpl 의 생성자 인수로 전달해 BusinessServiceImpl를 구현할 수 있게 됐다.

```java
DataService dataService =  new  DataServiceImpl();
BusinessServiceImpl businessService = new BusinessServiceImpl(dataService);
```

3. BusinessServiceImpl 의 인터페이스 작성

코드를 더 느슨하게 결합하려면 businessService 의 인터페이스를 만들고 BusinessServiceImpl 을 업데이트해 인터페이스를 구현해야 한다.

```java
public interface BusinessService {
  long calculateSum(User user);
}

public class BusinessServiceImpl implements BusinessService{
  private DataService dataService;
  public BusinessServiceImpl(DataService dataService){
    this.dataService = dataService;
  }
  public long calculateSum(User user) {
    long sum = 0;
    for (Data data : dataService.retrieveData(user)) {
      sum += data.getValue();
    }
    return sum;
  }
}
```

### 용어 이해

```java
의존 관계 주입

DataService dataService =  new  DataServiceImpl(); // 생성하기
BusinessServiceImpl businessService = new BusinessServiceImpl(dataService); // 생성하고 연결하기
```
이 코드의 의존 관계 주입 : DataServiceImpl 클래스의 인스턴스를 생성한 뒤, 이를 BusinessServiceImpl 클래스와 연결해 BusinessServiceImpl  클래스의 인스턴스를 만든다.

1. IoC(제어의 역전) : 사용자의 제어권을 다른 주체에게 넘겨 의존 관계를 만드는 책임을 프레임워크로 옮기는 것. 컴포넌트 의존 관계 결정, 설정 및 생명 주기를 해결하기 위한 디자인 패턴.

	-   일반적으로 처음에 배우는 자바 프로그램에서는  **각 객체들이 프로그램의 흐름을 결정하고 각 객체를 직접 생성하고 조작하는 작업(객체를 직접 생성하여 메소드 호출)을 했습니다**. 즉, **모든 작업을 사용자가 제어하는 구조**였습니다. 예를 들어 A 객체에서 B 객체에 있는 메소드를 사용하고 싶으면, B 객체를 직접 A 객체 내에서 생성하고 메소드를 호출합니다. 다시 말해, Class를 생성하고 new를 입력하여 원하는 객체를 직접 생성한 후에 사용했었습니다.
	-   하지만  **IOC가 적용된 경우, 객체의 생성을 특별한 관리 위임 주체(IoC 컨테이너 - 애플리케이션 컨텍스트)에게 맡깁니다.**  다시 말해, Spring에서는 직접 new를 이용하여 생성한 객체가 아니라,  IoC 컨테이너에 의해 관리당하는 자바 객체를 사용합니다. 이 경우  **사용자는 객체를 직접 생성하지 않고, 객체의 생명주기를 컨트롤하는 주체는 IoC 컨테이너**가 됩니다.
	- 예제에서 처음엔 BusinessServiceImpl 은 DataServiceImpl 의 인스턴스 생성을 담당했다. 코드를 느슨하게 결합시키는 과정을 통해 BusinessServiceImpl 은 DataServiceImpl 을 생성할 책임이 없어졌다.
	- Spring Framework 에서는 Spring Bean 을 얻기 위하여 ApplicationContext.getBean() 와 같은 메소드를 사용하여 Spring 에서 직접 자바 객체를 얻어서 사용합니다.
	- 빈을 생성하고 관리하는 기능을 컨테이너에 위임하면 생기는 장점
		- 클래스는 의존 관계 생성에 책임이 없기 때문에 느슨하게 결합돼 테스트할 수 있다. 디자인은 좋아지고 결함은 적어진다.
		- 컨테이너가 빈을 관리하므로 빈 주위에 몇 개의 기능을 더 일반적인 방식으로 도입할 수 있다. 로깅, 캐싱, 트랜잭션 관리 및 예외 처리와 같은 크로스 컷팅을 AOP를 사용해 빈 중심으로 구성할 수 있다. 이렇게 하면 유지 보스가 쉬운 코드가 된다.

2. 빈 : **Spring에 의하여 생성되고 IoC 방식으로 관리되는 자바 객체**
	- dataService 와 businessService 라는 두 개의 오브젝트를 생성했다. 두 인스턴스를 빈이라 한다.

3. IoC 컨테이너
	- 자바 객체에 대한 생성 및 생명 주기, 의존성을 관리
	- POJO의 생성, 초기화, 서비스 소멸에 대한 권한을 가진다.
	- 개발자들이 POJO를 직접 생성할 수 있지만 컨테이너에게 맡긴다.

6. POJO(Plain Old Java Object)
	- 객체지향 원리에 충실하면서, 특정 환경이나 규약에 종속되지 않고 필요에 따라 재활용될 수 있는 방식으로 설계된 객체
	- 스프링 컨테이너에 저장되는 POJO Java 객체는 특정한 인터페이스를 구현하거나, 특정 클래스를 상속받지 않아도 된다.

7. 와이어링 : **애플리케이션 객체 간의 연관간계 형성 작업**
	- 스프링을 사용하는 애플리케이션에서는 각 객체가 자신의 일을 하기 위해 필요한 다른 객체를 직접 찾거나 생성할 필요가 없다. 컨테이너가 협업할 객체에 대한 레퍼런스를 주기 때문이다.
	- dataService 에는 의존 관계가 없고 businessService 에는 dataService 라는 하나의 의존 관계가 있다. 
	- dataService 를 생성해 BusinessServiceImpl  생성자의 인수로 제공했는데, 이를 와이어링이라 한다.

#### 학습목표2

6. 의존 관계 주입 : **각 자바 객체 간의 의존 관계를 빈 설정 정보를 바탕으로 컨테이너가 자동으로 연결해주는 것. 빈 식별, 생성, 와이어링하는 의존 관계 프로세스**
	- 개발자들은 단지 빈 설정 파일에서 의존 관계가 필요하다는 정보를 추가하면 된다.
	- 객체 레퍼런스를 컨테이너로부터 주입 받아서, 실행 시에 동적으로 의존 관계가 형성된다.
	- 컨테이너가 흐름의 주체가 되어 애플리케이션 코드에 의존 관계를 주입해 주는 것이다.
		- 장점
			- 코드가 단순해진다.
			- 컴포넌트 간의 결합도가 제거된다.

## 스프링 프레임워크의 역할

의존 관계를 식별하고 함께 인스턴스화하고 연결한다.

### 질문 1. 스프링 IoC 컨테이너는 어떤 빈을 생성해야 하는지 어떻게 알까?

스프링 IoC 컨테이너는 클래스에서 어노테이션을 보고 그에 맞는 클래스의 인스턴스를 만든다.

@Component : 스프링 빈을 정의하는 가장 일반적인 방법이다. <bean> 태그와 동일한 역할을 한다.

@Service : 서비스 레이어, 비즈니스 로직을 가진 클래스에 사용한다.

@Repository : 퍼시스턴스 레이어, 영속성을 가지는 속성을 가진 DAO 클래스에 사용한다.

@Controller : 프레젠테이션 레이어, 웹 어플리케이션에서 웹 요청과 응답을 처리하는 클래스에 사용한다.

```java
@Component
public class DataServiceImpl implements DataService

@Component
public class BusinessServiceImpl implements BusinessService

@Repository
public class DataServiceImpl implements DataService

@Service
public class BusinessServiceImpl implements BusinessService
```

### 질문 2. 스프링 IoC 컨테이너는 빈의 의존 관계를 어떻게 알 수 있을까?

DataServiceImpl 클래스의 빈을 BusinessServiceImpl 클래스의 빈에 주입해야 한다.
-> BusinessServiceImpl 클래스에서 DataService 인터페이스의 인스턴스 변수에 @Autowired 어노테이션을 지정해 주입할 수 있다.

```java
@Service
public class BusinessServiceImpl {

  @Autowired // 데이터 레이어 dataService 가 비즈니스 레이어 BusinessServiceImpl 에 의존한다.
  private DataService dataService;
```

빈과 해당 와이어링을 정의했는데 테스트하려면 DataService 구현이 필요하다.
DataServiceImpl 은 몇 가지 데이터를 리턴한다.

```java
@Repository
public class DataServiceImpl implements DataService {
  public List<Data> retrieveData(User user) {
    return Arrays.asList(new Data(10), new Data(20));
  }
}
```

#### 학습목표3

@Component 어노테이션과 @Autowired을 사용해 올바르게 컴포넌트 간 의존 관계를 주입하면 애플리케이션을 테스트할 수 있다.

### 스프링 IoC 컨테이너( = Spring DI 컨테이너) 생성

스프링 IoC 컨테이너( = Spring DI 컨테이너)가 관리하는 객체를 빈(Bean) 이라고 하고, 이 빈들을 관리한다는 의미로 컨테이너를 빈 팩토리(BeanFactory)라고 부른다.

1. 빈 팩토리(BeanFactory)
	- 스프링의 IoC를 담당하는 핵심 컨테이너
	- Bean을 등록, 생성, 조회, 반환하는 기능을 담당
	- getBean() 메서드가 정의되어 있음.
	
2. 애플리케이션 컨텍스트(ApplicationContext)
	- BeanFactory를 확장한 IoC 컨테이너
	- Bean을 등록하고 관리하는 기능은 BeanFactory와 동일하지만, 스프링이 제공하는 각종 부가 서비스를 추가로 제공한다.
	- ApplicationContext 를 BeanFactory 보다 더 많이 사용한다.


#### 학습목표4

애플리케이션 컨텍스트를 사용해 스프링 IoC 컨테이너를 생성해보자.
메이븐으로 프로젝트를 빌드한다. 핵심 스프링 JAR 파일은 다음과 같다.

```java
<dependencies>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-beans</artifactId>
		</dependency>
</dependencies>
```

스프링 BOM 으로 의존 관계 버젼을 관리한다. dependencyManagement 에서 spring-framework-bom 을 가져오면 관리할 스프링 버젼을 지정할 필요가 없다.

```java
<properties>
	<spring.version>5.1.3.RELEASE</spring.version>
</properties>

<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-framework-bom</artifactId>
			<version>${spring.version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

#### 학습목표6

#### 애플리케이션 컨텍스트의 자바 구성(Java Configuration)

**설정 메타정보(Configuration metadata)** : 
- 애플리케이션 컨테스트 또는 빈 팩토리가 IoC를 적용하기 위해 사용하는 메타 정보를 말한다.
- 설정 메타정보는 IoC 컨테이너에 의해 관리되는 Bean 객체를 생성하고 구성할 때 사용된다.

**설정 메타 정보 구성 전략**
1. XML 설정 단독 사용
	- 모든 Bean을 명시적으로 XML에 등록하는 방법이다.
2. 어노테이션과 XML 설정 혼용
	- @Component 어노테이션이 선언된 클래스를 자동으로 찾아서 Bean으로 등록해주는 방식.
	- 빈 스캐닝(Bean Scanning)을 통한 자동 인식 Bean 등록 기능
3. 어노테이션 설정 단독 사용 -> 자바 컨텍스트 구성
	- @Configuration 어노테이션과 @Bean 어노테이션을 이용해서 스프링 컨테이너에 새로운 빈 객체를 제공할 수 있다.
	-  XML을 전혀 사용하지 않고도 자바 코드 내부에서 Bean의 등록 및 Bean들간의 의존 관계 연결 설정을 할 수 있다.

#### 전략 3. 자바 컨텍스트 구성

```java
@Configuration
class SpringContext {
}
```
@Configuration : 스프링 IoC 컨테이너가 해당 어노테이션이 선언된 클래스를 Bean 정의 설정용으로 사용한다는 것을 나타낸다.

애플리케이션 컨텍스트를 시작하려면 기본 메소드로 자바 클래스를 생성해야 한다.
자바 컨텍스트 기반으로 애플리케이션 컨텍스트를 작성하려면, **AnnotationConfigApplicationContext** 로 애플리케이션 컨텍스트를 시작하기 위해 메인 메소드를 사용한다.

```java 
public class LaunchJavaContext {

  private static final User DUMMY_USER = new User("dummy");

  public static Logger logger = Logger.getLogger(LaunchJavaContext.class);

  public static void main(String[] args) {
    ApplicationContext context = new AnnotationConfigApplicationContext( 
        SpringContext.class); 
        // 자바 컨텍스트 기반으로 애플리케이션 컨텍스트를 작성하려면, AnnotationConfigApplicationContext 를 사용

    BusinessService service = context.getBean(BusinessService.class); 
    // 컨텍스트가 시작되면 비즈니스 서비스 빈을 가져와야 한다. 이때 빈의 유형(BusinessService.class)을 전달하는 getBean 메소드를 인수로 사용한다.
    logger.debug(service.calculateSum(DUMMY_USER));
  }
}
```

#### 학습목표5

#### 컴포넌트 스캔

컴포넌트 스캔(@Component Scan) 을 정의해 스프링 IoC 컨테이너에게 빈과 의존 관계를 검색할 패키지를 알린다.

```java
@Configuration
@ComponentScan(basePackages = { "com.mastering.spring" })
class SpringContext {
}
```

**사전 작업**
1. com.mastering.spring 패키지의 컴포넌트 스캔으로 @Configuration 어노테이션을 사용해 스프링 구성 클래스 SpringContext를 정의했다.
2. 앞의 패키지에는 몇 개의 파일이 있다.
	- @Service 어노테이션이 있는 BusinessServiceImpl
	- @Repository 어노테이션이 있는 DataServiceImpl
3. BusinessServiceImpl 에는 DataService 인스턴스에 @Autowired 어노테이션이 있다.

**수행 프로세스**
4. com.mastering.spring 패키지를 스캔하고 BusinessServiceImpl 와 DataServiceImpl 빈을 찾는다.
5. DataServiceImpl 에는 의존 관계가 없다. 따라서 DataServiceImpl 을 위한 빈이 생성된다.
6. BusinessServiceImpl 은 DataService 에 의존한다. DataServiceImpl 은 DataService 인터페이스의 구현체다. 따라서 오토와이어링 기준과 일치한다. BusinessServiceImpl 빈이 생성되고 DataServiceImpl 로 생성된 빈이 setter를 통해 자동으로 연결된다.

#### 애플리케이션 컨텍스트 실행

실행 결과

```java
[2022-01-05 15:26:59,762] DEBUG com.mastering.spring.context.LaunchJavaContext 30 
```

요약

스프링 애플리케이션을 시작하면 다음 단계가 수행된다.
- 빈과 의존 관계를 식별하기 위해 컴포넌트 스캔이 수행된다.
- 빈이 생성되고 필요에 따라 의존 관계가 연결된다.

## 스프링 프레임워크

### 애플리케이션 컨텍스트에 XML 구성 사용

단계
1. XML 스프링 구성 정의
2. XML 구성으로 애플리케이션 컨텍스트 시작

#### 스프링 구성 정의
```java
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:context="http://www.springframework.org/schema/context"
	xmlns:tx="http://www.springframework.org/schema/tx" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd            
	                       http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd            
	                       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd            
	                   
	                       http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

	<context:component-scan base-package="com.mastering.spring"/>

</beans>
```
구성요소 스캔은 context: component-scan 을 사용해 정의된다.

### XML 구성으로 애플리케이션 컨텍스트 시작

ClassPathXmlApplicationContext를 사용해 애플리케이션 컨텍스트를 시작한다.
```java
public class LaunchXmlContext {

  private static final User DUMMY_USER = new User("dummy");

  public static Logger logger = Logger.getLogger(LaunchJavaContext.class);

  public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext(
        "BusinessApplicationContext.xml");

    BusinessService service = context.getBean(BusinessService.class);
    logger.debug(service.calculateSum(DUMMY_USER));
  }

}
```

### @Autowired 어노테이션

@Autowired 를 의존 관계에 사용하면 애플리케이션 컨텍스트는 일치하는 의존 관계를 검색한다. 기본적으로 오토와이어링된 모든 의존 관계가 필요하다.

가능한 결과
- 일치 항목이 1개 있다. -> 찾고 있는 의존 관계
- 일치 항목이 2개 이상 발견됐다 -> 오토와이어링 실패
	- @Primary 어노테이션 -> 후보 중 하나만 사용
	- @Qualifier -> 오토와이어링을 더욱 강화
- 일치하는 항목이 없다 -> 오토와이어링 실패

### @Primary 어노테이션

특정 의존 관계를 오토와이어링할 수 있는 후보가 둘 이상 있을 때, @Primary 어노테이션을 사용한 빈이 호출된다.

```java
interface SortingAlgorithm {

}

@Component
class MergeSort implements SortingAlgorithm {
  // Class code here
}

@Component
@Primary
class QuickSort implements SortingAlgorithm {
  // Class code here
}
```

컴포넌트 스캔에서 두 @Component를 찾으면 QuickSort 는 @Primary 어노테이션 때문에 SortingAlgorithm 의 의존 관계를 연결하는 데 사용된다.

### @Qualifier

@Qualifier 어노테이션은 스프링 빈에 참조를 주기 위해 사용된다. 참조는 오토와이어링할 필요가 있는 의존 관계를 규정하는 데 사용된다.

```java
@Component
@Qualifier("mergesort")
class MergeSort implements SortingAlgorithm {
  // Class code here
}

@Component
class QuickSort implements SortingAlgorithm {
  // Class code here
}

@Component
class SomeService {
  @Autowired
  @Qualifier("mergesort")
  SortingAlgorithm algorithm;
}
```

SomeService 클래스에서 @Qualifier("mergesort") 가 사용됐기 때문에 MergeSort 가 오토와이어링을 위해 선택된 후보 의존 관계가 된다.

### 의존 관계 주입 옵션

1. setter 주입
2. 생성자 주입

#### setter 주입

setter 주입은 setter 메소드를 통해 의존 관계를 주입하는 데 사용된다.

```java
public class BusinessServiceImpl {
  private DataService dataService;

  @Autowired
  public void setDataService(DataService dataService){
    this.dataService = dataService;
  }
}
```
DataService 인스턴스는 setter 주입을 사용한다.
실제로 setter 주입을 사용하기 위해 setter 메소들를 선언할 필요는 없다.
변수에 @Autowired 를 지정하면 스프링은 자동으로 setter 주입을 사용한다.

따라서 위의 코드는 아래와 같이 변경될 수 있다.

```java
public class BusinessServiceImpl {
  @Autowired
  private DataService dataService;
}
```

#### 생성자 주입

생성자 주입은 의존 관계 주입을 위해 생성자를 이용한다.

```java
public class BusinessServiceImpl {
  private DataService dataService;

  @Autowired
  public BusinessServiceImpl(DataService dataService){
    super();
    this.dataService = dataService;
  }
}
```

### 스프링 빈 스코프 사용자 정의

스프링 빈은 여러 스코프로 만들 수 있다.
기본 스코프는 싱글톤이다.

		스프링 프레임워크 컨텍스트의 싱글톤은 스프링 애플리케이션 컨텍스트당 하나의 인스턴스를 가진 클래스다.

싱글톤 빈의 인스턴스는 하나 뿐이므로 요청(request)와 관련된 데이터를 포함할 수 없다.
스코프는 모든 스프링 빈에서 @Scope 어노테이션과 함께 제공될 수 있다.

```java
@Service
@Scope("singleton")
public class BusinessServiceImpl implements BusinessService
```

**스코프의 종류**
- 싱글톤 : 기본적으로 모든 빈의 스코프는 싱글톤이다. 빈의 인스턴스는 스프링 IoC 컨테이너의 인스턴스당 하나만 사용된다. 빈에 여러 참조가 있어도 컨테이너당 한 번만 생성된다. 싱글톤 인스턴스는 캐싱돼 빈을 사용하는 모든 후속 요청에 사용된다. 스프링 싱글톤 스코프가 하나의 스프링 컨테이너당 하나의 객체라는 것을 지정하는 것은 중요하다. 단일 JVM에 여러 개의 스프링 컨테이너가 있다면 같은 빈의 인스턴스는 여러 개 있을 수 있다.

- 프로토타입 : 스프링 컨테이너에서 빈을 요청할 때마다 새 인스턴스가 생성된다. 빈에 상태가 포함되어 있는 경우, 프로토타입 스코프를 사용하는 것이 좋다.

- 리퀘스트 : 스프링 웹 컨텍스트에서만 사용할 수 있다. 모든 HTTP 요청마다 빈의 새 인스턴스가 생성된다. 빈은 요청 처리가 완료되는 즉시 폐기된다. 단일 요청과 관련된 데이터를 보유하고 있는 빈에 이상적이다.

- 세션 : 스프링 웹 컨텍스트에서만 사용할 수 있다. 모든 HTTP 세션(session)마다 빈의 새로운 인스턴스가 생성된다. 웹 애플리케이션의 사용자 권한과 같이 단일 사용자 고유의 데이터에 적합하다.

- 애플리케이션 : 스프링 웹 컨텍스트에서만 사용 가능하다. 웹 애플리케이션당 하나의 빈 인스턴스로 특정 환경의 애플리케이션 구성에 적합하다.

### 기타 중요한 스프링 어노테이션

- @ScopedProxy : 요청이나 세션 스코프의 빈을 싱글톤-스코프의 빈에 주입해야 할 때가 있다. @ScopedProxy 어노테이션은 싱글톤-스코프 빈에 주입되는 스마트 프록시를 제공한다.

- @PostConstruct : 모든 스프링 빈에서 @PostConstruct 어노테이션을 사용해 post construct 메소드를 제공한다. post construct 메소드는 빈이 의존 관계로 완전히 초기화된 후에 호출된다. 이것은 빈 생명 주기 동안 한 번만 호출된다.

- @PreDestroy : 모든 스프링 빈에서 @PreDestory 어노테이션으로 predestroy 메소드를 제공한다. predestroy 메소드는 컨테이너에서 빈이 제거되기 전에 호출된다. 빈이 보유한 모든 리소스를 해제하는 데 사용되는데 리소스 누수가 방지된다.

### CDI 탐색
생략

## 스프링 애플리케이션 컨텍스트의 단위 테스트

		TDD(Test Driven Development) 를 사용하면, 간단하고 유지 관리가 가능하며 테스트할 수 있는 코드가 된다.

### 단위 테스트의 개념

단위테스트에는 개별 클래스와 메소드에 돌립적인 자동 테스트 작성이 포함된다.
단위 테스트는 배포된 전체 애플리케이션을 테스트하는 대신 각 클래스와 각 메소드의 책임에 자동화된 테스트를 작성하는 데 중점을 둔다.

#### 단위 테스트의 장점
- 미래 결함에 대비한 안전망
- 결함의 조기 발견이 가능하다
- TDD를 통해 더 나은 디자인을 만든다.
- 잘 작성된 테스트는 코드와 기능의 문서화 역할을 한다.

### 스프링 컨텍스트를 사용해 JUnit 작성

스프링 프레임워크로 단위 테스트를 작성할 때는 애플리케이션 컨텍스트를 시작해야 한다. 애플리케이션 컨텍스트는 모든 빈과 의존 관계를 초기화한다. 애플리케이션 컨텍스트에서 빈을 가져와 예상 값을 리턴하는 지 확인할 수 있다.

단위 테스트에서 애플리케이션 컨텍스트를 시작하기 위해 SpringJUnit4ClassRunner.class 를 러너로 사용한다.

```java
@RunWith(SpringJUnit4ClassRunner.class)
```

XML 구성으로 애플리케이션 컨텍스트를 시작한다. 

```java
@ContextConfiguration(locations = { "/BusinessApplicationContext.xml" })
```

애플리케이션 컨텍스트가 시작되면 애플리케이션 컨텍스트에서 빈을 가져와 단위 테스트에 사용한다. @Autowired 어노테이션을 사용해 테스트에서 멤버 변수를 정의한다.

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "/BusinessApplicationContext.xml" })
public class BusinessServiceJavaContextTest {
  private static final User DUMMY_USER = new User("dummy");

  @Autowired
  private BusinessService service;

  @Test
  public void testCalculateSum() {
    long sum = service.calculateSum(DUMMY_USER);
    assertEquals(30, sum);
  }

}
```

여기서 DataService를 실제로 구현하지 않고 어떻게 BusinessServiceImpl 단위 테스트를 할까?
-> DataService의 모크를 생성하고 모크를 BusinessServiceImpl 에 오토와이어링한다.

### 모크를 사용한 단위 테스트

#### Mocking
실제 객체의 동작을 시뮬레이션하는 객체를 만든다.
이것은 런타임에 동적으로 생성될 수 있다.

모키토 어노테이션을 사용할 때는 MockitoJUnitRunner를 사용해야 한다.
MockitoJUnitRunner 는 @Mock 어노테이션으로 어노테이션이 달린 빈을 초기화하고 각 테스트 메소드를 실행한 후 프레임워크의 사용법을 검증한다.

```java
@RunWith(MockitoJUnitRunner.class)
public class BusinessServiceMockitoTest {
  private static final User DUMMY_USER = new User("dummy");

  @Mock // DataService의 모크를 생성
  private DataService dataServiceMock;

  @InjectMocks // 서비스의 모든 의존 관계를 오토와이어링
  private BusinessService service = new BusinessServiceImpl();

  @Test
  public void testCalculateSum() {
    // 메소드가 호출될 때 테스트 데이터를 리턴하는 모크를 만든다.
    // retrieveData 메소드를 모킹하기 위해 모키토가 제공하는 BDD 스타일 메소드를 사용한다.
    BDDMockito.given(dataServiceMock.retrieveData(Matchers.any(User.class)))
    .willReturn(Arrays.asList(new Data(10), new Data(15), new Data(25)));

    long sum = service.calculateSum(DUMMY_USER);
    assertEquals(10 + 15 + 25, sum);
  }
}
```




## Reference
1. [스프링 빈(Spring Bean)이란? 개념 정리](https://melonicedlatte.com/2021/07/11/232800.html)
2. [[스프링프레임워크] 빈 와이어링 -1.자동으로 빈 와이어링](https://m.blog.naver.com/kimnx9006/220600048654)
3. [백정숙 강사님의 스프링 프레임워크 입문과 활용](https://drive.google.com/file/d/18hQMQMOHg_jvDdn4eK6jVdncYmzzzfeT/view?usp=sharing)