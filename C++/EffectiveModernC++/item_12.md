#오버라이딩엔 override

C++에서 오버라이딩은 굉장히 흔히 쓰이는 개념이다. 상속받은 클래스가 기반 클래스의 함수를 재정의하는 것을 오버라이딩이라고 하는데, 오버라이딩이 일어나기 위해선 몇 가지 조건이 필요하다.

- 기반 클래스의 함수가 가상 함수(virtual function)이어야만 한다.
- 기반 함수와 상속된 함수(derived function)의 이름이 반드시 같아야한다(소멸자 제외).
- 기반 함수와 상속된 함수의 매개변수 타입도 반드시 같아야한다.
- 기반 함수와 상속된 함수의 상수성(constness)도 반드시 같아야한다.
- 기반 함수와 상속된 함수의 리턴 타입 및 예외 지정(exception specification)이 서로 호환되어야만 한다.

 굉장히 당연해보이는 조건들이다. 여기에 C++ 11에서는 한 가지 조건이 더 추가 되었다.

- 함수의 *레퍼런스 지정자(reference qualifiers)*도 똑같아야한다. 

## 레퍼런스 지정자(reference qualifiers)?

 레퍼런스 지정자가 뭔지 모르겠으니 레퍼런스 지정자부터 짚고 넘어가자. 이 역시도 C++11에서 lvalue, rvalue와 관련해서 추가된 개념이다. 해당 인스턴스가 lvalue일 때만, 혹은 rvalue일 때만 호출할 수 있는 멤버 함수를 선언할 때 이 레퍼런스 지정자를 이용한다.

```C++
class Widget
{
public:
    void doWork() &;  // *this가 lvalue일 때만 호출 가능하다

    void doWork() &&; // *this가 rvalue일 때만 호출 가능하다
};

Widget makeWidget();

Widget w;

w.doWork(); // w는 lvalue이므로 doWork() &가 호출.

makeWidget().doWork(); //makeWidget()의 리턴값은 rvalue이므로 doWork() &&가 호출.
```

 그리 복잡한 개념은 아니니 아마 이해하기 어렵진 않을 것이다. 글의 후반부에서 이 개념을 다시 한 번 상세하게 짚고 넘어갈 것이다.

## override

 어쨌건 오버라이딩의 이런 규칙들 때문에 자칫하면 미묘한 코딩 실수가 전혀 의도하지 않은 결과를 가져올 수 있다. 

```C++
class Base
{
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};

class Derived : public Base
{
public:
    virtual void mf1();
    virtual void mf2(unsigned int x);
    virtual void mf3() &&;
    void mf4() const;
};
```

얼핏 보기엔 Derived의 4개 함수가 제대로 오버라이딩이 될 것 같지만 실제로는 그렇지 않다. Derived의 4개 함수 전부 실제로는 오버라이딩이 되지 않은 상황이다. 하나씩 살펴보자.

- mf1은 Base에서 const였는데, Derived에서는 그렇지 않다. 즉 **상수성**이 다르다.
- mf2는 Base에서는 int를 인자로 받는데, Derived에서는 unsigend int를 받는다. 즉 **매개변수의 타입**이 다르다.
- mf3은 Base에서 lvalue로 지정되어있는데, Derived에서는 rvalue로 지정되어있다. 즉 **레퍼런스 지정자**가 다르다.
- mf4는 Base에서 **virtual**로 지정되어 있지 않다.

 규칙이 생각보다 세세하기 때문에 충분히 실수할 수 있고, 그 실수한 결과를 잘 알기 힘든 경우도 있다. 그래서 C++11에서는 오버라이딩하고 싶은 함수에 해당 함수가 오버라이딩되었음을 명시할 수 있는 override 키워드를 제공한다.

```C++
class Derived : public Base
{
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    void mf4() override;
};
```

아까 전 Derived 클래스에 override 키워드를 줘서 해당 함수들이 재정의되었음을 명시해주었다. 이 경우 **컴파일이 안 된다**. 오버라이딩이 목적임을 명확히했기 때문에 컴파일러가 위에서 명시한 오버라이딩에 필요한 규칙들을 체크하기 때문이다. 따라서 오버라이딩하고 싶은 함수에 override 키워드를 써 주면 의도치 않은 동작을 방지할 수 있다.

 따라서 아래와 같이 작성하는게 좋은 코드가 될 것이다.

```C++
class Base
{
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
};

class Derived : public Base
{
public:
    //오버라이딩된 함수에는 굳이 virtual을 쓸 필요는 없음.
    //물론 쓰고 싶으면 써도 된다.
    virtual void mf1() const override;
    virtual void mf2(int x) override;
    virtual void mf3() & override;
    void mf4() const override; // 상관없다!
};
```

 override 키워드를 씀으로써 얻을 수 있는 또다른 장점은, 기반 클래스의 변화에 유연하게 대처할 수 있다는 것이다. 만약 override 키워드를 안 쓴 상태에서 기반 함수의 리턴 타입이나 매개변수 타입 등이 바뀌었다고 생각해보자. 컴파일러는 어떤 에러도 내지 않고 컴파일을 할 것이다(경고는 낼지 몰라도). 이 경우 의도치 않은 동작이 발생함에도 불구하고 프로그래머가 그 사실을 알아차리기 힘들 수 있다. 반면에 override 키워드를 썼을 경우 기반 함수의 리턴 타입 등이 바뀌면 바로 컴파일 에러를 일으킨다. 따라서 프로그램에 문제가 생긴걸 런타임 이전에 확인 가능하므로 훨씬 빠르고 유연하게 대처할 수 있다.

> contextual keyword

C++ 11에는 override와 final(특정 함수를 더 이상 오버라이딩 못 하게 막거나, 특정 클래스를 더 이상 상속받지 못하게 만들고 싶을 때 사용하는 키워드)이라는 두 개의 contextual keyword를 제공한다. 이 contextual keyword들은 기존 코드들과의 호환성을 위해 특정한 상황에서만 키워드로 동작을 한다.

```C++
class Warning
{
public:
    void override(); //이런 게 가능하다!
};
```

override가 키워드긴 하지만 멤버 함수의 맨 뒤라는 위치에서만 키워드로 동작을 하는 것이다. 어느 상황에서나 키워드로 동작해버리면 기존에 override, final 등을 이름으로 사용하고 있던 코드에서 문제를 일으킬 수 있기 때문이다. 

## 레퍼런스 지정자(reference qualifiers)!

 위에서 레퍼런스 지정자가 어떤 개념인지는 이미 한 번 살펴보았다. 그런데 도대체 이 놈을 어디다 써 먹는 거지 싶을 수 있다. 실제로도 이걸 쓰는 상황이 그렇게 흔하진 않긴 하지만, 그래도 간혹 유용하게 쓰이는 경우가 있다. 예를 통해 살펴보자.

```C++
class Widget
{
public:
    using DataType = std::vector<double>;

    DataType& data() { return values; }

private:
    DataType values;
};

Widget w;

auto vals1 = w.data();
```

이 때 w.data가 리턴하는 값(std::vector<double>은 lvalue이기 때문에 vals1에 복사가 된다(auto -> 값 복사). 이 경우는 원래 w 자체가 lvalue니까 사실 이게 정상적이다. 하지만 다음과 같은 경우는 어떨까?

```C++
Widget makeWidget();

auto vals2 = makeWidget().data();
```

이 경우 makeWidget의 리턴값은 rvalue임에도 불구하고 data 함수의 리턴값이 lvalue이기 때문에 vals2를 초기화할 때 복사 생성자가 호출되어버린다. 그런데 makeWidget의 내부 값은 어차피 저 초기화를 끝내고 나면 사라져버릴 값인데 이 상황에서 복사 생성자가 호출되는게 좋을까? 가능하다면 rvalue를 리턴받아서 이동 생성자(move constructor)가 호출되는 편이 성능 상에 이득이 클 것이다(특히나 vector의 사이즈가 클 수록). 이럴 때 멤버 함수에 레퍼런스 지정자를 써주면 된다.

```C++
class Widget
{
public:
    using DataType = std::vector<double>;

    DataType& data() &
    {
        return values;
    }

    //rvalue이기 때문에 리턴 타입이 DataType&이 아니라 DataType이다.
    DataType data() &&
    {
        return std::move(values);
    }

private:
    DataType values;
};

Widget w;

Widget makeWidget(); // factory 함수

auto vals1 = w.data(); // 복사 생성자 호출

auto vals2 = makeWidget().data(); // 이동 생성자 호출
```

