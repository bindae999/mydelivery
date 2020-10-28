# mydelivery


## 서비스 시나리오
![image](https://user-images.githubusercontent.com/68535067/97397347-65aca200-192c-11eb-810a-049c2f12ae7b.png)

## saga
- 주문접수(Order)->주문확인 후 주문취소(shop)->주문상태값  RequestCancel로 변경(Order)
![image](https://user-images.githubusercontent.com/68535067/97409032-533c6380-1940-11eb-9a6f-8d12c5d5a98a.png)

## Request-Response


## PUB/SUB
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
![image](https://user-images.githubusercontent.com/68535067/97400165-fd60bf00-1931-11eb-9bd8-926c69149a7e.png)


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
