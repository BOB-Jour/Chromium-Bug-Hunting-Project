# WebIDL로_파이어폭스_퍼징하기.md
- [TL;DR, 들어가는 글](#tldr-들어가는-글)   
- [퍼징 기초](#퍼징-기초)   
  - [Fuzzer의 유형](#fuzzer의-유형)   
  - [블랙박스 퍼징](#블랙박스-퍼징)   
  - [화이트박스 퍼징](#화이트박스-퍼징)   
  - [그레이박스 퍼징](#그레이박스-퍼징)   
  - [문법 기반 퍼징(Grammar-Based Fuzzing)](#문법-기반-퍼징grammar-based-fuzzing)   
  - [기존 문법들의 문제점](#기존-문법들의-문제점)   
- [퍼징 문법으로서의 WebIDL](#퍼징-문법으로서의-webidl)   
  - [표준화된 문법](#표준화된-문법)   
  - [단순화된 문법 개발](#단순화된-문법-개발)   
  - [ECMAScript Extended Attributes](#ecmascript-extended-attributes)   
  - [자동으로 API 변경 여부 감지하기](#자동으로-api-변경-여부-감지하기)   
- [IDL을 JavaScript로 변환하기](#idl을-javascript로-변환하기)   
  - [제네릭 타입](#제네릭-타입)   
  - [객체 참조](#객체-참조)   
  - [콜백 핸들러](#콜백-핸들러)   
- [무설정(無設定) 퍼징](#무설정無設定-퍼징)   
- [GrIDL을 사용한 룰 패치(Rule Patching)](#gridl을-사용한-룰-패치rule-patching)   
- [실례](#실례)   
- [Contributing](#contributing)   
- [Jason Kratzer에 대해:](#jason-kratzer에-대해)   
---
원본 주소: [https://hacks.mozilla.org/2020/04/fuzzing-with-webidl/](https://hacks.mozilla.org/2020/04/fuzzing-with-webidl/)

위 주소의 글을 번역한 게시글입니다.

---

## **TL;DR, 들어가는 글**

퍼징(Fuzz Testing이라고도 함)이란 소프트웨어의 안전성과 안정성을 자동으로 테스트하는 방식 전반을 일컫습니다. 소프트웨어 내부의 예상치 못한 동작, 더 나아가서는 위험한 동작을 알아내기 위해 특수하게 조작해 둔 입력값을 넣어주는 것이 일반적인 퍼징 방식이지요. 

만일 퍼징에 익숙지 않으신 분이 있다면 [Firefox Fuzzing Docs](https://firefox-source-docs.mozilla.org/tools/fuzzing/index.html) 및 [Fuzzing Book](https://www.fuzzingbook.org/) 을 참조해 주세요.

지난 3년 간 Firefox 퍼징 팀은 Firefox에서 WebAPI를 사용할 때 발생하는 보안 취약점을 찾아내기 위해서 새로운 퍼저 개발에 매달려 왔습니다. 현재는 개발이 완료되어 Domino라고 명명했는데요, 이 퍼저에서는 Fuzzing Grammar로 WebAPI에 대하여 WebIDL로써 정의해 둔 것을 사용한다는 특징이 있습니다. 

저희가 만든 Domino, 즉 저희만의 고유한 접근 방식이 들어간 이 퍼저를 사용해 [850](https://bugzilla.mozilla.org/show_bug.cgi?id=1340565) 개 [이상의 버그를](https://bugzilla.mozilla.org/show_bug.cgi?id=1340565) 찾아낼 수 있었습니다. 이러한 버그 중 116개는 Security Rating을 받았지요.

이 글을 빌어 1) Domino의 몇몇 주요 기능들과 2)어째서 Domino가 이전까지의 WebAPI 퍼징 시도들과 다른지에 대해 설명하고자 합니다.

## **퍼징 기초**

Domino에 대해 자세히 파고 들어가기 전에, 먼저 현존하는 퍼징 기술의 유형에 대해 서술하겠습니다.

### **Fuzzer의 유형**

Fuzzer는 일반적으로 블랙박스, 그레이박스 또는 화이트박스 퍼저로 분류됩니다. 이 분류의 기준은 퍼저가 대상 애플리케이션에 대해 얼마나 잘 알고, 얼마나 맞춤화된 정보를 넣어주느냐인데요*, 가장 널리 쓰이는 두 가지 유형은 블랙박스와 그레이박스 퍼저입니다.

(* 역주: 원문은 ‘These designations are based upon the level of communication between the fuzzer and the target application.’이지만 의역을 거침.)

### **블랙박스 퍼징**

블랙박스 퍼징은 인풋으로 주고자 하는 데이터가 대상 애플리케이션에 어떤 영향을 미치는지에 대해 아무것도 모르는 채로 데이터를 넣어주는 것을 의미합니다. 이런 전제 때문에 블랙박스 퍼저의 효율성은 생성된 데이터가 얼마나 운좋게 애플리케이션의 구조에 걸맞는지에 전적으로 의지하게 됩니다.

하지만 이런 점이 오히려 장점이 될 수 있죠. 블랙박스 퍼징은 대규모의 비결정적 애플리케이션*이나, 고도로 구조화된 데이터를 처리하는 애플리케이션을 퍼징할 때 유용합니다.

(* 역주: 동일한 인풋인데도 다른 아웃풋이 나올 수 있는 프로그램)

### **화이트박스 퍼징**

화이트박스 퍼징은 퍼저와 대상 응용 프로그램이 서로의 코드에 대해 확실하게 알고 있는 상황에서, 퍼저가 응용 프로그램의 요구 사항을 정확히 충족하는 데이터를 생성하여 인풋으로 주는 방식을 일컫습니다. 

이 퍼징 방식에서는 응용 프로그램 코드 실행 시의 분기 조건을 체크해서, 데이터가 들어갔을 때 코드의 모든 분기를 실행하도록 모종의 이론적인 도구를 사용하는데요, 이를 통해 블랙박스 혹은 그레이박스 퍼저였다면 돌려 보지 못했을 분기를 테스트할 수 있다는 장점이 있습니다.

하지만 단점 또한 존재합니다. 바로 계산 비용이 많이 든다는 것인데요, 굉장히 복잡한 코드 분기점들을 가진 아주 무거운 프로그램을 테스트할 경우 정말 많은 시간이 걸릴 수 있기 때문입니다. 이렇게 되면 정해진 시간 안에 테스트할 수 있는 인풋 데이터의 수 또한 확연히 줄어들죠. 학문적으로 연구하는 경우를 제외하고, 화이트박스 퍼징은 실생활에서 사용하기엔 그리 적합하지 않습니다.

### **그레이박스 퍼징**

그레이박스 퍼징은 최근 들어서 가장 인기 있고 효과적인 퍼징 기법으로 인정받고 있죠. 현재 퍼징 상황을 확인해 퍼저에게 알려 주는 피드백 메커니즘을 이용해 이 다음에는 어떤 데이터를 만들 것인지 결정하는 식인데요, 더 많은 코드 분기를 이용하는 듯한* 인풋 데이터들은 이후 테스트에서 기반 코드로 재사용된다는 특징이 있습니다. 또한, 저조한 코드 분기 이용률을 보이는 데이터들은 바로 버려 버리지요.

(* 역주: 원어로는 ‘cover more code’.)

이 퍼징 방식은 정형화된 방식으로 찾아내긴 힘든 코드 경로에 도달하는 데 있어 높은 효율성과 속도를 보여주는데요, 이 점이 큰 인기를 끌고 있습니다. 하지만 모든 애플리케이션들이 그레이박스 퍼징의 대상으로 적합하진 않아요. 그레이박스 퍼징은 보통 [많은 수의 입력을 빠르게 처리할 수 있는, 작고 입력값에 따른 출력값이 정해져 있는 프로그램](https://llvm.org/docs/LibFuzzer.html#id22)을 퍼징하는 데 효율적이기 때문입니다. 

> *저희는 주로 파이어폭스의 구성 요소들, 그러니까 미디어 파서 같은 것들을 시험하기 위해 그레이박스 퍼저를 사용하고 있습니다. 만일 독자 분이 본인의 코드를 시험하는 데 어떻게 이런 류의 퍼저들을 사용하는지 알고 싶다면, [이쪽](https://firefox-source-docs.mozilla.org/tools/fuzzing/fuzzing_interface.html)으로 가셔서 Fuzzing Interface Documentation을 한 번 읽어보세요!*
> 

그런데 WebAPI를 퍼징할 때 그레이박스 퍼징을 적용하는 데에는 한계가 있습니다. 브라우저는 어쨌든 입력값 한 개에 따른 출력값이 어떻게 달라질지 모르고(Non-deterministic), 그 입력값도 고도로 구조화되어있는 애플리케이션이기 때문입니다. 그뿐일까요, 브라우저를 시작하고—테스트를 실행하고—문제점이 나오는지, 나온 문제점은 어떤지 모니터링하는 이 일련의 과정이 느릴 수밖에 없습니다. 보통 한 번 테스트하는데 몇 초에서 몇 분까지 걸리니까요.

따라서 WebAPI 퍼징에 쓰기에 그나마 나은 것은 블랙박스 퍼징입니다.

하지만, 이런 부류의 API에 들어가야 하는 입력은 고도로 구조화된 입력이어야 하기 때문에, 과연 현재 사용하는 퍼저가 **'적어도 구조라도 유효한 입력 데이터를 생성하는지'가 중요해집니다.***

(* 역주: 원문 'However, since the inputs expected by these APIs are highly structured, we need to ensure that our fuzzer generates data that is considered valid.' 문장 선후 맥락으로 미루어, valid하게 considered되는 데이터는 구조라도 유효한 입력 데이터일 것으로 생각하고 의역했다.)

### **문법 기반 퍼징(Grammar-Based Fuzzing)**

문법 기반 퍼징은 일반적인 프로그래밍 언어의 문법을 사용하여 생성할 데이터의 구조(형식)를 정의하는 퍼징 기술입니다. 

이런 문법은 보통 평문으로 쓰여 있고, 여러 기호와 상수의 조합을 이용해 데이터를 표현하게끔 합니다. 이렇게 정의된 문법을 퍼저는 파싱해서 퍼징된 결과물을 생성하는 데 사용합니다.

![https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/Untitled-drawing.svg](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/Untitled-drawing.svg)

여기 보이는 그림은 [Domato](https://github.com/googleprojectzero/domato) 와 [Dharma](https://github.com/MozillaSecurity/dharma) [fuzzers](https://github.com/googleprojectzero/domato) 에서 발췌해 단순화시킨 두 가지 문법을 표현한 것인데요, 이 문법들은 `[HTMLCanvasElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement)` 을 생성하고 이것의 속성과 작동을 조정하기 위해 사용되었어요.

### 기존 문법들의 문제점

하지만 문제가 있죠. 문법을 개발하는 데 필요한 노력의 크기는 이 문법으로 표현하려는 데이터의 복잡도와 크기에 정비례합니다. 그리고 이게 바로 문법 기반 퍼징의 가장 큰 문제점이죠. 파이어폭스에서 쓰는 WebAPI들이 730개가 넘는 인터페이스를 이용하고, 이 안에 6300개쯤 되는 멤버들이 있다는 걸 고려하면... 더더욱 큰 문제점입니다.*

(* 역주: Java에서의 인터페이스와 그 내부의 선언을 일컫는 멤버가 맞다.)

콜백이라든가, [enum](https://en.wikipedia.org/wiki/Enumerated_type)이라든가, 딕셔너리같은 요소들처럼 함수 등등에서 요구하는 데이터 구조까지 생각하게 되면 저 숫자들의 크기는 훨씬 커지겠죠. 이렇게나 다양하고 많은 API들을 정확하게 표현할 수 있는 문법을 개발하는 건 정말 정말로 대형 프로젝트가 되겠죠. 그런데 개발만이 문제가 아니죠. 이런 류의 것들은 에러가 엄청 많이 발생하고, 유지보수가 정말로 힘드니까요. 말 그대로 사람이 갈려 들어가는 것입니다.

저희 팀은, 이런 API들을 더 효과적으로 퍼징하려면 최대한 수작업으로 문법을 개발하는 방식은 피하는 게 좋겠다고 생각했습니다.

## **퍼징 문법으로서의 WebIDL**

```
typedef (BufferSource or Blob or USVString) BlobPart;

[Exposed=(Window,Worker)]
interface Blob {
 [Throws]
 constructor(optional sequence blobParts,
             optional BlobPropertyBag options = {});

 [GetterThrows]
 readonly attribute unsigned long long size;
 readonly attribute DOMString type;

 [Throws]
 Blob slice(optional [Clamp] long long start,
            optional [Clamp] long long end,
            optional DOMString contentType);
 [NewObject, Throws] ReadableStream stream();
 [NewObject] Promise text();
 [NewObject] Promise arrayBuffer();

};

enum EndingType { "transparent", "native" };

dictionary BlobPropertyBag {
 DOMString type = "";
 EndingType endings = "transparent";
};
```

[WebIDL](https://heycam.github.io/webidl/) 은 브라우저가 구현해 낸 API들을 설명하기 위해 만들어진 IDL([Interface Description Language](https://en.wikipedia.org/wiki/Interface_description_language))입니다. IDL이 API를 설명할 때는 당 API의 인터페이스, 멤버, 값, 그리고 구문을 나열하는 방식을 사용하지요.

WebIDL이 무엇인지는 브라우저 퍼징 커뮤니티에서 굉장히 유명한데요, WebIDL이 다양한 정보들을 그 안에 가지고 있다는 점 때문입니다.  예전에는 웹 브라우저 퍼징을 할 때, WebIDL이 가지고 있는 데이터를 추출해 퍼징 문법으로 사용하려는 시도가 있었어요. [Sensepost에서 나온 WADI fuzzer](https://sensepost.com/blog/2015/wadi-fuzzer/)가 꼭 이런 시도로 나온 결과물입니다. 하지만, 저희가 저 퍼저를 이용한 몇몇 사례를 조사해 보니, 이렇게 정의된 퍼징 문법 정보들은 퍼저 고유의 문법 구문으로 설정된 형식을 이용해 다시 한 번 추출과 재구현 과정을 거치더군요. 이런 식으로 퍼징하는 건 여전히 상당한 양의 수작업을 요합니다. 추가로, 어떤 인스턴스 내부에서 특정 WebAPI의 작동을 다른 WebAPI에게 정확히 전달해 주려 한다고 생각해 봅시다. 이 행위의 난이도는 퍼저 고유의 문법 구문으로 인해 훨씬 올라가게 되어 있어요.

이런 문제점들에 기반해서, 저희 팀은 위 퍼저의 방식이 아닌*, WebIDL의 정의를 곧장 퍼징 문법으로 가져다 쓰기로 결론을 내렸습니다. 이런 접근 방식 덕에 꽤 많은 이득을 볼 수 있었죠.

(* 역주: 한 마디로, WebIDL의 정의를 현재 사용중인 퍼저의 문법 구문 형식으로 바꿔버리는 방식.)

### **표준화된 문법**

첫째로, WebIDL Specification이 표준화된 문법을 알맞게 정의해 둔다는 점입니다. 이렇게 해둠으로써 raw WebIDL 정의들을 파싱해서 AST([Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree))형태로 만들 때 [WebIDL2.js](https://github.com/w3c/webidl2.js/)같은 도구들을 사용할 수 있게 되는 것이죠. 이렇게 만들어진 AST는 퍼저가 테스트 케이스들을 생성할 때 퍼저가 AST를 해석해서 이용하는 식으로 사용됩니다.

### **단순화된 문법 개발**

둘째로, WebIDL이 타겟으로 찍어 둔 API의 구조와 동작을 정의해 둔다는 점입니다. 이것 덕에 퍼징에 필요한 규칙 개발의 양이 크게 줄죠. 만약 이 방법을 사용하지 않고, 즉 앞에서 말했던 다른 퍼저의 방식대로, 같은 API를 서술하려면 이 API에 대한 맞춤형 규칙(API에서 정의된 인터페이스, 멤버, 값 등등에 대해서)을 만들어야 한다는 점에서 저희가 사용하는 방식이 훨씬 나은 걸 알 수 있습니다.

### **ECMAScript Extended Attributes**

셋째로, 데이터 구조만 정의해 주는 기존의 문법과는 다르게, WebIDL Specification은 ECMAScript Extended Attributes를 통해 인터페이스의 동작과 관련된 추가적인 정보를 알게 해 준다는 점입니다. Extended Attributes는 아래와 같은 다양한 동작 정보를 알려 주는데요:

- 특정 인터페이스가 사용될 수 있는 전후 맥락
- 리턴된 객체가 새로 생성된 건지/기존 인스턴스의 복제인지 여부
- 특정 멤버 인스턴스가 대체될 수 있는지의 여부

이와 같은 유형의 동작 정보들은 기존의 문법들로는 표현하기 힘들기 때문에 WebIDL 문법을 고스란히 사용하는 게 표현에 유리합니다.

### 자동으로 **API 변경 여부 감지하기**

마지막으로, WebIDL 파일이 브라우저에 의해 구현된 인터페이스들과 연결되어 있다는 점을 이득의 한 예로 들 수 있겠군요. 이 ‘연결되어 있다’는 특징으로 인해 웹 브라우저 인터페이스들의 업데이트 사항이 WebIDL에도 고스란히 반영되었으리라는 것이 기본 전제가 되니까요.

## **IDL을 JavaScript로 변환하기**

![https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/WebIDL-Inference2.svg](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/WebIDL-Inference2.svg)

퍼징에 WebIDL을 활용하려면 먼저 파싱을 거쳐야 합니다. 다행히도, [WebIDL2.js](https://github.com/w3c/webidl2.js/) 라이브러리라는 게 존재해서, raw IDL 파일을 AST([Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree))로 변환할 수 있습니다. 이렇게 WebIDL2.js를 이용해 만들어진 AST는 데이터를 트리 형태로 나열된 노드의 집합으로 표현해 주는데요, 여기에 존재하는 각각의 노드들은 WebIDL 문법상에서의 요소(construct)들을 나타냅니다.*

(* 역주: 자바스크립트에서의 constructor와는 다른 개념입니다. constructor(): 생성자 함수. construct: 특정 형태를 취하며, 어떠한 기능을 하는 임의의 코드의 집합들. JS에만 한정된 개념이 아님.)

> WebIDL2 AST 구조에 대해 더 알고 싶으신 분은 [이쪽을](https://github.com/w3c/webidl2.js/#ast-abstract-syntax-tree) 참고해 주세요.
> 

AST가 만들어지기만 하면 그 이후는 간단합니다. AST 상에서의 각각의 노드들, 즉 각각의 요소(construct)들이 어떻게 번역될지만 정의하면 되기 때문입니다. 저희는 이번에 개발한 Domino 퍼저에 AST 탐색과 AST 상의 노드를 자바스크립트로 변환하기 위한 여러 도구를 만들어 넣어두었답니다. 위에 보이는 사진이 저희 도구가 어떤 식으로 번역하는지를 예시 몇 개로 설명하고 있으니 한 번 확인해 보세요.

AST상에 존재하는 노드의 대부분은 '정적 변환', 그러니까, AST 상에 있는 특정 노드는 언제나 JS상에서 똑같은 의미를 지난다는 가정 하에 이루어지는 변환을 사용하면 목표 언어로 번역할 수 있습니다. 예를 들어 봅시다. 생성자(constructor) 키워드는 JS상의 'new' 연산자 인터페이스 이름이 합쳐진 형식과 언제나 동치입니다.

하지만 언제나 이런 것만은 아닙니다. WebIDL의 요소(construct)가 굉장히 많은 의미를 가지고 있고, 맥락에 따라 그 의미가 이리저리 달라진다면 반드시 앞뒤 관계를 고려해 동적으로 생성해야 하기 때문이에요.

### 제네릭 타입*

(* 역주: Generic Type는 매개변수화된 클래스나 인터페이스를 의미합니다. JS의 경우 클래스 이름 뒤에 <T>를 붙여 제네릭 타입의 사용을 알립니다. 여기에서 T는 Generic Type의 형식을 의미합니다.)

[WebIDL Specification](https://heycam.github.io/webidl/#idl-types)에 들어가 보면, generic value들을 표현하는 데 사용되는 자료형의 종류들을 볼 수 있습니다. 여기에 나온 각각의 자료형에 대해, 저희 Domino는 특정 함수를 이용해 요청받은 자료형 형태의 랜덤한 value를 리턴해 주거나, 예전에 이 자료형을 가지고 작업한 적이 있다면 이 자료형의 객체(object)를 기억해 뒀다가 이 객체형과 일치하는 랜덤한 value를 리턴해 줍니다. 예시와 함께 설명해 보겠습니다. AST상의 노드들을 번역할 때, 즉 AST를 순회할 때, 숫자 형식의 자료형, 즉, octec이나, short나, long같은 자료형들을 요구하는 요소(construct)가 존재하겠죠? 그렇다면 그 때, 이 자료형들의 범위 내에 존재하는 값을 우리 Domino가 리턴해 준다는 것입니다.

### 객체 참조

AST 상에서의 특정 노드, 그러니까 특정 요소(construct)의 자료형이 1) 특정한 다른 IDL 요소의 정의를 따와 쓰면서, 2) 함수에게 전달되는 인수로 쓰인다고 해 봅시다. 그렇다면 이 요소는 현재 이 요소가 따와서 쓰려는 IDL 요소에 대해, 그 요소가 표방하는 객체의 복사본이 필요하겠지요? 저희 Domino가 바로 그 일을 해 줍니다. 

Domino는 앞서 말한 방식대로 일해 주기도 하지만, 현재 번역하려는 요소가 요구하는 객체 형식을 리턴해 주는 다른 멤버 argument를 찾아내 엑세스함으로써 필요한 객체를 받아다 리턴해 주기도 합니다.

### 콜백 핸들러

앞서 이야기했던 WebIDL Specification에서는 함수를 표현하는 여러 자료형에 대해서도 찾아볼 수 있는데요(예: [promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise), [callbacks](https://developer.mozilla.org/en-US/docs/Glossary/Callback_function), and [event listeners](https://developer.mozilla.org/en-US/docs/Web/API/EventListener)), 이 각각의 자료형에 대해 Domino는 독특한 함수를 생성해서 리턴해 줍니다. 이렇게 생성된 함수는 인수가 없어도 기능하는 것이 있지만, 만일 인수를 받는 함수라면 그 인수에 대해 랜덤한 연산을 수행해 줍니다. 

모두 아시겠지만, 지금까지 이야기한 과정들이라든가, 동적으로 번역해야 하는 IDL요소들이라든가.... 이런 것들은 IDL을 JS로 완전히 번역하는 과정의 새발의 피에 불과합니다. Domino의 제너레이터는 WebIDL Specification 전체에 대해 번역을 지원하니까요, 전체 과정은 얼마나 방대하겠어요?

그럼 이제, 실제 적용 예를 한 번 봅시다. Blob WebIDL을 퍼징 문법으로 사용했을 때 어떻게 되는지 말이에요.

## 무설정(**無設定)** **퍼징**

```
> const { Domino } = require('~/domino/dist/src/index.js')
> const { Random } = require('~/domino/dist/src/strategies/index.js')
> const domino = new Domino(blob, { strategy: Random, output: '~/test/' })
> domino.generateTestcase()
…

const o = []
o[2] = new ArrayBuffer(8484)
o[1] = new Float64Array(o[2])
o[0] = new Blob([o[1]])
o[0].text().then(function (arg0) {
 o[0].text().then(function (arg1) {
   o[3] = o[0].slice()
   o[3].stream()
   o[3].slice(65535, 1, ‘foobar’)
 })
})
o[0].arrayBuffer().then(function (arg2) {
 o[3].text().then(function (arg3) {
   O[4] = arg3
   o[0].slice()
 })
})
```

위 코드에서 보이는 바와 같이, IDL로 알아낼 수 있는 정보는 그것만으로도 유효한 테스트 케이스를 생성하기에 충분한 수준입니다. 이런 식으로 생성된 테스트 케이스들은 Blob이 관여하는 코드들을 꽤 많이 실행해 주고요. 결과적으로 사람의 개입 없이, 처음 작업해 보는 API에 대한 fuzzer 기본틀을 빠르게 개발할 수 있겠죠.

하지만 이 매커니즘에게 입맛에 맞을 수준의 정확도를 바라기엔 부족한 감이 있습니다. 예를 들어 봅시다. 슬라이스 작업*에 필요한 값이 있다고 쳐요. [Blob Specification](https://w3c.github.io/FileAPI/#dfn-slice)을 확인해 보면, 시작과 끝으로 주어지는 인수(arguments)가 Blob의 크기를 감안한 상태에서 바이트 오더 순서로 놓여 있어야 한다는 걸 알 수 있는데요, 아무래도 퍼저를 사용하는 상황이다 보니 시작값과 끝값을 랜덤으로 생성할 수밖에 없어요. 이렇게 되면 Blob의 길이(크기) 범위에 맞지 않는 랜덤값을 반환하는 경우도 생기겠죠. 

이런 류의 문제가 생길 수 있다는 겁니다.

(* 역주: Slice operation. 문자열을 원하는 만큼 잘라서 리턴하는 작업이다.)

또한, 슬라이스 연산에서의 `contentType` 인수와 `BlobPropertyBag` 딕셔너리의 타입 속성은 모두 `[DOMString](https://developer.mozilla.org/en-US/docs/Web/API/DOMString)`의 한 종류라는 특징이 있어요. 그렇다면 퍼저를 사용할 때 숫자 값들처럼 스트링 값들도 랜덤으로 생성해 집어넣는다는 점에 집중해 봅시다. 스펙 문서를 잘 분석해 보면 지금 랜덤으로 생성되어 들어가고 있는 스트링 값들이 Blob 데이터의 미디어 유형을 나타내는 데 사용된다는 걸 알아낼 수 있는데요, 여기에서 정말 곤혹스러운 딜레마를 만나게 되죠. 과연 이 스트링 값은 Blob을 사용하는 API에 어떤 영향을 끼칠 것인가? 과연 아무 영향도 끼치지 않을까?

이런 문제를 해결하기 위해서 앞서 설명했던 제네릭 타입들을 구분하는 방식을 만들어야 했습니다.

## **GRIDL을 사용한 룰 패치(Rule Patching)**

![https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/GrIDL-Domino-Relationship2.png](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/GrIDL-Domino-Relationship2.png)

따라서 저희 팀은 GrIDL이라는 또 다른 도구를 하나 개발해 냈습니다. GrIDL은 IDL 정의를 AST로 변환할 때 WebIDL2.js 라이브러리를 이용하는데요, 거기에 추가로 AST를 최적화해 줌으로써 퍼징 문법으로 AST를 사용할 때 최대한의 효율을 낼 수 있게 한다는 특징이 있습니다.

그런데, GrIDL의 가장 재미있는 점은 다른 곳에서 찾아볼 수 있습니다. 바로 여러 맥락을 고려한 정확한 값을 설정하고 싶을 때 IDL 정의를 동적으로 패치할 수 있다는 것이죠. GrIDL은 규칙 기반 매칭 시스템(Rule-Based matching system)을 이용해 타겟 값을 설정하고 고유 식별자(Unique Identifier)라는 것을 삽입하는데, 이게 Domino에서 구현해 놨던 매칭 제너레이터(Matching generator)랑 서로 통신하거든요*. AST를 순회하는 도중에 이런 식별자와 마주치게 되면 Domino는 매칭 제너레이터를 콜해다가 알맞은 값을 만들어 리턴해 줍니다.

(* 역주: 원문 ‘Those identifiers correspond with a matching generator implemented by Domino.‘ correspond with는 일치한다는 뜻도 있으나 편지를 주고받는다는 의미도 있어 후자의 의미를 차용했다.)

![https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/GrIDL-Markup-and-Generators1.svg](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/GrIDL-Markup-and-Generators1.svg)

위 모식도는 GrIDL 식별자와 Domino의 제너레이터 간 상관 관계를 표현한 그림입니다. 제너레이터 두 개가 정의된 게 보이시죠? 하나는 바이트 오프셋을, 다른 하나는 유효한 MIME 타입을 반환해 주는 제너레이터입니다.

그리고 중요한 점. 각 제너레이터가 현재 퍼징중인 객체의 실황을 받아볼 수 있다는 것입니다. 이를 통해 객체의 현 상태를 반영한 값을 생성해 낼 수 있죠.

> 앞에서 확인한 예시에 따르면, 위 객체를 활용해 slice() 연산을 퍼징할 때 대상 스트링의 길이를 감안한 바이트 오프셋을 만들 수 있다는 사실을 알 수 있습니다. 그럼 더 복잡한 경우에 대해서는 어떨까요? WebGLRenderingContextBase 인터페이스와 연관된 애트리뷰트나 연산이 있다고 합시다. 이 인터페이스는 WebGL로도 WebGL2로도 구현할 수 있는데요, 이 경우 무엇을 선택해 구현하느냐에 따라 들어가는 인수가 정말 많이 달라지게 됩니다. 하지만, 현재 퍼징되고 있는 객체의 실 상황을 이용함으로써 여기 아래에 있는 코드처럼 컨텍스트 타입과 리턴 값을 알맞게 결정할 수 있어요.
> 

```
> domino.generateTestcase()
…
const o = []
o[1] = new Uint8Array(14471)
o[0] = new Blob([null, null, o[1]], {
'type': 'image/*',
'endings': 'transparent'
})
o[2] = o[0].slice((1642420336 % o[0].size), (3884321603 % o[0].size), 'application/xhtml+xml')
o[0].arrayBuffer().then(function (arg0) {
  setTimeout(function () { o[0].text().then(function (arg1) { o[0].stream() }) }, 180)
  o[2].arrayBuffer().then(function (arg2) {
    o[0].slice((3412050218 % o[0].size), (646665894 % o[0].size), 'text/plain')
    o[0].stream()
  })
  o[2].text().then(function (arg3) {
    o[2].slice((2025414481 % o[2].size), (2615146387 % o[2].size), 'text/html')
    o[3] = o[0].slice((753872984 % o[0].size), (883984089 % o[0].size), 'text/xml')
    o[3].stream()
  })
})

```

이렇게 만든 규칙을 사용하면, Specification 문서의 설명과 더 부합하는 값을 생성해낼 수 있습니다.

## **실례**

이번 글에서 소개한 예시들은 정말로 많이 단순화되었기 때문에, 실제 사용되는 복잡한 API에 대해서는 어떤 원리가 어떻게 적용되는지 확인이 어려울 수 있겠다는 생각이 들었습니다. 그러니 실제 상황에서 Domino로 발견한 복잡한 취약점 중 하나를 골라 소개해 드리려 해요.

![https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/Bug-15585221.svg](https://2r4s9p1yi1fa2jd7j43zph8r-wpengine.netdna-ssl.com/files/2020/04/Bug-15585221.svg)

bug [1558522](https://bugzilla.mozilla.org/show_bug.cgi?id=1558522) 입니다. 링크 들어가 보시면 저희 팀이 [IndexedDBAPI](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)에 영향을 주는 심각한 [Use-After-Free](https://en.wikipedia.org/wiki/Dangling_pointer) 취약점을 찾아냈다고 쓰여 있는데요, 이 취약점은 유발하는 과정이 복잡하기 때문에 이걸 퍼징으로 찾아냈다는 건 퍼징하는 사람 입장에서 굉장히 흥미로운 일이 아닐 수 없습니다. 

Domino가 이걸 어떻게 찾아냈냐고요? Domino는 global context 상에서 파일을 만들어서, 그 파일(객체가 되겠죠)을 IndexedDB 데이터베이스 연결을 수립해 주는 worker context에게로 넘겨줌으로써 취약점을 유발해 찾아냈습니다.

이런 수준의 컨텍스트 간 유기성은 기존의 문법으로는 어떻게 설명할 길이 없죠. 하지만 WebIDL이 알려 주는 API의 세세한 정보 덕에 Domino는 이런 취약점도 척척 찾아낼 수 있답니다.

## Contributing

마지막 참고 사항: Domino는 저희 브라우저에서 심각한 보안 취약점을 계속 찾고 있습니다. 그리고 안타깝게도 이건 우리가 아직 그것을 공개적으로 사용할 수 없다는 뜻이기도 해요. 그래도 최대한 빨리 일반 공개 버전을 출시할 계획은 있으니, 조금만 더 지켜봐 주세요. [Firefox 개발에 코드 기여를](https://codetribute.mozilla.org/) 시작하고 싶다면 저희 문은 언제든지 열려 있으니 편히 생각하시고요, 또 Mozilla 직원이거나 NDA 코드 기고자인데 Domino에서 작업하고 싶으신 분은 Riot(Matrix)의 Fuzzing 룸에 있는 팀에 언제든지 연락하십시오!

## Jason Kratzer에 대해:

- [@pyor_](http://twitter.com/pyoor_)

[Jason Kratzer의 더 많은 기사…](https://hacks.mozilla.org/author/jkratzermozilla-com/)
