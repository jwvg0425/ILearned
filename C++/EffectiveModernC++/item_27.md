# universal reference 오버로딩 회피법

## 오버로딩 회피

 말 그대로 오버로딩 자체를 안 해버리는 것이다. [item 26](item_26.md)을 기준으로 하자면, logAndAdd 함수에서 인자로 universal reference를 받는 경우는 함수 이름을 logAndAddName으로, index를 인자로 받는 경우는 함수 이름을 logAndAddNameIdx로 짓는 것이다. 이렇게 하면 애초에 오버로딩 자체가 안 일어나므로 아무런 문제 없이 잘 해결된다. 단, 이 해결법은 [item 26](item_26.md)에서도 언급된 클래스 생성자 관련 문제에는 사용할 수 없다는 단점이 있다.

## Pass By const T&

또 다른 해결책은 universal reference를 쓰지 않고, C++ 98때처럼 const T&타입을 넘기는 것이다. 이렇게 하면 오버로딩 문제는 해결되지만, 우리가 원하는 만큼 최고의 효율을 뽑아내지는 못한다는 문제가 있다. 하지만 다소 성능을 희생하더라도 코드를 심플하게 유지하는게 더 나은 경우에는 이것도 하나의 좋은 방책이 될 수 있다.

## Pass by value

좀 직관적이지 않을 순 있지만, 복사가 일어날 것임을 알고 있는 상황에서 가끔 어떤 복잡성의 증가도 없이 성능을 얻을 수 있는 방법이 바로 레퍼런스로 넘기는 인자를 값으로 넘기도록 바꾸는 것일 때가 있다. 자세한 내용은 [item 41](item_41.md)에서 다루니, 여기서는 이런 테크닉이 어떤 식으로 이용될 수 있는지 Person 예제를 통해 간단히 살펴보기만 하겠다.

```C++
class Person
{
public:
    explicit Person(std::string n)
    : name(std::move(n)) { }

    explicit Person(int idx)
    : name(nameFromIdx(idx)) { }

private:
    std::string name;
};
```

이렇게 하면 int 관련 타입은 모두 두 번째 생성자를, std::string 관련 타입은 모두 첫번째 생성자를 호출하게 되므로 아무런 이상 없이 동작하게 만들 수 있다.

## Tag Dispatch

 앞에서 언급한 방식들은 모두 perfect forwarding을 지원하지 않는다. perfect forwarding을 쓰고 싶다면 반드시 universal reference를 사용해야만하는데, 이러면 다시 [item 26](item_26.md)에서 다룬 오버로딩 문제에 직면하게 된다. 이를 해결하기 위해서는 함수 호출시 넘어온 인자의 타입이 어떤 함수에 가장 적합한지 따져서 가장 적합한 함수를 호출해줘야만 한다. universal reference는 거의 모든 타입을 받아들일 수 있다. 하지만, 해당 함수에 universal reference가 아닌 인자를 추가한다면 그 인자에 의해 어떤 함수가 호출될 지가 결정되도록 바꿀 수 있다. 이 인자가 일종의 어떤 함수를 호출할지를 결정하는 tag 역할을 하고, 이 tag에 따라 함수를 호출하는 방식이 바로 tag dispatch의 기본 아이디어이다. 아마 이렇게 말로 설명하는 것보다는 예제 코드를 보는게 훨씬 이해하기 편할 것이다.

```C++
std::multiset<std::string> names;

template<typename T>
void logAndAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

[item 26](item_26.md)에서 다뤘던 예제다. 이 logAndAdd 함수에 이제 index를 이용해서 이름을 추가할 수 있는 기능을 추가하려고 하면, 오버로딩 문제가 발생하게 된다. 이를 해결하기 위해 tag dispatch 방식은 내부적으로 실제 구현을 담당하는 logAndAddImpl 함수를 두고, 그 함수를 오버로딩하여 호출하도록 만든다. 이 logAndAddImpl 함수는 universal reference 인자와 별도로 universal reference가 아닌 두 번째 인자를 받음으로써, 그 두번째 인자에 의해 어떤 오버로딩 함수가 호출될 지가 결정되도록 한다. 대충 구현하면 아래와 같은 형태가 될 것이다(정확하지 않다).

```C++
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name),
                  std::is_integral<T>());
}
```

이 함수는 name을 forwarding한 다음, 두 번째 인자로 T가 정수형 타입인지 아닌지를 판별해서 logAndAddImpl 함수를 호출한다. 하지만, 단순히 이렇게 작성할 경우 인자로 들어온 name 값이 lvalue일 때 name은 int& 타입이 될 수 있다. 이 경우 int&는 레퍼런스기 때문에 integral 타입이 아니게 된다. 이런 경우를 방지하기 위해 [item 9](item_09.md)에서 소개된 type traits중 std::remove_reference를 사용해야한다. 

```C++
//C++ 11
template<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name),
                  std::is_integral<typename std::remove_reference<T>::type>());
}

//C++14
typename<typename T>
void logAndAdd(T&& name)
{
    logAndAddImpl(std::forward<T>(name),
                  std::is_integral< std::remove_reference_t<T> >());
}
```
이제 logAndAddImpl이 어떻게 구현되는지 살펴보자.

```C++
//name이 integral 타입이 아닌 경우
template<typename T>
void logAndImpl(T&& name, std::false_type)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}
```

std::is_integral에 의해 넘어온 인자가 false라면 T는 integral 타입이 아니라는 뜻이고, 그렇지 않다면 integral 타입이라는 뜻이다. 그런데, 그냥 true와 false라는 값은 런타입에 결정되는 값이고, 그냥 true false를 받았다간 logAndImpl 함수를 오버로딩할 수가 없다. 오버로딩이 일어나려면 타입이 달라야하는데 true나 false나 런타임에 결정되는 값일 뿐 둘 다 bool 타입인 건 마찬가지기 때문이다. 그래서, std::is_integral은 true나 false라는 값을 돌려주는 함수가 아니다. 사실 std::is_integral은 구조체다. 만약 std::is_integral<T>에서 T 인자가 integral 타입이라면 std::true_type을 상속받은 구조체가 되고, 그렇지 않다면 std::false_type을 상속받은 구조체가 되어(템플릿 특수화를 이용) logAndAddImpl함수의 두 번째 인자의 타입이 T가 integral 타입이냐 아니냐에 따라 바뀌게 만들어버리는 것이다. 그래서 logAndImpl 함수는 두 번째 인자로 std::false_type을 받음으로써 T가 integral 타입이 아닐 때만 호출되도록 만들었다. 그러니 T가 intgeral 타입일 때 호출될 logAndAddImpl 함수는 아래와 같이 작성하면 될 것이다.

```C++
std::string nameFromIdx(int idx);

void logAndAddImpl(int idx, std::true_type)
{
    logAndAdd(nameFromIdx(idx));
}
```

여기서 std::true_type, std::false_type이 바로 ***tag***가 된다. 이 tag는 오버로딩된 함수중 어떤 걸 이용할지 결정할 때만 이용되지 실제 함수 내에서 사용되는 인자가 아니므로, 아예 인자에 이름을 붙이지 않음으로써 컴파일러가 런타임에 좀 더 최적화된 코드를 만들도록 할 수 있다.
 이렇게 tag에 따라 함수 내부에서 호출할 함수를 정하는(dispatch) 방식이 바로 tag dispatch다. 이렇게 하면 우리가 원했던 것처럼 효율적이면서도 오버로딩 문제를 회피하는 코드를 짤 수 있다. tag의 각 타입에 따라 매칭될 수 있는 오버로딩 함수가 하나밖에 존재하지 않게 만듦으로써 각각의 tag가 가장 적합한 함수를 호출할 수 있도록 만드는 것이다.

## universal reference를 받는 템플릿을 제한하기

tag dispatch 기법의 핵심은 클라이언트 API는 오버로딩되지 않은 하나의 함수만 제공하고, 그 내부에서 적절한 구현 함수를 호출해서 동작하게 만드는 것에 있다. 그러나 이 기법이 제대로 적용될 수 없는 상황이 있다. 바로 생성자에 적용하는 경우다. 생성자에 적용할 경우, 컴파일러가 자동으로 생성하는 이동 및 복사 생성자 때문에 하나의 생성자만 만들고 거기에 tag dispatch를 적용한다해도 컴파일러가 생성한 함수가 tag dispatch와 상관없이 호출될 수 있기 때문이다. 사실 가장 큰 문제는, 컴파일러가 만들어낸 생성자가 ***항상*** tag dispatch를 우회해서 동작하는게 아니라, ***가끔*** 우회하는 것이다. [item 26](item_26.md)에서 언급한 것처럼, const Person을 넘길 땐 복사 생성자가, Person을 넘길 땐 perfect forwarding 생성자가 호출돼서 일관적이지 않다. 차라리 const Person을 넘기든 Person을 넘기든 무조건 Person 관련 타입을 넘기면 복사/이동 생성자가 호출되고 그렇지 않을 때만 perfect-forwarding 생성자가 호출되면 아무 문제가 없을 것이다.

 이렇게 universal reference가 모든 타입을 다 받아들이는게 아니라 받아들이는 타입에 약간의 제약을 주고 싶다면, 또 다른 테크닉이 필요하다. 어떤 특정한 조건 하에서만 universal reference 함수 템플릿이 허용되도록 만들려면, std::enable_if를 사용해야한다.

 std::enable_if는 특정 함수 템플릿이 존재하지 않는 것처럼 동작하도록 컴파일러에게 강제할 수 있다. 이런 템플릿들은 사용불가능하게 됐다(to be disabled)고 표현한다. 기본적으로 모든 템플릿들은 사용가능하지만(enable), std::enable_if를 사용하는 템플릿의 경우 std::enable_if에 명시된 조건을 만족할 때만 사용가능하다. 우리가 다루고 있는 예제의 경우 Person의 perfect forwarding 생성자가 받는 타입 T가 Person 타입이 아닐 때에만 사용가능하도록 만들면 될 것이다. 이렇게 하면 해당 템플릿은 무시되므로 컴파일러가 생성한 이동 / 복사 생성자가 제때 호출될 것이기 때문이다. 

 개념 자체는 그리 어렵지 않지만 그걸 사용하는 건 제법 어렵다. 하나씩 차근차근 따라가보자. 일단 Person의 perfect forwarding 생성자가 std::enable_if를 사용해야하는 건 확실하다.

```C++
class Person
{
public:
    template<typename T,
             typename = typename std::enable_if<condition>::type>
    explict Person(T&& n);
};
```

함수의 실제 구현은 달라질게 없으므로 함수의 선언만 살펴보자.std::enable_if의 condition 부분에 우리가 원하는 조건을 넣어야한다. 우리가 원하는 조건은 아까도 언급했듯이, ***T가 Person 타입이 아닐 것**이다. 이걸 간단히 표현하면 ```!std::is_same<Person, T>::value```가 될 것이다. 하지만, 이걸로는 해결이 안 된다.

```C++
Person p("Nancy");

auto cloneOfP(p);
```

위 코드에서 p는 lvalue기 때문에, 타입 추론 과정에서 T는 Person&으로 추론될 것이다. ```std::is_same<Person, Person&>::value```의 값은 false기 때문에(레퍼런스와 값이 같은 타입일리 없으므로) 이것만 가지고는 우리가 원하는 결과를 얻을 수가 없다. 정확히 우리가 무시해야하는 케이스를 따져보면 아래와 같다.

- ***reference인 경우*** 우리의 목적(Person 타입이 들어오면 복사/이동 생성자를 호출하는 것)을 위해서는 Person 타입 뿐만 아니라 Person&, Person&& 타입도 모두 무시해야한다.

- ***CV 지정자가 붙은 경우*** 만약 const Person, 혹은 volatile Person(혹은 const volatile Person)이 인자로 들어왔을 경우에도 마찬가지로 복사 / 이동 생성자가 호출되게 만들어야한다. 따라서 이런 CV 지정자가 붙은 경우도 모두 무시해야한다.

즉 T에 CV 지정자가 붙거나 레퍼런스 타입이거나 한 경우는 모두 그냥 T 타입으로 취급해야한다는 것이다. 이런 용도로 쓸 수 있는 std::decay라는 type_traits가 존재한다. ```std::decay<T>::type```은 T에 붙은 CV 지정자 및 레퍼런스를 모두 제거한 타입을 돌려준다(정확하게는 하는 일이 좀 더 있다. array, function type -> pointer 등의 동작도 수행하지만 여기선 관련없는 내용이니 간략히만 설명. 자세한 내용은 구글신 검색!). 

 즉, 우리가 원하는 조건은 아래와 같이 될 것이다.

```C++
//이게 기본 조건. 여기서 T -> std::decay<T>::type이 되어야
//우리가 의도한 결과가 나온다.
!std::is_same<Person, T>::value

//여기서 std::decay<T>::type은 nested type이므로 typename 키워드를
//붙여줘야 컴파일러가 제대로 타입임을 인식한다.
!std::is_same<Person, typename std::decay<T>::type>::value

//C++ 14
!std::is_same<Person, std::decay_t<T> >::value
```

이제 이 조건을 앞의 Person 생성자에 집어넣으면 될 것이다.

```C++
class Person
{
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_same
                       <Person,
                        typename std::decay<T>::type
                       >::value
                    >::type
     >
     explicit Person(T&& n);
};
```

이렇게 하면 모든 문제가 해결될 것 같지만, 아직 해결되지 않은 한 가지 문제가 있다. 바로 상속받은 자식의 생성자에서 부모 생성자를 호출하는 경우이다([item 26](item_26.md) 참조).

```C++
class SpecialPerson : public Person
{
public:
    SpecialPerson(const SpecialPerson& rhs)
    : Person(rhs)
    { ... }
    SpecialPerson(SpecialPerson&& rhs)
    : Person(std::move(rhs))
    { ... }
};
```
여기서 Person(rhs)에서 T 타입이 SpecialPerson과 관련된 타입으로 추론되어버리기 때문에 앞의 조건에 걸리지 않고 기어코 perfect forwarding 생성자를 호출하게 된다. 이걸 해결하기 위해선 타입 T가 Person 타입인지 아닌지 뿐만 아니라 T가 Person의 자식 타입인지 아닌지까지 따져야한다. 따라서 std::is_same이 아니라 std::is_base_of를 써야한다. ```std::is_base_of<T1, T2>::value```은 T2가 T1의 자식 타입이면 true가 된다. T1과 T2가 동일한 타입일 때도 true이므로 std::is_same->std::is_base_of로 바꿔쓰기만 하면 해결된다.

```C++
class Person
{
public:
    template<
        typename T,
        typename = typename std::enable_if<
                       !std::is_base_of
                       <Person,
                        typename std::decay<T>::type
                       >::value
                    >::type
     >
     explicit Person(T&& n);
};

//C++14
class Person
{
public:
    template<
        typename T,
        typename = std::enable_if_t<
                       !std::is_base_of
                       <Person,
                        std::decay_t<T>
                       >::value
                    >
    >
     explicit Person(T&& n);
};
```

이제 이걸로 끝!이라고 하고 싶지만, 아직 하나가 더 남았다. 제일 처음의 문제, 그러니까 index를 인자로 받아 초기화하는 경우에 대한 오버로딩 문제를 해결해야 한다. 이건 그렇게 어렵지 않다. T가 integral 타입일 때도 해당 템플릿을 사용할 수 없도록 만들어주면 되니까, 단순히 관련 조건을 하나만 더 추가해주면 된다.

```C++
class Person
{
public:
    template
    <
        typename T,
        typename = std::enable_if_t
        <
            !std::is_base_of<Person, std::decay_t<T>>::value &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)
    : name(std::forward<T>(n)) { ... }

    explicit Person(int idx)
    : name(nameFromIdx(idx)) { ... }
};
```

드디어 해결됐다. 이렇게 하면 효율성을 지키면서도 오버로딩이 가능하다. 하지만 이런 TMP를 이용한 프로그래밍은 코드를 굉장히 복잡하게 만든다. TMP 페티쉬같은게 있는 인간이라면 위 코드를 보고 아름답다고 느끼겠지만 대다수 사람들은 이해하기 힘든 코드로 느낄 것이다. 따라서 생성자와 오버로딩, perfect forwarding이 겹친, 위와 같이 정말 이 방법이 아니고서는 해결할 수 없는 상황이 아니라면 다른 방법을 써서 해결하자.

## trade-offs

여기서 처음 소개한 3가지(오버로딩 회피, Pass by const T&, Pass by value)는 함수에서 호출될 인자를 확실히 명시하는 방법이고, 나머지 두 가지(tag dispatch, 템플릿에 제약 걸기)는 perfect forwarding을 써서 호출될 인자를 명시하지 않는 방법이다. 즉 크게 봤을 때 함수의 인자를 명시할 거냐 말거냐의 2가지 선택이 되는 것이다.

 기본적으로 perfect forwarding이 더 효율적인건 사실이지만, 이 역시 문제점을 갖고 있다. 첫번째는 몇몇 인자들의 경우 perfect forward될 수 없다는 것이다. 타입을 명시한 함수에서는 전달이 됨에도 불구하고 말이다. 어떤 경우에 이런 문제가 발생하는지는 [item 30](item_30.md)에 기술되어 있다.

 두 번째 문제는 잘못된 인자를 넘겼을 때 나타나는 에러 메시지를 이해하기 힘들다는 것이다. 예를 들어, 아래와 같은 호출을 했을 때를 생각해보자.

```C++
Person p(u"Konrad Zuse");
```

위와 같이 16비트 문자로 구성된 문자열을 넘겼을 경우, 처음 소개한 3가지 방법의 경우 char16_t[12]를 std::string이나 int로 변환할 수 없다는 에러 메시지가 뜬다. 굉장히 간결해서 이해하기 쉽다. 

반면에 perfect forwarding을 사용할 경우, 타입 추론까지는 잘 되다가, 막상 name 멤버를 초기화할 때 std::string의 적합한 생성자가 존재하지 않는다는 걸 확인하고, 호출자가 넘긴 타입과 필요한 타입간의 불일치를 발견하게 된다. 그 결과로 나타나는 에러 메시지는 굉장히 길고 복잡해서 어떤 에러가 발생했는지 쉽게 알아내기 힘들다(scott meyers의 말로는 무려 160줄의 에러 메시지를 내는 컴파일러도 있었다고 한다). 고작 하나의 universal reference를 쓰는 Person 예제도 이정도인데 이게 3중 4중으로 겹치면 어떤 결과가 나타날 지 생각만 해도 끔찍하다. 

 Person 예제의 경우 static assert를 적절히 활용하여 에러 메시지를 명확하게 나타낼 수 있다. Person에서 perfect forwarding 생성자는 std::string과 관련된 타입만 인자로 들어와야한다. 따라서 이를 std::is_constructible이라는 type traits를 이용해서 에러를 검출할 수 있다. 이 type traits는 컴파일 타임에 어떤 타입의 객체가 다른 어떤 타입의 객체들로부터 생성될 수 있는지를 판별해낸다.

```C++
```C++
class Person
{
public:
    template
    <
        typename T,
        typename = std::enable_if_t
        <
            !std::is_base_of<Person, std::decay_t<T>>::value &&
            !std::is_integral<std::remove_reference_t<T>>::value
        >
    >
    explicit Person(T&& n)
    : name(std::forward<T>(n))
    {
        //T 타입 객체로부터 std::string이 생성될 수 있어야만 함
        static_assert(
            std::is_constructible<std::string, T>::value,
            "인자 n은 std::string을 생성하는데 사용될 수 없습니다.");
    }

    explicit Person(int idx)
    : name(nameFromIdx(idx)) { ... }
};
```

이렇게 하면 복잡한 에러 메시지들 대신에 깔끔하고 명확한 에러 메시지를 보여줄 수 있다. 하지만, 위 예제의 경우 생성자의 멤버 초기화 리스트 다음에 static_assert가 나오기 때문에, 복잡한 에러 메시지가 나오고 나서 맨 마지막에서야 static_assert로 인한 에러 메시지가 나온다. 아쉽지만 뭐 없는 것보다는 낫지 않겠는가.