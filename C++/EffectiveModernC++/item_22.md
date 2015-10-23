#pImpl idiom

pImpl(pointer to Implementation) idiom은 프로그램 빌드에 걸리는 시간을 줄여주기 위한 기법중 하나다. 클래스의 데이터 멤버를 구현 객체(implementation class or struct)에 옮긴 후 그걸 가리키는 포인터를 멤버로 들고 있게 만드는 것이다. 예를 들어, Widget 클래스가 다음과 같이 선언되어 있다고 하자.

```C++
class Widget
{
public:
    Widget();

private:
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};
```

Widget은 멤버로 std::string, std::vector, Gadget 타입을 갖고 있기 때문에 ```<string>, <vector>, "Gadget.h"``` 세 개의 파일을 include해 주어야만 한다. std::string, std::vector 두 가지는 표준 라이브러리니 내부 내용이 별로 바뀔 일이 없어 상관없지만, Gadget.h의 경우 사용자 정의 타입이기 때문에 구현이 수시로 바뀔 염려가 있다. 이 Gadget.h의 구현이 바뀔 때마다 Widget.h를 include한 모든 파일들이 다 다시 컴파일되어야하기 때문에 컴파일에 드는 시간이 늘어나는 것이다. 이걸 C++98 스타일로 pImpl idiom을 적용시키면 아래와 같은 구조가 된다.

```C++
class Widget
{
public:
    Widget();
    ~Widget();

private:
    struct Impl;
    Impl* pImpl;
};
```

이렇게 바꾸면 일단 Widget.h에는 std::string, std::vector, Gadget이 없어지므로 컴파일 속도에 향상을 갖고 오며, Gadget이 바뀌었다고 해서 Widget을 include한 파일들까지 모두 다시 컴파일해야하는 일도 없어지게 된다.

 struct Impl; 처럼 선언은 했으나 구현하지 않은 타입을 불완전한 타입(incomplete type)이라고 한다. 이렇게 불완전한 타입의 포인터를 멤버로 갖고 있게 한 다음, 실제 이 타입의 정의는 cpp파일에서 하고 이걸 동적으로 할당 받아 사용하는게 pImpl idiom의 일반적인 구현 형태이다.

```C++
// Widget.cpp 파일
#include "Widget.h"
#include "Gadget.h"
#include <string>
#include <vector>

//Widget의 실제 데이터 멤버들을 여기다가 정의
struct Widget::Impl
{
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

//pImpl 동적으로 할당
Widget::Widget() : pImpl(new Impl)
{
}

Widget::~Widget()
{
    //동적으로 할당한 pImpl 소멸될 때 해제
    delete pImpl;
}
```

이제 .h 파일에 있던 include들이 다 .cpp 파일로 이동했다. 실제 해당 클래스들을 참조해서 사용하는 .cpp파일만 구현이 바뀔 때 다시 컴파일되므로 컴파일 속도가 훨씬 빨라질 것이다. 단 여기서 한 가지 걸리는 부분이 있다. pImpl을 직접 동적으로 할당하고 해제하는데, 이건 너무 옛날 방식이다. C++ 11에서 도입된 스마트 포인터를 적극적으로 이용하여 자원이 자동으로 해제되게 만들어주는게 훨씬 깔끔할 것이다. Widget이 Impl에 대해 배타적 소유권을 갖고 있는 것이므로(자신의 데이터 멤버니까) std::unique_ptr을 쓰면 적절할 것 같다.

```C++
//Widget.h

class Widget
{
public:
    Widget();

private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

//Widget.cpp

#include "Widget.h"
#include "Gadget.h"
#include <string>
#include <vector>

struct Widget::Impl
{
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

//make_unique를 이용해 std::unique_ptr 생성
Widget::Widget()
: pImpl(std::make_unique<Impl>())
{
}

//소멸자는 만들지 않아도 자동으로 자원을 해제해준다.
```

## 소멸자 문제

이렇게 만들고 Widget을 컴파일해보면 컴파일 잘 된다. 그러나 아직 문제가 있다.

```C++
#include "Widget.h"

Widget w; //error 발생!
```

막상 Widget을 쓰려고 하면 컴파일할 때 에러가 발생할 것이다. 일반적으로 불완전한 타입에 대해 delete를 했다는 에러 메시지가 발생할 텐데, 에러가 발생하는 건 자동으로 생성되는 소멸자 때문이다. 이 자동으로 생성되는 소멸자는 파괴되면서 내부 멤버들을 정리하기 위해 std::unique_ptr<Impl>의 소멸자를 호출하게 되고, custom deleter를 쓰지 않았을 경우 이 std::unique_Ptr<Impl>의 소멸자는 기본적으로 delete를 이용해서 자원을 해제하게 된다. 그리고 이 부분이 바로 문제가 되는 지점이다. 암시적으로 생성된 소멸자는 ***inline*** 속성을 갖게 되는데, 생성한 w가 파괴되는 시점에서 inline으로 호출되는 소멸자는 Impl이 어떻게 생겨먹은 놈인지 알 방법이 없기 때문에 pImpl을 파괴하는 과정에서 에러를 내뱉게 되는 것이다. 이를 해결하기 위해서는 소멸자가 Impl 구조체가 어떻게 생겼는지 알게 만들어줄 필요가 있다. 따라서, Widget 소멸자의 구현부는 반드시 Impl 구조체의 정의보다 아래에 위치해있어야만 한다.

```C++
//Widget.h
class Widget
{
public:
    Widget();
    ~Widget(); //소멸자 만들어줌

private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

//Widget.cpp
#include "Widget.h"
#include "Gadget.h"
#include <string>
#include <vector>

struct Widget::Impl
{
    std::string name;
    std::vector<double> data;
    Gadget g1, g2, g3;
};

Widget::Widget()
: pImpl(std::make_unique<Impl>())
{
}

//소멸자 추가! 여기 구현함으로써 
//소멸자가 Impl 구조체의 정의를 알 수 있게 함.
Widget::~Widget()
{
}
```

이렇게 하면 해결된다. 근데 아무 동작도 안하는 소멸자를 만드는 것보다는, default 동작을 수행하는 소멸자라면 ```= default```를 이용하는 편이 더 좋을 것이다.

```C++
//Widget.cpp 파일 구현부에서 아래와 같이 작성.
Widget::~Widget() = default;
```

## 이동 작업

 이걸로 소멸자와 관련된 문제는 해결되었다. 그러나 아직 자동생성되는 special member function들은 더 남아있다. 바로 이동 및 복사와 관련된 생성자, 대입 연산자들이다. 이 녀석들 역시 자동으로 생성되며 소멸자와 거의 비슷한 이유로 컴파일 에러를 일으키게 된다. 우선 이동 작업부터 해결해보자. 이 역시 소멸자와 마찬가지 방식으로 문제를 해결할 수 있다.

```C++
//Widget.h
class Widget
{
public:
    Widget();
    ~Widget();

    //선언만 해둔다.
    Widget(Widget&& rhs);
    Widget& operator=(Widget&& rhs);

private:
   struct Impl;
   std::unique_ptr<Impl> pImpl;
};

//Widget.cpp

...

Widget::~Widget() = default;

//이동 작업 관련 함수들도 마찬가지로 = default 이용해 구현.
Widget::Widget(Widget&& rhs) = default;
Widget& Widget::operator=(Widget&& rhs) = default;
```

## 복사 작업

 pImpl idiom을 이용하든 이용하지 않든 클래스가 하는 작업에는 변함이 없다. 그 말은, 원래 멤버 std::string, std::vector<double>, Gadget 멤버를 갖고 있던 클래스가 복사를 지원한다면 pImpl idiom을 이용한다해도 제대로 복사를 지원해야한다는 것이다. 그런데 문제는, std::unique_ptr은 move-only 타입이기 때문에 컴파일러가 자동으로 복사 생성자 및 복사 대입 연산자를 생성해주지 않는다는 것이다. 혹여 자동으로 만들어준다해도 그건 얕은 복사가 될 것이기 때문에 마찬가지로 문제를 일으킬 것이다. 

```C++
//Widget.h
class Widget
{
public:
    Widget();
    ~Widget();

    Widget(Widget&& rhs);
    Widget& operator=(Widget&& rhs);

    //역시 선언만 해둔다.
    Widget(Widget& rhs);
    Widget& operator=(const Widget& rhs);

private:
   struct Impl;
   std::unique_ptr<Impl> pImpl;
};

//Widget.cpp

...

Widget::~Widget() = default;

Widget::Widget(Widget&& rhs) = default;
Widget& Widget::operator=(Widget&& rhs) = default;

//복사 생성자 깊은 복사 하도록 구현
Widget::Widget(const Widget& rhs)
: pImpl(std::make_unique<Impl>(*rhs.pImpl))
{
}

//복사 대입 연산자도 마찬가지
Widget& Widget::operator=(const Widget& rhs)
{
    *pImpl = *rhs.pImpl;
    return *this;
}
```

Impl 구조체는 자동으로 생성되는 복사 생성자 및 복사 대입 연산자를 갖고 있으므로 이를 활용하면 위와 같이 일일히 복사하지 않고도 깔끔하게 구현할 수 있다.

## std::shared_ptr은 어떨까?

 Widget 클래스가 Impl 구조체에 대한 배타적 소유권을 갖고 있으므로 std::unique_ptr을 쓰는게 자연스럽고, 그래서 std::unique_ptr을 써서 구현을 했지만, 만약 여기 std::shared_ptr을 쓴다면 어떨까? 재미 삼아 한 번 고려해봄직한 주제다.

```C++
class Widget
{
public:
    Widget();

private:
    struct Impl;
    std::shared_Ptr<Impl> pImpl;

//사용

Widget w1;
auto w2(std::move(w1));
w1 = std::move(w2);
};
```

놀랍게도 std::shared_ptr을 이용할 경우 std::unique_ptr을 이용할 때와는 다르게 기본 생성하는 함수들을 그대로 내버려둬도(구현부를 Impl 구조체보다 아래 두는 등의 처리를 하지 않아도) 제대로 컴파일되고 결과도 이상하지 않게 잘 동작한다.
 std::unique_ptr과 std::shared_ptr의 이런 차이가 발생하는 가장 큰 이유는, std::unique_ptr의 타입에는 custom deleter의 타입이 포함되지만 std::shared_ptr의 타입에는 포함되지 않는다는 것이다. std::unique_ptr은 타입에 deleter의 타입이 포함되기 때문에 컴파일러가 자동으로 함수들을 생성할 때 불완전한 타입이 포함되어 있으면 안 된다. 반면 std::shared_ptr은 deleter의 타입이 자신의 타입에 포함되지 않기 때문에 컴파일러가 자동으로 함수를 생성할 때 deleter의 타입이 불완전해도 괜찮다.