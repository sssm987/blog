---
layout: post
title: "[Concurrency] 재고 차감에서 동시성 문제가 발생하는 이유"
date: 2026-04-27 22:10:00 +0900
categories: [Concurrency]
description: Concurrency가 일어나는 상황을 재현해보고 어떤 현상이 일어나는지 알아보자 
---
개요
--
실무에서 가장 조심해야하고 또 빈번히 일어나 늘 고려해야하는것중 하나가 바로 이 동시성 문제이다.    
실무에서는 이론상 알고있어 방지를 하지만 실제로 어떤식으로 문제가 일어나고 또 서버, DB에서는 어떤 일이 일어나는지 잘 모르기 때문에 실험을 진행하게 되었다.    
이번글에서는 실험을 통해 문제를 발생시키고 측정할것이며 다음 글에 해결 방법과 성능 비교 식으로 진행 할 것이다.
---
실험 환경
-----
실험 환경은 Spring boot,Grafana,Prometheus,k6,PostGreSQL이며 k6를 제외한 나머지를 로컬 미니 PC에 구축하고
동일한 환경에 있는 맥북에 k6를 이용하여 부하를 주어 테스트를 진행할 것 이다.   
![erd]({{ '/assets/images/concurrency/erd.png' | relative_url }})
최대한 간단하게 구현하기 위해 회원은 따로 구현하지않고 숫자로 구분할것이며 상품과 주문은 일대다 관계로 구성했다.
 
아래가 핵심 로직인 주문생성 로직이며 일부러 동시성 문제를 일으키기 위해 서비스에서 상품을 조회 후 재고를 차감 그리고 주문을 저장하는 식으로 구성했다.
~~~
@Transactional
    public void create(OrderCreateCmd cmd){
        Product product = productRepository.findProductsById(cmd.getProductId())
                .orElseThrow(() -> new DomainException(ErrorCode.PRODUCT_NOT_FOUND));

        product.decrease();

        Order order = Order.create(cmd.getMemberId(),product);
        orderRepository.save(order);
    }
~~~
