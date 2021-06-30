## SOLID 5원칙
<ol>
  <li>
    SRP: 단일 책임 원칙(single responsibility principle)
    <ul>
      <li>
        한 클래스는 하나의 책임만 가져야 한다.
      </li>
      <li>
        하나의 책임이라는 것은 모호함. 클 수도 있고, 작을 수 있기 때문에 문맥과 상황에 따라 다르다.
      </li>
      <li>
        중요한 기준은 변경이다. 변경이 있을 때 파급 효과가 적으면 단일 책임 원칙을 잘 따른것
      </li>
      <li>
        ex) UI 변경, 객체 생성과 사용을 분리
      </li>
    </ul>
  </li>
  <li>
    OCP: 개방-폐쇄 원칙(Oen/closed principle)
    <ul>
      <li>
        소프트웨어 요소는 확장에는 열려 있으나, 변경에는 닫혀 있어야 한다.
      </li>
      <li>
        인터페이스를 구현한 새로운 클래스를 하나 만들어서 새로운 기능을 구현 
      </li>
      <li>
        지금까지 배운 역할과 구현의 분리를 생각해보자
      </li>
    </ul>
  </li>
  <li>
    LSP: 리스코프 치환 원칙(Liskov substitution principle)
    <ul>
      <li>
        프로그램의 객체는 프로그램의 정확성을 깨뜨리지 않으면서 하위 타입의 인스턴스로 바꿀 수 있어야 한다.
      </li>
      <li>
        다형성에서 하위 클래스는 인터페이스 규약을 다 지켜야 한다는 것, 다형성을 지원하기 위한 원칙, 인터페이스를 구현한 구현체는 믿고 사용하려면, 이 원칙이 필요.
      </li>
      <li>
        단순히 컴파일에 성공하는 것을 넘어서는 이야기
      </li>
      <li>
        ex) 자동차 인터페이스의 엑셀은 앞으로 가라는 기능이지, 뒤로 가게 구현하면 LSP 위반, 느리더라도 앞으로 가야 한다.
      </li>
    </ul>
  </li>
  <li>
    ISP: 인터페이스 분리 원칙(Interface segregation principle)
    <ul>
      <li>
        특정 클라이언트를 위한 인터페이스 여러 개가 범용 인터페이스 하나보다 낫다
      </li>
      <li>
        자동차 인터페이스 -> 운전 인터페이스, 정비 인터페이스로 분리
      </li>
      <li>
        사용자 클라이언트 -> 운전자 클라이언트, 정비사 클라이언트로 분리
      </li>
      <li>
        분리하면 정비 인터페이스 자체가 변해도 운전자 클라이언트에 영향을 주지 않음
      </li>
      <li>
        인터페이스가 명확해지고, 대체 가능성이 높아진다.
      </li>
    </ul>
  </li>
  <li>
    DIP: 의존관계 역전 원칙(Dependency inversion principle)
    <ul>
      <li>
        프로그래머는 "추상화에 의존해야지, 구체화에 의존하면 안된다." 의존성 주입은 이 원칙을 따르는 방법 중 하나다.
      </li>
      <li>
        쉽게 이야기해서 구현 클래스에 의존하지 말고, 인터페이스에 의존하라는 뜻
      </li>
      <li>
        역할(Role)에 의존하게 해야 한다는 것과 같다. 객체 세상도 클라이언트가 인터페이스에 의존해야 유연하게 구현체를 변경할 수 있다! 
        구현체에 의존하게 되면 변경이 아주 어려워진다.
      </li>
    </ul>
  </li>
</ol>

## 다형성 정리
<ul>
  <li>
    객체 지향의 핵심은 다형성
  </li>
  <li>
    다형성 만으로는 쉽게 부품을 갈아 끼우듯이 개발할 수 없다.
  </li>
  <li>
    다형성 만으로는 구현 객체를 변경할 때 클라이언트 코드도 함께 변경된다. (DIP 위반)
  </li>
  <li>
    다형성 만으로는 OCP, DIP를 지킬 수 없다.
  </li>
  <li>
    다형성 말고 뭔가 더 필요하다.
  </li>
</ul>

## 스프링
<ul>
  <li>
    스프링은 다음 기술로 다형성 + OCP, DIP를 가능하게 지원
    <ul>
      <li>
        DI(Dependency Injection):의존 관계, 의존성 주입
      </li>
      <li>
        DI 컨테이너 제공
      </li>
    </ul>
  </li>
  <li>
    클라이언트 코드의 변경 없이 기능 확장
  </li>
  <li>
    쉽게 부품을 교체하듯이 개발
  </li>
  <li>
    객체 지향 개발을 하기 위해서 OCP, DIP원칙을 지키면서 개발을 하다보면 결국엔 스프링이라는 프레임워크를 하나 만들게 되는 것이다.
  </li>
</ul>

## AppConfig
- 애플리케이션의 전체 동작 방식을 구성(config)하기 위해, "구현 객체를 생성하고" "연결"하는 책임
- 클라이언트 객체의 역할이 명확해짐 ( 실행하는 것에만 집중 가능 )
- 애플리케이션이 크게 "사용영역"과, 객체를 생성하고 구성하는 "구성"영역으로 분리 해줌
- OCP, DIP원칙을 지킬 수 있도록 해줌.

## SOLID 5원칙 적용
1. SRP
   - "한 클래스는 하나의 책임만 가져야 한다"
   - 구현 객체를 생성하고 연결 = AppConfig / 실행 = 클라이언트 객체 로 분리함으로써 각 객체(클래스)는 하나의 책임만 갖도록 해주었다.

2. DIP
    - "프로그래머는 추상화에 의존해야지, 구체화에 의존하면 안된다." 의존성 주입은 이 원칙을 따르는 방법 중 하나다"
    - Appconfig가 `FixDiscountPolicy` 객체 인스턴스를 클라이언트 코드 대신 생성해서 클라이언트 코드에 의존관계를 주입했다. 이렇게 해서 DIP 원칙을 따르면서 문제도 해결함
    - 클라이언트 코드가 `DiscountPolicy`와 같은 추상화 인터페이스에만 의존해야함.

3. OCP
    - "소프트웨어 요소는 확장에는 열려 있으나 변경에는 닫혀 있어야 한다."
    - AppConfig가 의존관계를 `FixDiscountPolicy` -> `RateDiscountPolicy`로 변경해서 클라이언트 코드에 주입하므로 클라이언트 코드는 변경하지 않아도 됨
    - 소프트웨어 요소를 새롭게 확장해도 사용 영역의 변경은 닫혀 있다.
   
## IoC, DI, And Container
1. 제어의 역전: IoC(Inversion of Control)
   - 기존 프로그램은 클라이언트 구현 객체가 스스로 필요한 서버 구현 객체를 생성하고, 연결하고 실행했다. 한마디로 구현 객체가 프로그램의 제어 흐름을 스스로 조종했다 .개발자 입장에서는 자연스러운 흐름이다.
   - 반면에 AppConfig가 등장한 이후에 구현 객체는 자신의 로직을 실행하는 역할만 담당한다. 프로그램의 제어 흐름은 이제 AppConfig가 가져간다. 예를 들어서 `OrderServiceImpl`은 필요한 인터페이스들을 호출하지만 어떤 구현 객체들이 실행 될지 모른다.
   - 프로그램에 대한 제어 흐름에 대한 권한은 모두 AppConfig가 가지고 있다. 심지어 `OrderServiceImpl`도 AppConfig가 생성한다. 그리고 AppConfig는 `OrderServgiceImpl`이 아닌 OrderService 인터페이스의 다른 구현 객체를 생성하고 실행할 수 도 있다. 그런 사실도 모른체 `OrderServiceImpl` 은 묵묵히 자신의 로직을 실행할 뿐이다.
   - 이렇듯 프로그램의 제어 흐름을 직접 제어하는 것이 아니라 외부에서 관맇나느 것을 제어의 역전(IoC)라고 한다.
   
2. 프레임워크 vs 라이브러리
   - 프레임워크가 내가 작성한 코드를 제어하고, 대신 실행하면 그것은 프레임워크가 맞다. (JUnit)
   - 반면에 내가 작성한 코드가 직접 제어의 흐름을 담당한다면 그것은 프레임워크가 아니라 라이브러리다.
   - 결국, 프레임워크는 분명한 "제어의 역전(IoC)" 개념이 적용되어 있어야 한다.
   
3. 의존관계 주입DI(Dependency Injection)
   - 의존관계는 정적인 클래스 의존 관계와, 실행 시점에 결정되는 동적인 객체(인스턴스) 의존 관계 둘을 분리해서 생각해야 한다.
   - 정적인 클래스 의존관계: 클래스가 사용하는 import 코드만 보고 의존관계를 쉽게 판단할 수 있다. 정적인 의존관계는 애플리케이션을 실행하지 않아도 분석이 가능
   - 동적인 클래스 의존관계: 애플리케이션 실행 시점에 실제 생성된 객체 인스턴스의 참조가 연결된 의존 관계다.
   - 애플리케이션 "실행 시점(런타임)"에 외부에서 실제 구현 객체를 생성하고 클라이언트에 전달해서 클라이언트와 서버의 실제 의존관계가 연결 되는 것을 "의존관계 주입" 이라 한다.
   - 의존관계 주입을 사용하면 클라이언트 코드를 변경하지 않고, 클라이언트가 호출하는 대상의 타입 인스턴스를 변경할 수 있다.
   - 의존관계 주입을 사용하면 정적인 클래스 의존관계를 변경하지 않고, 동적인 객체 인스턴스 의존관계를 쉽게 변경할 수 있다.
   
4. Ioc 컨테이너, DI 컨테이너
   - AppConfig 처럼 객체를 생성하고 관리하면서 의존관계를 연결해주는 것을 IoC 컨테이너 또는 DI 컨테이너라고 한다.
   - 의존관계 주입에 초점을 맞추어 최근에는 주로 DI 컨테이너라 한다.
   - 또는 어셈블러, 오브젝트 팩토리 등으로 불리기도 한다.
   
## 스프링 컨테이너
- `ApplicationContext` 를 스프링 컨테이너라 한다.
- 기존에는 개발자가 `AppConfig` 를 사용해서 직접 객체를 생성하고 DI를 했지만, 이제부터는 스프링 컨테이너를 통해서 사용 한다.
- 스프링 컨테이너는 `Configuration`이 붙은 `AppConfig`를 설정(구성) 정보로 사용한다. 여기서 `@Bean`이라 적힌 메서드를 모두 호출해서 반환된 객체를 스프링 컨테이너에 등록한다. 이렇게 스프링 컨테이너에 등록된 객체를 스프링 빈이라 한다.
- 스프링 빈은 `@Bean` 이 붙은 메서드의 명을 스프링 빈의 이름으로 사용한다. (`memberService`, `orderService`)
- 이전에는 개발자가 필요한 객체를 `AppConfig` 를 사용해서 직접 조회했지만, 이제부터는 스프링 컨테이너를 통해서 필요한 스프링 빈(객체)를 찾아야 한다. 스프링 빈은 `applicationContext.getBean()`메서드를 사용해서 찾을 수 있다.
- 기존에는 개발자가 직접 자바코드로 모든것을 했다면 이제부터는 스프링 컨테이너에 객체를 스프링 빈으로 등록하고, 스프링 컨테이너에서 스프링 빈을 찾아서 사용하도록 변경되었다.
- 코드가 약간 더 복잡해진 것 같은데, 스프링 컨테이너를 사용하면 어떤 장점이 있을까? 

## 스프링 컨테이너 생성
- `ApplicationContext` 는 인터페이스이다.
- 스프링 컨테이너는 XML을 기반으로 만들 수 있고, 애노테이션 기반의 자바 설정 클래스로 만들 수 있다.
- 직전에 `AppConfig` 를 사용했던 방식이 애노테이션 기반의 자바 설정 클래스로 스프링 컨테이너를 만든 것이다.
- 자바 설정 클래스를 기반으로 스프링 컨테이너(`ApplicationContext`)를 만들어보자.
   - `new AnnotationConfigApplicationContext(AppConfig.class);`
   - 이 클래스는 `ApplicationContext` 인터페이스의 구현체이다.
   
## 스프링 컨테이너의 생성 과정
1. 스프링 컨테이너 생성
   - `new AnnotationConfigApplicationContext(AppConfig.class)`
   - 스프링 컨테이너를 생성할 때는 구성 정보를 지정해주어야 한다.
   - 여기서는 `AppConfig.class`를 구성 정보로 지정했다.
2. 스프링 빈 등록
   - 스프링 컨테이너는 파라미터로 넘어온 설정 클래스 정보를 사용해서 스프링 빈을 등록한다.
3. 스프링 빈 의존관계 설정 - 준비
   - 빈 객체 생성
4. 스프링 빈 의존관계 설정 - 완료 
   - 스프링 컨테이너는 설정 정보를 참고해서 의존관계를 주입(DI)한다.
   - 단순히 자바 코드를 호출하는 것 같지만, 차이가 있다. 이 차이는 뒤에 싱글톤 컨테이너에서 설명
   
## 스프링 빈 조회 - 상속관계
- 부모 타입으로 조회하면, 자식 타입도 함께 조회한다.
- 그래서 모든 자바 객체의 최고 부모인 `Object` 타입으로 조회하면, 모든 스프링 빈을 조회한다.

## BeanFactory 와 ApplicationContext
1. BeanFactory
   - 스프링 컨테이너의 최상위 인터페이스다.
   - 스프링 빈을 관리하고 조회하는 역할을 담당한다.
   - `getBean()` 을 제공한다.
   - 지금까지 우리가 사용했던 대부분의 기능은 BeanFactory가 제공하는 기능이다.
   - 직접 BeanFactory를 직접 사용할 일은 거의 없다. 부가기능이 포함된 ApplicationContext를 사용한다.
   
2. ApplicationContext
   - BeanFactory 기능을 모두 상속받아서 제공한다
   - 빈을 관리하고 검색하는 기능을 BeanFactory가 제공해주는데, 그러면 둘의 차이가 뭘까?
   - 애플리케이션을 개발할 때는 빈은 관리하고 조회하는 기능은 물론이고, 수 많은 부가기능이 필요하다

3. ApplicationContext가 제공하는 부가기능
   - 메시지소스를 활용한 국제화 기능
      - 예를 들어서 한국에서 들어오면 한국어로, 영어권에서 들어오면 영어로 출력
   - 환경변수
      - 로컬, 개발, 운영등을 구분해서 처리
   - 애플리케이션 이벤트
      - 이벤트를 발행하고 구독하는 모델을 편리하게 지원
   - 편리한 리소스 조회
      - 파일, 클랙스패스, 외부 등에서 리소스를 편리하게 조회
   
4. BeanFactory나 ApplicationContext를 스프링 컨테이너라 한다.

## 다양한 설정 형식 지원 - 자바 코드, XML
- 스프링 컨테이너는 다양한 형식의 설정 정보를 받아드릴 수 있게 유연하게 설계되어 있다.
   - 자바 코드, XML, Groovy 등등
- 애노테이션 기반 자바 코드 설정 사용
   - 지금까지 한 내용.
   - `new AnnotationConfigApplicationContext(Appconfig.class)`
   - `AnnotationConfigApplicationContext` 클래스를 사용하면서 자바 코드로된 설정 정보를 넘기면 된다.
   - 팩토리 메서드를 활용해서 등록 하는 방법
- XML 설정 사용
   - 최근에는 스프링 부트를 많이 사용하면서 XML기반의 설정은 잘 사용하지 않는다. 아직 많은 레거시 프로젝트 들이 XML로 되어 있고, 또 XML을 사용하면 컴파일 없이 빈 설정 정보를 변경할 수 있는 장점도 있으므로 한번쯤 배워두는 것도 괜찮다.
   - `GenericXmlApplicationContext`를 사용하면서 `xml`설정 파일을 넘기면 된다.
   - 직접 스프링 빈에 등록 하는 방법
   

## 스프링 빈 설정 메타 정보 - BeanDefinition
- 스프링은 어덯게 이런 다양한 설정 형식을 지원하는 것일까? 그 중심에는 `BeanDefinition` 이라는 추상화가 있다.
- 쉽게 이야기해서 "역할과 구현을 개념적으로 나눈것" 이다.
    - XML을 읽어서 BeanDefinition을 만들면 된다.
    - 자바 코드를 읽어서 BeanDefinition을 만들면 된다.
    - 스프링 컨테이너는 자바 코드인지, XML인지 몰라도 된다. 오직 BeanDefinition만 알면 된다.
- `BeanDefinition`을 빈 설정 메타정보라 한다.
    - `@Bean`, `<bean>`당 각각 하나씩 메타 정보가 생성된다.
- 스프링 컨테이너는 이 메타정보를 기반으로 스프링 빈을 생성한다.


## BeanDefinition 정보
- BeanClassName: 생성할 빈의 클래스 명(자바 설정 처럼 팩토리 역할의 빈을 사용하면 없음)
- factoryBeanName: 팩토리 역할의 빈을 사용할 경우 이름, 예) appConfig
- factoryMethodName: 빈을 생성할 팩토리 메서드 지정, 예) memberService
- Scope: 싱글톤(기본값)
- lazyInit: 스프링 컨테이너를 생성할 때 빈을 생성하는 것이 아니라, 실제 빈을 사용할 때 까지 최대한 생성을 지연처리 하는지 여부
- InitMethodName: 빈을 생성하고, 의존관계를 적용한 뒤에 호출되는 초기화 메서드 명
- DestroyMethodName: 빈의 생명주기가 끝나서 제거하기 직전에 호출되는 메서드 명
- Constructor arguments, Properties: 의존관계 주입에서 사용한다. (자바 설정 처럼 팩토리 역할의 빈을 사용하면 없음)
- BeanDefinition에 대해서는 너무 깊이있게 이해하기 보다는, 스프링이 다양한 형태의 설정 정보를 BeanDefinition으로 추상화 해서 사용하는 것 정도만 이해하면 된다.
- 가끔 스프링 코드나 스프링 관련 오픈 소스의 코드를 볼 때, BeanDefinition 이라는 것이 보일 때가 있다. 이때 이러한 메커니즘을 떠올리면 된다.

## 웹 애플리케이션과 싱글톤
- 스프링은 태생이 기업용 온라인 서비스 기술을 지원하기 위해 탄생했다.
- 대부분의 스프링 애플리케이션은 웹 애플리케이션이다. 물론 웹이 아닌 애플리케이션 개발도 얼마든지 개발할 수 있다.
- 웹 애플리케이션은 보통 여러 고객이 동시에 요청을 한다
- 우리가 만들었던 스프링 없는 순수한 DI 컨테이너인 AppConfig는 요청을 할 때 마다 객체를 새로 생성한다.
- 고객 트래픽이 초당 100이 나오면 초당 100개 객체가 생성되고 소멸된다! -> 메모리 낭비가 심하다.
- 해결방안은 해당 객체가 딱 1개만 생성되고, 공유하도록 설계하면 된다 -> 싱글톤 패턴 

## 싱글톤 패턴
- 클래스의 인스턴스가 딱 1개만 생성되는 것을 보장하는 디자인 패턴이다.
- 그래서 객체 인스턴스를 2개 이상 생성하지 못하도록 막아야 한다.
    - private 생성자를 사용해서 외부에서 임의로 new 키워드를 사용하지 못하도록 막아야 한다.
- 방법 예시: 이방법은 객체를 미리 생성해두는 가장 단순하고 안전한 방법이다.
    1. static 영역에 객체 instance를 미리 하나 생성해서 올려둔다.
    2. 이 객체 인스턴스가 필요하면 오직 `getInstance()` 메서드를 통해서만 조회할 수 있다. 이 메서드를 호출하면 항상 같은 인스턴스를 반환한다.
    3. 딱 1개의 객체 인스턴스만 존재해야 하므로, 생성자를 private으로 막아서 혹시라도 외부에서 new 키워드로 객체 인스턴스가 생성되는 것을 막는다.
- 싱글톤 패턴을 적용하면 고객의 요청이 올 때 마다 객체를 생성하는 것이 아니라 ,이미 만들어진 객체를 공유해서 효율저긍로 사용 할 수 있다. 하지만 싱글톤 패턴은 다음과 같은 수 많은 문제점들을 가지고 있다.
- 싱글톤 패턴 문제점
    1. 싱글톤 패턴을 구현하는 코드 자체가 많이 들어간다.
    2. 의존관계상 클라이언트가 구체 클래스에 의존한다 -> DIP 위반( 클라이언트에서 getInstance 를 불러와야 하기 떄문)
    3. 클라이언트가 구체 클래스에 의존해서 OCP 원칙을 위반할 가능성이 높다.
    4. 테스트하기 어렵다.
    5. 내부 속성을 변경하거나 초기화 하기 어렵다.
    6. private 생성자로 자식 클래스를 만들기 어렵다.
    7. 결론적으로 유연성이 떨어진다. (Dependency Injection 와 같은 것을 적용하기 어렵다)
    8. 안티패턴으로 불리기도 한다.
- 스프링 컨테이너를 쓰면 자동으로 싱글톤 패턴으로 동작하기 떄문에, 동작 원리만 이해 해두면 된다.
- 스프링 프레임워크는 싱글톤의 문제점을 모두 해결 하고 있는 동시에 객체를 싱글톤으로 관리 해준다.

## 싱글톤 컨테이너
- 스프링 컨테이너는 싱글톤 패턴의 문제점을 모두 해결하면서, 객체 인스턴스를 싱글톤(1개만 생성)으로 관리한다.
- 지금까지 우리가 학습한 스프링 빈이 바로 싱글톤으로 관리되는 빈이다.
- 스프링 컨테이너는 싱글턴 패턴을 적용하지 않아도, 객체 인스턴스를 싱글톤으로 관리한다.
    - 이전에 설명한 컨테이너 생성 과정을 자세히 보면, 컨테이너는 객체를 하나만 생성해서 관리한다.
- 스프링 컨테이너는 싱글톤 컨테이너 역할을 한다. 이렇게 싱글톤 객체를 생성하고 관리하는 기능을 "싱글톤 레지스트리"라 한다.
- 스프링 컨테이너의 이런 기능 덕분에 싱글턴 패턴의 모든 단점을 해결하면서 객체를 싱글톤으로 유지할 수 있다.
    - 싱글톤 패턴을 위한 지저분한 코드가 들어가지 않아도 된다.
    - DIP, OCP, 테스트, private 생성자로 부터 자유롭게 싱글톤을 사용할 수 있다.
- 스프링 컨테이너 덕분에 고객의 요청이 올 때 마다 객체를 생성하는 것이 아니라, 이미 만들어진 객체를 공유해서 효율적으로 재사용 할 수 있다.
- 스프링의 기본 빈 등록 방식은 싱글톤이지만, 싱글톤 방식만 지원하는 것은 아니다. 요청할 때 마다 새로운 객체를 생성해서 반환하는 기능도 제공한다. 자세한 내용은 빈 스코프에서 정리
- 스프링은 99%이상 싱글톤을 사용한다. 특별한 기능을 사용할 때 httpRequest요청에 따라서 빈 라이프 사이클을 할 때 정도나 기타 특수 기능을 위해서만 다른 기능을 활용한다.


## 싱글톤 방식의 주의점
- 싱글톤 패턴이든, 스프링 같은 싱글톤 컨테이너를 사용하든, 객체 인스턴스를 하나만 생성해서 공유하는 싱글톤 방식은 여러 클라이언트가 하나의 같은 객체 인스턴스를 공유하기 때문에 싱글톤 객체는 상태를 유지(stateful)하게 설계하면 안된다.
- 무상태(stateless)로 설계해야 한다.
    - 특정 클라이언트에 의존적인 필드가 있으면 안된다.
    - 특정 클라인언트가 값을 변경할 수 있는 필드가 있으면 안된다!
    - 가급적 읽기만 가능해야 한다.
    - 필드 대신에 자바에서 공유되지 않는, 지역변수, 파라미터, ThreadLocal 등을 사용해야 한다.
- 스프링 빈의 필드에 공유 값을 설정하면 정말 큰 장애가 발생할 수 있다!!
    - ThreadA가 10000원 주문을 넣으면서 orderService라는 인스턴스를 생성한다.
    - ThreadB가 20000원 주문을 넣으면서 orderService라는 인스턴스를 생성한다.
    - ThreadA가 본인이 주문을 넣은 가격을 조회했을 때 10000원이 아닌 20000원이 등장하게 된다.
    - 이러한 상황이 price 필드값을 공유하도록 설계되어 있기 때문이다.
    - 그러므로 Spring은 항상 무상태(stateless)로 설계 해야 한다.
    - 위에 상황을 해결하기 위해서는 orderService의 price값을 전역변수를 사용하지 말고, 주문이 들어가는순간 반환(return)값으로 price를 던져주는 방식으로 해야한다. 즉, order를 진행한다음에 나중에 조회를 못하도록 해야 한다. (따로 price값을 계속 유지시키면 안됨)
    
## Configuration과 싱글톤
- `@Configuration` 은 싱글톤을 위해서 존재하는 것이다.

## @Configuration과 바이트코드 조작의 마법
- 스프링 컨테이너는 싱글톤 레지스트리다. 따라서, 스프링 빈이 싱글톤이 되도록 보장 해주어야 한다. 그런데, 스프링이 자바 코드까지 어떻게 하기는 어렵다. 
- `AnnotationConfigApplicationContext`에 파라미터로 넘긴 값은 스프링 빈으로 등록된다. 그래서 `AppConfig`도 스프링 빈이 된다.
- `AppConfig` 빈을 조회해보면, 순수 클래스랑은 다른 클래스명 인 것을 볼 수 있다. 이것은 내가 만든 클래스가 아니라 스프링이 CGLIB라는 바이트코드 조작 라이브러리를 사용해서 AppConfig 클래스를 상속받은 임의의 다른 클래스를 만들고, 그 다른 클래스를 스프링 빈으로 등록한 것이다.
- 바이트 코드 조작을 통해 임의의 다른 클래스가 싱글톤이 보장되도록 해준다. 이것은 하나의 빈을 등록할떄 이미 등록된 빈이라면 등록된 빈을 반환하고, 아니라면 우리가 작성한 new 키워드로 생성하는 코드를 통해서 새롭게 빈을 등록하는 방식으로 되어 있다.
- @Bean이 붙은 메서드마다 이미 스프링 빈이 존재하면 존재하는 빈을 반환하고, 스프링 빈이 없으면 생성해서 스플이 빈으로 등록하고 반환하는 코드가 동적으로 만들어지고, 이것을 통해서 싱글톤이 보장!
- AppConfig@CGLIB는 AppConfig의 자식 타입이므로, AppConfig 타입으로 조회 할 수 있다.
- @Configuration을 빼버리면, 순수 클래스인 AppConfig가 생성되고, 한번씩만 등록이 되던 빈이 여러번 등록이 된다. (싱글톤 보장 깨짐)
- @Configuration을 배버리면, Spring Container에서 관리하는것이 아니게 되버린다.


## 컴포넌트 스캔과 의존관계 자동 주입 시작하기
- 지금까지 스프링 빈을 등록할 때는 자바 코드의 @Bean이나 XML의 <bean> 등을 통해서 설정 정보에 직접 등록할 스프링 빈을 나열했다.
- 예제에서는 몇개가 안되었지만, 이렇게 등록해야 할 스프링 빈이 수십, 수백개가 되면 일일이 등록하기도 귀찮고, 설정 정보도 커지고, 누락하는 문제도 발생했다.
- 그래서 스프링은 설정 정보가 없어도 자동으로 스프링 빈을 등록하는 컴포넌트 스캔이라는 기능을 제공한다.
- 또 의존관계도 자동으로 주입하는 `@Autowired`라는 기능도 제공한다.
- 컴포넌트 스캔을 사용하려면 먼저 `@ComponentScan` 을 설정 정보에 붙여주면 된다.
- 기존의 AppConfig와는 다르게 @Bean으로 등록한 클래스가 하나도 없다.
- 컴포넌트 스캔을 사용하면 `@Configuration` 이 붙은 설정 정보도 자동으로 등록되기 때문에(Configuration도 Component이기 떄문), AppConfig, TestConfig 등 앞서 만들어두었던 설정 정보도 함께 등록되고, 실행되어 버린다. 그래서 `excludeFilters`를 이용해서 설정 정보는 컴포넌트 스캔 대상에서 제외해야 한다. 보통 설정 정보를 컴포넌트 스캔 대상에서 제외하지는 않지만, 기존 예제 코드를 최대한 남기고 유지하기 위해서 여기에서는 이방법으로 처리했다.
- 컴포넌트 스캔을 사용하려면 결국 객체(class)에 `@Component`를 붙여주고, 생성자에 `@Autowired`를 붙여주어야한다. 왜냐하면, Component만으로는 의존관계 주입을 어떤것으로 해야 할지 지정이 안되기 때문에 Autowired로 해주어야 한다.
- 이전에 AppConfig에서는 `@Bean` 으로 직접 설정 정보를 작성했고, 의존관계도 직접 명시했다. 이제는 이런 설정 정보 자체가 없기 때문에, 의존관계 주입도 클래스 안에서 해결 해야 한다.
- `@Autowired`는 의존관계를 자동으로 주입해준다. 자세한 룰은 뒤에 정리
1. @ComponentScan
    - `@ComponentScan`은 `@Component`가 붙은 모든 클래스를 스프링 빈으로 등록한다.
    - 이때 스프링 빈의 기본 이름은 클래스명을 사용하되 맨 앞글자만 소문자를 사용한다.
        - 빈 이름 기본 전략: MemberServsiceImpl클래스 -> memberServiceImpl
        - 빈 이름 직접 지정: 만약 스프링 빈의 이름을 직접 지정하고 싶으면 `@Component("memberService2")` 이런식으로 이름을 부여하면 된다.
2. @Autowired 의존관계 자동 주입
    - 생성자에 `@Autowired`를 지정하면, 스프링 컨테이너가 자동으로 해당 스프링 빈을 찾아서 주입한다.
    - 이때 기본 조회 전략은 타입이 같은 빈을 찾아서 주입한다.
        - `getBean(MemberRepository.class)`와 동일하다고 이해하면 된다.
        - 더 자세한 내용은 뒤에서..
    

