# Universal Reference는 오버로딩을 피해라

이름을 인자로 받아서, 로그를 남기고 해당 이름을 어떤 전역 자료구조에 저장하는 함수를 생각해보자.

```C++
std::multiset<std::string> names;

void logAndAdd(const std::string& name)
{
    auto now = 
        std::chrono::system_clock::now();

    log(now, "logAndAdd");

    names.emplace(name);
}
```

위 코드는 동작은 정상적으로 하지만, 별로 효율적인 코드는 아니다. 이 함수를 호출할 때 있을 수 있는 3가지 경우를 생각해보자.

```C++
std::string petName("Darla");

//case 1 : lvalue std::string 넘기는 경우
logAndAdd(petName); 

//case 2 : rvalue std::string 넘기는 경우
logAndAdd(std::string("Persephone"));

//case 3 : 리터럴 문자열 넘기는 경우
logAndAdd("Patty Dog");
```

case1의 경우는 아무런 문제가 없다. 인자 자체가 lvalue이므로 내부적으로도 반드시 copy가 일어나야만 한다. case2의 경우에는 인자로 넘어온 값이 rvalue인데, 함수 내부에서는 const std::string&으로 받고 있으므로 move를 할 수 있음에도 불구하고 copy가 일어나므로 비효율적이다. case 3의 경우는 더하다. 인자로 넘어가는 값이 리터럴 문자열임에도 불구하고 임시 std::string이 생성되어야만 하며 심지어 그걸 copy까지 한다. 만약 리터럴 문자열이 직접적으로 emplace 함수의 인자로 넘어갔다면 copy는 커녕 move조차도 일어나지 않았을 것이다. 이런 비효율을 없애기 위해 logAndAdd를 universal reference를 이용해 다시 작성해보자.

```C++
template<typename T>
void logAnddAdd(T&& name)
{
    auto now = std::chrono::system_clock::now();
    log(now, "logAndAdd");
    names.emplace(std::forward<T>(name));
}

std::string petName("Darla");

//case 1 : copy가 일어난다
logAnddAdd(petName);

//case 2 : rvalue이므로 move가 일어난다
logAndAdd(std::string("Persephone"));

//case 3 : multiset 내부에서 std::string 생성.
logAndAdd("Patty Dog");
```

이렇게 하면 가장 최적화된 코드가 만들어지게 된다. 여기서 끝나면 가장 이상적인 상황이지만, 현실은 그렇게 녹록하지 않다. 여기서 만약 이름이 아니라 인덱스를 통해 이름을 받아와서 그걸 전역 자료구조에 추가해야하는 경우가 생겼다고 하자. 가장 간단한 해결책은 아마 함수 오버로딩일 것이다.

```C++
//인덱스에 따라 이름을 돌려주는 함수
std::string nameFromIdx(int idx);

void logAndAdd(int idx)
{
    auto now = std::chrono::system_click::now();
    log(now, "logAndAdd");
    names.emplace(nameFromIdx(idx));
}

//호출시

std::string petName("Darla");

//앞과 마찬가지로 템플릿 함수 호출
logAndAdd(petName);
logAndAdd(std::string("Persephone"));
logAndAdd("Patty Dog");

//int형에 대한 오버로딩 함수 호출
logAndAdd(22);
```

하지만, 함수 오버로딩에서 어떤 함수가 호출될 지 선택되는 과정은 그렇게 형편좋게 예상대로만 진행되지 않는다. index로 int 타입 대신에 short 타입이 넘어갔을 경우를 생각해보자.

```C++
short nameIdx;

//error 발생!!
logAndAdd(nameIdx);
```

이 경우 universal refernce(T&&)를 인자로 받는 함수와 int를 인자로 받는 함수가 오버로딩 되어있다. short 타입 역시 T&& 타입을 받는 함수로 호출 가능하며, int를 인자로 받는 함수 역시 short과 호환이 된다. 그러나 이 경우, short-> int는 암시적 변환을 거쳐 호출이 되어야하지만 T&&를 인자로 받는 함수는 암시적 변환이 필요없이 바로 호출이 가능하다. 그래서 T&&를 인자로 받는 함수가 우선적으로 호출되고, emplace 함수에서 short을 인자로 하는 std::string의 생성자는 없으므로 에러가 발생하는 것이다. 이런 문제 때문에 universal reference에 오버로딩을 적용하는 건 별로 좋은 생각이 아니다.

## 생성자 문제

 하지만, 한 가지 큰 문제가 있다. 이런 universal reference를 인자로 받는 생성자를 만들고 싶은 경우, 절대 오버로딩을 피할 수 없는 것이다. 생성자의 이름은 클래스의 이름과 똑같아야만하는 걸로 언어 차원에서 정해져 있으니 이는 피할 수가 없다.

```C++
class Person
{
public:
    template<typename T>
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}

    explicit Person(int idx)
    : name(nameFromIdx(idx)) { }

private:
    std::string name;
};
``` 

이름을 멤버로 갖고 있는 Person 클래스를 위와 같이 작성하면, 아까 전까지 본 logAndAdd와 동일한 문제가 발생한다. idx가 int가 아니라 short, std::size_t, long, 기타 등등 int 외의 정수 타입이면 무조건 perfect-forwarding 생성자가 호출되어버리는 것이다. 여기에 문제는 또 있다. [item 17](item_17.md)에 따르면 템플릿 멤버 함수는 어떤 special member function의 자동 생성도 막지 않으므로, 위의 경우 컴파일러가 자동으로 복사 및 이동 생성자를 만들어낸다.

```C++
class Person
{
public:
    template<typename T>
    explicit Person(T&& n)
    : name(std::forward<T>(n)) {}

    explicit Person(int idx)
    : name(nameFromIdx(idx)) { }

    //보이진 않지만 아래와 같은 함수들이 컴파일러에 의해 생성되어 있다
    Person(const Person& rhs);
    Person(Person&& rhs);

private:
    std::string name;
};
```

여기서 굉장히 이해하기 힘든 컴파일 오류가 발생해버린다.

```C++
Person p("nancy");

auto cloneOfP(p); //p로부터 Person 생성. 하지만 컴파일이 안된다!
```

정말 이상해보이는 현상이지만, 근본 원리는 아까전의 short-int 문제와 같다. cloneOfP의 생성자 호출에서 인자로 넘어간 p는 Person 타입이다. perfect-forwarding 생성자는 여기서 T&&를 Person&으로 추론해내지만, 컴파일러가 생성한 복사 생성자는 인자로 const Person&를 받는다. perfect-forwarding 생성자는 Person&를 인자로 받으므로 아무런 변환없이 인자를 넘길 수 있지만, 컴파일러가 생성한 복사 생성자는 const Person&을 인자로 받으므로 암시적인 변환이 필요하다. 따라서 perfect-forwarding 생성자가 호출되어버리고, 멤버 std::string을 Person&로 초기화하려다가 컴파일 에러가 발생하는 것이다. 물론 이 문제는 아래와 같이 작성할 경우 해결 가능하다.

```C++
const Person p("nancy");

auto cloneOfP(p); //이건 가능!
```

const Person을 인자로 넘길 경우 perfect-forwarding 생성자와 복사 생성자 모두 const Person&를 인자로 받게 되는데, 함수의 서명이 동일한 경우 템플릿 함수보다 템플릿이 아닌 일반 함수의 호출에 우선권이 있다. 따라서 컴파일러가 생성한 복사 생성자가 호출되는 것이다(물론 무조건 P를 const Person으로 생성해야한다는 점에서 완벽한 해결책이라고 보긴 힘들 듯).

이와 유사한 또다른 문제가 상속 관계에서도 발생한다.

```C++
class SpecialPerson : public Person
{
public:
    SpecialPerson(const SpecialPerson& rhs)
    : Person(rhs) { ... }

    SpecialPerson(SpecialPerson&& rhs)
    : Person(std::move(rhs)) { ... }
};
```

위와 같이 Person을 상속받은 SpecialPerson이라는 클래스가 있다고 하자. SpecialPerson 클래스의 이동 및 복사 생성자에서 자기 부모의 이동/ 복사 생성자를 호출해 부모 부분을 초기화 하려고 하는 경우 문제가 발생한다. 아까전과 마찬가지 이유로 인해 Person(rhs) 또는 Person(std::move(rhs))부분에서 T&&이 const SpecialPerson& 또는 SpecialPerson&&으로 추론되어 Person의 perfect-forwarding 생성자가 호출되어버리는 것이다. 이것참 굉장히 골치 아프다. 

이런 여러가지 이유들 때문에 Universal reference를 인자로 받는 함수에 오버로딩을 적용시키는 건 정말 좋은 방법이 아니다. 하지만 문제는 그런 방식의 동작이 필요할 때가 존재한다는 것이다. 이럴 때 위에서 언급한 문제들을 회피하면서 그런 상황에 적합한 동작을 구현하는 방법은 [item 27](item_27.md)에 기술되어 있다.