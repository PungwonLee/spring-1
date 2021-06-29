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
   
