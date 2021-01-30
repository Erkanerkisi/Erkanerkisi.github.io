---
layout: post
title: Hystrix-Circuit-Breaker-Bulkhead-patterns
featured_img: /images/view1.jpg
---

Merhaba,

Bu yazımda hystrix, circuit breaker ve bulkhead patternlerinden bahsedecek ve bir case üzerinde uygulamaya çalışacağız.

Hystrix, netflix'in open source olarak yazılım dünyasına sunduğu ve dağıtık sistemlerde bağımlılıkların hata durumlarını tolere etmek veya yönetmek anlamında ortaya çıkmış bir kütüphanedir. Ortaya çıkma sebeplerini netflix şu şekilde bahsediyor;

- Bağımlılıklardan kaynaklı gecikmelerden ve hatalardan korunmak
- Komplex dağıtık bir mimaride hataların katlanarak büyümesini engellemek
- Hata ortaya çıktığında hemen fail vermesi ve düzeltilmesini sağlamak
- Hata ortaya çıktığında fallback uygulamak
- gerçek zamanlı görüntüleme ve alert mekanizması kurmak


Aslında hystrix'in çözmeye çalıştığı ana sorun, dağıtık mimaride yüzlerde servisin olduğu bir uygulamada bir veya bir kaç mikro servisin geç yanıt dönmesi veya dönememesinin verdiği hasarı kullanıcı deneyimine yansıtmamak ve kullanıcının diğer servislere hala erişebiliyor olması.(Netflix için kullanıcının film veya dizileri izlemesi gibi.)


Peki hystrix bunu nasıl başarıyor? Tüm dışarıya çıkan veya içerindeki çağırımları command pattern ile sararak farklı bir thread de çalıştırıyor. Çağırımları kendisi sardığı için bizim belirlediğimiz değişkenlere göre hızlıca aksiyon alabiliyor mesela timeout süresi veya thread sayısının belirli bir threshold üzerine çıkmaması gibi.

Circuit Breaker pattern

Circuit breaker bir pattern olmak ile birlikte hystrix bu pattern'i implemente ederek yazılım dünyasına açmıştır. Buradaki amaç iki sistem arasında çağıran taraf, çağrılan taraftan hata almaya başlarsa ve bu hata oranları bizim belirlediğimiz çıtayı aşarsa bir fallback metodunun tetiklenmesi sağlanabileceği gibi daha fazla istek atmak yerine çağrılan tarafın kendini iyileştirmesi beklenir. Bu sayede çağrılan tarafda bir yoğunluk oluşmasının önüne de geçilmiş olur.

Yine bizim belirlediğimiz değerlere göre servisin düzelip düzelmediğini anlamak için servis örneklem yapar ve üst üste bir sayıda başarılı olursa tekrardan devreyi kapatarak servise istekler yönlendirilir.

Bulkbead pattern

Bulkhead pattern ise bir servisin geç cevap vermesi veya timeout'a düşmesi halinde tüm threadlerin kullanılmasını engellemek veya farklı bir deyişle ilgili servisinin eşzamanlı maksimum n kadar requeste/thread'e cevap verebilir olmasının ayarlanmasını sağlamak. Hystrix'in burda iki farklı isolation davranışı mevcut. 

Thread isolation, defult olarak isolation stratejisidir ve her gelen çağırımlar ayrı, fixed bir threadpool'a taşınır ve orada yönetilir. Ana thread'den izole etmenin avantajı eğer çağrılan servis timeout alır ise ana thread yoluna devam eder.

Semaphore isolation, ayrı bir thread'de yönetmez(örneğin tomcat threadinden devam eder) ve ana thread'de hystrix komutunu çağırır. Burda belirlenen max eşzamanlı çağırım sayısına kadar izin verir. Ayrı bir thread olmadığından timeout alırsa thread isolationdaki gibi yoluna devam edemez.


CASE

Şekildeki gibi istekler gateway üzerinden X uygulamasına, ordan ise belirlenen uygulamalara rest call'lar ile responselar dönülmektedir. Bu yapıda başlıca olabilecekleri sıralayalım.

- Her gelen istek X uygulamasından dağıtıldığı için herhangi bir bağımlılıkta beliren hata, geç cevap verme veya timeoutların uygulamadaki tüm threadleri kullanabilecek olması ve diğer uygulamalarda sorun olmamasına rağmen X uygulamasında kullanacak thread kalmaması.

- Bağımlılıklarda oluşan hatalarda veya timeoutlarda daha bağımlı olan uygulamaları istekler ile boğma problemi. Ve akabinde ne yapılması gerektiği belirsizliği


Ayrı ayrı uygulamalara circuit breaker pattern'i uygulayabileceğimiz gibi, X uygulamasına da hem circuit breaker hem bulkhead pattern i uygulayabiliriz. Spring bize bazı anotasyonlar ile kolay bir şekilde hystrix command oluşturarak circuit breaker metodu oluşturmamıza yardımcı oluyor fakat her gatewayden gelen isteği ayrıştırarak hangi uygulamaya gitmesini istediğimizi biliyoruz ama dinamik olarak nasıl circuit breaker uygulayabiliriz?

Bizim case'imizde X e gelen her isteğin header bilgisinden bilgisini pars ederek hangi threadpool'u kullanması gerektiğini söylemek gerekiyordu. Ana amaç A,B,C,D ve E uygulamalarının belirli bir threadpool'lara ayrılmasını sağlamak. Örneğin;

X uygulaması üzerinde A isimli bir threadpool oluşturarak değişkenleri tanımlıyoruz. Diğer uygulamalar içinde kullanması gereken threadpool özelliklerini belirliyoruz.

hystrix: 
  threadpool:
    A: 
	  coreSize: 
	  maximumSize: 
	  maxQueueSize: 
	  queueSizeRejectionThreshold: 
	  keepAliveTimeMinutes: 
	  allowMaximumSizeToDivergeFromCoreSize: 
	  timeInMilliseconds: 
	B:
	  coreSize: 
	  maximumSize: 
	  maxQueueSize: 
	  queueSizeRejectionThreshold: 
	  keepAliveTimeMinutes: 
	  allowMaximumSizeToDivergeFromCoreSize: 
	  timeInMilliseconds: 
	default: ...

Bu değerlere göre gelen isteklerin hangi uygulamalara ve kullanması gereken threadpoollarını belirlemiş oluyoruz. Hysrix de bu şekilde A uygulamasına gelen çağrıların thread kullanımlarını sınırlayabilmemizi sağlıyor. A uygulamasına gelen yoğun isteklerin ve akabinde timeout almaya başlamasıyla X deki tüm threadleri tüketmesini engelleyerek diğer isteklerin diğer uygulamalara gitmesini sağlamış oluyoruz.




Teşekkürler.




 