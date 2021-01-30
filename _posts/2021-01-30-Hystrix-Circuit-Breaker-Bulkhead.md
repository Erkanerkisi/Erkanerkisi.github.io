---
layout: post
title: Hystrix-Circuit-Breaker-Bulkhead-patterns
featured_img: /images/view1.jpg
---

Merhaba,

Bu yazımda hystrix, circuit breaker ve bulkhead patternlerinden bahsedecek ve bir case üzerinde uygulamaya çalışacağız.

Hystrix, netflix'in open source olarak yazılım dünyasına sunduğu ve dağıtık sistemlerde bağımlılıkların hata durumlarına karşı önlemler almak veya yönetmek anlamında ortaya çıkmış bir kütüphanedir. Ortaya çıkma sebeplerini netflix şu şekilde bahsediyor;

- Bağımlılıklardan kaynaklı gecikmelerden ve hatalardan korunmak
- Komplex dağıtık bir mimaride hataların katlanarak büyümesini engellemek
- Hata ortaya çıktığında hemen fail vermesi ve düzeltilmesini sağlamak
- Fallback metodu belirleyerek hata durumlarında veya sınırların aşılması durumlarında yönlendirme yapmak
- gerçek zamanlı görüntüleme ve alert mekanizması kurmak


Aslında hystrix'in çözmeye çalıştığı problem, dağıtık mimaride yüzlerce servisin olduğu bir uygulamada bir veya bir kaç mikro servisin geç yanıt dönmesi veya dönememesinin kaynaklanan hasarı kullanıcı deneyimine yansıtmamak ve kullanıcının diğer servislere hala erişebiliyor olmasını sağlamak.(Netflix için kullanıcının film veya dizileri izlemesi gibi.)


Peki hystrix bunu nasıl başarıyor? Tüm dışarıya çıkan veya içerindeki çağırımları command pattern ile sararak farklı bir thread de çalıştırıyor. Çağırımları kendisi sardığı için bizim belirlediğimiz değişkenlere göre hızlıca aksiyon alabiliyor mesela timeout süresi veya thread sayısının belirli bir threshold üzerine çıkmaması gibi. Adım adım neler yaptığına bakalım;


![_config.yml]({{ site.baseurl }}/images/hystrix-command-flow-chart.png.png)

- HystrixCommand veya HystrixObservableCommand oluşturulur.
- Command tetiklenir veya çalıştırılır.(execute, queue, observe, toObservable)
- Bu command için request caching açık ise response cache de var mı kontrol eder ve varsa anında döner.
- Cache yok ise command execute edildiğinde hystrix circuit breaker açık mı diye kontrol eder. Eğer açık ise (buna tripped de denir.) komut çalıştırılmaz ve direk fallback metoduna yönlendirilir. Eğer circuit breaker kapalı ise thread veya queue kapasite kontrolü yapılır.
- threadpool veya queue tam kapasitedeyse hystrix komutu çalıştırmaz ve direk fallback metoduna yönlendirir.
- Bu noktada artık hystrix command çalıştırılır.
- Command çalıştırıldığında (ayrı thread kullanıldığında) eğer çalıştırma belirlenen timeout süresinde bitmez ise TimeoutException fırlatılır ve fallback metoduna yönlendirilir.
- Herhangi bir hata alınmaz ise hystrix loglama ve metrik toplama işlemlerinden sonra responsu döner.
- Yukarıdaki metriklerden kastım aslında circuit breaker için success, failure, rejection ve timeout gibi bilgileri toplar ki bir sonraki command de bu bilgiler ile gidişatı belirleyecektir.

### Circuit Breaker pattern

Circuit breaker bir pattern olmak ile birlikte hystrix bu pattern'i implemente ederek yazılım dünyasına açmıştır. Buradaki amaç iki sistem arasında çağıran taraf, çağrılan taraftan hata almaya başlarsa ve bu hata oranları bizim belirlediğimiz çıtayı aşarsa bir fallback metodunun tetiklenmesi sağlanabileceği gibi daha fazla istek atmak yerine çağrılan tarafın kendini iyileştirmesi beklenir. Bu sayede çağrılan tarafda bir yoğunluk oluşmasının önüne de geçilmiş olur.

Yine bizim belirlediğimiz değerlere göre servisin düzelip düzelmediğini anlamak için servis örneklem yapar ve üst üste bir sayıda başarılı olursa tekrardan devreyi kapatarak servise istekler yönlendirilir.

### Bulkbead pattern

Bulkhead pattern ise bir servisin geç cevap vermesi veya timeout'a düşmesi halinde tüm threadlerin kullanılmasını engellemek veya farklı bir deyişle ilgili servisinin eşzamanlı maksimum n kadar requeste/thread'e cevap verebilir olmasının ayarlanmasını sağlamak ile ilgilidir. Hystrix'in burda iki farklı isolation davranışı mevcut. 

Thread isolation, defult olarak isolation stratejisidir ve her gelen çağırımlar ayrı, fixed bir threadpool'a taşınır ve orada yönetilir. Ana thread'den izole etmenin avantajı eğer çağrılan servis timeout alır ise ana thread yoluna devam eder.

Semaphore isolation, ayrı bir thread'de yönetmez(örneğin tomcat threadinden devam eder) ve ana thread'de hystrix komutunu çağırır. Burda belirlenen max eşzamanlı çağırım sayısına kadar izin verir. Ayrı bir thread olmadığından timeout alırsa thread isolationdaki gibi yoluna devam edemez.


### Request Collapsing

Her histrix command'i bir thread açarak ilgili bağımlılığa gidiyor ve istek gönderiyor. Çok kısa zamanlarda(ki histrix dökümantasyonunda 10ms ve daha az) birden fazla command geliyor ise bu requestleri toplayıp tek thread açarak ilgili bağımlılığa gitmek request collapsing ile mümkündür(Hystrix Collapser). Kullanılmasının en önemli nedeni eşzamanlı çalıştırılan histrix commandleri için thread ve network connection kullanım sayılarını azalmak.

Netflix ideal olarak globalde tüm tomcat threadlerinde bunu uyguluyor. Fakat tek bir user ve threadinde de yapmak mümkün.

![_config.yml]({{ site.baseurl }}/images/collapser.png)


CASE

![_config.yml]({{ site.baseurl }}/images/hystrix.png)

Şekildeki gibi istekler gateway üzerinden X uygulamasına, ordan ise belirlenen uygulamalara rest call'lar atarak istenen responselar dönülmektedir. Bu yapıda başlıca olabilecekleri sıralayalım.

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


`deneme`

`
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
`

Bir factory class'ı yaratarak CircuitBreakerFactory class'ını extend ediyoruz ve create metodumuzu oluşturuyoruz. Burda bunu yapmamızdaki amaç her gelen istekte eğer configürasyonu yapmamış isek onu yapmak, eğer yapmış isek onu get ederek bir circuit breaker oluşturmak. Şimdi çağırdığımız yere bakalım;

```
public CircuitBreaker createCircuitBreaker(String name) {
    return erkanHystrixCircuitBreakerFactory.create(name);
}
```
	
"name" aslında bizim uygulama adımız. A ve B gibi. A değerini vererek factory class'ı A nın threadpool'unu kullanarak bir istek oluşturuyor.
	
	
    public Response run(String name, Supplier<Response> supplier) {
        String id = setNameIfItIsNull(name);// burda gruplayabiliriz veya default adında bir threadpool açarak bazılarını default'a yönlendirebiliriz.
        return createCircuitBreaker(name).run(supplier, t -> callFallback(id, t));
    }

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


X uygulamasının diğer herhangi bir servisin geç cevap vermesi veya timeout alması halinde diğer tüm servislerin etkilenmemesi için bu şekilde bir strateji oluşturduk.

Teşekkürler.