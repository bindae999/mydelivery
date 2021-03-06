# mydelivery


## 서비스 시나리오
![image](https://user-images.githubusercontent.com/68535067/97397347-65aca200-192c-11eb-810a-049c2f12ae7b.png)

## saga
- 주문접수(Order)->주문확인 후 주문취소(shop)->주문상태값  RequestCancel로 변경(Order)
![image](https://user-images.githubusercontent.com/68535067/97525815-0bbee180-19eb-11eb-8520-7eb821d2a376.png)

![image](https://user-images.githubusercontent.com/68535067/97525856-27c28300-19eb-11eb-82fe-175aea1a1cdb.png)

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

![image](https://user-images.githubusercontent.com/68535067/97533101-f2be2c80-19fa-11eb-996a-74d947a5aa2e.png)

## CQRS
View인 orderBoard는 실시간으로 주문상태를 확인할 수 있음.
--콘솔화면으로 대체--

## SelfHealing-Liveness Probe
- 소스화면
![image](https://user-images.githubusercontent.com/68535067/97511878-77915200-19cb-11eb-9a1e-af62a283fe8a.png)
- resatrt 시도 횟수
![image](https://user-images.githubusercontent.com/68535067/97511814-516bb200-19cb-11eb-9849-44edc9d2a128.png)
- describe 확인 : Liveness는 8081을  확인, 실제  서비스포트는 8080
![image](https://user-images.githubusercontent.com/68535067/97512001-bf17de00-19cb-11eb-9ab8-55785e97bc71.png)

## 무정지 배포-Readnees Probe
- 소스 확인
```
          readinessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
```
- 증빙화면 : 
  1. Order의 deployment.yaml파일의 ReadneesProbe를 주석처리하여 오류 발생시킴
  2. siege로 모니터링 걸어둠
  3. 8080으로 배포하여  정상  서비스로  복원  : 15%정도  실패하다가  정상으로전환하는것을 확인
![image](https://user-images.githubusercontent.com/68535067/97514497-2cc70880-19d2-11eb-8ed8-bf02d8b89f2f.png)

## CB
-  소스설정
```
# application.yml
```
![image](https://user-images.githubusercontent.com/68535067/97445003-1e480500-1970-11eb-8825-eb922e480085.png)

- 증빙화면
![image](https://user-images.githubusercontent.com/68535067/97458004-9ff25f80-197d-11eb-9bdc-6c9890a6fa3a.png)


## 오토스케일러 (HPA)
-autoscle적용
```
kubectl autoscale deploy order --min=1 --max=10 --cpu-percent=15
```
- sige이용 과부하 적용
![image](https://user-images.githubusercontent.com/68535067/97515645-d27b7700-19d4-11eb-9804-ecb4025cfe0b.png)

-  모니터링
![image](https://user-images.githubusercontent.com/68535067/97515733-0b1b5080-19d5-11eb-87ce-91888e880dae.png)


## CI/CD

- CI
![image](https://user-images.githubusercontent.com/68535067/97433120-58110f80-1960-11eb-8c54-389ce5ecb63a.png)
- CD
![image](https://user-images.githubusercontent.com/68535067/97433936-9bb84900-1961-11eb-9666-93b474411843.png)

## configMap
- configmap.yml파일
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: apiurl
data:
  url:  http://delivery:80
  fluentd-sever-ip: 10.xxx.xxx.xxx
```

-  depoyment.yml 파일
```
          env:
            - name: configurl
              valueFrom:
                configMapKeyRef:
                  name: apiurl
                  key: url
```

- application.yml파일 
```
#PSS : REQ/RES 장애격리 처리
api:
  url:
    #Delivery: http://delivery:8080
    Delivery: ${configurl}

feign:
  hystrix:
    enabled: true
```

- DeliveryService.java 파일
```
@FeignClient(name="Delivery", url="${api.url.Delivery}")
//@FeignClient(name="Delivery", url="http://Delivery:8080")
public interface DeliveryService {

    @RequestMapping(method= RequestMethod.POST, path="/deliveries")
    public void delivery(@RequestBody Delivery delivery);

}
```

- 80포트로 설정하여 오류발생확인
![image](https://user-images.githubusercontent.com/68535067/97522634-8dab0c80-19e3-11eb-9320-9438c2b0876b.png)

- configmap 적용확인
![image](https://user-images.githubusercontent.com/68535067/97522567-66543f80-19e3-11eb-83b0-4433d9a7ca96.png)

## 폴리글랏
- Order의  pom.xml에 H2  대신 hsqldb 적용
![image](https://user-images.githubusercontent.com/68535067/97516070-ad3b3880-19d5-11eb-818f-37b0706fdd39.png)

-증빙화면 : DB변경후에도 오더가 정상 저장됨을 확인
![image](https://user-images.githubusercontent.com/68535067/97517037-cb099d00-19d7-11eb-90a5-ff08cbf3301c.png)
