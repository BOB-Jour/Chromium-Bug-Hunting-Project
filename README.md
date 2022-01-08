# 웹 브라우저 취약점 탐지 자동화 프로젝트

> 취약점 분석 트랙 BOB-Jour팀은 [BoB(Best of the Best)](https://www.kitribob.kr/) 프로젝트로 약 3개월간 자동화를 이용하여 웹 브라우저의 취약점 탐지를 하고자 했다.
> 

## Overview

프로젝트의 목표는 크롬 브라우저 버그 헌팅과 브라우저 보안 시장의 인식 향상을 위해 자동화를 이용하여 취약점을 탐지하는 것이다.

해당 프로젝트를 진행하기에 앞서, 우리는 모두 웹 브라우저 취약점 분석 경험이 없었다. 따라서 크롬 브라우저 구조학습과 원데이 분석을 진행한 후 어택 벡터 선정을 하였다. 또한 Chrome Release CVE 정보를 수집함으로써 어택 벡터를 선정하기도 하였다. 

상대적으로 어려운 크롬의 구현 코드와 메모리 구조 덕분에 우리는 자동화 기술 중 퍼징을 이용했고, 프로젝트 후반부에 CodeQL까지 적용하였다. 결국 취약점 탐지는 실패하였지만 이후 이 분야를 접하는 분들에게 도움이 되기 위해 우리가 공부하고 노력한 내용들을 공유하고자 한다.

## BOB-Jour

- 신정훈 멘토님, 이상섭 멘토님
- 임준오 PL
- 손현지 PM, 심준용, 이윤희, 이주협, 홍택균


## Contents

1. [원데이 분석 보고서](./1_vulnerability_analysis_whitepapers)   
  1.1 [CVE-2020-15999](./1_vulnerability_analysis_whitepapers/cve-2020-15999)   
  1.2 [CVE-2020-16002](./1_vulnerability_analysis_whitepapers/cve-2020-16002)   
  1.3 [CVE-2020-6449](./1_vulnerability_analysis_whitepapers/cve-2020-6449)   
  1.4 [CVE-2021-30519](./1_vulnerability_analysis_whitepapers/cve-2021-30519)   

2. [퍼저 개발](./2_fuzzer_development)   
  2.1 [Domino](./2_fuzzer_development/Domino)   
  2.2 [WatTF](./2_fuzzer_development/watTF)   
  2.3 [Glitch](./2_fuzzer_development/Glitch)   
  2.4 [Pleaser](./2_fuzzer_development/Pleaser)   

3. [코드큐엘](./3_codeql)   
  3.1 [Overview](./3_codeql/Overview.md)   
  3.2 [라이브러리를 이용한 쿼리 작성](./3_codeql/Designing_CodeQL_queries.md)   

4. [결론](./4_conclusion/README.md)   
