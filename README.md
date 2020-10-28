# mydelivery


## 서비스 시나리오
![image](https://user-images.githubusercontent.com/68535067/97397347-65aca200-192c-11eb-810a-049c2f12ae7b.png)

## saga
- 주문접수(Order)->주문확인 후 주문취소(shop)->주문상태값  RequestCancel로 변경(Order)
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


-증빙화면


## CQRS

## SelfHealing-Liveness Probe

## 무정지 배포-Readnees Probe

## CB

## 오토스케일러 (HPA)

## CI/CD

## configMap

## 폴리글랏
