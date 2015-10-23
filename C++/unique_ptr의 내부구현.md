# 최대한 간략하게

unique_ptr 내부에 굉장히 다양한 기능들이 들어가 있어서 한번에 그걸 죄다 살펴보려고 하면 굉장히 힘들다. 그래서 unique_ptr 클래스에서 내가 이해한 부분만 간추려서 새롭게 간략한 버전의 unique_ptr을 한 번 만들어보았다.

아래의 설명은 내가 만든 간략한 버전의 unique_ptr(UniquePtr)을 기준으로 한 설명이다. 기초 구조는 완전히 똑같은 상황에서 내가 이해한 부분에 관한 내용만 동일한 방식으로 구현 및 설명하였으니 참고 바람.

## 구조

UniquePtr은 아래와 같은 계층 구조로 이루어져 있다.

```C++
template<class Resource, class Deleter, bool isZero>
class UniquePtrBase 
{ ... }

template<class Resource, class Deleter>
class UniquePtr
 : private UniquePtrBase<Resource, Deleter, std::is_empty<Deleter>::value>
{ ... }
```

딱 봐도 복잡하다. UniquePtrBase는 Resource, Deleter, isZero라는 3가지 값을 받는데, 그 3가지 각각의 의미는 다음과 같다.

 - **Resource** : 관리하고자 하는 자원의 타입이다. int, char, 기타 등등 여러가지가 있을 수 있다.
 - **Deleter** : 관리하는 자원을 파괴할 때 사용할 함수 객체이다. 원래 unique_ptr은 기본 deleter가 존재하지만 그 부분 구현이 나한테는 너무 복잡해서 생략했다. 그래서 일단 무조건 deleter를 넘겨줘야만 하게 되어 있다.
 - **isZero** : 넘겨 받은 Deleter의 크기가 0인지 아닌지 결정한다. 이 부분이 중요. 이 인자를 이용해 부분 특수화를 함으로써 stateless lambda를 이용할 경우 unique ptr의 크기가 커지지 않도록 만들었다.

사실상 중요한 구현은 UniquePtrBase 내부에 있고 UniquePtr은 그걸 상속받아서 이용하는 방식으로 이루어져 있다. 이 때 UniquePtr이 상속받는 부분을 잘 보면 마지막 isZero 인자를 ```std::is_empty<Deleter>::value```를 이용해 정함을 알 수 있다. ```std::is_empty<T>```는 해당 타입의 크기가 0인지 아닌지 판단하며 0일 경우 ```std::is_empty<T>::value```값이 true, 아닐 경우 false로 정해진다.


### UniquePtrBase

```C++
template<class Resource, class Deleter, bool isZero>
class UniquePtrBase
{
public:
	using Deleter_noRef = 
		typename std::remove_reference<Deleter>::type;
	using Pointer = Resource*;

	UniquePtrBase(Pointer resource_, Deleter deleter_)
		: resource(resource_), deleter(deleter_)
	{	
	}

	//호환되는 유사 타입에 대한 처리
	template<class Pointer2, class Deleter2>
	UniquePtrBase(Pointer2 resource_, Deleter2 deleter_)
		: resource(resource_), deleter(deleter_)
	{
	}

	Deleter_noRef& getDeleter()
	{
		return (deleter);
	}

	const Deleter_noRef& getDeleter() const
	{
		return (deleter);
	}

	Pointer resource;
	Deleter deleter;
};
```

일단 구현 코드는 위와 같다. 위 코드는 Deleter의 크기가 0이 아닌 경우의 구현이다. 이 경우는 사실 굉장히 간단해서 큰 설명이 필요 없다. 관리할 resource와 deleter를 인자로 받아 저장해둔 다음 getDeleter함수를 통해 자신이 저장하고 있는 deleter를 돌려주도록 되어 있다. 그런데 이 부분에서 솔직히 잘 이해가 안되는게, 그냥 Deleter를 리턴해도 될 것 같은데 레퍼런스를 제거한 타입인 Deleter_noRef에다가 다시 &를 붙여 참조형으로 돌려주고 있다. 혹시나 싶어서 그냥 Deleter를 리턴하게 해봤는데 그래도 잘 동작하더라.

```C++
template<class Resource, class Deleter>
class UniquePtrBase<Resource, Deleter, true> : public Deleter
{
public:
	using MyBase = Deleter;
	using Deleter_noRef = 
		typename std::remove_reference<Deleter>::type;
	using Pointer = Resource*;

	UniquePtrBase(Pointer resource_, Deleter deleter_)
		: resource(resource_), MyBase(deleter_)
	{
	}

	//호환되는 유사 타입에 대한 처리
	template<class Pointer2, class Deleter2>
	UniquePtrBase(Pointer2 resource_, Deleter2 deleter_)
		: resource(resource_), MyBase(deleter_)
	{	
	}

	Deleter_noRef& getDeleter()
	{
		return (*this);
	}

	const Deleter_noRef& getDeleter() const
	{
		return (*this);
	}

	Pointer resource;
};
```

이번엔 isZero가 true인 경우다. false랑은 좀 다르다. 이걸 어떻게 구현했지 하고 생각했었는데 보면 ```: public Deleter```를 통해 Deleter를 상속받고 있는 것을 알 수 있다. 저장하는 state가 하나도 없으면 size가 0이므로 상속받은 후 새로운 멤버로 관리할 포인터를 더함으로써 그대로 4바이트를 유지하는 것이다. 그래서 따로 Deleter를 저장하지 않고 초기화 리스트에서 Deleter의 생성자를 호출하는 것을 알 수 있다. 람다를 상속받았을 때 저런식으로 생성자 호출해서 람다 부분을 초기화할 수 있다는 것도 처음 알았는데 굉장히 신기하게 느껴졌다. 어쨌든 그래서 이 경우 자기자신이 deleter가 되므로, 리턴할 때 (*this)를 돌려준다.

###UniquePtr

난 역참조 연산(*)만 넣었다. 실제로는 UniquePtrBase나 UniquePtr이나 넘어온 타입이 T타입인지 T[]인지, 그리고 사용하는 Deleter가 사이즈가 0인지 아닌지, 기본 제공 Deleter인지 등등에 따라 다양한 처리를 하고 있다. 근데 그 걸 다 다루려니 너무 복잡하고 잘 이해가 안 가는 부분이 많아 가장 단순하고 기본적인 케이스 및 기능만 구현했다. 그래서 내가 만든 것에서는 Deleter도 일일히 넘겨줘야하고 배열(T[])도 사용할 수 없다. (소유권 이동 등등의 기능도 없음)

```C++
template<class Resource, class Deleter>
class UniquePtr 
: private UniquePtrBase<Resource, Deleter, std::is_empty<Deleter>::value>

public:
	using MyBase 
		= UniquePtrBase<Resource, Deleter, std::is_empty<Deleter>::value>;
	using Pointer = MyBase::Pointer;
	using MyBase::getDeleter;

	UniquePtr(Pointer resource_,
		typename std::_If<std::is_reference<Deleter>::value, Deleter,
		const typename std::remove_reference<Deleter>::type&>::type deleter_)
		: MyBase(resource_, deleter_)
	{
	}

	~UniquePtr()
	{
		this->getDeleter()(this->resource);
	}

	typename std::add_reference<Resource>::type operator*() const
	{
		return (*this->resource);
	}

	UniquePtr(const UniquePtr& rhs) = delete;
	UniquePtr& operator=(const UniquePtr& rhs) = delete;
```

다른 건 사실 그렇게 복잡하지 않은데, 생성자의 ```typename std::_If<std::is_reference<Deleter>::value, Deleter, const typename std::remove_reference<Deleter>::type&>::type``` 이 부분이 굉장히 복잡하게 느껴질 수 있다. 사실 나도 저게 무슨 뜻인지는 알겠는데 왜 저렇게 쓰는 지는 모르겠다. 일단 하나하나 살펴보자.

```std::_If<bool, _Ty1, _Ty2>::type```는 그냥 if문이랑 똑같다고 생각하면 된다. 첫번째 인자 bool 값이 true면 type이 _Ty1이 되고, false면 _Ty2가 된다. ```std::is_reference<T>::value```는 T가 reference 타입이면 value 값이 true, 아니면 false가 된다. 즉, 저 코드는 만약 Deleter 타입이 레퍼런스 타입이면 그냥 Deleter 타입을, 그렇지 않다면 ```const std::remove_reference<Deleter>::type&``` 타입을 취하겠다는 것이다. 근데 std::is_reference가 false면 reference 타입이 아니라는 뜻 아닌가? 왜 다시 remove_reference를 하는 지 모르겠다. 내가 잘 이해하지 못해서 생략한 부분들과 관련된 내용인 듯도 하고. 심오한 TMP의 세계.. 정확히 이해하게 되면 다시 내용을 수정, 보완해야겠다. 일단은 이 정도로만 알고 넘어가야지.

아무튼 이런식으로 초기화만 하고 나면 간단하다. 파괴될때는 UniquePtrBase로부터 getDeleter함수를 호출해서 자원을 해제하고, 역참조 연산의 경우에도 간단히 ```return (*this->resource);```를 수행하는 것으로 해결된다.

 결국 핵심은 UniquePtrBase에서 Deleter의 크기가 0이냐 아니냐에 따라 템플릿 부분 특수화를 이용해 uniquePtr의 크기를 조정해주는 부분인 듯 하다. 이 부분 뭔가 함수형 프로그래밍에서 패턴 매칭 이용하는 거랑 비슷한 느낌이 들어서 재밌다. 나머지 잘 이해 안 가는 부분은 템플릿에 대해 좀 더 심도있게 공부한 다음 보완할 예정..

### 적용 코드

그래서 아래는 위 코드를 기반으로 짠 예시 코드.

```C++
int main()
{
	int state = 1;
	auto del = [](int* pa)
	{
		printf("stateless deleter.\n");
		delete pa;
	};

	auto del2 = [&state](int* pa)
	{
		printf("deleter. state = %d.\n", state);
		delete pa;
	};

	int* a = new int(3);
	int* b = new int(5);

	UniquePtr<int, decltype(del)> pa(a, del);
	UniquePtr<int, decltype(del2)> pb(b, del2);

	printf("sizeof (pa) = %d, sizeof (pb) = %d \n", sizeof(pa), sizeof(pb));

	printf("값 참조도 정상 동작. a = %d, b = %d\n", *pa, *pb);
}
```

수행결과는 아래와 같이 나온다.

>sizeof(pa)=4, sizeof(pb)=8  
값 참조도 정상 동작. a = 3, b = 5  
deleter. state = 1.  
stateless deleter.  