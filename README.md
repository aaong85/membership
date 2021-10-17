# Membership

## 고객 유료 멤버십 관리 시스템   

# Table of contents

- [고객 유료 멤버십 관리 시스템](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)

# 서비스 시나리오

1. 관리자는 멤버십상품을 등록/삭제/수정할 수 있다.
2. 고객은 멤버십 상품을 구매한다.
3. 결제승인된 내역을 자동으로 판매수량을 update한다.
4. 멤버십상품을 구매하면 결제단계로 넘어간다.
5. 멤버십상품을 취소하면 결제가 취소된다.
6. 고객은 모든 진행사항을 조회 가능하다.
7. 구매가 완료되면 만료일자는 1년으로 설정한다.
8. 만료된 멤버십상품은 구매할 수 없다.

# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
![image](https://user-images.githubusercontent.com/89369983/133180330-14093197-2864-4b9c-8e5b-6ec6555e49f5.png)

## TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/90189785/137633685-b1c3ced9-5fd1-419c-b9a0-63eae2027628.PNG)

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: https://www.msaez.io/#/storming/sBSoKIMynLR2owUN1kjnRfJohAj2/cdf97eea37ca9195aa3030282c1c9412

### 이벤트 도출
![event도출](https://user-images.githubusercontent.com/90189785/137633872-1a8b125d-5101-4c7b-a578-371b14698b03.PNG)

### 부적격 이벤트 탈락
![부적격event](https://user-images.githubusercontent.com/90189785/137633898-0c29505e-4f9e-439f-9058-170cc2328425.PNG)

### 액터, 커맨드 부착하여 읽기 좋게
![action](https://user-images.githubusercontent.com/90189785/137633932-9b34fef0-12cc-4469-bb1e-5337487fe6cd.PNG)

### 어그리게잇으로 묶기
![aggregate](https://user-images.githubusercontent.com/90189785/137633952-131da630-b7ea-4c89-9bc7-2810c349ae8f.PNG)

### 바운디드 컨텍스트로 묶기
![boundedct](https://user-images.githubusercontent.com/90189785/137633989-7d8ef459-df53-4ee6-942f-a35b4864f781.PNG)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![policy_ct](https://user-images.githubusercontent.com/90189785/137634807-ee27786b-f9f7-44f2-9441-65dbc3289ffb.PNG)

### 완성된 1차 모형
![complete](https://user-images.githubusercontent.com/90189785/137634349-0064e697-e679-4103-8d27-246bad63e343.PNG)

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
![검증1](https://user-images.githubusercontent.com/88808251/133179151-31b820fe-0863-436a-9090-51b9a7f372ea.png)
![검증2](https://user-images.githubusercontent.com/88808251/133173230-9f45a13a-2d64-4475-99ae-87cb756e0706.png)

# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라,구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8086이다)
