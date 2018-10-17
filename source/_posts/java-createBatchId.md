---
title: 生成16位数字订单号
date: 2016-12-30 10:16:07
tags: UUID
categories: java
---


UUID是指在一台机器上生成的数字，它保证对在同一时空中的所有机器都是唯一的，这个不重复性全世界人民都知道。当然，既然字符串值不重复，那对应的hashCode也是一样，不会重复。

```java
public static String getOrderIdByUUId() {
         int hashCodeV = UUID.randomUUID().toString().hashCode();  
         if(hashCodeV < 0) {//有可能是负数  
             hashCodeV = - hashCodeV;  
         }   
         return String.format("%016d", hashCodeV);  
     }
```
该方法出自一下三种方法的第三种方案



__前提背景__  
相信做过银联支付的都知道，银联的订单号要求商户提供一个不重复的16位数字订单号（不重复指的是对商户本身，不用考虑银联有多个商户会与其他商户的订单号重复）。16位数其实很短，要考虑每秒并发1w或者10w或100万时，重复订单号将数不过来。

需要考虑的因素：  
 若使用数据库保存流水号，集群部署时，同步关键字不再有效。当然同步对性能也有非常大的影响；  
 若使用时间，必须要精确到毫秒、微妙级别，长度就不止16位了。  
 若使用数据库字段自增，数据库并发时硬件将吃不消。  
 获取订单号时检查表的最大值，这种方案是最不可取的。  

以下将给出本人经过深入研究的三种方案，按顺序，最优的方案为第三个。  
备注：  
如果要测试产生重复订单号的情况，可以建立一个表，把订单号字段设置为唯一性，然后开启1000或10000或更多的线程去请求方法，每个线程循环5次或10次来请求，在方法里面写插入语句。或者可以使用Apache的ab工具并发测试。
使用方法：ab -n5000 -c5000 http://192.168.1.102:8888/kjcx/aaa.action

__1. 可选方案一__  
本方案使用的是当前时间，包括毫秒数、纳秒数，不需要数据库参与计算，性能不用说。
算法：  
```java
 OrderId=  
machineId+  
 (System.currentTimeMillis()+"").substring(1)+  
(System.nanoTime()+"").substring(7,10);  
```

讲解：  
参数machineId：是集群时的机器代码，可以1-9任意。部署时，分别为部署的项目手动修改该值，以确保集群的多台机器在系统时间上不一致的问题（毫无疑问每台机器的毫秒数基本上不一致）。  
参数System.currentTimeMillis():这是java里面的获取1970年到目前的毫秒数,是一个13位数的数字，与Date.getTime()函数的结果一样，比如1378049585093。经过研究，在2013年，前三位是137，在2023年是168，到2033年才199.所以，我决定第一位数字1可以去掉，不要占位置了。可以肯定绝大多数系统用不了10年20年。这样，参数2就变成了12位数的数字，加上参数1machineId才13位数。
参数System.nanoTime()：这是java里面的取纳秒数，经过深入研究，在同一毫秒内，位置7,8,9这三个数字是会变化的。所以决定截取这三个数字出来拼接成一个16位数的订单号。  
总结：理论上此方案在同一秒内，可以应对1000*1000个订单号，但是经过测试，在每秒并发2000的时候，还是会出现2-10个重复。

__2. 可选方案二__  
本方案使用的是获得会话ID（sessionId）来产生hashCode。  
算法：  
```java
 OrderId=  
 machineId+  
 session().getId().hashCode();
```
讲解：  
参数machineId不再讲解，与方案一致。  
参数2 session().getId().hashCode()是值在web系统中获取用户浏览器与web容器的唯一会话编号，再把该会话ID转换为该字符串的hashCode值，如1939354961。该值可能是一个11位数的或10位数的，或者在前面还会出现-号，也就是有可能该值是负数，没关系，取正。然后再对该值进行左补0到15位数，基本上可以应对位数不一致的问题。  
我们知道，hashCode是jdk根据对象的地址或者字符串或者数字算出来的int类型的数值。可以想象，hashCode的值如果出现重复，那就是一个值了，而不是不同的值。又因为sessionId是客户端、与浏览器有关联的，所以基本上不会出现重复，但是如果用户在同一个会话有效期内、同一个版本的浏览器，生成2次就无效了，因为会话ID是一致的。
总结：该算法，可以确保不重复的概率很小，但是需要自己特殊处理同会话同浏览器生成1次以上订单号的问题，此算法没有经过调试，略过，您请看方案三。  

__3. 可选方案三__  
本方案在基于方案二的基础上做了修改，使用的使用UUID而不是会话id。  
UUID是指在一台机器上生成的数字，它保证对在同一时空中的所有机器都是唯一的，这个不重复性全世界人民都知道。当然，既然字符串值不重复，那对应的hashCode也是一样，不会重复。  
算法：  
```java
 OrderId=  
 machineId+  
 UUID.randomUUID().toString().hashCode();  
```

讲解：  
参数1不再解释。  
参数2是值生成UUID然后取它的hashCode值，经过测试，完全没有一点问题。您可以开1000w的并发去测试插入吧，只要数据库不会报唯一性错误，那就没问题。  
总结：  
hashCode这个算法从搞软件开始到现在这么多年，一直没派上用场，这次大大的用上了。解决了问题。请同志们以后善用这个东西。  

 __. 附录：方案三的算法代码__  
 ```java
   public static String getOrderIdByUUId() {  
           int machineId = 1;//最大支持1-9个集群机器部署  
           int hashCodeV = UUID.randomUUID().toString().hashCode();  
           if(hashCodeV < 0) {//有可能是负数  
              hashCodeV = - hashCodeV;  
          }  
          // 0 代表前面补充0       
          // 4 代表长度为4       
          // d 代表参数为正数型  
         return machineId+String.format("%015d", hashCodeV);  
      }  
 ```
 方案三其实也就一个函数，很简便。

参考 [http://www.cezuwang.com/listFilm?page=1&areaId=906&filmTypeId=1](http://www.cezuwang.com/listFilm?page=1&areaId=906&filmTypeId=1)