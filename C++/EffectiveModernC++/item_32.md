# move에는 init capture

가끔 by-value 캡쳐도, by-reference 캡쳐도 마음에 안 들 때가 있다. move-only 객체(std::unique_ptr이나 std::future 등)를 클로져 내부로 이동시키고 싶은 경우가 바로 그런 경우인데, 문제는 C++11은 이런 경우에 대한 방법을 지원하지 않는다. 클로져 내부로 복사는 비싸지만 이동은 싼(vector 같은 컨테이너) 객체를 이동시키고 싶은 경우에도 마찬가지로 C++11에서는 언어적 차원의 기능 지원이 없다. 하지만 C++14는 다르다. C++14에서는 이런 경우에 클로져 내부로 객체를 이동시키는 방법을 제공한다.

## init capture

C++14에 추가된 init capture 기능을 이용하면 외부의 객체를 클로져 내부로 이동시킬 수 있다. init capture를 사용하면 C++11 의 캡쳐가 할 수 있는 거의 모든 것들 이상의 기능을 구현할 수 있다. init capture는

1. 람다로부터 생성되는 클로져 클래스 내부의 데이터 멤버 이름
2. 해당 데이터 멤버를 초기화하기 위한 표현식(expression)

을 기술함으로써 사용할 수 있다. 아래는 std::unique_ptr을 클로져 내부로 이동시키기 위해 init capture를 사용하는 예제다.

```C++
class Widget
{
public:
    bool isValidated() const;
    bool isProcessed() const;
    bool isArchived() const;

private:
};

auto pw = std::make_unique<Widget>();

//closure 내부의 멤버 pw를 std::move(pw)로 초기화
auto func = [pw = std::move(pw)]
            { return pw->isValidated()
                     && pw->isArchived(); };

```

init capture에서 = 왼쪽은 클로져 클래스 내부의 멤버 이름이고, 오른쪽은 해당 멤버를 초기화하기 위한 표현식이다. 이 때 =의 왼쪽과 오른쪽의 scope가 다르다는 것이다. = 왼쪽의 scope는 해당 람다 표현식으로 만들어지는 클로져 클래스의 내부이고, =의 오른쪽 편은 람다가 정의된 영역과 같은 scope를 가진다. 이런 특성을 이용해서 위와 같이 표현식으로부터 클로져 클래스 내부 멤버의 초기화를 할 수 있는 것이다. 이렇게 표현식의 결과로도 내부 멤버를 초기화할 수 있게 해주는 특성 때문에 init capture는 일반화된 람다 캡쳐(generalized lambda capture)라고도 불린다. 

## In C++ 11

하지만 이건 어디까지나 C++14에서의 이야기다. C++11에서는 언어적으로 이런 기능을 지원해주지 않는데, 그런 상황에서 위와 같은 기능을 구현하려면 어떻게 해야할까? 가장 간단한 방법은 직접 함수 객체를 구현하는 것이다.

```C++
class IsValAndArch
{
public:
    using DataType = sstd::unique_ptr<Widget>;

    explicit IsValAndArch(DataType&& ptr)
    : pw(std::move(ptr)){}

    bool operator()() const
    { return pw->isValidated() && pw->isArchived(); }

private:
    DataType pw;
};

auto func = IsValAndArch(std::make_unique<Widget>());
```

어차피 람다 표현식의 목적 자체가 함수 객체를 쉽게 만들기 위한 것이므로 정 방법이 없다면 이렇게 해결하는 것도 나쁘지 않다. 타이핑을 좀 많이해야하긴 하지만 어쨌든 원래의 목적은 달성했으니까. 하지만 꼭 람다를 이용하고 싶고 저렇게 긴 코드를 쓰는게 싫다면 조금 다른 방법도 있다. 람다 표현식과 std::bind를 적절히 응용하면 C++11에서도 C++14 스타일의 캡쳐를 흉내낼 수 있다. 그 방법은 다음과 같다.

1. 캡쳐할 객체를 std::bind로 생성된 함수 객체로 이동시키고
2. 람다에 **캡쳐된** 객체에 대한 레퍼런스를 주기

std::bind에 익숙하다면 코드가 굉장히 깔끔해 보이겠지만 익숙하지 않다면 조금 헷갈릴 수도 있다. 일단 std::vector 컨테이너를 클로져 내부로 이동시키는 예제부터 시작해보자.

```C++
std::vector<double> data;

auto func = [data = std::move(data)]{};
```
C++14의 경우 위와 같이 init capture를 이용해 간단히 해결할 수 있다. C++11에서는 이를 아래와 같은 방식으로 흉내낸다.

```C++
std::vector<double> data;

auto func =
    std::bind(
        [](const std::vector<double>& data)
        { }, std::move(data));
```

람다 표현식과 비슷하게 std::bind 역시 함수 객체를 리턴한다. std::bind가 리턴하는 함수 객체를 bind object라고 하자. std::bind의 첫번째 인자는 호출 가능한 오브젝트이다. 그 뒤에 오는 인자들은 해당 오브젝트로 pass될 값들을 나타낸다.

 bind object는 std::bind로 넘어간 모든 인자들의 복사본을 포함하고 있다. 각각의 lvalue 인자들에 대해, bind object는 그걸 복사해서 들고 있게 되고, rvalue 오브젝트에 대해서는 그걸 이동시켜 들고 있게 된다. 위 예제에서 두 번째 인자는 rvalue이므로 bind object는 data 값을 이동시켜 들고 있게 된다. 이게 C++11에서 클로져 내부로 멤버를 이동시키기 위한 핵심적인 아이디어다. bind 객체는 호출될 때 처음 std::bind에서 인자로 넘겼던 이들을 그대로 인자로 해서 함수를 호출한다. 위 코드를 예제로 들자면 func이 호출될 때 첫번째 인자로 넘긴 람다에 두번째 인자로 넘긴 값(std::move(data))이 이동 생성되어 람다의 인자로 넘어간다는 것이다. 그래서 람다 객체가 인자로 move capture하고 싶은 값의 const reference를 취해야만 한다.

 람다 표현식으로 인해 클로져 클래스 내부에 생성되는 operator() 함수는 기본적으로 const이다. 이 특성 때문에 클로져 내부의 모든 데이터 멤버들은 람다의 body에서 const로 취급된다. 하지만 bind object안에 이동 생성된 data의 복사본은 const가 아니다. 이 때문에 라다 내부에서 data가 수정되는걸 막기 위해 인자를 const로 설정해야했다. 하지만 람다가 mutable로 선언된다면, 클로져 클래스 내부의 operator()는 const로 선언되지 않는다. 따라서 이 땐 람다의 매개변수 선언에 const를 빼도 된다.

```C++
auto func =
    std::bind(
        [](std::vector<double>& data) mutable
        { }, std::move(data));
```

std::bind는 자신의 인자로 넘어온 모든 인자들을 bind object에 저장하기 때문에, 첫번째 인자로 넘어온 클로져도 내부적으로 저장하게 된다. 따라서 클로져의 생명 주기는 bind object와 같아진다. 이 말인즉슨 클로져가 존재하는한 move capture를 흉내낸 방식으로 저장한 해당 객체 역시 사라지지 않고 존재한다는 뜻이기 때문이다(즉 댕글링이 절대 발생하지 않는다). 

 요약하자면 다음과 같다.

 - C++11에서 클로져 내부로 객체를 이동시키는 건 불가능하다. 대신에 bind object를 이용하면 흉내는 낼 수 있다.

 - move capture를 C++11에서 흉내내려면 이동시키려는 객체를 bind object 내부로 이동시키고, 람다가 해당 객체에 대한 람다를 인자로 받도록 만들자.

 - bind object의 생명주기가 클로져와 같기 때문에, 해당 bind object안에 있는 객체들을 클로져안에 있는 객체인 것마냥 대해도 상관없다.

std::bind를 이용해 move capture를 흉내내는 두번째 예제는 다음과 같다. C++14 버젼과 비교해보자.

```C++
//C++14
auto func = [pw = std::make_unique<Widget>()]
            { return pw->isValidated()
                     && pw->isArchived(); };

//C++11
auto func = std::bind(
              [](const std::unique_ptr<Widget>& pw)
              {return pw->isValidated()
                      && pw->isArchived(); },
              std::make_unique<Widget>());
```

