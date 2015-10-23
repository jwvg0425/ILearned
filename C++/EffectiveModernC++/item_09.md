#typedef보다는 alias declaration을 써라

`std::unique_ptr<std::unordered_map<std::string, std::string>>` 같은 타입을 생각해보자. STL 컨테이너와 스마트 포인터가 섞이게 되면 타입 이름이 굉장히 길어지게 되고, 이건 코드 가독성을 심하게 훼손한다. C++98까지는 이런 상황에 typedef를 썼다.

```C++
typedef
  std::unique_ptr<std::unordered_map<std::string, std::string>>
  UPtrMapSS;
```

C++11에서는 이런 경우에 typedef대신 alias declaration을 사용한다.

```C++
using UPtrMapSS =
  std::unique_ptr<std::unordered_map<std::string, std::string>>
```

typedef나 alias declaration이나 하는 역할은 완전히 똑같고, 기술적으로도 무슨 차이가 생기거나 하는 건 아니다. 하지만 typedef에 비해 alias declaration이 훨씬 가독성이 좋고, 헷갈리는 일이 덜하다. 특히 함수 포인터같은 경우 typedef는 문법이 눈에 잘 안들어오고, 기억하기도 힘들다.

```C++
typedef void (*FP)(int, const std::string&);

using FP = void (*)(int, const std::string&);
```

어떤 코드가 가독성이 더 높은 지는 자명할 것이다. 여기에 더불어, alias declaration은 typedef가 따라 올 수 없는 굉장한 장점이 있다. 바로 **템플릿**을 이용할 수 있다는 것이다. 

```C++
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

MyAllocList<Widget> lw;
```

위와 같이 alias declaration과 템플릿을 같이 이용할 수 있다. 물론 적절하게 응용하면 typedef으로도 이와 유사한 구현을 할 수 있다(실제로도 자주 이용되는 방법이다).

```C++
template<typename T>
struct MyAllocList
{
    typedef std::list<T, MyAlloc<T>> type;
};

MyAllocList<Widget>::type list;
```

여기서 typedef를 이용한 위 코드의 경우 템플릿과 함께 쓰면 귀찮은 일이 많이 생긴다.

```C++
template<typename T>
class Widget
{
private:
   typename MyAllocList<T>::type list;
};
```

여기서 MyAllocList<T>::type은 T에 의존적이다. 여기서 typename 키워드를 통해 MyAllocList<T>::type이 타입이라는 걸 알려주지 않으면 에러가 발생한다. 왜냐하면, 우리는 MyAllocList<T>::type이 타입이라는 걸 알지만 컴파일러는 이게 타입을 가리키는 지 알 수 없기 때문이다. 누군가가 MyAllocList의 특정한 T 타입에 대해서만 템플릿 구체화를 통해 type을 멤버 변수로 선언해버릴지도 모를 일 아닌가? 그렇게 되면 MyAllocList<T>::type은 더 이상 타입이 아니게 된다. 즉, 컴파일러 스스로는 템플릿 내부에서 이런 의존적인 타입에 대해 그게 실제로 타입인지 아닌지 판단할 수 없으므로 typename 키워드를 통해 그게 타입임을 컴파일러에게 알려주어야하는 것이다.

반면에 alias declaration을 이용하면 typename도, ::type도 필요없다.

```C++
template<typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

template<typename T>
class Widget
{
private:
    MyAllocList<T> list; //simple!
};
```

이 경우 MyAllocList는 alias declaration을 이용해 정의된 것이기 때문에 컴파일러는 이게 타입임을 다른 지시자 없이도 확실히 알 수 있다. 따라서 typename을 붙여주지 않아도 정상적으로 동작한다.

##type traits

템플릿 메타 프로그래밍(TMP)에서, 어떤 T 타입에 레퍼런스를 붙이 거나 그걸 const T로 만들거나 하는 등등 타입을 건드려야하는 경우가 왕왕 있다. C++ 11에서는 이런 경우의 편의성을 위해 <type_traits> 헤더 파일에 이와 관련된 타입 변환 작업을 하는 녀석들을 만들어 놓았다.

```C++
std::remove_const<T>::type //const T -> T

std::remove_reference<T>::type // T&,T&& -> T

std::add_lvalue_reference<T>::type // T -> T&
```

척보면 알겠지만 C++11의 type traits는 내부적으로 struct와 typedef를 중첩해서 구현해놓았다. 그래서 템플릿에서 이 놈들을 쓰기 위해선 앞에 typename 키워드를 꼭 붙여줘야한다. 근데 이건 누가봐도 귀찮다. 그래서 C++14에서는 이 부분을 수정해서, alias declaration을 이용해서 구현해놓은 버전을 제공한다.

```C++
std::remove_const<T>::type //C++11
std::remove_const_t<T>     //C++14

std::remove_reference<T>::type //C++11
std::remove_reference_t<T>     //C++14

std::add_lvalue_reference<T>::type // C++11
std::add_lvalue_reference_t<T>     // C++14
```

간단히 말하자면 std::transformation<T>::type 꼴이 std::transformation**_t**<T> 꼴로 바뀌었다고 보면 된다.

만약 C++11에서도 C++14 스타일로 위의 type traits를 쓰고 싶다면 아래와 같은 코드를 쓰면 된다. 사실 굉장히 간단하다.

```C++
template <class T>
using remove_const_t = typename remove_const<T>::type;

template <class T>
using remove_reference_t = typename remove_reference<T>::type;

template <class T>
using add_lvalue_reference_t =
    typename add_lvalue_reference<T>::type;
```