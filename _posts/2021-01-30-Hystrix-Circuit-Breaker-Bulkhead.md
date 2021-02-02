---
layout: post
title: Hystrix ve Resilient Patterns
featured_img: /images/hystrixlogo.jpg
---

Merhaba,

Bu yazıda hystrix ve resilient patternlerden olan circuit breaker ve bulkheadden bahsedecek ve bir case üzerinde uygulamaya çalışacağız.

## 1. Hystrix

Hystrix, netflix'in open source olarak yazılım dünyasına sunduğu ve dağıtık sistemlerde bağımlılıkların hata durumlarına karşı önlemler almak veya yönetmek anlamında ortaya çıkmış bir kütüphanedir.

## 1.1 Hystrix Neden Ortaya Çıktı?

Ortaya çıkma sebeplerini Netflix kendi github hesabında(link koy) şu şekilde bahsediyor;

- Bağımlılıklardan kaynaklı gecikmelerden ve hatalardan korunmak.

- Kompleks dağıtık bir mimaride hataların katlanarak büyümesini engellemek.

- Hata ortaya çıktığında hemen fail vermesi ve düzeltilmesini sağlamak.

- Fallback metodu belirleyerek hata durumlarında veya sınırların aşılması durumlarında yönlendirme yapmak.

- Gerçek zamanlı görüntüleme ve alert mekanizması kurmak.


Hystrix'in çözmeye çalıştığı problem, dağıtık mimaride yüzlerce servisin olduğu bir uygulamada bir veya bir kaç mikro servisin geç yanıt dönmesi veya dönememesinin kaynaklanan hasarı kullanıcı deneyimine yansıtmamak ve kullanıcının diğer servislere hala erişebiliyor olmasını sağlamak.(Netflix için kullanıcının film veya dizileri izlemesi gibi.)


## 1.3 Resilient Patterns

Uygulamaların daha dayanıklı olmasını sağlamak için aşağıda popüler olan resilient patternler gösterilmiştir. Bunlardan ilk ikisine bakacağız.

* Circuit Breaker Pattern
* Bulkhead Pattern
* Retry
* Fallback
* Timeout
* Stateless Services


### 1.3.1 Circuit Breaker

Circuit breaker, servisler arasındaki iletişimde kullanıcın belirlediği hata oranlarının aşılması durumunda aktif duruma geçerek aradaki bağlantıyı kesen(devreyi açan) bir yönetim şeklidir. Hata alınan servise yapılan istekler engellenir ve kullanıcın tanımladığı bir fallback metoduna yönlendirilir. Bu sayede hata alınan servise sürekli istek atılması engellenmiş olur. Bir süre sonra servisin durumunun düzelip düzelmediğini anlamak için birkaç örnek request yönlendilir ve yanıt beklenir. Eğer yanıtlar başarılı ise devre kapanır ve akış tekrardan eski halini alır. Servisten yine hata gelmesi durumunda devre açılır ve yine kullanıcın belirlediği süre(sleep window) kadar gelen requestler reddedilir.


### 1.3.2 Bulkhead

Bulkhead pattern, bir servisin geç cevap vermesi veya timeout'a düşmesi halinde tüm threadlerin kullanılmasını engellemek veya farklı bir deyişle ilgili servisinin eşzamanlı maksimum n kadar requeste/thread'e cevap verebilir olmasının ayarlanmasını sağlamak ile ilgilidir. Bulkhead sayesinde ilgili uygulamanın belli bir kısmı yanıt veremez olduğunda diğer kısımlarının hala çalışıyor olması sağlanır. Hystrix'in burda iki farklı isolation davranışı mevcut. 

Thread isolation, defult olarak isolation stratejisidir ve her gelen çağırımlar ayrı, fixed bir threadpool'a taşınır ve orada yönetilir. Ana thread'den izole etmenin avantajı eğer çağrılan servis timeout alır ise ana thread yoluna devam eder.

Semaphore isolation, ayrı bir thread'de yönetmez(örneğin tomcat threadinden devam eder) ve ana thread'de hystrix komutunu çağırır. Burda belirlenen max eşzamanlı çağırım sayısına kadar izin verir. Ayrı bir thread olmadığından timeout alırsa thread isolationdaki gibi yoluna devam edemez.

## 1.4 Hystrix Nasıl Çalışıyor?


Hystrix, tüm dışarıya çıkan veya içerindeki çağırımları command pattern yardımıyla ile sararak farklı bir thread de çalıştırıyor. Bu sayede kendi içerisinde tuttuğu metriklere göre de gelen isteklerin belirlediğimiz değerleri aşıp aşmadığını bilebiliyor. Örneğin timeout süresi veya thread sayısının belirli bir threshold üzerine çıkmaması gibi. Adım adım neler yaptığını inceleyelim;


![_config.yml]({{ site.baseurl }}/images/hystrix-command-flow-chart.png)

- HystrixCommand veya HystrixObservableCommand oluşturulur.

- Command tetiklenir veya çalıştırılır.(execute, queue, observe, toObservable)

- Bu command için request caching açık ise response cache de var mı kontrol eder ve varsa anında döner.

- Cache yok ise command execute edildiğinde hystrix circuit breaker açık mı diye kontrol eder. Eğer açık ise (buna tripped de denir.) komut çalıştırılmaz ve direk fallback metoduna yönlendirilir. Eğer circuit breaker kapalı ise thread veya queue kapasite kontrolü yapılır.

- Threadpool veya queue tam kapasitedeyse hystrix komutu çalıştırmaz ve direk fallback metoduna yönlendirir.

- Bu noktada artık hystrix command çalıştırılır.

- Command çalıştırıldığında (ayrı thread kullanıldığında) eğer çalıştırma belirlenen timeout süresinde bitmez ise TimeoutException fırlatılır ve fallback metoduna yönlendirilir.

- Herhangi bir hata alınmaz ise hystrix loglama ve metrik toplama işlemlerinden sonra responsu döner.

- Yukarıdaki metriklerden kastım aslında circuit breaker için success, failure, rejection ve timeout gibi bilgileri toplar ki bir sonraki command de bu bilgiler ile gidişatı belirleyecektir.


## 1.4 Hystrix Configs

## 1.4.1 Threadpool Config

	**coreSize:** Minimum hazırda bulunması gereken thread sayısı.
	**maximumSize:** coreSize geçildiğinde çıkılacak maksimum thread sayısı.
	**allowMaximumSizeToDivergeFromCoreSize:** coreSize aşıldığında maximumSize'ın kullanılması için bir flag. true ise maximumSize a çıkar. false ise maximumSize çalışmaz.
	**keepAliveTimeMinutes:** coreSize aşıldığında açılan threadlerin kullanılmadığı taktirde ne kadar dakikada kapatılacağı bilgisi.

## 1.4.2 Circuit Breaker Config
 
	**requestVolumeThreshold:** Hystrix'in hata oranlarına bakacağı request sayısı(örneğin sondan itibaren 20 requeste bak gibi.).
	**sleepWindowInMilliseconds:** Circuit breakerın açık kalacağı süre.
	**errorThresholdPercentage:**  Hata yüzde sınırı.
	
	
Diğer tüm configler için Hystrix'in ilgili github wiki sayfasından bakılabilir(Link).

### 1.4.3 Configleri Nasıl Ayarlamalı?

Hystrix'i ilk implemente edildiğinde config değerlerinin nasıl ayarlanması gerektiği ile alakalı kafamızda soru işaretleri oluşuyor. Yapılması gerekenden daha az kaynak verirsek uygulamalarımızın çoğu isteği reddetme ihtimali var. Yada kaynağı çok verirsek histrixden bir fayda alamama gibi de bir ihtimalimiz var. Hystrix dökümantasyonunda bu konuda bir kesinlik olmadığını söylerken prod sistemlerde zaman içinde doğru değerlerin bulunacağından bahsediyor.

Hystrix'i hayatımızı dahil ettikten sonra uygulamalarımızı canlı ortamda izlerken hata oranlarının arttığını görürüz ve hystrix'in reject ettiği isteklerin artışı bizi ister istemez config değerlerini arttırmaya zorlayabilir. Değerleri doğru bir şekilde vermiş isek bu kütüphaneyi kullanarak ortaya çıkarmak istediğimiz strateji de bu zaten. Uygulamalarımızı olağan dışı hallerden korumak. Her olağan dışı halde parametreleri arttırırsak kütüphanenin kullanımı bir anlam ifade etmeyebilir.

Peki optimum değerleri en başta nasıl vereceğiz? Hystrix'in github sayfasında bununla ilgili bir formül ortaya çıkarılmış.

	**Request per second at peak when healty** * **99th percentile latency in seconds** + **breathing room**

Uygulamamızı zaten bir tool yardımıyla veya loglar ile monitor ediyorsak, gelen request sayıları ortalama olarak bellidir. Yoğun bir günde saniyedeki request sayısını, servisin responselarının %99 nu kapsayan süre(%99th percentile latency şeklinde geçer. Response'ların %99'unun bu süreden az olması öngörülüyor. ) ile çarparak nefes alacak kadar da bir ekleme sonucu ortaya bir değer çıkıyor. Bu değer optimal olmamakla birlikte başlangıç olarak bir fikir verebilir. 


![_config.yml]({{ site.baseurl }}/images/thread-configuration.png)

## 1.5 Tomcat ve Hystrix Threadleri

Tomcat ve Hystrix threadlerinin nasıl çalıştığını inceleyelim.

## 1.5.1 Tomcat configs

	tomcat:
      max-threads: Maksimum thread sayısı.
	  connection-timeout: Thread'in connection timeout süresi (ms).
	  min-spare-threads: Minimum hazırda bekleyen thread sayısı.
	  max-connections: Servera yapılabilecek maksimum bağlantı sayısı.
	  accept-count: Kuyrukta bekletilen connection sayısı.

## 1.5.2 Threadler

Hystrix komutu çalıştırıldığında "Hystrix-commandname" şeklinde bir thread açılır. Buna paralel olarak "HystrixTimer" adında bir thread daha açılır. Timer threadi sizin komutunuz çalıştırıldığında timeout alıp almamasını kontrol eden threaddir.

Hystrix, her komut geldiğinde coreSize değerine kadar thread açar ve bu threadler de müsait olan tomcat threadlerini kullanırlar. Hystrix coreSize değerine kadar her komutta diğer hystrix threadleri müsait olsalarda farklı bir thread açılır.

Eşzamanlı olarak komut gönderildiğinde eğer tomcat tarafında yeterli thread yok ise tomcat threadleri açılmaya başlanıyor. coreSize değeri minimum tomcat spare değerinden büyük olduğunda hystrix threadleri halen tomcat threadlerini kullandığı için var olan müsait tomcat threadleri tekrardan minimum seviyesine inmiyor. Örneğin coreSize 20, tomcat minimum spare thread değeri 15, maximum 25 olsun. Eşzamanlı gelinen 20 komuttan sonra bekleyen hystrix threadi 20 olacaktır. Bu nedenle bekleyen tomcat thread sayısı da 20 olacaktır.

coreSize değeri aşıldığında maximumSize değerine kadar hystrix thread açabiliyor(Eğer allowMaximumSizeToDivergeFromCoreSize değeri true yapılmış ise). Bu da yeterli gelmediğinde tomcat threadi açıyor anlamına geliyor. Sonrasında keepAliveTimeMinutes değerine göre kullanılmayan hystrix threadleri kendini terminate ediyor ve tomcat threadini de kapatmış oluyor.


## 1.6 Request Collapsing

Hystrix'de çok kısa süren fakat yoğun istek alan servisler için hystrix commandlerini toplayarak tek thread ve networkde kullanım mümkündür. Buna Request Collapsing deniyor ve Netflix'in ihtiyaçları neticesinde ortaya çıkmıştır. Normalde her histrix komutu bir thread açarak ilgili bağımlılığa gidiyor ve istek gönderiyor. Çok kısa zamanlarda(histrix dökümantasyonunda 10ms ve daha az olarak belirtilmiş) birden fazla command geliyor ise bu requestleri toplayıp tek thread açarak ilgili bağımlılığa gitmek request collapsing ile mümkün oluyor. Kullanılmasının en önemli nedeni eşzamanlı çalıştırılan histrix commandleri için thread ve network connection kullanım sayılarını azalmak.

Netflix ideal olarak globalde tüm tomcat threadlerinde bunu uyguluyor. Fakat tek bir user ve threadinde de yapmak mümkün.


![_config.yml]({{ site.baseurl }}/images/collapser.png)


## 2.0 Use Case


Aşağıdaki gibi istekler gateway üzerinden X uygulamasına, ordan ise belirlenen uygulamalara rest istekler atarak istenen cevaplar dönülmektedir. Bu yapıda başlıca olabilecekleri sıralayalım.


![_config.yml]({{ site.baseurl }}/images/hystrix.png)


- Her gelen istek X uygulamasından dağıtıldığı için herhangi bir bağımlılıkta beliren hata, geç cevap verme veya timeoutların uygulamadaki tüm threadleri kullanabilecek olması ve diğer uygulamalarda sorun olmamasına rağmen X uygulamasında kullanılacak thread kalmaması.

- Bağımlılıklarda oluşan hatalarda veya timeoutlarda  uygulamaları sürekli istekler ile boğma problemi. Ve akabinde ne yapılması gerektiği belirsizliği.


Ayrı ayrı uygulamalara circuit breaker pattern'i uygulayabileceğimiz gibi, X uygulamasına da hem circuit breaker hem bulkhead pattern i uygulayabiliriz. Spring bize bazı anotasyonlar ile kolay bir şekilde hystrix command oluşturarak circuit breaker metodu oluşturmamıza yardımcı oluyor fakat burada durum biraz farklı. gatewayden gelen her isteğin ayrıştırılarak dinamik olarak hangi uygulamaya gitmesi gerektiğine karar veriyoruz ve o uygulamaya rest call atıyoruz. ilgili bilgiler de yine db de tutularak karar verildiğini farz edelim. Ama dinamik olarak nasıl circuit breaker uygulayabiliriz?

Bizim case'imizde X e gelen her isteğin header bilgisinden bilgisini pars ederek hangi threadpool'u kullanması gerektiğini söylemek gerekiyordu. Ana amaç A,B,C,D ve E uygulamalarının belirli bir threadpool'lara ayrılmasını sağlamak. Örneğin;

X uygulaması üzerinde A isimli bir threadpool oluşturarak değişkenleri tanımlıyoruz. Diğer uygulamalar içinde kullanması gereken threadpool özelliklerini belirliyoruz.

	hystrix: 
	  threadpool:
		A: 
		  coreSize: 50
		  maximumSize: 200
		  keepAliveTimeMinutes: 1
		  allowMaximumSizeToDivergeFromCoreSize: true
		B:
		  coreSize: 400
		  maximumSize: 600
		  keepAliveTimeMinutes: 1
		  allowMaximumSizeToDivergeFromCoreSize: true
		default: ...



Bu değerlere göre gelen isteklerin hangi uygulamalara ve kullanması gereken threadpoollarına kadar belirlemiş oluyoruz. Hysrix de A uygulamasına gelen çağrıların thread kullanımlarını sınırlayabilmemizi olanak tanıyor. A uygulamasına gelen yoğun isteklerin ve akabinde timeout almaya başlamasıyla X deki tüm threadleri tüketmesini engelleyerek diğer isteklerin hala çalışabilmesini sağlıyoruz.


Kısaca değişkenlere bakalım;

	coreSize: minimum hazırda bulunması gereken thread sayısı.
	maximumSize: coreSize geçildiğinde çıkılacak maksimum thread sayısı.
	allowMaximumSizeToDivergeFromCoreSize: coreSize aşıldığında maximumSize'ın kullanılması için bir flag. true ise maximumSize a çıkar. false ise maximumSize çalışmaz.
	keepAliveTimeMinutes: coreSize aşıldığında açılan threadlerin kullanılmadığı taktirde ne kadar dakikada kapatılacağı bilgisi.


Şimdi X uygulamasına gelen isteklerin nasıl dinamik olarak hystrix command'ine çevrildiğine bakalım;

```
@Component
public class ErkanHystrixCircuitBreakerFactory extends CircuitBreakerFactory<HystrixCommand.Setter, ErkanHystrixCircuitBreakerFactory.ErkanHystrixConfigBuilder> {

	@Override
	public CircuitBreaker create(String id) {
		HystrixCommand.Setter setter;
		Assert.hasText(id, "A CircuitBreaker must have an id.");

		if (this.getConfigurations().containsKey(id)) {
			setter = this.getConfigurations().get(id);
		} else {
			setter = HystrixCommand.Setter.withGroupKey(HystrixCommandGroupKey.Factory.asKey(id + "-group"))
					.andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey(id))
					.andCommandKey(HystrixCommandKey.Factory.asKey(id));
			this.getConfigurations().put(id, setter);
		}
		return new HystrixCircuitBreaker(setter);

	}

	@Override
	protected ErkanHystrixConfigBuilder configBuilder(String id) {
		return new ErkanHystrixCircuitBreakerFactory.ErkanHystrixConfigBuilder(id);
	}

	@Override
	public void configureDefault(Function<String, HystrixCommand.Setter> defaultConfiguration) {
	}

	public static class ErkanHystrixConfigBuilder extends AbstractHystrixConfigBuilder<HystrixCommand.Setter> {
		public ErkanHystrixConfigBuilder(String id) {
			super(id);
		}

		public HystrixCommand.Setter build() {
			return HystrixCommand.Setter.withGroupKey(this.getGroupKey()).andCommandKey(this.getCommandKey()).andCommandPropertiesDefaults(this.getCommandPropertiesSetter());
		}
	}
}
```


Bir factory class'ı yaratarak CircuitBreakerFactory class'ını extend ediyoruz ve create metodumuzu oluşturuyoruz. Burda bunu yapmamızdaki amaç her gelen istekte eğer configürasyonu yapmamış isek onu yapmak, eğer yapmış isek onu get ederek bir circuit breaker oluşturmak. Şimdi çağırdığımız yere bakalım;


```
public CircuitBreaker createCircuitBreaker(String name) {
    return erkanHystrixCircuitBreakerFactory.create(name);
}
```

	
"name" aslında bizim uygulama adımız. A ve B gibi. A değerini vererek factory class'ı A nın threadpool'unu kullanarak bir istek oluşturuyor.
	
	
```
public Response run(String name, Supplier<Response> supplier) {
    String id = setNameIfItIsNull(name);// burda gruplayabiliriz veya default adında bir threadpool açarak bazılarını default'a yönlendirebiliriz.
    return createCircuitBreaker(name).run(supplier, t -> callFallback(id, t));
}
```

Ayrıca bir fallback metodumuz da mevcut. Burda circuit open olduğunda veya herhangi bir hatada fallback metodumuza düşecek ve gerekli aksiyonu almış olacağız.(Alert, log,  default response gibi)

circuit breaker ve isolation stratejilerini tanımlayalım.

	requestVolumeThreshold: hystrix' değerlendireceği request sayısı.
	sleepWindowInMilliseconds: Circuit open kalacağı süre.
	errorThresholdPercentage:  hata yüzde sınırı.
	
	
Hystrix, aşağıdaki değerleri baz alırsak A için son 20 isteğe bakacak. 16 dan fazla istek hatalı ise circuit open olacak ve diğer gelen istekler reddedilecek. Bu süre de 2 saniye.

	hystrix: 
	  command:
	    A:
	      circuitBreaker:
		    requestVolumeThreshold: 20
			sleepWindowInMilliseconds: 2000
			errorThresholdPercentage: 80
			
		  execution:
		    isolation: 
			  thread: 
			    timeoutInMilliseconds: 30000
	    B:
	      circuitBreaker:
		    requestVolumeThreshold: 20
			sleepWindowInMilliseconds: 2000
			errorThresholdPercentage: 80
			
		  execution:
		    isolation: 
			  thread: 
			    timeoutInMilliseconds: 30000


	timeoutInMilliseconds: Streteji default olarak THREAD'dir. Semaphore' a çevrilmesi istenirse ayrıca belirtilmesi gerekiyor. Biz thread ile ilerledik ve timeoutInMilliseconds değerini 30 saniye olarak belirledik. Ayrı olarak açılan thread'in time out süresi 30 sn.
	
	maxConcurrentRequests: Sadece strateji semaphore olduğunda çalışır. eşzamanlı karşılanacak maksimum request sayısı.



Sonuç olarak, X uygulamasının diğer herhangi bir servisin geç cevap vermesi veya timeout alması halinde diğer tüm servislerin etkilenmemesi için bu şekilde bir strateji oluşturduk.

Teşekkürler.