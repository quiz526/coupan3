# 쿠폰판매 사이트

![image](https://user-images.githubusercontent.com/84000890/124267649-80967c80-db73-11eb-9e1c-748e37af6e45.png)
# 쿠판 

본 프로젝트는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 프로젝트입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 내용을 포함합니다.

# 서비스 시나리오

기능적 요구사항
1. 점주는 쿠폰의 가격, 재고수량 등과 함께 쿠폰을 등록할 수 있다. 
2. 고객은 쿠폰을 구매을 하고 구매가 완료되면 해당 쿠폰의 재고 수량은 변경된다. 
3. 고객 쿠폰 구매 후, 결제가 진행된다. 
4. 결제가 완료되면 고객의 쿠폰 구매 상태가 변경된다. 
5. 고객은 구매 취소가 가능하다. 
6. 구매가 취소되면 관련 구매 정보는 삭제되지 않고 구매 상태가 변경되며, 결제 취소가 발생된다.
7. 구매 취소 시 쿠폰 재고 수량은 취소된 쿠폰 갯수만큼 증가한다.
8. 고객은 쿠폰 구매 및 결제 정보 등을 마이페이지를 통해 수시로 조회한다. ex) 수량, 상태, 예약일시,예약취소일시, 결제일시,결제취소일시, 결제금액등 

비기능적 요구사항
1. 트랜잭션
    1. 쿠폰이 구매되면 전체 쿠폰 수량은 예약된 수만큼 변경되어야 한다. Sync 호출
2. 장애격리
    1. 쿠폰등록이 정상적으로 수행되지 않더라도 쿠폰구매는 365일 24시간 받을 수 있어야 한다.  Async (event-driven), Eventual Consistency
    2. 쿠폰등록이 과중되면 사용자를 잠시동안 받지 않고 차량 등록을 잠시후에 하도록 유도한다.  Circuit breaker
3. 성능
    1. 고객이 쿠폰 구매 이력을 별도의 고객페이지에서 확인할 수 있어야 한다. CQRS

# 체크포인트

1. Saga
2. CQRS
3. Correlation
4. Req/Resp
5. Gateway
6. Deploy/ Pipeline
7. Circuit Breaker
8. Autoscale (HPA)
9. Zero-downtime deploy
10. f Map / Persistence Volume
11. Polyglot
12. Self-healing (Liveness Probe)

# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/84000863/121344166-65fb2a00-c95e-11eb-97b3-8d1490beb909.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/2g5SbEbUqzXH8UjIkG1viaXlcoq2/mine/b90af39ae23bc6868258e50d1882d9e7


### 이벤트 도출
![ddd1](https://user-images.githubusercontent.com/84000890/124348168-5f8d6480-dc23-11eb-8e4f-f04f4e8f907a.jpg)

### 부적격 이벤트 탈락
![ddd2](https://user-images.githubusercontent.com/84000890/124348178-6a47f980-dc23-11eb-9982-3978d8dfcd67.jpg)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
    - 구매이력을 조회함 :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외

### 액터, 커맨드 부착하여 읽기 좋게
![ddd3](https://user-images.githubusercontent.com/84000890/124348179-6caa5380-dc23-11eb-88eb-0fdf5ae6fb2d.jpg)

### 어그리게잇으로 묶기
![ddd4](https://user-images.githubusercontent.com/84000890/124348180-6f0cad80-dc23-11eb-9f66-e840ab1499e9.jpg)

    - 쿠폰, 구매, 결제와 같이 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![ddd5_바운드추가](https://user-images.githubusercontent.com/84000890/124348184-7338cb00-dc23-11eb-90f5-9a9881d73d93.jpg)

    - 도메인 서열 분리 
        - Core Domain: 쿠폰, 구매, 결제 : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 쿠폰, 구매, 결제의 경우 1주일 1회 미만
        - Supporting Domain:  -- : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.

### 폴리시 부착및 컨텍스트 매핑

![ddd6](https://user-images.githubusercontent.com/84000890/124348188-7764e880-dc23-11eb-876e-0d187d4a7ead.jpg)
↓ View Model 추가
![ddd7_고객센터추가](https://user-images.githubusercontent.com/84000890/124348191-7b910600-dc23-11eb-92ec-3843190a626b.jpg)


### 완성된 1차 모형(영문으로 변경)

![ddd최종](https://user-images.githubusercontent.com/84000890/124348194-8055ba00-dc23-11eb-99e9-7419fea37ba7.jpg)
  

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![ddd점검1](https://user-images.githubusercontent.com/84000890/124348200-8481d780-dc23-11eb-899a-0b14361429a1.jpg)

    - 고객이 쿠폰을 구매한다 (ok)
    - 구매가 완료되면 결제가 진행된다 (ok)
    - 구매 내역이 결제정보에 전달되는 동시에, 쿠폰 재고 수량이 변경된다. (ok)
    - 결제는 구매내역(구매수량*금액)통해 결제가 진행된다. (ok)
    - 결제가 완료되면, 결제완료상태(PayCompleted)로 변경된다. (ok)

![ddd점검2](https://user-images.githubusercontent.com/84000890/124348202-86e43180-dc23-11eb-997e-a357528acb8f.jpg)

    - 고객이 쿠폰 구매를 취소할 수 있다 (ok)
    - 고객이 쿠폰 구매를 취소하면 쿠폰 재고 수량이 변경된다. (ok)
    - 고객은 구매 상태와 결제 상태 등을 중간중간 조회한다. (View-green sticker 의 추가로 ok) 


### 비기능 요구사항에 대한 검증

![ddd점검3](https://user-images.githubusercontent.com/84000890/124348204-89468b80-dc23-11eb-83e6-4cbac11247fb.jpg)

    - ① : 구매가 완료되면 쿠폰의 구매 가능한 수량이 변경되어야 한다. (Req/Res)
    - ② : 결제 시스템이 정상 수행되지 않더라도 쿠폰구매는 365일 24시간 받을 수 있어야 한다. (Pub/sub)
    - ③ : 쿠폰 구매 시스템이 과중되면 잠시동안 구매를 받지 않고 잠시후에 진행하도록 유도한다 (Circuit breaker)
    - ④ : 고객이 쿠폰구매정보를 별도의 고객페이지에서 확인할 수 있어야 한다 (CQRS)


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/84000863/122320373-15d32780-cf5d-11eb-95b7-23935cda9bb4.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084이다)

```
cd coupon
mvn spring-boot:run

cd order
mvn spring-boot:run 

cd pay
mvn spring-boot:run  

cd customercenter
mvn spring-boot:run

```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 아래 coupon가 그 예시이다.

```
package coupan;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Coupon_table")
public class Coupon {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long couponId;
    private String couponName;
    private Integer amt;
    private Integer stock;


.....


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getCouponId() {
        return couponId;
    }

    public void setCouponId(Long couponId) {
        this.couponId = couponId;
    }
    public String getCouponName() {
        return couponName;
    }

    public void setCouponName(String couponName) {
        this.couponName = couponName;
    }
    public Integer getAmt() {
        return amt;
    }

    public void setAmt(Integer amt) {
        this.amt = amt;
    }
    public Integer getStock() {
        return stock;
    }

    public void setStock(Integer stock) {
        this.stock = stock;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package coupan;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="coupons", path="coupons")
public interface CouponRepository extends PagingAndSortingRepository<Coupon, Long>{
    Coupon findByCouponId(Long couponId);

}
```
- 적용 후 REST API 의 테스트 : 쿠폰구매(order) 시, 쿠폰(coupon) 시스템 내 coupon 수량 수정되므로 order 서비스에 대한 정상 처리 여부를 확인한다.

```
# order 서비스의 쿠폰구매 처리
http POST http://localhost:8082/orders couponId=1 customerId=2 amt=15000 qty=2 orderDate=202107020010 status=Ordered

# 구매 상태 확인
http GET http://localhost:8082/orders

```


## 폴리글랏 퍼시스턴스

coupon 서비스와 order 서비스는 h2 DB로 구현하고, 그와 달리 pay 서비스의 경우 Hsql DB로 구현하여, MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다.

- coupon, order 서비스의 pom.xml 설정
```
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
```
- pay 서비스의 pom.xml 설정
```
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>

```
## CQRS

Viewer를 별도로 구현하여 아래와 같이 view가 출력된다.

- myPage 구현

↓ 쿠폰구매 완료 시, 데이터 생성
```
  public void whenOrdered_then_CREATE_1 (@Payload Ordered ordered) {
        try {

            if (ordered.isMe()) {            

                // view 객체 생성
                Mypage mypage = new Mypage();
                // view 객체에 이벤트의 Value 를 set 함
                mypage.setOrderId(ordered.getId());
                mypage.setCustomerId(ordered.getCustomerId());
                mypage.setStatus(ordered.getStatus());
                mypage.setQty(ordered.getQty());
                mypage.setOrderDate(ordered.getOrderDate());
                // view 레파지 토리에 save
                mypageRepository.save(mypage);
            }

        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
↓ 쿠폰결제 완료 시 또는 쿠폰구매취소 시에 상태값과 취소처리일시 등이 변경된다.
```
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPayed_then_UPDATE_1(@Payload Payed payed) {
        try {
            if (payed.isMe()) {
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByOrderId(payed.getOrderId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setPayAmt(payed.getPayAmt());
                    mypage.setPayDate(payed.getPayDate());
                    mypage.setStatus(payed.getStatus());
                    mypage.setPayCancelDate(payed.getPayCancelDate());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }   

        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderCancelled_then_UPDATE_2(@Payload OrderCancelled orderCancelled) {
        try {
            if (orderCancelled.isMe()){
                // view 객체 조회
                List<Mypage> mypageList = mypageRepository.findByOrderId(orderCancelled.getId());
                for(Mypage mypage : mypageList){
                    // view 객체에 이벤트의 eventDirectValue 를 set 함
                    mypage.setOrderCancelDate(orderCancelled.getOrderCancelDate());
                    mypage.setStatus(orderCancelled.getStatus());
                    // view 레파지 토리에 save
                    mypageRepository.save(mypage);
                }
            }    

        }catch (Exception e){
            e.printStackTrace();
        }
    }

```

- 쿠폰구매 후의 myPage

![image](https://user-images.githubusercontent.com/84000890/124371590-32da5b00-dcbe-11eb-8566-f1981887c6e0.png)

- 구폰구매 취소 후의 myPage (변경된 상태 노출값 확인 가능)

![image](https://user-images.githubusercontent.com/84000890/124371594-3a99ff80-dcbe-11eb-971f-67e1623526dd.png)



## 동기식 호출(Req/Resp)

분석단계에서의 조건 중 하나로 구매(order)→쿠폰(coupon)간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 쿠폰서비스를 호출하기 위하여 FeignClient를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (order) CouponService.java

package coupan.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.Date;

//@FeignClient(name="coupon", url="http://localhost:8081")
@FeignClient(name="coupon", url="http://coupon:8080")

public interface CouponService {
    @RequestMapping(method= RequestMethod.GET, path="/chkAndModifyStock")
    public boolean modifyStock(@RequestParam("couponId") Long couponId,
                            @RequestParam("qty") Integer qty);

}
```

- 예약된 직후(@PostPersist) 재고수량이 업데이트 되도록 처리 (modifyStock 호출)
```
# Order.java

 @PostPersist
    public void onPostPersist(){
        boolean rslt = OrderApplication.applicationContext.getBean(coupan.external.CouponService.class)
        .modifyStock(this.getCouponId(), this.getQty());

        if (rslt) {
            this.setStatus("ordered");
            //this.setStatus(System.getenv("STATUS"));

            Ordered ordered = new Ordered();
            SimpleDateFormat format1 = new SimpleDateFormat ( "yyyyMMddHHmmss");
            String orderDate = format1.format (System.currentTimeMillis());
            this.setOrderDate(orderDate);
            BeanUtils.copyProperties(this, ordered);
            ordered.publishAfterCommit();            
        }


        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.


    }
    
```

- 재고수량은 아래와 같은 로직으로 처리
```
public boolean modifyStock(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
                boolean status = false;
                Long couponId = Long.valueOf(request.getParameter("couponId"));
                int qty = Integer.parseInt(request.getParameter("qty"));

                Coupon coupon = couponRepository.findByCouponId(couponId);

                if(coupon != null){
                        if (coupon.getStock() >= qty) {
                                coupon.setStock(coupon.getStock() - qty);
                                couponRepository.save(coupon);
                                status = true;
                        }
                }

                return status;
        }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 쿠폰 시스템(coupon)이 장애가 나면 예약도 못하는 것을 확인

쿠폰 서비스(coupon)를 잠시 내려놓음 (ctrl+c)

↓쿠폰구매하기(order)
```
http POST http://localhost:8082/orders couponId=2 customerId=5 amt=18000 qty=2 orderDate=202107022100 status=ordered
```
< Fail >
↓ 쿠폰구매 시 500 error 발생
![image](https://user-images.githubusercontent.com/84000890/124350012-0e826e00-dc2d-11eb-8500-d58fc1bfe71c.png)


- 쿠폰(coupon) 서비스 재기동
```
cd coupon
mvn spring-boot:run
```

- 쿠폰구매하기(order)
```
http POST http://localhost:8082/orders couponId=1 customerId=1 amt=15000 qty=2 orderDate=2021070100100 status=Ordered
```
< Success >

![image](https://user-images.githubusercontent.com/84000890/124350126-cd3e8e00-dc2d-11eb-9800-5864beba66b1.png)

- 쿠폰 수량 확인을 통해 정상 req/res 처리 여부 확인
↓ 최초 쿠폰 등록 시 1,000건 등록

![image](https://user-images.githubusercontent.com/84000890/124350557-4939d580-dc30-11eb-967e-9d4f15006dde.png)

↓ 동일 쿠폰에 대해 2건 구매 완료

![req_res_2_쿠폰구매_빨간박스추가](https://user-images.githubusercontent.com/84000890/124350323-06c3c900-dc2f-11eb-8b1e-cc07db587734.jpg)

↓ 쿠폰 재고가 998건으로 구매된 쿠폰 수만큼 감소

![image](https://user-images.githubusercontent.com/84000890/124350614-856d3600-dc30-11eb-917d-93905da88747.png)



## Gateway 적용

- gateway > applitcation.yml 설정

```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: coupon
          uri: http://localhost:8081 
          predicates:
            - Path=/coupons/** , /chkAndModifyStock/**
        - id: order
          uri: http://localhost:8082
          predicates:
            - Path=/orders/** 
        - id: pay
          uri: http://localhost:8083
          predicates:
            - Path=/payments/** 
        - id: customercenter
          uri: http://localhost:8084
          predicates:
            - Path= /mypages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: coupon
          uri: http://coupon:8080
          predicates:
            - Path=/coupons/** , /chkAndModifyStock/** 
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/** 
        - id: pay
          uri: http://pay:8080
          predicates:
            - Path=/payments/** 
        - id: customercenter
          uri: http://customercenter:8080
          predicates:
            - Path= /mypages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
```

- gateway 테스트

```
http POST http://localhost:8088/orders couponId=2 customerId=3 amt=18000 qty=4 orderDate=202107022050 status=Ordered 
```
↓ gateway 8080포트로 order 처리 시,  8082 포트로 링크되어 정상 처리

![image](https://user-images.githubusercontent.com/84000890/124371661-0115c400-dcbf-11eb-8184-d1a396996a52.png)


## 비동기식 호출(Pub/Sub)

- 구매 시스템(order)에서 처리 후에 결제 시스템(pay)에서 결제되는 행위는 동기식이 아니라 비 동기식으로 처리하여 결제 시스템(pay)의 결제 처리로 인해 구매시스템이 블로킹 되지 않도록 처리한다.

- 이를 위하여 구매완료 되었음을 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
   @PostPersist
    public void onPostPersist(){
        boolean rslt = OrderApplication.applicationContext.getBean(coupan.external.CouponService.class)
        .modifyStock(this.getCouponId(), this.getQty());

        if (rslt) {
            this.setStatus("ordered");
            //this.setStatus(System.getenv("STATUS"));

            Ordered ordered = new Ordered();
            SimpleDateFormat format1 = new SimpleDateFormat ( "yyyyMMddHHmmss");
            String orderDate = format1.format (System.currentTimeMillis());
            this.setOrderDate(orderDate);
            BeanUtils.copyProperties(this, ordered);
            ordered.publishAfterCommit();            
        }
    }
```
- 결제 시스템(pay)에서는 구매가 발생하면 이벤트를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다. (subscribe)
```
 public void wheneverOrdered_PreparePayment(@Payload Ordered ordered){

        if(ordered.isMe()){            
            Payment pay = new Payment();
            pay.setOrderId(ordered.getId());
            pay.setCouponId(ordered.getCouponId());        
            pay.setStatus("PayCompleted");
            pay.setPayAmt(ordered.getAmt() * ordered.getQty());

            SimpleDateFormat format1 = new SimpleDateFormat ( "yyyyMMddHHmmss");
            String payDate = format1.format (System.currentTimeMillis());
            pay.setPayDate(payDate);
            
            paymentRepository.save(pay);
        }
    }
    
...

```
- 구매 시스템(order)는 결제 시스템(pay)와 완전히 분리되어있으며(sync transaction 없음) 이벤트 수신에 따라 처리되기 때문에, pay 서비스가 유지보수로 인해 잠시 내려간 상태라도 쿠폰 구매가 진행해도 문제 없다.(시간적 디커플링)
  
↓ 업체(store) 서비스를 잠시 내려놓음(ctrl+c)

![image](https://user-images.githubusercontent.com/84000890/124371908-4d620380-dcc1-11eb-81d0-4236ceb29d47.png)

↓ 쿠폰 구매하기(order)
```
http POST http://localhost:8088/orders couponId=2 customerId=4 amt=18000 qty=1 orderDate=202107022055 status=Ordered
```

< Success >

![image](https://user-images.githubusercontent.com/84000890/124371962-b3e72180-dcc1-11eb-929b-afd287346282.png)


## Deploy / Pipeline

- 소스 가져오기 [이미지 및 경로 : 수정] !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```
git clone https://github.com/kary000/car.git
```
![image](https://user-images.githubusercontent.com/84000863/122197088-cea05480-ced2-11eb-8a64-9c2d55b41240.png)

- 빌드하기
```
cd coupon
mvn package

cd customercenter
mvn package

cd gateway
mvn package

cd order
mvn package

cd pay
mvn package
```
[수정 :  이미지 } !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
![image](https://user-images.githubusercontent.com/84000863/122197418-1de68500-ced3-11eb-8b10-7820e8a354b8.png)

- 도커라이징(Dockerizing) : Azure Container Registry(ACR)에 Docker Image Push하기
```
cd booking
az acr build --registry user05skccacr --image user05skccacr.azurecr.io/booking:latest .

cd customercenter
az acr build --registry user05skccacr --image user05skccacr.azurecr.io/customercenter:latest .

cd gateway
az acr build --registry user05skccacr --image user05skccacr.azurecr.io/gateway:latest .

cd product
az acr build --registry user05skccacr --image user05skccacr.azurecr.io/product:latest .

cd store
az acr build --registry user05skccacr --image user05skccacr.azurecr.io/store:latest . 
```
![image](https://user-images.githubusercontent.com/84000863/122322876-22597f00-cf61-11eb-90ff-bb7b26b1c21f.png)

- 컨테이너라이징(Containerizing) : Deployment 생성
```
cd product
kubectl apply -f kubernetes/deployment.yml

cd booking
kubectl apply -f kubernetes/deployment.yml

cd store
kubectl apply -f kubernetes/deployment.yml

cd customercenter
kubectl apply -f kubernetes/deployment.yml

cd gateway
kubectl create deploy gateway --image=user05skccacr.azurecr.io/gateway:latest

kubectl get all
```

- 컨테이너라이징(Containerizing) : Service 생성 확인
```
cd product
kubectl apply -f kubernetes/service.yaml

cd booking
kubectl apply -f kubernetes/service.yaml

cd store
kubectl apply -f kubernetes/service.yaml

cd customercenter
kubectl apply -f kubernetes/service.yaml

cd gateway
kubectl expose deploy gateway --type=LoadBalancer --port=8080

kubectl get all
```

![image](https://user-images.githubusercontent.com/84000863/122323130-9136d800-cf61-11eb-9dd6-edb2f60952c4.png)


## 서킷 브레이킹(Circuit Breaking)

* 서킷 브레이킹 : istio방식으로 Circuit breaking 구현함

시나리오는 차량등록시 요청이 과도할 경우 CB 를 통하여 장애격리.

outlierDetection 를 설정: 1초내에 연속 한번 오류발생시, 호스트를 10초동안 100% CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정

```
# dr-htttpbin.yaml

outlierDetection:
        consecutive5xxErrors: 1
        interval: 1s
        baseEjectionTime: 10s
        maxEjectionPercent: 100

```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 30초 동안 실시

```
$ siege -c100 -t30S -v --content-type "application/json" 'http://52.231.76.211:8080/products POST {"productId": "1001", "stock":"50", "name":"IONIQ"}'
```

앞서 설정한 부하가 발생하여 Circuit Breaker가 발동, 초반에는 요청 실패처리되었으며
밀린 부하가 product에서 처리되면서 다시 요청을 받기 시작함

- Availability 가 높아진 것을 확인 (siege)

![image](https://user-images.githubusercontent.com/84000863/122497960-454f6600-d029-11eb-9a72-09992ab1baba.png)

- 모니터링 시스템(Kiali) : EXTERNAL-IP:20001 에서 Circuit Breaker 뱃지 발생 확인함.

![image](https://user-images.githubusercontent.com/84000863/122333368-01018e80-cf73-11eb-94c9-c99368bb6636.png)


### Autoscale (HPA)
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 상품(product) 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 1프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy product --min=1 --max=10 --cpu-percent=1

kubectl get hpa
```
![image](https://user-images.githubusercontent.com/84000863/122494497-71b4b380-d024-11eb-948d-97ef4a8628ae.png)

- CB 에서 했던 방식대로 워크로드를 30초 동안 걸어준다.
```
siege -c100 -t30S -v --content-type "application/json" 'http://52.231.76.211:8080/products POST {"productId": "1001", "stock":"50", "name":"IONIQ"}'
```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
watch -n 1 kubectl get pod
```

- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![image](https://user-images.githubusercontent.com/84000863/122336985-a1a67d00-cf78-11eb-8be9-d1700f67ceb2.png)


- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
  (동일 워크로드 CB 대비 2.70% -> 84.10% 성공률 향상)

![image](https://user-images.githubusercontent.com/84000863/122337016-ad923f00-cf78-11eb-86b9-c3a6e08373a0.png)


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c10 -t360S -v --content-type "application/json" 'http://20.41.96.68:8080/products POST {"productId": "1001", "stock":"50", "name":"IONIQ"}'
```
- Readiness가 설정되지 않은 yml 파일로 배포 중 버전1에서 버전2로 업그레이드 시 서비스 요청 처리 실패
```
kubectl set image deploy product user05skccacr.azurecr.io/product:v2
```

![image](https://user-images.githubusercontent.com/84000863/122378900-53f23a80-cfa1-11eb-81ab-2c8b60a8a79b.png)

- deployment.yml에 readiness 옵션을 추가
```
 readinessProbe:
   tcpSocket:
     port: 8080
   initialDelaySeconds: 10
   timeoutSeconds: 2
   periodSeconds: 5
   failureThreshold: 5          
```

- readiness 옵션을 배포 옵션을 설정 한 경우 Availability가 배포기간 동안 변화가 없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

![image](https://user-images.githubusercontent.com/84000863/122379442-d418a000-cfa1-11eb-8b67-fad7b18e64b1.png)

- 기존 버전과 새 버전의 product pod 공존 중

![image](https://user-images.githubusercontent.com/84000863/122379354-c06d3980-cfa1-11eb-97cc-2e28e1902117.png)


## ConfigMap

- Booking 서비스의 deployment.yml 파일에 아래 항목 추가
```
env:
   - name: STATUS
     valueFrom:
       configMapKeyRef:
         name: storecm
         key: status
```

- Booking 서비스에 configMap 설정 데이터 가져오도록 아래 항목 추가

![image](https://user-images.githubusercontent.com/84000863/122492593-04ebea00-d021-11eb-8000-9e6b55d75ffb.png)

- ConfigMap 생성 및 조회
```
kubectl create configmap storecm --from-literal=status=Booked
kubectl get configmap storecm -o yaml
```

![image](https://user-images.githubusercontent.com/84000863/122506477-6455f400-d039-11eb-93b2-2145d21f8e36.png)

- ConfigMap 설정 데이터 조회

![image](https://user-images.githubusercontent.com/84000863/122492475-c5bd9900-d020-11eb-9131-16619f8324ab.png)



## Self-Healing (Liveness Probe)

- 상품(product) 서비스의 deployment.yaml에 liveness probe 옵션 추가

![image](https://user-images.githubusercontent.com/84000863/122364364-a842ed80-cf94-11eb-8a45-980901aeaf3b.png)
 
- product에 liveness 적용 확인

![image](https://user-images.githubusercontent.com/84000863/122364275-95301d80-cf94-11eb-9312-7bbcab0a00d8.png)

- product 서비스에 liveness가 발동되었고, 포트에 응답이 없기에 Restart가 발생함

![image](https://user-images.githubusercontent.com/84000863/122365346-8d24ad80-cf95-11eb-855a-e8e724d273f6.png)
