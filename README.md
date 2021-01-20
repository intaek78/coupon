
쿠폰발행

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 쿠폰발행](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
     1. SAGA
     2. CQRS
     3. Correlation
     4. Req/Res
  - [운영](#운영)
     5. Gateway     
     6. Deploy     
     7. CB     
     8. HPA     
     9. Readiness
     


# 서비스 시나리오


기능적 요구사항

1. 고객이 멤버로 가입한다
2. 멤버로 가입하면 쿠폰이 발행된다
3. 멤버가 가입하면 관리자에게 알림을 발송한다.
4. 쿠폰이 발행되면 멤버의 상태를 변경한다.
5. 알림이 발송되면 멤버의 알림발송여부를 Y로 수정한다.
6. 멤버는 탈퇴 할 수 있다.
7. 멤버가 탈퇴하면 쿠폰발행이 취소된다.
8. 쿠폰발행이 취소되면 멤버의 쿠폰취소여부를 Y로 수정한다.
9. 멤버는 쿠폰발행여부를 조회할 수 있다.

비기능적 요구사항
1. 트랜잭션
    1. 쿠폰발행이 취소되어야 멤버탈퇴가 가능하다.  Sync 호출
    
1. 장애격리
    1. 쿠폰발행 기능이 수행되지 않더라도 멤버등록은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 쿠폰 취소시스템이 과중되면 멤버탈퇴가 잠시 중단되고 잠시 후에 탈퇴하도록 한다  Circuit breaker, fallback
1. 성능
    1. 고객이 수시로 주문상태를 마이페이지에서 확인할 수 있어야 한다  CQRS




## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  


### 완성된 1차 모형
![이벤트스토밍-쿠폰](https://user-images.githubusercontent.com/39254844/105127295-ed690d80-5b23-11eb-9f6f-bfce69390ba2.png)


    - View Model 추가

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![흐름1](https://user-images.githubusercontent.com/39254844/105127704-c19a5780-5b24-11eb-804f-d16be904a609.png)

    - 고객이 멤버로 등록한다 (ok)
    - 멤버로 등록하면 쿠폰이 발행된다. (ok)
    - 쿠폰이 등록되면 멤버의 상태를 수정한다 (ok)

![흐름2](https://user-images.githubusercontent.com/39254844/105127771-de368f80-5b24-11eb-9636-0f8d16ed4b73.png)

    - 멤버는 탈퇴 가능하다 (ok)
    - 탈퇴가 이루어지면 쿠폰발행이 취소된다. (ok)
    - 쿠폰발행이 취소되면 멤버의 쿠폰발행취소여부를 Y로 수정한다. (ok)
    
![흐름3](https://user-images.githubusercontent.com/39254844/105127807-f3abb980-5b24-11eb-81e9-d56ef64922a0.png)

    - 멤버가 등록하면 관리자에게 알림을 발송한다. (ok)
    - 알림이 발송되면 멤버의 알림발송여부를 Y로 수정한다. (ok)



### 비기능 요구사항에 대한 검증

![비기능](https://user-images.githubusercontent.com/39254844/105128455-6b2e1880-5b26-11eb-936a-add7ab75c28f.png)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 멤버등록 취소 시 쿠폰발행취소 처리 : 쿠폰발행이 취소되지 않을 경우 멤버 탈퇴 불가하다는 방침에 따라, ACID 트랜잭션 적용. 
        - 멤버등록 취소시 쿠폰발행 취소처리에 대해서는 Request-Response 방식 처리
        - 멤버 등록 시 이벤트 드리븐 방식으로 쿠폰발행 서비스의 상태에 관계없이 처리됨
        - 쿠폰 취소시스템이 과부하 걸리면 멤버탈퇴가 잠시 중단되도록 처리
        - 멤버정보는 수시로 마이페이지에서 조회 가능함





# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 


1,3. 멤버->쿠폰->멤버 캡쳐



 - 멤버 등록

![멤버등록](https://user-images.githubusercontent.com/39254844/105129334-27d4a980-5b28-11eb-84ed-81e681225e2d.png)

 - 멤버 등록 후 쿠폰 발행 

![쿠폰등록](https://user-images.githubusercontent.com/39254844/105129570-b47f6780-5b28-11eb-88c1-9f986f464e89.png)

 - 멤버등록 후 관리자 알림 발송

![관리자일림발송](https://user-images.githubusercontent.com/39254844/105133465-2c04c500-5b30-11eb-9044-60d0023b3041.png)

 - 쿠폰발행 후 멤버 조회

![멤버재조회](https://user-images.githubusercontent.com/39254844/105133629-6c644300-5b30-11eb-9af5-4cc74db8c4f2.png)

 - 멤버탈퇴

![멤버탈퇴](https://user-images.githubusercontent.com/39254844/105133757-9ddd0e80-5b30-11eb-9e94-ccfab7a22245.png)

 - 멤버탈퇴 시 쿠폰발행 취소 반영

![쿠폰취소반영](https://user-images.githubusercontent.com/39254844/105133933-eac0e500-5b30-11eb-8ccb-99b800f8303b.png)


2. 마이페이지 조회

![마이페이지최종조회](https://user-images.githubusercontent.com/39254844/105134011-117f1b80-5b31-11eb-9167-4c63a193f472.png)


3. 쿠폰취소 서비스 장애 시 멤버탈퇴 불가

![쿠폰서비스장애시멤버탈퇴불가](https://user-images.githubusercontent.com/39254844/105134636-12fd1380-5b32-11eb-9972-6b19331ba0d8.png)



   

5.6 Gateway, Deploy

product 상품 등록 
 - LoadBalanced로 노출된 퍼블릭IP로 상품등록 API 호출

![image](https://user-images.githubusercontent.com/75401920/105001534-42008000-5a73-11eb-8ab7-c955745e7703.png)


애져에 배포되어 있는 상황 조회 kubectl get all

![image](https://user-images.githubusercontent.com/75401920/105000728-06b18180-5a72-11eb-8609-e527c48f7060.png)



7. Istio 적용 캡쳐

  - Istio테스트를 위해서 Payment에 sleep 추가
  
![image](https://user-images.githubusercontent.com/75401920/105005616-e89b4f80-5a78-11eb-82cb-de53e5881e3f.png)

 - payments 서비스에 Istio 적용
   
![image](https://user-images.githubusercontent.com/75401920/105006822-7f1c4080-5a7a-11eb-9191-db35233773d3.png)

 - Istio 적용 후 seige 실행 시 대략 50%정도 확률로 CB가 열려서 처리됨

![image](https://user-images.githubusercontent.com/75401920/105006958-b2f76600-5a7a-11eb-99f3-c8b81a4ec270.png)

8. AutoScale

 - AutoScale 적용된 모습

![image](https://user-images.githubusercontent.com/75401920/105006642-4714fd80-5a7a-11eb-8424-aa2dede45666.png)

 - AutoScale로  order pod 개수가 증가함

![image](https://user-images.githubusercontent.com/75401920/105006308-cf46d300-5a79-11eb-96db-77d865c9bfe9.png)


9. Readness Proobe
 
  - Readiness 적용 전: 소스배포시 500 오류 발생
  
![image](https://user-images.githubusercontent.com/75401920/105004548-7d04b280-5a77-11eb-95cb-d5fe19a40557.png)


  - 적용 후: 소스배포시 100% 수행됨

![image](https://user-images.githubusercontent.com/75401920/105004912-f0a6bf80-5a77-11eb-88ee-f0bcd8f67f45.png)

