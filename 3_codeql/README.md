# 3_codeql

## Intro


프로젝트를 설계할 때 우리는 프로젝트 후반부에 CodeQL을 취약점 찾기에 적용하는 것을 목표로 두었다. 그 이유는 주관적으로 CodeQL이 흥미로운 기술이기도 했지만 [BlackHat USA 2021](https://www.blackhat.com/us-21/)에서 ‘[CodeQL로 크롬 브라우저 버그 헌팅하기](https://www.blackhat.com/us-21/briefings/schedule/#put-in-one-bug-and-pop-out-more-an-effective-way-of-bug-hunting-in-chrome-22855)’라는 주제가 목표 선정에 큰 영향을 주었기 때문이다.

CodeQL이라는 쿼리 언어의 특성상 어떤 쿼리를 작성하는지에 따라 원하는 정보를 가져올 수 있음을 판가름한다. 따라서 우리는 쿼리를 작성하기에 앞서 충분한 공부가 필요했고, 위에 소개한 BlackHat USA 2021 발표와 유명 리서처의 글을 통해 실제로 취약점을 찾아낸 CodeQL 쿼리를 분석하며 사전 지식을 쌓았다. 

우리는 최종적으로 전반적으로 CodeQL을 이용하여 취약점 찾기를 위한 어떠한 노력을 했는 지, 결론적으로 어떤 CodeQL 쿼리를 작성할 수 있었는 지에 대한 문서를 작성하였다.

## Contents


1. [Overview](./Overview.md)
2. [라이브러리를 이용한 쿼리 작성](./Designing_CodeQL_queries.md)

### Related Work


- [Blackhat_usa_2021_chrome.md](./related_work/Blackhat_usa_2021_chrome.md)
- [Review_of_chromium_IPC_vulnerabilities.md](./related_work/Review_of_chromium_IPC_vulnerabilities.md)
- [Variant_analysis_of_WebAudio_callback_vulnerabilities_in_chrome_studylog.md](./related_work/Variant_analysis_of_WebAudio_callback_vulnerabilities_in_chrome_studylog.md)
