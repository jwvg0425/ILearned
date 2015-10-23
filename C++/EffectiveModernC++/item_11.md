#정의하지 않기보단 deleted function

C++에서는 정의하지않아도 자동으로 생성되는 몇몇 함수들이 있다. 그 대표적인게 생성자 소멸자 시리즈들과 복사에 관련된 녀석들인데, 이 놈들은 정의하지 않아도 자동으로 생성되어버리기 때문에 때로는 골치아픈 문제를 야기한다.

C++98에서는 이런 녀석들을 의도적으로 사용하지 못하게 막고 싶을 때, 이걸 private으로 선언하고 정의는 하지 않는 방법을 많이 택했다. 대표적으로 표준 라이브러리의 stream과 관련된 것들이 있다. 입,출력 스트림과 관련된 클래스를 복사한다는 것 자체가 논리적으로 말이 안 되기 때문에 C++98에서는 이들 클래스의 조상 클래스인 basic_ios를 다음과 같이 정의했다.

```C++
template<class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base
{
private:
    basic_ios(const basic_ios&); //복사 생성자. 정의 안함.
    basic_ios& operator = (const basic_ios&); // 복사 대입 연산자. 정의 안함.
};
```

 그러나 C++11에서는 이런 임기응변같은 방식보다 더 나은 방식인 '= delete'를 쓰는 방법을 제공한다. 말 그대로 '지워진' 함수로 만드는 것이다. 위의 똑같은 클래스가 C++11에서는 아래와 같이 만들어진다.

```C++
template<class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base
{
public: //private->public으로!
    basic_ios(const basic_ios&) = delete;
    basic_ios& operator = (const basic_ios&) = delete;
};
```

 deleted function을 이용하면 일단 링크 타임에 '함수가 정의되지 않았다'라는 메시지를 보는 것보다 의도를 더 명확히 알수 있다는 장점이 있다.

 그리고 deleted function은 public에 선언하는 게 좋다. C++ 컴파일러는 함수가 삭제됐는지(deleted) 아닌지 검사하는 것보다 해당 함수에 대한 접근성(accessibility)을 먼저 검사한다. 즉 삭제가 되어서 호출하지 못하는 함수임에도 불구하고 private에 두면 컴파일 에러 메시지가 'private'이라서 접근 불가능하다는 메시지가 나올 수 있다는 것이다. 그게 private이든 아니든 사용 불가능한 함수인데도 말이다. 즉 public에 둬서 해당 함수가 **삭제되었기 때문에** 사용할 수 없다 라고 하는 편이 해당 클래스의 사용자에게 더 명확하게 느껴질 것이다(링크타임에 '함수가 정의되지 않았다' 라는 말을 보는 것보다 '해당 함수는 삭제되어서 쓸 수 없다'라는 말을 보는 게 더 명확한 것이랑 마찬가지).

##deleted function의 활용

 위에선 꼭 멤버 함수에 대해서만 delete를 쓸 수 있는 것처럼 말했지만, delete 키워드는 사실 어느 함수에서든 쓸 수 있다. 멤버 함수의 경우 위에서 언급한 것같은 경우에 유용하게 쓸 수 있겠지만, 멤버 함수가 아닌 경우에는 어떤 유용함을 갖고 있을까?

```C++
bool isLucky(int number);
```

 위와 같은 멤버 함수가 아닌 함수가 있다고 하자. 이 함수는 int형을 인자로 받지만, 사실 암시적 형변환에 의해 int뿐 아니라 float, double, 심지어는 bool 타입까지 int와 호환되는 모든 종류의 타입을 인자로 넘길 수 있다. 이런 상황에서 의도치 않은 암시적 변환을 막기 위해 delete 키워드를 사용할 수 있다.

```C++
bool isLucky(int number);

bool isLucky(char) = delete;
bool isLucky(bool) = delete;
bool isLucky(double) = delete; // double을 막으면 float도 막힘
                               // float->double이 float->int보다
                               // 우선순위가 높기 때문
```

 delete된 함수들도 오버로딩의 규칙을 따르기 때문에, double이나 bool등 원하지 않는 타입으로 이 함수를 호출할 경우 삭제된 함수를 호출하려는 걸로 간주되어 컴파일 에러를 낸다. 이 방법은 심지어 템플릿에서도 이용할 수 있다. 

```C++
template<typename T>
void processPointer(T* ptr);

template<>
void processPointer<void>(void*) = delete;

template<>
void processPointer<char>(char*) = delete;
```

void*는 역참조가 불가능한 특수한 포인터고, char*는 C에서 문자열을 의미하는 용도로 쓰이는 경우가 많아 이런 경우 템플릿의 구체화를 방지하고 싶을 수 있다. delete 키워드가 template 특수화에 대해서도 동작하기 때문에 이럴 때 delete 키워드를 쓰면 깔끔하게 해결된다. 물론 여기서 좀 더 정확히 하기 위해서는 아래 경우도 delete해 줘야 할 것이다.

```C++
template<>
void processPointer<const void>(const void*) = delete;

template<>
void processPointer<const char>(const char*) = delete;
```

좀 더 정확히 하려면 const volatie char* 라든지 std::wchar_t, std::char16_t, std::char32_t 같은 타입에 대해서도 다 방지해줘야겠지만 그건 너무 깊이 들어가는 것같으니 생략. 필요하다면 해주는 게 맞을 것 같다.

C++ 98에서는 클래스의 템플릿 멤버 함수를 일부만 쓰지 못하게 할 방법이 없었다. C++ 98에서는 private 선언 후 정의하지 않는 방법을 써야만 이런게 가능한데, 같은 함수를 어떤 것만 private 어떤것만 public으로 정의할 수는 없기 때문이다. 그러니까 다음과 같은 구문이 불가능하다.

```C++
class Widget
{
public:
    template<typename T>
    void processPointer(T* ptr)
    { ... }

private:
    template<>
    void processPointer<void>(void*); //error!
};
```

당연히 deleted function은 이런 문제가 없다.

```C++
class Widget
{
public:
    template<typename T>
    void processPointer(T* ptr)
    { ... }

    tepmlate<>
    void processPointer<void>(void*) = delete;
};
```

어떤 면으로 보나 C++ 98에서 이용하던 private 선언 후 정의 안하기 보다는 deleted function이 좋다. deleted function을 적극적으로 활용하자.