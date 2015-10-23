# 공유 자원 관리에는 std::shared_ptr

요즘 나오는 언어들은 거의 대부분 garbage collection 기능을 갖고 있다. 반면 C++은 언어 차원에서 garbage collection을 지원해주지 않기 때문에 사용한 자원을 일일히 해제해줘야하는 어려움이 있었는데, 이런 어려움을 해결하기 위해 등장한 것이 shared_ptr이다. shared_ptr은 garbage collection처럼 여럿이 공유하는 자원을 더 이상 필요없을 때 자동으로 해제해줄 뿐만 아니라, 어느 시점에 해제되는지도 알 수 있다(garbage collection은 자원이 정확히 어느 시점에 해제되는지 알기 힘들다. 자동으로 일정 규칙에 따라 관리하다가 해제할 때가 되면 해제하기 때문에).

shared_ptr의 동작 원리는 간단하다. 각 자원에 대해 해당 자원을 참조하고 있는 개수를 계산하고(reference counting) 그 값이 0이 되면 누구도 해당 자원을 가리키고 있지 않다는 의미이므로 이 때 해당 자원을 파괴한다. 그래서 일반적인 경우 std::shared_ptr을 생성할 때 레퍼런스 카운트를 증가시키고, 파괴될 때 레퍼런스 카운트를 감소시키게 된다. ```sp1 = sp2;``` 와 같은 대입 구문에서는 sp1이 가리키고 있던 오브젝트의 레퍼런스 카운트를 감소시키고, sp2가 가리키고 있던 오브젝트의 레퍼런스 카운트를 증가시키게 된다. 이 과정에서 레퍼런스 카운트가 0이 되면 해당 자원을 파괴하는 것이다. 이 때 '이동'을 통한 생성의 경우에는 레퍼런스 카운트에 변화가 일어나지 않는다. 단순히 자원을 가리키는 개체가 바뀔 뿐이지 가리키는 개수가 변하진 않기 때문이다.

## 특징

 - **std::shared_ptr은 일반 포인터의 2배 크기를 갖게 된다.** 왜냐하면 내부적으로 자원에 대한 포인터와 해당 포인터의 레퍼런스 카운트 두 개를 관리해야만 하기 때문이다.

 - **레퍼런스 카운트를 관리하기 위한 메모리는 동적으로 할당된다.** 레퍼런스 카운트와 std::shared_ptr이 가리키는 오브젝트는 서로 연관되어있는 대상이지만, 해당 오브젝트가 레퍼런스 카운트를 기억하고 있을 수는 없다. 모든 오브젝트를 설계할 때 레퍼런스 카운트를 세도록 만들수도 없는 일이니 말이다. [item 21](item_21.md)에서 std::make_shared를 이용해서 std::shared_ptr을 만들 때 동적 메모리 할당을 하는데 드는 비용을 피하는 방법을 소개하고 있으나, std::make_shared를 쓸 수 없는 상황도 존재한다. 어쨌든, 레퍼런스 카운트는 동적으로 할당된 데이터로 관리된다.

 - **레퍼런스를 증가시키고 감소시키는 동작은 반드시 원자적(atomic)이어야 한다.** 여러 개의 스레드에서 동시에 읽고 쓰는 동작이 발생할 수 있기 때문이다. 어느 한 쪽에서 std::shared_ptr이 파괴되면서 레퍼런스 카운트를 1깎는 동시에 다른 쪽에서 1을 증가시키거나 하는 식으로 연산이 꼬이면 문제가 발생할 수 있기 때문에 이 동작은 원자적으로 수행되어야한다.


## control block

std::unique_ptr처럼 std::shared_ptr도 기본적으로 자원 파괴시 delete를 이용한다. 하지만 역시 std::unique_ptr과 마찬가지로 custom deleter도 지원한다. 이 때 한 가지 다른 점은, std::unique_ptr의 경우 custom deleter의 타입이 자신의 타입에 포함이 되지만, std::shared_ptr의 경우 custom deleter의 타입이 포인터 자체의 타입에 영향을 끼치지 않는다는 것이다.

```C++
//custom deleter.
auto loggingDel = [](Widget* pw)
{
    makeLogEntry(pw);
    delete pw;
};

//unique_ptr은 deleter의 타입이 포인터의 타입에 영향을 미친다
std::unique_ptr<Widget, decltype(loggingDel)>
    upw(new Widget, loggingDel);

//shared_ptr은 그렇지 않다
std::shared_ptr<Widget> spw(new Widget, loggingDel);
```

std::shared_ptr의 이런 특징 때문에 서로 custom deleter가 다른 std::shared_ptr들을 하나의 컨테이너에 다같이 담아 보관할 수 있다.

```C++
//custom deleters.
auto customDeleter1 = [](Widget* pw) { ... };
auto customDeleter2 = [](Widget* pw) { ... };

std::shared_ptr<Widget> pw1(new Widget, customDeleter1);
std::shared_ptr<Widget> pw2(new Widget, customDeleter2);

//다 같이 담을 수 있음.
std::vector< std::shared_ptr<Widget> > vpw{ pw1, pw2 };
```

custom deleter가 달라도 타입이 같기 때문에 서로 대입하는 거나 함수에 인자로 넘기거나 하는 것들을 모두 어떤 deleter를 쓰는가에 상관없이 수행할 수 있다. std::unique_ptr과의 또 다른 중요한 차이점 중 하나는, std::shared_ptr의 경우 custom deleter의 사이즈가 std::shared_ptr의 크기에 **영향을 끼치지 않는다**는 것이다.

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/shared_ptr.png)  

실제 std::shared_ptr의 메모리 구조는 위와 같은 형태를 갖고 있다. 아까 레퍼런스 카운트를 내부에 저장해서 관리하고 있다고 했는데, 실제로는 레퍼런스 카운트 외의 여러 가지 std::shared_ptr을 관리하기 위한 정보를 control block이라고 해서 따로 할당한 다음 그 곳을 가리키는 포인터를 들고 있는 식으로 구성된다. 이렇게 control block이 메모리의 다른 공간에 따로 할당되기 때문에 std::shared_ptr의 크기가 custom deleter의 크기에 영향을 받지 않는 것이다.

 어떤 객체의 컨트롤 블록은 해당 오브젝트를 가리키는 std::shared_ptr이 처음으로 생성될 때 초기화진다. 하지만 처음 std::shared_ptr을 만들 때, 해당 오브젝트를 이미 가리키고 있는 다른 std::shared_ptr이 있는지 아닌지를 알 수 있는 방법이 없다. 그래서 컨트롤 블록을 생성할지 말지의 여부를 결정하는 것은 아래의 규칙을 따른다.

 - **std::make_shared ([item 21](item_21.md) 참조)는 항상 컨트롤 블록을 생성한다.** std::make_shared는 가리킬 새로운 객체를 생성하기 때문에, 이 시점에서 해당 객체와 관련된 컨트롤 블록이 절대 존재하지 않을 것임을 확신할 수 있다. 

 - **std::shared_ptr이 std::unique_ptr이나 std::auto_ptr로부터 생성될 경우 컨트롤 블록이 생성된다.** std::unique_ptr이나 std::auto_ptr의 경우 따로 컨트롤 블록을 사용하지 않는다. 따라서 std::shared_ptr이 std::unique_ptr, std::auto_ptr로부터 생성될 경우 컨트롤 블록을 생성해야함이 자명하다(이 경우 생성에 이용된 std::unique_ptr과 std::auto_ptr은 null이 된다).

 - **std::shaerd_ptr의 생성자가 raw pointer를 인자로 호출되었을 경우 컨트롤 블록을 생성한다.** 이미 컨트롤 블록을 갖고 있는 객체에 대한 std::shared_ptr을 만들고 싶다면, std::shared_ptr이나 std::weak_ptr([item 20](item_20.md) 참조)을 인자로 생성자를 호출할 것이다. 따라서 raw pointer를 인자로 넘길 경우 이미 만들어진 컨트롤 블록이 없을 거라고 보고 새롭게 컨트롤 블록을 생성한다.

마지막 룰이 야기하는 문제가 한 가지 있다. 아래 예제를 보자.

```C++

//Widget에 대한 raw pointer를 생성
auto pw = new Widget;
...
//pw에 대한 컨트롤 블록 생성
std::shared_ptr<Widget> spw1(pw, loggingDel);
...
//pw에 대한 컨트롤 블록을 또 생성
std::shared_ptr<Widget> spw2(pw, loggingDel);
```

이렇게 같은 raw pointer로 두 개의 std::shared_ptr을 만들 경우, 해당 오브젝트에 대한 레퍼런스 카운트를 포함한 컨트롤 블록이 2개가 생성된다. 이 2개 각각이 spw1,spw2의 소멸자가 호출될 때 오브젝트를 파괴하려고 들 것이고, 결국 이 건 이미 해제한 자원을 또 해제하려드는 문제를 발생시킨다.

 이런 문제 때문에 std::shared_ptr을 쓸 때 생성자의 인자로 raw pointer를 넘기는 건 별로 좋지 않다. 웬만하면 std::make_shared([item 21](item_21.md) 참조)를 쓰는게 좋으나, 이 경우 custom deleter를 쓰고 있는데 std::make_shared는 custom deleter와 같이 쓸 수 없다는 문제가 있다. 이런 경우에는 raw pointer를 넘길 때 다른 변수에 저장해서 넘기지말고 직접 바로 넘기는게 좋다.

```C++
//따로 대입하지 않고 바로 new를 이용해 넘김
std::shared_ptr<Widget> spw1(new Widget, loggingDel);

//spw2는 spw1과 같은 컨트롤 블록을 씀.
std::shared_ptr<Widget> spw2(spw1);
```

이렇게 new로 바로 넘기는 식으로 초기화할 경우 같은 객체에 대해 두 개 이상의 컨트롤 블록이 생기는 문제를 피할 수 있다.

## this 포인터

위에서 말한 raw pointer를 생성자의 인자로 넘기는 경우가 클래스의 this 포인터와 연관되어 까다로운 문제를 일으킬 수 있다. 프로그램에서 현재 동작 중인 Widget들을 추적하는 데이터 구조를 갖고 있다고 하자.

```C++
//현재 동작중인 Widget들을 관리하는 벡터.
std::vector< std::shared_ptr<Widget> > processedWidgets;

class Widget
{
public:
    //Widget 클래스의 동작.
    void process()
    {
        ...
        //동작중인 Widget의 목록에 이 Widget 추가.
        processedWidgets.emplace_back(this);
    }
};
```

여기서 ```processedWidgets.emplace_back(this)```구문이 치명적인 문제를 일으킬 수 있다. 현재 동작중인 Widget의 목록에 자기 자신을 추가한다는 의도 자체는 이상한게 없으나, 여기서 **raw pointer**를 넘기기 때문에 무조건 새로운 컨트롤 블록이 만들어진다는게 큰 문제다. 여기서 this를 추가하기 전에 이미 외부에서 이 객체를 가리키는 std::shared_ptr이 있었다면 하나의 객체에 대해 두 개의 컨트롤 블록이 만들어지게 되고 이건 앞에서 언급한 이미 해제한 자원을 또 해제하는 등의 골치 아픈 버그를 일으킬 것이다. 이 문제를 해결하기 위해 C++에는 ```std::enable_shared_from_this```라는 템플릿 클래스를 제공한다.

 이 템플릿 클래스를 사용하면 this 포인터로부터 안전하게 std::shared_ptr을 생성할 수 있다.

```C++
class Widget : public std::enable_shared_from_this<Widget>
{
public:

    void process()
    {
        //현재 오브젝트에 대한 std::shared_ptr을 proccesdWidget에 추가.
        processWidgets.emplace_back(shared_from_this());
    }
};
```

std::enable_shared_from_this 클래스에 타입 인자로 해당 타입 그 자체를 넘겨서 상속 받으면 this 포인터에 대해 컨트롤 블록을 다시 생성하는 불상사를 일으키지 않을 수 있다. shared_from_this 멤버 함수를 이용하면 된다. 내부적으로 shared_from_this 함수는 현재 오브젝트에 대한 컨트롤 블록이 있는지 검사한 후, 해당 컨트롤 블록을 가리키는 std::shared_ptr 포인터를 생성한다. 한 가지 문제는 이 때 반드시 해당 컨트롤 블록이 존재해야만 한다는 것이다. 만약 이 오브젝트를 가리키는 std::shared_ptr이 하나도 생성된 적이 없어 컨트롤 블록이 존재하지 않는다면, 이 함수는 일반적으로 예외를 던지긴 하지만 정의되지 않은 동작을 일으키게 된다. 

 이 문제를 해결하기 위해 보통 다음과 같은 방식으로 코드를 작성한다.

```C++
class Widget : public std::enable_shared_from_this<Widget>
{
public:
    //create를 통해서만 Widget을 생성할 수 있게 만든다
    template<typename... Ts>
    static std::shared_ptr<Widget> create(Ts&&.. params);

    //이전과 동일
    void process();

private:
    Widget(); //생성자를 private으로
};
```

 이렇게 하면 새로 Widget 자원을 할당받을 때 무조건 create 함수를 사용해야하고, 이 함수는 반드시 std::shared_ptr 형태로 돌려주므로 std::shared_ptr이 반드시 존재함을 보장받을 수 있다.

## std::shared_ptr의 성능

 이 앞까지 내용을 읽으면서 아마 std::shared_ptr의 성능에 대해 상당한 의구심이 생겼을 것이다. 컨트롤 블록의 크기는 custom deleter의 크기에 따라 상당히 비대해질 수 있으며, 레퍼런스 카운팅과 관련된 연산을 할 때는 atomic한 연산을 해야하기 때문에 여기서 드는 비용도 상당해 보인다.

 컨트롤 블록의 구현은 생각보다 훨씬 복잡하다. 대개의 경우 워드 몇 개 정도 크기지만 custom deleter와 allocator를 통해 크기가 상당히 커질 수 있으며, 일반적인 컨트롤 블록의 내부 구현은 생각보다 훨씬 복잡하게 되어 있다. 자신이 가리키고 있는 객체를 제대로 파괴하기 위해서 상속을 이용하며 심지어 가상함수도 쓴다. 이는 std::shared_ptr을 쓰면 컨트롤 블록에 의한 가상 함수 호출 비용까지 발생한다는 것이다.

 그래서 모든 자원 관리 문제에 std::shared_ptr을 적용시키는 건 사실 별로 좋은 방법이 아니다. 하지만 std::shared_ptr은 그 기능을 제공하기위해 굉장히 합리적인 비용을 지불하는 것이다. 기본적인 deleter와 allocator를 쓰고 std::make_shared를 이용해 생성되는 일반적인 조건 하에서 std::shared_ptr의 컨트롤 블록은 기껏해야 워드 3개 정도 크기이며 그 할당 비용은 거의 없다고 봐도 좋다(자세한 내용은 [item 21](item_21.md) 참조). std::shared_ptr의 역참조(dereference) 비용은 raw pointer와 동일하며 레퍼런스 카운팅에 드는 비용은 대개 1,2개 정도의 atomic한 연산에 드는 비용인데, 하드웨어에 따라 다르겠지만 대부분의 경우 이 연산은 CPU 명령어 하나로 대치되기 때문에 그리 큰 비용은 아니다. 컨트롤 블록에서의 가상 함수 호출은 일반적으로 std::shared_ptr이 관리하는 오브젝트 하나당 한 번(파괴될 때)만 일어난다.

 이런 비용을 지불하는 대신에 동적으로 할당한 자원에 대한 자동화된 lifetime 관리 시스템을 얻을 수 있는 것이다. 대부분의 경우, std::shared_ptr은 공유 자원의 lifetime을 관리하기에 아주 좋은 수단이다. 하지만 만약 배타적 소유권이 더 어울린다는 생각이 조금이라도 든다면 std::unique_ptr이 나은 선택일 수 있다. std::unique_ptr의 성능은 raw pointer와 거의 동일하며, std::shared_ptr로 변경하기도 아주 쉽다(std::unique_ptr로부터 std:shared_ptr을 만들 수 있으므로). 하지만 std::shared_ptr을 std::unique_ptr로 변환하는 것은 불가능하다.

 또 한 가지 std::shared_ptr은 배열에 대해서는 동작하지 않는다. 즉 std::shared_ptr<T[]>같은 건 없다. 뭐 delete[]를 수행하는 custom deleter를 만들면 꼭 못할 것도 없긴 하지만 굳이 그런 이상한 짓을 하면서까지 T[]에 대한 std::shared_ptr을 쓸 이유가 없다. 저런식으로 써봤자 std::shared_ptr은 []연산을 지원하지 않기 때문에 배열을 참조하기 위해선 굉장히 이상한 문법을 쓰게 된다. [item 19](item_19.md)에서도 말했지만 배열의 경우 대부분 std::vector, std::array, std::string 같은 것을 쓰는것이 훨씬 좋다.