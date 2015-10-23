# std::weak_ptr의 사용

 std::shared_ptr처럼 동작하지만 막상 레퍼런스 카운팅은 되지 않는 포인터가 유용하게 쓰일 때가 있다. 이런 포인터를 std::weak_ptr이라고 부른다. std::weak_ptr은 역참조(dereference)도 안 되고 nullptr인지 아닌지 테스트하는 것도 안 된다. std::weak_ptr은 그 자체로 홀로 쓰이는 스마트 포인터가 아니기 때문이다. 이까지만 들으면 당최 이 놈이 어디서 유용하다는 것인지 전혀 이해가 안 될 수 있다. 하지만 std::weak_ptr은 굉장히 유용한 스마트 포인터이다. 어떤 경우에 활용될 수 있는지 하나씩 살펴보자.

## dangling 감지

 std::weak_ptr은 이미 파괴됐을 가능성이 있는 자원을 참조해야할 경우가 있을 때 해당 포인터가 dangling된 포인터인지 아닌지를 파악할 때 유용하게 쓰일 수 있다. std::weak_ptr은 보통 std::shared_ptr로부터 생성되며 자신이 가리키는 객체의 레퍼런스 카운트를 변화시키지 않는다.

```C++

//shared_ptr 생성. 이 시점에서 reference count는 1.
auto spw = std::make_shared<Widget>();

...

//wpw는 spw와 같은 객체를 가리키게 됨. 여전히 reference count는 1.
std::weak_ptr<Widget> wpw(spw);

...

//여기서 reference count는 0이 되고 Widget은 파괴된다.
//wpw는 댕글링(dangling)됨.
spw = nullptr;

```

이 상황에서 std::weak_ptr은 expired 멤버함수를 통해 자신이 가리키는 객체가 댕글링되었는지 확인 가능하다.

```C++
if(wpw.expired())
{
    //wpw가 제대로 된 오브젝트를 가리키고 있지 않은 경우
}
```

 하지만 실제로는 해당 포인터가 댕글링되었는 지 여부를 파악하는 것뿐만 아니라, 댕글링되어있지 않다면 해당 오브젝트를 참조하는 연산을 하고자 하는 경우가 대부분이다. std::weak_ptr은 기본적으로 역참조 기능을 제공하지 않기 때문에 이 경우 expired 체크와 사용을 별도로 진행해야하는데, 이게 경쟁 상태(race condition)을 일으킬 수 있다. 멀티 스레드 환경의 경우 expired 체크를 통과한 시점에서, 다른 스레드가 해당 자원을 해제해버리면 이미 해제된 자원을 참조하는 문제가 충분히 발생할 수 있다는 것이다.

 이걸 해결하는 건 굉장히 쉽다. std::weak_ptr로부터 std::shared_ptr을 생성하는 멤버 함수인 lock을 쓰면 된다. lock은 해당 자원이 댕글링됐는지 아닌지 검사한 뒤 댕글링됐으면 nullptr를, 그렇지 않으면 해당 자원을 가리키는 포인터를 돌려주는 atomic한 멤버 함수다.

```C++
//만약 wpw가 이미 댕글링되었다면, spw1은 null이 된다.
std::shared_ptr<Widget> spw1 = wpw.lock();

auto spw2 = wpw.lock(); //마찬가지. auto를 썼을 뿐

//이렇게 생성자의 인자로 넘길 경우,
//wpw가 댕글링됐다면 std::bad_weak_ptr 예외를 던진다.
std::shared_ptr<widget> spw3(wpw);
```

## 팩토리 함수에서의 활용

어떤 unique ID에 따라 읽기만 가능한 객체에 대한 스마트 포인터를 제공하는 팩토리 함수를 생각해보자. [item 18](item_18.md)에 따라 이 팩토리 함수의 리턴 타입은 std::unique_ptr로 하는 것이 좋을 것이다.

```C++
std::unique_ptr<const Widget> loadWidget(WidgetID id);
```

이 때 만약 loadWidget이 한 번 호출하는데 상당히 큰 비용이 드는 함수라면, 그리고 각 ID가 반복적으로 재사용되는게 일반적인 상황이라면 이 경우 가장 합리적인 최적화는 **캐싱**을 이용하는 것이다. 생성한 후 캐싱해서 그걸 돌려주되 캐싱한 객체가 쓸모없어지면 그 때 메모리를 해제해주는 방식으로 동작하는 것이다.

 이렇게 캐싱을 기반으로 동작하는 팩토리 함수에서 리턴타입으로 std::unique_ptr을 쓰는 건 별로 적합하지 않다. 함수 호출하는 쪽에서는 캐시된 오브젝트의 포인터를 받아서 써야하고, 그 lifetime을 관리해야하지만 한편으론 팩토리 함수도 캐시된 오브젝트의 포인터를 알아야하며, 해당 포인터가 가리키는 오브젝트가 파괴됐는지 아닌지 여부도 알아야만 한다. 따라서 배타적 소유권을 기반으로 동작하는 std::unique_ptr은 적합하지 않다. 이 경우에 적합하게 이용될 수 있는 것이 std::weak_ptr이다. std::weak_ptr은 캐시된 오브젝트가 파괴되었을 때 그걸 감지해줄 수 있기 때문이다. std::weak_ptr을 기반으로 작성된 팩토리 함수는 아래와 같다.

```C++
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
    //캐시된 오브젝트 관리
    static std::unordered_map<WidgetID,
                              std::weak_ptr<const Widget> > cache;

    //이미 캐시되어있는지 여부 확인
    auto objPtr = cache[id].lock();

    //캐시되어 있지 않다면 캐시한다
    if(objPtr == nullptr)
    {
        objPtr = loadWidget(id);
        cache[id] = objPtr;
    }

    return objPtr;
}
```

## Observer Pattern의 구현

옵저버(Observer) 디자인 패턴에서 가장 중요한 객체는 Subject(상태가 바뀔 수 있는 객체)와 Observer(상태가 바뀌었을 때 이를 통지 받는 객체)이다.  대부분의 구현에서 각각의 Subject들은 그 멤버로 자신의 Observer에 대한 포인터를 갖고 있게 된다. 이렇게 하면 상태가 바뀌었을 때 Observer에 통지를 하기 쉬워지기 때문이다. 각각의 Suject들은 Observer를 언제 파괴해야할지 같은 거에는 별 관심이 없으나, 해당 Observer가 파괴되었는지 아닌지에는 민감하다. Observer가 파괴되었으면 더 이상 상태 변화 통지를 하지 않아야하기 때문이다. 따라서 각각의 Subject는 자신의 상태변화를 통지할 Observer들의 포인터를 std::weak_ptr로 들고 있는 것이 가장 좋을 것이다. 

## 순환 참조 해결

 마지막 예제로 std::shared_ptr의 순환 참조 문제를 생각할 수 있다. 일단 아래와 같은 상황을 생각해보자.

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/weak_ptr1.png)  

A,B,C라는 세 개의 오브젝트가 있고 A,C가 각각 B에 대한 포인터를 std::shared_ptr을 통해 갖고 있다. 이 때 만약 B가 다시 A에 대한 포인터를 갖고 있는 것이 더 유용하다고 하자. 그럼 이 포인터는 어떤 종류가 되는게 좋을까?

![screenshot](https://github.com/jwvg0425/ModernCppStudy/blob/master/weak_ptr2.png)  

3가지 정도의 선택이 있을 수 있다.

 - **raw pointer** : 이 경우, A가 파괴되었다 해도 C는 여전히 B에 대해 참조할 권한을 갖고 있고, 여기서 다시 B가 갖고 있는 A의 포인터(이미 파괴된 객체를 가리키는 - dangling)를 참조할 위험이 생긴다. 그리고 이건 raw pointer기 때문에 B는 A가 이미 파괴되었는지 여부를 알 수 없다.

 - **std::shared_ptr** : 이 경우, A와 B는 서로에 대한 std::shared_ptr을 갖게 된다. 이렇게 순환 참조가 발생하면 A,B 두 개의 자원 모두 더 이상 접근 불가능한 자원이 되었다 해도 절대 파괴되지 않게 된다. 서로가 서로를 참조함으로써 파괴되어야함에도 불구하고 레퍼런스 카운트가 1이 남게 되기 때문이다. 이는 memory leak 현상의 발생으로 이어진다.

 - **std::weak_ptr** : 위의 두 문제를 모두 피할 수 있는 방법이다. A가 파괴되었을 때, B의 A에 대한 포인터는 댕글링되지만, std::weak_ptr이기 때문에 댕글링 여부를 파악가능하다. 또 std::weak_ptr은 레퍼런스 카운트에 영향을 끼치지않기 때문에 까다로운 순환 참조 문제도 발생하지 않는다.

위 세가지 선택지 중에서 std::weak_ptr을 쓰는게 가장 좋은 선택이리라는 것이 자명하다. 하지만 이런 경우가 그리 흔한 것은 아니다. 보통 트리와 같은 엄격한 수직적 계층 구조의 경우 자식 노드는 보통 부모 노드의 포인터를 가지게 된다. 그리고 부모 노드가 파괴되었을 때, 자식 노드도 같이 파괴되는 것이 정상이다. 따라서 부모 노드의 자식에 대한 포인터는 std::unique_ptr로 나타내는 것이 좋다(부모가 파괴될 때 자식도 자동으로 같이 파괴될 것이므로). 자식 노드의 부모에 대한 포인터는 raw pointer로 해도 된다. 자식이 부모보다 긴 lifetime을 가질 리가 없기 때문이다. 물론 모든 경우에 이런 엄격한 수직적 계층 구조를 사용하는 것은 아니므로 적절한 상황에는 std::weak_ptr을 쓰는게 좋다(위에서 언급한 observer pattern 같은 경우 등).

std::weak_ptr에 대해 알아두어야할 또 한가지는, std::weak_ptr이 std::shared_ptr에서 설명한 특성을 거의 그대로 갖고 있다는 것이다. std:;weak_ptr 역시 std::shared_ptr처럼 워드 2개분의 크기를 가지며 std::shared_ptr과 동일한 컨트롤 블록을 이용한다. 또 생성자, 소멸자 및 대입 연산 등도 역시 std::shared_ptr과 마찬가지로 atomic한 레퍼런스 카운팅 연산을 수행한다. 처음에 std::weak_ptr은 레퍼런스 카운팅에 참여하지 않는다고 했는데 이게 뭔 귀신 씨나락 까먹는 소린가 싶을 것이다. std::weak_ptr은 **해당 객체를 가리키고 있는 참조의 숫자(pointed-to object's reference count)**에는 영향을 끼치지 않지만 컨트롤 블록의 또다른 레퍼런스 카운트인 weak count에는 영향을 끼친다. 자세한 내용은 [item 21](item_21.md)에서 계속.