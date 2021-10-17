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
![검증1](https://user-images.githubusercontent.com/90189785/137635075-8d99df2f-d40b-4db0-a42a-8fdc5b575f66.jpg)

1. 관리자는 멤버십상품을 등록/삭제/수정할 수 있다. (ok)
2. 고객은 멤버십 상품을 구매한다. (ok)
3. 결제승인된 내역을 자동으로 판매수량을 update한다. (ok)
4. 멤버십상품을 구매하면 결제단계로 넘어간다. (Async, ok)
5. 멤버십상품을 취소하면 결제가 취소된다. (Async, ok)
6. 고객은 모든 진행사항을 조회 가능하다. (ok)
7. 구매가 완료되면 만료일자는 1년으로 설정한다. (ok)
8. 만료된 멤버십상품은 구매할 수 없다. (sync, 비기능적)

# 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라,구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084, 8088이다)

```shell
cd membership
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd product
mvn spring-boot:run 

cd view 
mvn spring-boot:run

cd gateway
mvn spring-boot:run 
```

## DDD(Domain-Driven-Design)의 적용
msaez Event-Storming을 통해 구현한 Aggregate 단위로 Entity 를 정의 하였으며,
Entity Pattern 과 Repository Pattern을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.

membership 서비스의 membership.java

```java
package membership.system;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name = "Membership_table")
public class Membership {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long memId;
    private Integer productId;
    private String productName;
    private Integer price;
    private Integer customerId;
    private Boolean memStatus;

    @PostPersist
    public void onPostPersist() {
        MemPurchased memPurchased = new MemPurchased();
        BeanUtils.copyProperties(this, memPurchased);
        memPurchased.publishAfterCommit();

        MemChanceled memChanceled = new MemChanceled();
        BeanUtils.copyProperties(this, memChanceled);
        memChanceled.publishAfterCommit();

    }

    @PostUpdate 
    public void onPostUpdate(){

        if("RETURN".equals(this.memStatus)){           // 구매처리 Publish
            MemPurchased memPurchased = new MemPurchased();
            BeanUtils.copyProperties(this, memPurchased);
            memPurchased.publishAfterCommit();

        } else if("DELAY".equals(this.memStatus)){     // 취소처리 Publish
            MemChanceled memChanceled = new MemChanceled();
            BeanUtils.copyProperties(this, memChanceled);
            memChanceled.publishAfterCommit();
        }
    } 
     .. getter/setter Method 생략
```
Payment 서비스의 PolicyHandler.java rental 완료시 Payment 이력을 처리한다.

```java
```
