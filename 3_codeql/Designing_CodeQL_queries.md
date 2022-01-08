# Designing_CodeQL_queries.md
# 목차
1. [Intro](#1-intro)   
2. [Iterator Invalidation](#2-iterator-invalidation)   
2.1 [C++ STL](#21-c-stl)   
2.2 [Iterator Invalidation으로 인한 UAF](#22-iterator-invalidation으로-인한-uaf)   
3. [CodeQL](#3-codeql)   
3.1 [Data Flow](#31-data-flow)   
3.2 [Recursion](#32-recursion)   
4. [Itergator 소개](#4-itergator-소개)   
5. [Designing](#5-designing)   
6. [Conclusion](#6-conclusion)   
7. [Reference](#7-reference)   

# 1. Intro


우리가 Chrome 브라우저에서 CodeQL을 이용하여 검색하여 찾고자 했던 취약점은 C++의 Iterator Invalidation이다. 그리고 그 중에서도 연속 컨테이너인 vector에서 발생하는 것을 노려보고자 했다. 쿼리 작성에 앞서, Iterator Invalidation 정의를 먼저 언급한 후, 쿼리 작성에 참고했던 itergator 라이브러리 소개 및 쿼리 결과물을 공유한다.

# 2. Iterator Invalidation


## 2.1 C++ STL


C++ STL(Standard Template Library)는 Contatiner, Iterator, Algorithm, Functor를 제공한다.

### Contatiner : Vector

C++의 컨테이너는 연속 컨테이너와 연관 컨테이너로 분류된다. 그 중에서도 연속 컨테이너인 vector에 집중한다. 

vector는 현재 가지고 있는 데이터의 개수보다 더 많은 공간을 할당하고 있는데, 이를 capacity라고 한다. vector에 데이터를 insert하여 capacity를 다 채우게되면, 데이터를 복사해서 다른 메모리 공간에 할당을 해야한다. 취약점을 찾기 위해서 다른 메모리 공간에 할당이 될 때 집중해야 한다.

### Iterator

iterator는 c++에서 컨테이너의 데이터를 순회하는 표준 방법이다. iterator 객체는 최소한 두 가지 작업을 지원한다.

1. 컨테이너의 데이터를 가져오기 위한 역참조
2. 다음 데이터에 대한 iterator를 얻기 위한 증가

```cpp
std::vector<int> vec{1, 2, 3, 4, 5};
 
for (std::vector<int>::iterator it = vec.begin(), end = vec.end(); it != end; ++it) {
    std::cout << *it << " ";
}
```

C++11에서는 아래와 같이 작성할 수 있다. 이전 코드와 동일하지만 모든 Iterator 연산은 컴파일러에서 생성되고, 반복에 대한 세부 정보는 개발자에게 숨겨져 있다.

```cpp
for (auto i : vec) {
    std::cout << i << " ";
}
```

## 2.2 Iterator Invalidation으로 인한 UAF


### CVE-2020-6551

```cpp
void XRSystem::FocusedFrameChanged() {
  // Tell all sessions that focus changed.
  for (const auto& session : sessions_) {
    session->OnFocusChanged(); // [1]
  }
	...
}
```

```cpp
void XRSession::UpdateVisibilityState() {
  ...
    DispatchEvent( // [2]
        *XRSessionEvent::Create(event_type_names::kVisibilitychange, this));
  }
}
```

`XRSystem::FocusedFrameChanged` 함수에서 iterator인 `session`을 얻어와 `OnFocusChanged` 함수를 호출하는데[1], 그 안에서 JS callback을 호출한다. 그 안에서 동기적으로 `XRSessionEvent::Create` 함수를 호출하기 위해 `DispatchEvent` 함수를 호출한다.[2] 

```cpp
XRSessionEvent::XRSessionEvent(const AtomicString& type,
                               const XRSessionEventInit* initializer)
    : Event(type, initializer) {
  if (initializer->hasSession())
    session_ = initializer->session(); // [3]
}
```

이렇게 호출된 함수에서 `sessions_`에 새로운 element를 추가하게 되는데[3], 이것은 `XRSystem::FocusedFrameChanged` 함수의 range-based 루프에서 사용되는 iterator를 invalid하게 만든다. 그 다음 loop에서 iterator는 증가해서 다음 element를 가리키게 되고, UAF가 발생한다.

### [CVE-2020-6449](./../1_vulnerability_analysis_whitepapers/cve-2020-6449/README.md)

```cpp
void DeferredTaskHandler::BreakConnections() {
	...
	for (auto* finished : finished_source_handlers_) {
      // Break connection first and then remove from the list because that can
      // cause the handler to be deleted.
      finished->BreakConnectionWithLock();     //<-- `active_source_handlers_` may have been cleared, and finished is already freed
      active_source_handlers_.erase(finished); //<-- assumes `active_source_handlers_` contains finished
    }
	...
}
```

`active_source_handlers_` 는 `DeferredTaskHandler::ClearHandlersToBeDeleted` 메서드에 의해 지워질 수 있는 반면, `finished_source_handlers_` 는 동시에 지워지지 않아 `finished_source_handlers_`에 댕글링 포인터가 남게 되어 `DeferredTaskHandler::BreakConnections` 메서드에서 UAF가 발생한다.

# 3. CodeQL


CodeQL 개념, 사용 방법 등은 [Overview.md](./Overview.md)에서 언급한다. 여기서는 공개된 Trail of Bits의 [itergator](https://github.com/trailofbits/itergator) 라이브러리 분석을 위한 Data Flow와 recursion 대해 간단히 개념을 짚고자 한다.

## 3.1 Data Flow


Data Flow 분석은 변수가 프로그램의 다양한 지점에서 보유할 수 있는 가능한 값을 계산하여 이러한 값이 프로그램을 통해 전파되는 방식과 사용되는 위치를 결정한다. CodeQL에서는 Local Data Flow와 Global Data Flow를 모두 모델링할 수 있다.

1. Local Data flow : 하나의 함수에서의 data flow를 계산한다.
2. Global Data flow : 프로그램 전체에서의 data flow를 계산한다.

Global Data flow는 프로그램 전체, 즉 scope가 없기 때문에 다음과 같은 `DataFlow::Configuration` 클래스를 확장하여 사용된다.

```cpp
import semmle.code.cpp.dataflow.DataFlow

class MyDataFlowConfiguration extends DataFlow::Configuration {
  MyDataFlowConfiguration() { this = "MyDataFlowConfiguration" }

  override predicate isSource(DataFlow::Node source) {
    ...
  }

  override predicate isSink(DataFlow::Node sink) {
    ...
  }
}
```

- `isSource` : data flow가 시작되는 지점을 정의
- `isSink` : data flow가 끝나는 지점을 정의
- `isBarrier` : 특정 조건에서 data flow를 제한
- `isBarrierGuard` : 특정 조건에서 data flow를 제한
- `isAdditionalFlowStep` : 추가적인 data flow step을 정의

predicate `MyDataFlowConfiguration()` 은 configuration의 이름을 정의하기 때문에 `MyDataFlowConfiguration` 클래스 이름으로 대체되어서 사용되어야 한다. predicate `hasFlow(DataFlow::Node source, DataFlow::Node sink)`을 이용하여 분석이 가능하다.

```cpp
from MyDataFlowConfiguration dataflow, DataFlow::Node source, DataFlow::Node sink
where dataflow.hasFlow(source, sink)
select source, "Data flow to $@.", sink, sink.toString()
```

CodeQL의 Data Flow에 대한 자세한 내용은 [여기](https://codeql.github.com/docs/writing-codeql-queries/about-data-flow-analysis/)에서 확인할 수 있다.

## 3.2 Recursion


CodeQL은 recursion(재귀)를 강력하게 지원한다. QL의 predicate는 직접 또는 간접적으로 그 자체에 의존하는 경우 재귀적이라고 한다.

### Transitive closures

predicate의 transitive closure은 기존의 predicate를 반복적으로 적용하여 결과를 얻는 재귀 predicate이다. 기존의 predicate에는 두 개의 인수(this 또는 result 값)가 있어야 하고, 해당 인수에는 호환 가능한 타입이 있어야한다. 

transitive closure는 일반적인 재귀 형식인데, QL에는 두 가지 유용한 약어가 있다.

#### Transitive closure `+`

predicate를 한 번 이상 적용하려면 predicate 이름에 `+` 를 추가한다.

> 예를 들어, 멤버 predicate `getAParent()`가 있는 Person 클래스가 있다고 가정한다. `p.getAParent()`는 p의 부모를 반환한다. transitive closures `p.getAParent+()`는 p의 부모, p의 부모의 부모 등을 반환한다.
> 

#### Reflexive transitive closure `*`

predicate를 자신에게 0번 이상 적용하는데 사용할 수 있다는 것을 제외하면 위의 transitive closure 연산자와 유사하다.

> 예를 들어, p.getAParent*()의 결과는 p의 부모, p의 부모의 부모 ... 또는 p 자체이다.
> 

CodeQL의 Recursion에 대한 자세한 내용은 [여기](https://codeql.github.com/docs/ql-language-reference/recursion/)에서 확인할 수 있다.

# 4. Itergator 소개


- [깃허브 주소](https://github.com/trailofbits/itergator)

Itergator는 C++ 코드 베이스에서 iterator invalidation을 감지하고 분석하기 위한 CodeQL 라이브러리이다. 이것은 실제 코드에서 버그를 찾는데 사용이 되었다.

CodeQL에서 제공하는 Global Data Flow 라이브러리와 recursion을 이용하여 쿼리를 쉽게 작성할 수 있다. Itergator는 잠재적으로 invalidation되는 모든 함수 호출(`std::vector::push_back`과 같은 invalidation 함수에 대한 호출을 초래할 수 있는 호출)의 그래프를 구성하고 쿼리에 사용할 클래스를 정의할 수 있다.

- `Iterator`: a variable that stores an iterator
- `Iterated`: where a collection is iterated, e.g. vec in vec.begin()
- `Invalidator`: a potentially invalidating function call in the scope of an iterator
- `Invalidation`: a function call that directly invalidates an iterator

# 5. Designing


분석했던 CVE 중 CVE-2020-6551를 깊게 참고하여 아래와 같이 분석하고자하는 부분을 itergator 라이브러리를 이용하여 검색하고자 했다.

> 1.  for문 조건식에서 std::vector 타입 변수의 iterator를 사용하는 경우
> 2.  for문 body에서 iterator가 On.*\(\) 함수를 호출하는 경우
> 3.  DispatchEvent를 이용하여 자바스크립트 콜백을 호출하는 경우
> 4.  호출한 자바스크립트 콜백이 vector의 요소를 추가하거나 삭제하는 함수일 경우
> 

아쉽게도 세 번재, 네 번째 조건은 쿼리로 작성하는데에 어려움이 있어 나머지 조건만 충족하여 쿼리를 작성하였다. 작성한 쿼리는 공개를 하지 않을 예정이다.

검색 결과로 총 159개가 나왔고, 세 번째와 네 번째 조건을 충족하는 지 확인하기 위해 159개 모두 분석을 진행하였으나 취약점으로 의심되는 부분은 찾아보기 힘들었다. 

# 6. Conclusion


결과적으로 CodeQL을 이용하여 취약점 찾기에 실패했지만, CodeQL을 이용하여 어떤 것을 하고자 한 우리의 시행착오와 공부한 것들을 공유하는데에 의의를 둔다. 

# 7. Reference


[https://github.com/trailofbits/itergator](https://github.com/trailofbits/itergator)

[https://blog.trailofbits.com/2020/10/09/detecting-iterator-invalidation-with-codeql/](https://blog.trailofbits.com/2020/10/09/detecting-iterator-invalidation-with-codeql/)

[https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-cpp/](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-cpp/)

[https://codeql.github.com/docs/ql-language-reference/recursion/](https://codeql.github.com/docs/ql-language-reference/recursion/)
