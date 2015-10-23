# const_iterator를 쓰자!

STL에서 const_iterator는 포인터로 따지면 const T* 와 같다. 즉 iterator가 가리키는 대상을 수정하지 못함(데이터의 상수성)을 의미하는 것이다. Effective C++에서도 다룬 내용이지만 const는 많이 쓸 수록 좋다. 해당 iterator가 가리키는 대상을 꼭 수정할 필요가 없다면 const_iterator를 사용하는 편이 더 안정적인 코드가 될 것이다.

## C++ 98에서는 써먹을 게 못 됐다

 const를 쓸 수 있으면 가능한 한 많이 쓰는 게 좋다는 일반적인 원칙에도 불구하고 C++ 98에서는 const_iterator를 쓰기가 힘들었다. 사용하기도 굉장히 불편했고 기능이 제대로 구현되어 있다고 보기 힘든 상태였기 때문이다. 예제를 통해 C++ 98에서 const_iterator가 어땠는지 한 번 살펴보자.

```C++
std::vector<int> values;

std::vector<int>::iterator it = 
    std::find(values.begin(), values.end(), 1983);
values.insert(it, 1998);
```

int형의 vector에서 1983이라는 숫자가 있는 위치를 찾은 후 그 자리에 1998을 삽입하는 코드다(1983이 존재하지 않는다면 벡터의 맨 마지막에 1998이 삽입될 것이다). 이 코드는 단순히 vector에서 위치를 찾고 삽입할 뿐 iterator가 가리키는 원소의 값을 바꾸는 동작은 절대 하지 않는다. const는 많이 쓸 수록 이로운 것이라 했으니 const_iterator로 바꿔서 쓰자.

```C++
//C++ 98에서의 예니까 using 대신 typedef를 씀
typedef std::vector<int>::iterator       IterT;
typedef std::vector<int>::const_iterator ConstIterT;

std::vector<int> values;

//casting을 이용한 방법
ConstIterT ci = std::find(static_cast<ConstIterT>(values.begin()),
                        static_cast<ConstIterT>(values.end()), 1983);

//const reference를 이용한 방법
const std::vector<int>& cvalues = values;
ConstIterT ci2 = std::find(cvalues.begin(), cvalues.end(), 1983);

values.insert(static_cast<IterT>(ci), 1998);
```

C++ 98에서의 문제점이 적나라하게 드러나 있다. 심지어 이렇게 써도 컴파일이 안 된다. C++ 98에서 iterator 대신에 const_iterator를 쓰는게 왜 이렇게 힘든지 한 번 살펴보자.

#### const가 아닌 container에서 const_iterator는 못 가져온다

코드에서 values는 std::vector<int>이다. 이 vector에는 const 설정이 붙어있지 않기 때문에(const std::vector<int>가 아니기 때문에) const_iterator를 쉽게 가져올 방법이 없다(C++ 98). 그래서 캐스팅을 하든 values를 const형 레퍼런스로 가리킨 후 거기서 const_iterator를 가져오든 뭔가 변환 과정이 필요하다. 이 과정이 일단 코드를 복잡하게 만든다.

#### const가 아닌 container에는 그냥 iterator만 삽입 가능하다

두번째 문제는 이렇게 해서 찾은 const_iterator를 바로 container에 삽입할 방법이 없다는 것이다. const가 아닌 container는 삽입, 삭제등의 인자로 넘기는 iterator에 const_iterator를 허용하지 않는다. 그래서 위 코드에서 다시 iterator 타입으로 캐스팅해서 넘기는 것이다.

#### const_iterator -> iterator는 불가능!

이건 C++ 11에서도 해당되는 이야기다. const_iterator를 iterator로 캐스팅하려고 해도 이 둘을 서로 변환할 방법이 없기 때문에  여기서 컴파일 에러가 발생한다. 이런 문제들 때문에 C++ 98에서는 const_iterator를 효과적으로 사용하기가 힘들다.

## C++ 11에서는 편하게 쓸 수 있다

 C++11 에서는 const_iterator를 굉장히 편리하게 사용할 수 있다. cbegin과 cend라는 멤버 함수가 있어서 이 함수를 이용하면 const가 아닌 container라도 const_iterator를 쉽게 가져올 수 있다. 그리고 insert, erase 같이 iterator를 인자로 받는 멤버 함수들도 C++ 98과는 다르게 const_iterator를 인자로 받게 되어있어서 쉽게 사용가능하다. C++11 스타일로 위 코드를 다시 작성해보자.

```C++
std::vector<int> values;

//cbegin, cend 이용해서 const_iterator 받아오기
auto it = std::find(values.cbegin(),values.cend(), 1983);

values.insert(it, 1998);
```

이 덕분에 const_iterator를 굉장히 실용적으로 쓸 수 있게 됐다.

## template에서의 const_iterator

template 등에서 container 및 유사 container 타입(예를 들자면 내장 타입(built-in type)의 배열, 서드파티 라이브러리의 컨테이너 류 타입 등)에 대해 최대한 일반적으로 동작하는 코드를 작성하고 싶을 수 있다. 이 경우 멤버 함수 begin, end 등을 쓰는 것보다 멤버가 아닌 begin, end를 쓰는게 좋다. 라이브러리 등에서 제공하는 코드가 멤버 함수로 반드시 이런 함수를 포함하고 있으리라는 보장이 없으니까.

```C++
template<typename C, typename V>
void findAndInsert(C& container,
                    const V& targetVal,
                    const V& insertVal)
{
    using std::cbegin;
    using std::cend;

    auto it = std::find(cbegin(container),cend(container),targetval);

    container.insert(it, insertVal);
}
```

하지만 위 코드는 C++ 14에서만 돌아간다. 왜 그랬는지는 몰라도 C++ 11에서는 멤버 함수가 아닌 cbegin, cend, rbegin, rend, crbegin, crend가 다 빠졌다.

C++11에서 위와 같은 방식으로 동작하는 코드를 짜고 싶다면 아래 코드를 추가하면 된다.

```C++
template <class C>
auto cbegin(const C& container) -> decltype(std::begin(container))
{
    return std::begin(container);
}
```

컨테이너 타입이 C일 때 const C& 타입 컨테이너의 begin 리턴값을 돌려줬으니, 그 결과 값은 당연히 const_iterator가 된다. C++ 11의 비멤버 함수 begin, end는 내장 배열 타입에 대해서도 템플릿 특수화를 통해 T* const의 값을 돌려주도록 되어 있으므로 이 템플릿 함수는 내장 타입에 대해서도 잘 동작한다.