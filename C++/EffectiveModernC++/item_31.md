# Lambda Expression

람다는 함수 객체를 만드는데 굉장히 유용한 방법이다. 람다에 관련된 용어들은 서로 헷갈리는 경우가 많으니 간단히 한 번 짚고 넘어가자.

 - *람다 표현식(lambda expression)*은 단순한 표현식이다. 이는 소스코드의 일부이다. ```std::find_if(container.begin(), container.end(), [](int val){ return 0 < val && val < 10; });``` 이 코드에서 ```[](...){ ~ }``` 부분이 람다 표현식이다. 
 - *클로져(closure)*는 람다에 의해 런타임에 생성되는 객체이다. 캡쳐 모드에 따라 클로져는 캡쳐된 데이터의 복사본 또는 레퍼런스를 갖고 있게 된다. 위 코드의 std::find_if의 호출에서 런타임에 세 번째 인자(람다 표현식으로 정의된)로 넘어가는 객체가 바로 클로져다.
 - *클로져 클래스(closure class)*는 클로져가 인스턴스화된 클래스다. 각각의 람다는 컴파일러가 고유한 클로져 클래스를 만들게 한다. 람다 표현식 내부의 문장(statement)들은 그 람다 표현식으로 인해 생성되는 클로져 클래스의 멤버 함수 속의 실행 가능한 명령문(instruction)이 된다.

람다로부터 생성된 클로져는 복사가 가능하다. 따라서 하나의 클로져 타입에 대해 여러 개의 클로져가 존재할 수도 있다.

```C++
int x;

auto c1 = [x](int y){ return x * y > 55; };

//c1 람다에 대한 복사본
auto c2 = c1;
auto c3 = c2;
```

# default capture mode를 피하라

C++11의 람다에는 by-reference와 by-value라는 두 가지의 기본 캡쳐 모드가 있다. 그런데 이 두 가지의 기본 캡쳐 모두 잠재적인 문제점을 갖고 있다. 하나씩 살펴보자.

## by-reference

by-reference 기본 캡쳐 모드는 댕글링의 위험이 있다. 람다가 정의된 영역에서 사용가능한 모든 지역 변수, 매개 변수들에 대한 레퍼런스를 포함하는 클로져를 만들기 때문이다. 만약 람다에 의해 생성된 클로져의 생명주기가 지역변수나 매개변수보다 더 길다면, 이 변수들에 대한 레퍼런스가 댕글링을 일으키게 된다. 예를 들어, 필터링 함수들에 대한 컨테이너를 생각해보자. 이 컨테이너는 int를 받아서 bool을 리턴하는(해당 인자가 조건을 만족하는지 검사하는) 함수들로 구성되어 있다.

```C++
using FilterContainer = std::vector<std::function<bool(int)>>;

FilterContainer filters;

//이런 식으로 필터 함수 추가 가능
filters.emplace_back([](int value){ return value % 5 == 0; });
```

위 필터 함수는 5로 나눈 나머지가 0인 애들만 걸러낸다. 근데 실제로는 5같은 상수를 쓰기보단 특정 변수 값으로 체크하는 경우가 많을 것이다. 아래 예제를 보자.

```C++
void addDivisorFilter()
{
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();

    auto divisor = computeDivisor(calc1, calc2);

    filters.emplace_back
    (
        [&](int value) { return value % divisor == 0; }
    );
}
```

어떤 과정을 통해 divisor 값을 계산한 다음 이걸 by-reference 기본 캡쳐 모드로 받아서 썼다. 이렇게 되면 [&]에 의해 지역 변수 divisor의 레퍼런스가 클로져에 저장되고 그걸 실행 코드에서 사용하게 되는데, 이로 인해 댕글링 문제가 발생할 수 있다. filters의 생명주기가 divisor보다 길기 때문이다. 실제 저 필터 함수가 호출되는 시점에서는 divisor 변수는 이미 해제되고 없는 상황일 것이므로 정의되지 않은 동작을 일으킬 것이다. 이 문제는 물론 명시적인 by-reference 캡쳐를 해도 똑갈이 발생하는 문제다.

```C++
filters.emplace_back
(
    [&divisor](int value) { return value % divisor == 0; }
);
```

하지만 명시적으로 캡쳐를 할 경우 이 람다가 divisor의 생명주기와 연관이 있다는 걸 확인하기가 훨씬 쉽다.

 만약 by-reference 기본 캡쳐 모드로 만들어진 람다와 그 람다가 캡쳐하는 지역 변수들의 생명주기가 같거나 람다가 더 짧다는 걸 알고 있는 상황이라면 기본 캡쳐 모드를 써도 아무 상관 없는 것 아니냐? 라고 할 수도 있다. 물론 안전하다. 생명 주기 때문에 댕글링 문제가 발생하는 거니까 모든 값들의 생명 주기를 완벽히 알고 있다면 아무런 문제가 생기진 않는다. 그러나 프로그램 코드라는  건 항상 그 상태로 고정되어있는 것이 아니고 나 말고 다른 사람이 코드를 건드릴 수도 있으며, 내가 나중에 그런 세세한 사항들을 모두 기억하지 못한 채로 어 이 람다 유용한데? 하고 복사 붙여넣기해서 쓸 수도 있다. 그러니까 애초에 코드를 위험하지 않도록 짜는게 중요하다.

## by-value

 이런 댕글링 문제를 해결할 수 있는 방법 중 하나는 by-value 기본 캡쳐 모드를 사용하는 것이다. 이 경우 코드는 아래와 같다.

```C++
filters.emplace_back
([=](int value){ return value % divisor == 0; });
```

이렇게 하면 이 예제에서는 문제 없이 잘 동작하지만, by-value 기본 캡쳐 모드를 사용한다 하더라도 댕글링 문제에서 벗어날 수는 없다. 가장 대표적인 경우가 바로 포인터 변수를 by-value로 캡쳐하는 경우인데, 이 경우 포인터 변수는 주소 값이므로 by-value로 캡쳐한다해도 결국 해당 포인터를 참조할 때 댕글링 문제가 발생할 수 있다. 하지만 모던 C++에서는 구시대의 유물인 raw pointer를 쓰지 않는다! smart pointer를 쓴다면 아무 문제 없는 거 아니냐! 라고 할 수도 있다. 그러나 이 역시 틀린 말이다. 어쩔 수 없이 쓸 수 밖에 없는 raw pointer가 존재하기 때문이다. 예를 통해 알아보자.

```C++
class Widget
{
public:
    void addFilter() const;

private:
    int divisor;
};
```

이 클래스는 addFilter에서 자신의 멤버 divisor를 통해 filter 함수를 추가한다.

```C++
void Widget::addFilter() const
{
    filters.emplace_back
    ( [=](int value){ return value % divisor == 0; });
}
```

이렇게 하면 내부적으로 divisor를 쓰긴 하지만 이건 by-value로 캡쳐했으니 아무런 문제를 일으키지 않을 것 같다. 그러나 그 생각은 절대적으로 **틀린 생각**이다. 람다가 캡쳐하는 것은 해당 람다가 생성된 범위에서 볼 수 있는 static이 아닌 **지역 변수**이다(매개 변수 포함). 그러나, Widget::addFilter에서 divisor는 지역 변수가 아니라 Widget 클래스의 데이터 멤버이다. 즉, 이건 캡쳐되지 않는다는 것이다. 만약 기본 캡쳐 모드를 없앤다면 이 코드는 컴파일되지 않을 것이다.

```C++
void Widget::addFilter() const
{
    filters.emplace_back
    ( [](int value){ return value % divisor == 0; }); //compile error!
}
```

심지어, divisor를 명시적으로 캡쳐한다해도(값이든 레퍼런스든) 컴파일되지 않는다. divisor는 지역 변수도 매개변수도 아니기 때문이다.

```C++
void Widget::addFilter() const
{
    filters.emplace_back
    ( [divisor](int value){ return value % divisor == 0; });
}
```

기본 캡쳐모드가 divisor를 캡쳐하지 않는데 어떻게 람다 내부에서 divisor를 쓸 수 있는걸까? 그것은 바로 **this**덕분이다. 클래스의 멤버 변수는 호출될 때 자동으로 this를 함수의 첫번째 인자로 받는데(thiscall) 이 this가 캡쳐되어 this의 멤버 변수인 divisor를 쓸 수 있게 되는 것이다. by-value 기본 캡쳐 모드를 사용했을 때 컴파일러는 내부적으로 코드를 아래와 같이 취급한다.

```C++
void Widget::addFilter() const
{
    auto currentObjectPtr = this;

    filters.emplace_back
    ([currentObjectPtr](int value)
    { 
        return value % currentObjectPtr->divisor == 0; 
    });
}
```

즉, 저 필터 함수가 사용될 때 캡쳐한 this의 생명주기가 더 짧다면 이 역시 댕글링 문제를 일으킬 수 있는 것이다. this라는 raw pointer를 쓰기 때문에 스마트 포인터를 쓴다고 해도 댕글링 문제가 발생하게 된다.

```C++
using FilterContainer = std::vector<std::function<bool(int)>>;

FilterContainer filters;

void doSomeWork()
{
    auto pw = std::make_unique<Widget>();

    pw->addFilter();
}
```

위와 같은 코드가 있을 때, pw를 스마트 포인터로 만들었지만 addFilter에서는 pw의 this를 캡쳐한다. 그리고 doSomeWork 함수가 끝나면 pw는 파괴될 것이고, 그 이후 부분에서 이 때 추가된 필터 함수를 쓴다면 역시 댕글링 문제가 발생해버리는 것이다. 이런 문제는 캡쳐하고 싶은 멤버 변수를 지역 변수에 복사한 후 이를 캡쳐함으로써 해결할 수 있다.

```C++
void Widget::addFilter() const
{
    auto divisorCopy = divisor;

    filters.emplace_back(
        [divisorCopy](int value){ return value % divisorCopy == 0; }
    );
}
```

C++14에서는 더 좋은 방법인 일반화된 람다 캡쳐(generalized lambda capture - [item 32](item_32.md) 참조)를 지원한다. 

```C++
void Widget::addFilter() const
{
    filters.emplace_back(
        [divisor = divisor](int  value)
        { return value % divisor == 0; }
    );
}
```

by-value 기본 캡쳐 모드의 또다른 문제점은 해당 클로져가 독립적이고 외부 데이터의 변화로부터 어떤 영향도 받지 않을 것처럼 보이게 만든다는 것이다. 하지만, 이건 사실이 아니다. 람다는 지역 변수와 매개변수 뿐만 아니라 정적 객체(static object)들로부터도 영향을 받는다. 아래 코드를 보자.

```C++
void addDivisorFilter()
{
    static auto calc1 = computeSomeValue1();
    static auto calc2 = computeSomeValue2();

    static auto divisor = 
        computeDivisor(calc1, calc2);

    filters.emplace_back(
        [=](int value)
        { return value % divisor == 0; }
    );

    ++divisor;
}
```

위 코드에서 람다 내부에 쓰인 divisor는 캡쳐된게 **아니다**. divisor는 정적 변수기 때문에 캡쳐되지 않고, 해당 외부의 값을 그냥 가리킬 뿐이다. 즉, divisor 값이 변하면 람다 내부의 결과도 변한다. 외부 데이터의 변화에 완전히 독립적이라고 할 수 없는 것이다. 하지만 ```[=]```을 보면 왠지 모르게 해당 람다가 외부 데이터로부터 독립적이며 내부에서 쓰는 값들은 전부 복사본인 것처럼 느껴진다. 코드를 오해하기 쉽게 만드는 것이다.

결론 : **기본 캡쳐 모드 쓰지 마라!**