# Variant_analysis_of_WebAudio_callback_vulnerabilities_in_chrome_studylog.md

- 원본 링크: [https://securitylab.github.com/research/chrome_task_queue_uaf/](https://securitylab.github.com/research/chrome_task_queue_uaf/)
- 시간의 부족으로 인해 이 글을 공부할 당시 번역을 거치지 않고 핵심만 정리해 두었다.
- 이 글을 읽음으로써 <CodeQL을 통한 버그 헌팅 프로젝트 수행 단계>의 초안을 작성할 수 있었다는 점을 강조하고 싶다.

---
1. [핵심 요약](#1-핵심-요약)
2. [제언](#2-제언)
---

# 1. 핵심 요약

1. 아래 문단들의 핵심은 여러 개의 크롬 웹오디오에서 발생한 UAF 취약점을 종합해서 살펴봤더니, 취약점들의 공통된 패턴이 있었다는 것으로 정리될 수 있다. 
    - these issues occurred because of invalid assumptions about the ownership of `AudioHandler`. Namely, it assumes either `AudioNode` or `rendering_orphan_handlers_` are responsible to keeping an `AudioHandler` alive. When `AudioNode` is alive, it keeps both the `AudioHandler` and the untraced `BaseAudioContext` alive, so no UaF there. **After `AudioNode` is destroyed, the lifetime of `context_` is no longer guaranteed.**
        - `context_` : `AudioScheduleSourceHandler` 내부에 있는 객체.
    - **To prevent UaF, `context_` must be cleared out from `AudioHandler` when it is destroyed.** The way this is done in the destructor is to **clear out `context_` in `rendering_orphan_handlers_`.**
    - This implicitly assumed that **after `AudioNode` is gone, `rendering_orphan_handlers_` is the only object that holds a `scoped_refptr` to `AudioHandler`**, (원래는 이런데)which is invalid in this case, (취약점들에서는)where **`AudioHandler` can be kept alive by a callback even after it is removed from `rendering_orphan_handlers_`.**
        - `scoped_refptr`: 현재까지는 `context_` 의 역할을 하는 것으로 추정 중.
    - At the time when 105788 was fixed, I was manually auditing the code and wasn’t thinking too much about variant analysis, but after I saw the second issue in [1057627](https://bugs.chromium.org/p/chromium/issues/detail?id=1057627), it became apparent to me that **this pattern may be more common**.
2. From these bugs, it appears that **following combinations are likely to cause issues:**
    1. **A member function of an `AudioHandler`** that is 1) being posted to the main thread, which 2) accesses `context_`
        - `context_` 의 수명이 더는 보장되지 않는 상황이라는 전제가 있다.
    2. The **`AudioHandler` is posted as a `scoped_refptr`**, which means that it will be alive even when `BaseAudioContext` is destroyed and cleared away `rendering_orphan_handlers_` that holds it.
        - **`AudioHandler` (=posted as a `scoped_refptr` ) can be kept alive by a callback even after it is removed from `rendering_orphan_handlers_`.**
3. 위 조건들을 취합한다면 
    
    ```cpp
    CrossThreadBindOnce(& FunctionsRelatedToAudioHandler , SomethingReturnsScoped_refptr , ...) 
    ```
    
    꼴의 것이 있어야 UAF가 난다는 결론을 내릴 수 있다.
    
    즉, `context_` 에 접근하는 `Audiohandler` 중 하나를, `scoped_refptr` 형태로 main thread에 post하는 코드 부분에서 UAF가 날 가능성이 높으므로 그 패턴과 일치하는 부분을 찾아내야 한다.
    
    추가. 위 코드 패턴에서, 각 요소가 의미하는 바는 아래와 같다.
    
    - `CrossThreadBindOnce` : This method posts a member function of an `AudioHandler` to the main thread
    - `FunctionsRelatedToAudioHadler` : This callback acesses `context_`
    - `SomethingReturnsScoped_refptr` : This argument mostly means `WrapRefCounted` in wild.
        
        ```cpp
        scoped_refptr<T> WrapRefCounted(T* t){
        	return scoped_refptr<T>(t);
        }
        ```
        
4. 글의 흐름, 연구 과정 정리:
    1. 취약점들의 UAF 원인을 분석한다.
    2. 그것을 막기 위해서는 어떻게 해야 하는지 알아낸다.
    3. 방지책을 역으로 이용해 UAF가 날 수밖에 없는 상황의 조건을 간주한다.
        
        → 이 글을 기반으로 CodeQL 이용 계획을 세우면 좋을 것 같다.
        
5. This is a fairly simple pattern that can be surfaced by CodeQL(확정한 패턴을 쿼리화):

```sql
from FunctionCall fc, FunctionCall wrapRef// fc(~wrapRef)꼴 찾기
      //Look for cross thread task being posted: CrossThreadBindOnce는 특정 메소드를 task queue에 올려주는 역할.
where fc.getTarget().hasName("CrossThreadBindOnce") and
      //===========Look for `AudioHandler` posted as `scoped_refptr`==================
      wrapRef.getTarget().hasName("WrapRefCounted") and
			//wrapRef이 WrapRefCounted 여야 하는데, 이건 CrossThreadBindOnce의 인자로 있어야 한다는 뜻
      wrapRef = fc.getAnArgument() and
			// 1. 공통 패턴: CrossThreadBindOnce(&Task, WrapRefCounted(this), ...)
			// 2. 타입 상관없이 존재하는 e에 대해서, 직계 상위 클래스가 AudioHandler인 클래스를 CrossThreadBindOnce()의 첫 번째 인자로 받는 경우
      exists(Expr e | e.getType().stripType().(Class).getABaseClass*().getName() = "AudioHandler" and
                      e = wrapRef.getArgument(0))
//검색 결과로 보여주는 건 CrossThreadBindOnce |  <-얘의 첫 번째 인자(이게 AudioHandler 하위 클래스일 테니까)
select fc, fc.getArgument(0)
```

![상기 쿼리 중 exists의 의미와 용법. 사진 [출처](https://codeql.github.com/docs/ql-language-reference/formulas/#exists)(링크)](./img/webaudio1.png)

상기 쿼리 중 exists의 의미와 용법. 사진 [출처](https://codeql.github.com/docs/ql-language-reference/formulas/#exists)(링크)

(정리하는 말. 마오 윤 본인이 이상함을 눈치채고 저 패턴을 잡아내는 쿼리를 만들어 돌려봤으나, 소수의 결과만 나왔으며 그것조차 거의 즉시 패치되었다고 한다. 그래도 진짜로 저 패턴과 일치하는 모든 코드들이 다 패치되었는지 확인하는 차원에서 쿼리를 만들어 돌려보는 것은 꽤 좋은 일이라고 말하고 있다.)

In the above, I omitted the condition of accessing `context_` to simplify the query and make it run faster. As it is, there aren’t many results, and at this point, it had also become apparent to the developers that these issues may be more widespread — they found and fixed them independently in this [commit](https://source.chromium.org/chromium/chromium/src/+/b75436e554d54b2d8d3590d7e61607e1ce67a2fe). Nevertheless, having a query to confirm that all cases were fixed is plenty useful.

---

# 2. 제언

이 문서는 확실히 CodeQL을 이용한 버그헌팅 계획을 세우는 데 참고할 만하다. 그러나 현실적인 한계, 즉, 시간이 부족하다는 한계를 감안해야 할 필요성 또한 존재한다. 차후 팀원들과 논의가 필요하다.

확실한 건, 현재로서는 이 방식을 그대로 적용하기 어려우리란 것이다. 왜냐하면:

문서 전반의 리서치 과정을 정리하면,

> 1.  취약점이 자주 발생한 공격벡터를 선정한다.
> 
> 1. 그 공격벡터에서 발생한 취약점들을 리스트업해 근본 원인을 분석한다.
> 2. 취약점들의 근본 원인에서 공통적으로 확인되는 패턴을 확정한다.
> 3. 이러한 패턴의 취약점에 어떤 패치가 되었는지 확인해 분석한다.
> 4. 패치 내역을 피할 수 있으면서, 패턴을 만족하는 코드 내용을 쿼리화한다.

이렇게 되는데, 1~3 과정에는 우리가 CodeQL 프로젝트에 할당한 2주 이상의 시간이 필요하기 때문이다.
