 포인터는 C/C++을 강력한 언어로 만들어주는 도구이면서 동시에 굉장히 다루기 까다로운 언어로 만들어주는 도구이다. 왜 포인터는 어렵고 짜증나는 개념인가? 그 이유를 대충 요약하자면 아래 6가지 정도의 항목이 나온다.

 1. 포인터가 가리키는 주소의 값이 배열인지 하나의 객체인지 판별이 불가능하다.

 2. 포인터가 가리키고 있는 주소의 값을 내가 다 쓰고 나서 파괴해야할지 아닐지 판단하는게 불가능하다. 즉, 포인터가 가리키는 대상을 내가 **소유하고 있는지 아닌지** 알 수 있는 방법이 없다.

 3. 만약에 포인터가 가리키는 대상을 파괴하기로 마음먹었다고 하자. 문제는, 그걸 파괴하기 위해 어떤 방법을 써야할 지 알아낼 방법이 없다는 것이다. 모든 자원을 다 delete를 써서 제거하는 것은 아니다. release 등의 특정 함수를 호출해서 파괴해야한다든지 다 사용한 자원을 파괴하는 방식도 따져보면 가지각색이다.

 4. delete를 써서 제거하는게 옳은 방법이라는 걸 알아냈다고 해도, 한 가지 문제가 더 있다. delete를 써야할지, delete []를 써야할지 알 수 없다. 1번 이유과 연결되는 부분인데, 결국 가리키고 있는 주소의 값이 단일 객체인지 배열인지 알 수 없기 때문이다.

 5. 설령 위의 모든 어려움을 뚫고 내가 해당 포인터의 자원을 소유하고 있어서 그걸 파괴해야하며, 어떤 방식으로 파괴해야하는 지도 다 알아냈다고 가정하자. 이 때 생기는 또다른 문제점은 자원 해제는 **정확히 한 번만** 해야한다는 것이다. 굉장히 긴 코드의 흐름 속에서 자원 해제가 정확히 한 번만 일어남을 보장하는 것은 상당히 어려운 문제다. 중간에 함수가 return 되거나 예외를 던지거나 해서 자원을 해제하는 코드에 도달하지 못할 수도 있다. 혹은 이 때문에 여기 저기 자원 해제 코드를 넣었다가 자원 해제가 두 번이상 일어나버릴 수도 있다.

 6. 이미 해제된 자원의 주소를 가리키는 포인터인 댕글링 포인터(dangling pointer)라는 까다로운 문제가 있다. 요는 포인터가 가리키고 있는 자원이 이미 해제된 자원인지 아닌지 판별할 수가 없다는 것이다.

포인터가 강력한 도구임은 부인할 수 없다. 하지만 동시에 위에 언급한 문제점들이 보여주듯이 굉장히 어렵고 불안정한 도구이기도 하다. 이런 문제점을 해결하기 위해 C++ 11에서는 **스마트 포인터**라는 것이 도입되었다. 스마트 포인터는 포인터의 강력함을 유지하면서 위의 곤란한 문제들을 해결하기 위한 다양한 방법들을 제공해준다. 

# 배타적 소유권에는 std::unique_ptr

std::unique_ptr은 거의 대부분의 경우 일반적인 포인터와 크기, 성능이 모두 동일하다는 강력한 장점을 갖고 있다. std::unique_ptr의 또다른 중요한 특징은, std::unique_ptr이 **배타적 소유권(exclusive ownership)**을 구현한 것이라는 점이다. 즉 std:unique_ptr은 nullptr를 가리키는 게 아닌 이상 항상 포인터가 가리키는 대상을 소유하고 있다. 이 때문에 std::unique_ptr은 복사가 불가능하다. 위에서도 말했듯이 **배타적 소유권**을 갖고 있기 때문에 한 번에 여러 개체가 자신이 관리하고 있는 자원을 참조하면 안되는 것이다. 하지만 복사가 아닌 이동은 허용된다. 이동을 시킬 경우 소유권이 다른 포인터로 넘어가는 것이다(원래 해당 개체를 가리키던 std::unique_ptr은 nullptr이 된다). std::unique_ptr은 기본적으로 자신이 소멸될 때 자기가 소유하고 있던 자원을 delete를 이용해서 파괴시킨다.

## 활용 예

std::unique_ptr은 계층적인 구조를 가진 객체들의 팩토리 함수 리턴타입으로 활용될 수 있다. 예를 들어, Investemet라는 기반 클래스를 가진 투자를 위한 타입들의 계층이 있다고 하자. 

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/investment.png)  


```C++
class Investment { ... };

class Stock : public Investment { ... };

class Bond : public Investment { ... };

class RealEstate : public Investment { ... };
```

팩토리 함수는 보통 어떤 계층 구조에 대해 객체의 메모리를 힙에 할당한 다음 그 포인터를 리턴해준다. 팩토리 함수를 호출한 쪽은 리턴받은 객체에 대해 다 쓰고 그걸 파괴해 주어야 한다. 이 상황은 std::unique_ptr이 쓰이기 정말 적합한 상황이다. 왜냐하면 팩토리 함수의 결과로 나온 객체는 그걸 호출한 쪽이 **소유**하게 되는 것이고, std::unique_ptr은 소멸될 때 자동으로 자원을 해제해주기 때문이다. 위 계층 구조에 대한 팩토리 함수는 다음과 같은 형태가 될 것이다.

```C++
//주어진 인자로부터 객체를 생성한 후 해당 포인터를 std::unique_ptr<Investment> 형태로 돌려줌
template<typename... Ts>
std::unique_ptr<Investment> makeInvestment(Ts&&... params);

//사용시
{
    //pInvestment는 std::unique_ptr<Investment> 타입을 갖게 됨.
    auto pInvestment = makeInvestment( arguments );

    ...

} // <- 이 시점에서 std::unique_ptr의 소멸자가 호출되며 이 때 할당한 자원이 파괴됨.

```

이 예제의 경우 리턴 받은 std::unique_ptr을 바로 사용하고 그 소멸자가 호출되는 시점이 비교적 명확히 보이지만 경우에 따라 std::unique_ptr이 제대로 자원을 해제할 지 의심이 갈 수도 있다. 한 가지 예로 팩토리 함수를 통해 받은 std::unique_ptr을 컨테이너 안에 넣어놓고 보관하면서 쓰다가 나중에 해제하거나 하는 경우가 있을 수 있다. 아니면 소유권 이전을 통해 계속해서 std::unique_ptr이 여기 저기로 이동하고 있는 와중에 함수나 loop의 이른 종료(return, break, 예외 발생 등)등이 발생하는 경우도 있다. 하지만 그 어떤 경우든 결과적으로 std::unique_ptr의 소멸자는 호출되고, 해당 std::unique_ptr이 소유하고 있던 자원의 해제 역시 이 때 정상적으로 이루어지게 된다.

> 사실 항상 해제된다고 말할 수는 없다. 비정상적인 프로그램 종료 상황 등에서는 소멸자가 호출되지 않을 수 있기 때문이다. 스레드의 primary function(main 함수 같은)로부터 예외가 전파되거나, 혹은 noexcept가 위반(violate - [item 14](item_14.md) 참조)되는 경우, 지역 객체들이 제대로 파괴되지 않을 수 있다. 그리고 std::abort 함수나 다른 exit 함수들(std::_Exit, std::quick_exit)의 경우 명백하게 지역 객체들이 파괴되지 않는다.

 기본적으로 std::unique_ptr의 자원 해제 동작은 ```delete```를 통해 이루어지지만, std::unique_ptr을 생성할 때 **custom deleters**를 명시해줄 경우 해당 custom deleter를 이용해 자원을 해제한다.

```C++
//lambda expression을 통한 custom deleter 정의
auto delInvmt = [](Investment* pInvestment)
{
    //삭제하기 전에 삭제 로그를 남기고 싶다!
    makeLogEntry(pInvestment);
    delete pInvestment;
};


//custom deleter의 타입도 타입 정의에 포함된다.
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt)>
makeInvestment(Ts&&... params)
{
    //리턴할 포인터
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);

    if( /* Stock object가 생성되어야 할 경우*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* Bond object가 생성되어야 할 경우*/ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* RealEstate object가 생성되어야 할 경우*/ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }

    return pInv;
}

```

 일단 왜 구현을 이런식으로 했는지가 잘 이해가지 않을 수 있다.

 - delInvmt는 makeInvestment로부터 리턴된 객체를 파괴하기 위한 custom deleter다. custom deleter는 제거될 객체의 포인터(raw pointer)를 받으며 해당 객체를 파괴하는데 필요한 동작을 수행한다. 이 경우 간단한 로그를 남기는 동작을 수행했으며 lambda expression을 이용해서 정의했다.

 - custom deleter를 이용할 때는 반드시 std::unique_ptr의 타입 인자로 해당 custom deleter의 타입을 명시해주어야한다. 그래서 이 경우 decltype(delInvmt)를 이용해 custom deleter의 타입을 명시해주었다.

 - makeInvestement 함수는 기본적으로 nullptr을 가리키는 std::unique_ptr을 만든 다음 조건에 따라 적절한 객체를 생성해서 std::unique_ptr이 그걸 가리키게 하는 방법을 사용했다. custom deleter delInvmt를 std::unique_ptr과 연결하기 위해 처음 생성할 때 두 번째 인자로 delInvmt를 넘겨주었다.

 - raw pointer를 std::unique_ptr에 대입하는 것은 컴파일되지 않는다. raw pointer로부터 스마트 포인터로의 암시적 변환은 허용되지 않기 때문이다. 그래서 reset 함수를 이용해서 std::unique_ptr이 가리킬 객체를 지정해주었다.

 - 각각의 new에서 자원을 넘길 때 std::forward를 썼다. [item 25](item_25.md)에서 언급되지만, 생성될 객체에서 함수 호출자가 넘긴 인자들을 모두 이용가능하게 만들기 위해 std::forward(perfect forward)를 사용한다.

 - custom deleter는 Investment* 타입의 인자를 받는다. makeInvestment 내부에서 만들어지는 객체의 실제 타입이 무엇이든지에 상관없이(Stock이든, Bond든, RealEstate든) 생성된 객체들은 결과적으로 custom deleter에 의해서 파괴된다. 이 말은 즉 부모 타입의 포인터로 자식 타입의 포인터를 파괴할 수 있어야한다는 뜻이다. 즉, Investment 클래스는 파괴자를 virtual로 선언해야한다.

```C++
class Investment
{
public:
    ...
    virtual ~Investment():
    ...
};
```

어쨌든 결과적으로 makeInvestment는 굉장히 멋진 인터페이스를 갖게 된다. 이 팩토리 함수를 통해 자원을 제공받는 쪽에서는 auto 키워드를 통해 실제 타입이 뭔지 하나도 신경쓰지 않고(즉, 파괴 과정에 어떤 특별한 일이 이루어진다해도 그런 파괴 작업에 전혀 신경쓸 필요 없이) 자원을 할당받고 사용할 수 있으며 해당 자원이 항상 제대로 파괴됨을 보장받을 수 있게 된다.

 C++ 14에서는 return type deduction([item 3](item_03.md) 참조)을 이용해 좀 더 깔끔하게 makeInvestment 함수를 작성할 수 있다.

```C++
template<typename... Ts>
auto makeInvestment(Ts&&... params)
{
    //이제 lambda expression이 함수 안으로 들어왔다.
    auto delInvmt = [](Investment* pInvestment)
    {
        makeLogEntry(pInvestment);
        delete pInvestment;
    };

    //뒷부분은 이전과 마찬가지
    std::unique_ptr<Investment, decltype(delInvmt)> pInv(nullptr, delInvmt);

    if( /* Stock object가 생성되어야 할 경우*/)
    {
        pInv.reset(new Stock(std::forward<Ts>(params)...));
    }
    else if ( /* Bond object가 생성되어야 할 경우*/ )
    {
        pInv.reset(new Bond(std::forward<Ts>(params)...));
    }
    else if ( /* RealEstate object가 생성되어야 할 경우*/ )
    {
        pInv.reset(new RealEstate(std::forward<Ts>(params)...));
    }

    return pInv;
}
```

default deleter를 쓰는 경우에는 std::unique_ptr의 크기가 정확히 raw pointer와 같다고 할 수 있으나 custom deleter를 쓰는 경우에는 이야기가 달라진다. custom deleter가 함수 포인터인 경우 일반적으로 std::unique_ptr의 크기를 워드(word) 하나 크기에서 2개 크기로 늘린다. 함수 객체인 경우 해당 함수 객체가 내부에 저장하고 있는 상태의 개수가 얼마나 많은가에 따라 std::unique_ptr의 크기가 달라진다. 하지만 아무것도 capture 하지 않은 lambda expression같이 Stateless한 함수 객체는 크기에 있어서 어떤 패널티도 없다. custom deleter을 함수로 짜는 것보다는 captureless lambda expression으로 짜는게 더 좋다는 것이다.

```C++

//stateless lambda를 이용한 custom deleter
auto delInvmt1 = [](Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
};

// 이 경우 unique_ptr의 크기가 Investment*와 같다.
template<typename... Ts>
std::unique_ptr<Investment, decltype(delInvmt1)> 
makeInvestment(Ts&&... args);

void delInvmt2(Investment* pInvestment)
{
    makeLogEntry(pInvestment);
    delete pInvestment;
}

//이 경우 Investment*보다 최소 function pointer 크기만큼 커짐. 
template<typename... Ts>
std::unique_ptr<Investment, void(*)(Investment*)> 
makeInvestment(Ts&&... args);
```

>이 부분 굉장히 신기해서 unique_ptr 헤더 파일을 좀 뒤져봤는데, 기본적으로 stateless lambda expression은 size가 0이다. 이걸 이용해서 size가 0인 custom deleter가 타입 인자로 들어왔을 때에는 해당 custom deleter를 내부적인 멤버로 저장하는게 아니라 그걸 상속받는 방식으로 구현. 이게 TMP를 이용한 것 같은데 아직 TMP를 잘 몰라서 그런지 코드에 이해가 안 가는 부분이 많긴 한데 대충 이런 개념으로 구현해서 stateless lambda expression을 이용할 경우 size에 변화가 생기지 않는 것 같다. 관련 내용이 더 궁금하면 [unique_ptr의 내부 구현](../unique_ptr의 내부구현.md)을 참조. 아는 만큼만 정리해둠..

만약 custom deleter가 엄청난 양의 state를 가지고 있을 경우 std::unique_ptr의 크기도 어마어마해질 것이다. 보통 이런 경우는 설계가 잘못된 것이므로 설계를 바꾸는 편이 낫다.

std::unique_ptr은 두 가지 형태를 갖고 있다. 하나는 ```std::unique_ptr<T>```이고, 다른 하나는 ```std::unique_ptr<T[]>```이다. 대상이 단일 객체일 경우 delete를 이용해서 파괴하고 배열일 경우 delete[]를 이용한다. 그 외에도 ```std::unique_ptr<T[]>```은 [] 연산자를 제공하고 ```std::unique_ptr<T>```의 경우 * 연산과 -> 연산을 제공하는 등의 편의성도 있다. 근데 ```std::unique_ptr<T[]>```의 경우 별로 쓸 일이 없을 것이다. std::vector나 std::array, std::string같이 배열에 대해선 더 안정적이고 유용한 녀석들이 이미 존재하기 때문이다.

 std::unique_ptr의 또다른 장점은 이게 std::shared_ptr과 호환이 된다는 점이다.

```C++
//std::unique_ptr이 std::shared_ptr로 변환된다.
std::shared_ptr<Investment> sp = makeInvestment( arguments );
```

 이건 std::unique_ptr이 팩토리 함수의 리턴값으로 쓰이기 좋은 이유이기도 하다. 팩토리 함수는 함수 호출자가 생성한 자원을 배타적으로 소유해서 사용할지 아니면 공유해서 사용할 지 알 수없다. 하지만 std::unique_ptr로 일단 리턴하면 두 경우 모두에 호환되기 때문에 리턴 타입으로 정확히 들어맞는 것이다. std:shared_ptr에 관한 자세한 내용은 바로 다음 item인 [item 19](item_19.md)를 참고.

 또 std::unique_ptr의 활용에는 팩토리 함수 외에도 Pimple Idiom의 구현도 있는데, 이 관련 내용은 [item 22](item_22.md)에서 다룰 예정.