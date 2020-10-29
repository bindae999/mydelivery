# mydelivery


## 서비스 시나리오
![image](https://user-images.githubusercontent.com/68535067/97397347-65aca200-192c-11eb-810a-049c2f12ae7b.png)

## saga
- 주문접수(Order)->주문확인 후 주문취소(shop)->주문상태값  RequestCancel로 변경(Order)
![image](https://user-images.githubusercontent.com/68535067/97414001-cba62300-1946-11eb-8bd8-2ba7a9d2cde5.png)
![image](https://user-images.githubusercontent.com/68535067/97409032-533c6380-1940-11eb-9a6f-8d12c5d5a98a.png)

## 동기식 호출 Request-Response
- Shop의 DeliveryService.java
```
package mydelivery.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="Delivery", url="${api.url.Delivery}")
//@FeignClient(name="Delivery", url="http://Delivery:8080")
public interface DeliveryService {

    @RequestMapping(method= RequestMethod.POST, path="/deliveries")
    public void delivery(@RequestBody Delivery delivery);

}
```
-- shop의 Shop.java
```
@PostUpdate
    public void onPostUpdate(){
        RequestCanceled requestCanceled = new RequestCanceled();
        BeanUtils.copyProperties(this, requestCanceled);

        if(requestCanceled.getOrderStatus().equals("request Cancel")){
            requestCanceled.setOrderStatus(this.getOrderStatus());
            requestCanceled.publishAfterCommit();
        }else{
            DeliveryRequested deliveryRequested = new DeliveryRequested();
            BeanUtils.copyProperties(this, deliveryRequested);
            deliveryRequested.publishAfterCommit();

            //Following code causes dependency to external APIs
            // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
            mydelivery.external.Delivery delivery = new mydelivery.external.Delivery();
            System.out.println("배달요청 접수!!"+this.getOrderId());
            delivery.setOrderId(this.getOrderId());
            delivery.setOrderStatus("Delivery Request");

            // mappings goes here
            ShopApplication.applicationContext.getBean(mydelivery.external.DeliveryService.class)
                    .delivery(delivery);

        }
    }
```
-증빙화면
- 배달요청(Shop)-배달접수(Delivery)는 동기식으로 연동되어있음, delivery서비스를 내려놓고 shop에서 요청을 하게되면 장애발생
![image](https://user-images.githubusercontent.com/68535067/97410951-04dc9400-1943-11eb-8a83-bd369d22dec8.png)

## 비동기식 호출 PUB/SUB(장애격리)
- Order의 Order.java
```
    @PostPersist
    public void onPostPersist(){
        OrderRequested orderRequested = new OrderRequested();
        BeanUtils.copyProperties(this, orderRequested);
        //pss : 오더주문할때는  기본값  "Order"로 세팅
        orderRequested.setOrderStatus("Order");
        orderRequested.setOrderId(this.getOrderId());
        orderRequested.publishAfterCommit();

    }
```
 - Shop의 PolicyHandler.java
 ```
 @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderRequested_CreateOrder(@Payload OrderRequested orderRequested){

        if(orderRequested.isMe()){
            System.out.println("##### listener CreateOrder : " + orderRequested.toJson());
            int flag=0;

            //pss : 오더접수 수신시 객체 생성(update)
            Iterator<Shop> iterator = shopRepository.findAll().iterator();
            while(iterator.hasNext()){
                Shop shopTmp = iterator.next();
                if(shopTmp.getOrderId() == orderRequested.getOrderId()){
                    System.out.println("$$$$$"+shopTmp.getOrderId());
                    flag=1;
                    Optional<Shop> ShopOptional = shopRepository.findById(shopTmp.getOrderId());
                    Shop shop = ShopOptional.get();
                    shop.setOrderStatus(orderRequested.getOrderStatus());
                    shopRepository.save(shop);
                }
            }

            //pss : 기존오더가 없다면 신규생성(insert)
            if (flag==0){
                System.out.println("신규 접수!");
                Shop shop = new Shop();
                shop.setOrderStatus(orderRequested.getOrderStatus());
                shop.setFoodName(orderRequested.getFoodName());
                shop.setFoodQty(orderRequested.getFoodQty());
                shop.setOrderId(orderRequested.getOrderId());
                shopRepository.save(shop);
            }

        }
    }
 ```

-증빙화면
- 주문요청(order)-주문접수(Shop)는 비동기식으로 연동되어있음, Shop서비스를 내려놓고 주문요청을 하여도 주문서비스는 가능함
![image](https://user-images.githubusercontent.com/68535067/97409648-54ba5b80-1941-11eb-95ac-458e226783dd.png)


## Gateway
-소스적용
![image](https://user-images.githubusercontent.com/68535067/97443175-0d968f80-196e-11eb-8891-c668474d2bfb.png)

-증빙화면
![image](https://user-images.githubusercontent.com/68535067/97443014-d45e1f80-196d-11eb-8e04-3acf7a92c3bb.png)

## CQRS
View인 orderBoard는 실시간으로 주문상태를 확인할 수 있음.
![image](https://user-images.githubusercontent.com/68535067/97413158-c7c5d100-1945-11eb-93ed-4a0cdf8d26c2.png)

## SelfHealing-Liveness Probe
- 소스화면
![image](https://user-images.githubusercontent.com/68535067/97511878-77915200-19cb-11eb-9a1e-af62a283fe8a.png)
- resatrt 시도 횟수
![image](https://user-images.githubusercontent.com/68535067/97511814-516bb200-19cb-11eb-9849-44edc9d2a128.png)
- describe 확인 : Liveness는 8081을  확인, 실제  서비스포트는 8080
![image](https://user-images.githubusercontent.com/68535067/97512001-bf17de00-19cb-11eb-9ab8-55785e97bc71.png)

## 무정지 배포-Readnees Probe

## CB
-  소스설정
```
# application.yml
```
![image](https://user-images.githubusercontent.com/68535067/97445003-1e480500-1970-11eb-8825-eb922e480085.png)

- 증빙화면
![image](https://user-images.githubusercontent.com/68535067/97458004-9ff25f80-197d-11eb-9bdc-6c9890a6fa3a.png)

## 오토스케일러 (HPA)

## CI/CD
- CI
![image](https://user-images.githubusercontent.com/68535067/97433120-58110f80-1960-11eb-8c54-389ce5ecb63a.png)
- CD
![image](https://user-images.githubusercontent.com/68535067/97433936-9bb84900-1961-11eb-9666-93b474411843.png)

## configMap

## 폴리글랏
