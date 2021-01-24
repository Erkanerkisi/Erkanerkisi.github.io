---
layout: post
title: Dynamic Scheduler and Shedlock
featured_img: /images/view1.png
---

Merhaba,

Bu yazımda ihtiyaç duyduğumuz dinamik scheduler mekanizması ve yine aynı şekilde dinamik bir lock mekanizmasına bakacağız.

Elimizde spring annotasyonları ile kurgulanan bir scheduler uygulaması mevcut (@Scheduler). Siz bu tetikleme tanımlarını bir rest ile tanımlıyor olabilirsiniz veya her yeni bir iş için uygulamaya dokunmak gerekiyor olabilir.
Bunun için aslında db den yönetmek bir çözüm olabilir. Tablolara gerekli eklemeyi yaptığımızda job'ımızın tanımının otomatik olarak yapılacak olması varmak istediğimiz bir sonuç olacaktır.

Böyle bir yapı için arka tarafta @Scheduled anotasyonunun da kullandığı "TaskScheduler" kullanılabilir. Herhangi bir sınıf içerisinde uygulama ayağa kalkarken db den çekilen jobların TaskScheduler ile schedule edilebilmesi sağlanabilir.

	tasks.put(name, taskScheduler.schedule(() -> 
	{System.out.println("NAME IS : " + name);}, 
	new CronTrigger("20 * * * * *")));

Sonrasında  burada tanımını yaptığınız jobların refereanslarını bir yerde tutarak ta yönetebilirsiniz. Örneğin "taskScheduler.schedule" bir "scheduledfuture" dönüyor ve bunları objelere map ederek Set collection'ında tutabilirsiniz.
Bunun için springin ScheduledTaskRegistrar sınıfına göz atmanızı tavsiye ederim. Dinamik olarak tabloya eklendiği anda farkları bulacak ve schedule edecek veya var olan bir job kaldırılmış ise silecek bir job tanımı da yapılabilir(Her 1 dk da çalışan mesela).


Bunları tasarladıktan sonra scheduler uygulamanızı birden fazla instance'da kaldırıyorsanız joblarınız ne kadar instance var ise o kadar sayıda çalışacaktır.
Örneğin saat 1 de kritik bir taskı schedule etmişsiniz ama vakti geldiğinde, 3 instance var ise 3 kere jobınız tetiklenmiş olacaktır.

Bunu engellemek için ihtiyaç görülen nokta bir lock sistemi. Eğer sizin bir instance'ınız jobınızı başlatmış ise diğer instance'larınız pas geçmeli.
Bunun için bir çok kütüphane var fakat siz kendiniz de bir lock sistemi yapabilirsiniz. Tüm instance'ların aynı db ye baktığını varsayarsak bir lock tablosu yapılıp yönetilebilir.

Ben burda scheduled annotasyonu ile sık kullanılan shedlock kütüphanesinden bahsedeceğim. İnternette bir çok yerde ve kendi github hesabından nasıl kullanıldığına bakabilirsiniz.(https://github.com/lukas-krecan/ShedLock)


	@Scheduled(cron = "0 */15 * * * *")
	@SchedulerLock(name = "scheduledTaskName", lockAtMostFor = "14m", lockAtLeastFor = "14m")
	public void scheduledTask() {
	// do something
	}
	
	
name: locklanacak job ismi.
lockAtMostFor: en fazla ne kadar locklı kalacak.
lockAtLeastFor: en az ne kadar locklı kalmalı.


scheduled kısmını anotasyondan kurtardığımız için bunu da anotasyondan kurtarmak gerekecek. Verdiğim github hesabından baktığınızda detaylı olarak nasıl kullanılmış örnekleri ile görebilirsiniz.
LockableTaskScheduler sınıfını kullanmaya çalıştım ama her bir job için yeni bir lockabletaskscheduler yaratmamız gerekiyor. Sebebi ise bu sınıfın cons parametrelerinde, ben de değişkenlik gösteren değişkenler max ne kadar süre locklı kalacağı ve lock ismi.
Bu sınıf şuan bir taskscheduler ve lock manager bekliyor constructor'ında. Lock manager detalarına bakarsanız bu sınıfında sizin değişkenlerinize göre değişiklik göstereceğini fark edeceksiniz.

Bunun yerine ben de framework bağımsız kullanabileceğiniz aşağıdaki yapıyı kullanmaya çalıştım.

	LockingTaskExecutor executor = new DefaultLockingTaskExecutor(lockProvider);

	...

	Instant lockAtMostUntil = Instant.now().plusSeconds(600);
	executor.executeWithLock(runnable, new LockConfiguration("lockName", lockAtMostUntil));
	
	
Şu şekilde bir yapı kurmaya çalıştım.

	class LockRunnable implements Runnable {

		Runnable runnable;
		LockProvider lockProvider;
		String lockName;

		public LockRunnable(Runnable runnable, String lockName, LockProvider lockProvider) {
			this.runnable = runnable;
			this.lockName = lockName;
			this.lockProvider = lockProvider;
		}

		@Override
		public void run() {
			LockingTaskExecutor executor = new DefaultLockingTaskExecutor(lockProvider);
			Instant lockAtMostUntil = Instant.now().plusSeconds(60);//Değişebilir
			executor.executeWithLock(runnable, new LockConfiguration(Instant.now(), lockName, Duration.ofSeconds(60), Duration.ofMillis(1000)));//Değişebilir
		}
	}

	@Service
	public class SchedulerManager {

		private TaskScheduler taskScheduler;
		private Map<String, ScheduledFuture> tasks = new HashMap<>();
		private LockProvider lockProvider;

		public SchedulerManager(TaskScheduler taskScheduler,
								LockProvider lockProvider) {
			this.taskScheduler = taskScheduler;
			this.lockProvider = lockProvider;
		}


		@PostConstruct
		void load() {
			
			String name = "First Job!";
			tasks.put(name, taskScheduler.schedule(wrap(() -> {
				System.out.println("NAME IS : " + name);
			},name), new CronTrigger("20 * * * * *")));
			//*******************************************************************
			String name2 = "Second Job!";
			tasks.put(name2, taskScheduler.schedule(
					wrap(() -> {System.out.println("NAME IS : " + name2);
						}, name2),
					new CronTrigger("30 * * * * *")));
		}

		public Runnable wrap(Runnable runnable, String lockName) {
			return new LockRunnable(runnable, lockName, lockProvider);
		}
	}
	
taskScheduler.schedule a verdiğimiz runnable kısmını wrapleyip lock kullanan mekanizmamızı çalıştırdık. Bu sayede diğer değişkenleri de dinamik olarak verebilmiş oluyoruz.
Testleri yaparken schedulermanager sınıfını çoğaltırsanız aynı job ve isimleri ile aynı zamanda çalışmasını test edebilirsiniz.
Ben bu testleri yaparken joblar sadece "system.out" yazdırdıkları için çok kısa sürede tamamlanıyorlardı. Bu da tüm jobların çalışmasına sebep oluyordu. Ne demek istiyorum loglardan bakalım.


Testlerimi yaparken aşağıdaki metotda farklı bir constructor veriyordum ve lockAtLeastFor parametresini kullanmıyordum. Sonrasında gördüm ki farklı instancedaki joblar aynı anda çalışıyorlardı. Logları incelediğimde;

	new LockConfiguration(Instant.now(), lockName, Duration.ofSeconds(60), Duration.ofMillis(1000))


2021-01-24 12:15:20.007 DEBUG 13900 --- [lTaskScheduler3] n.j.s.p.j.SqlStatementsSource            : Using SqlStatementsSource
2021-01-24 12:15:20.033 DEBUG 13900 --- [lTaskScheduler1] n.j.s.core.DefaultLockingTaskExecutor    : Locked 'First Job!', lock will be held at most until 2021-01-24T09:16:20.002212600Z
NAME IS : First Job!
2021-01-24 12:15:20.037 DEBUG 13900 --- [lTaskScheduler1] n.j.s.core.DefaultLockingTaskExecutor    : Task finished, lock 'First Job!' released
2021-01-24 12:15:20.094 DEBUG 13900 --- [lTaskScheduler2] n.j.s.core.DefaultLockingTaskExecutor    : Locked 'First Job!', lock will be held at most until 2021-01-24T09:16:20.002212600Z
NAME IS : First Job!
2021-01-24 12:15:20.094 DEBUG 13900 --- [lTaskScheduler3] n.j.s.core.DefaultLockingTaskExecutor    : Not executing 'First Job!'. It's locked.
2021-01-24 12:15:20.095 DEBUG 13900 --- [lTaskScheduler2] n.j.s.core.DefaultLockingTaskExecutor    : Task finished, lock 'First Job!' released
2021-01-24 12:15:30.002 DEBUG 13900 --- [lTaskScheduler5] n.j.s.core.DefaultLockingTaskExecutor    : Locked 'Second Job!', lock will be held at most until 2021-01-24T09:16:30.001720100Z
NAME IS : Second Job!
2021-01-24 12:15:30.003 DEBUG 13900 --- [lTaskScheduler6] n.j.s.core.DefaultLockingTaskExecutor    : Not executing 'Second Job!'. It's locked.
2021-01-24 12:15:30.003 DEBUG 13900 --- [lTaskScheduler5] n.j.s.core.DefaultLockingTaskExecutor    : Task finished, lock 'Second Job!' released
2021-01-24 12:15:30.004 DEBUG 13900 --- [lTaskScheduler4] n.j.s.core.DefaultLockingTaskExecutor    : Not executing 'Second Job!'. It's locked.

İlk job'ın 2021-01-24 12:15:20.033 saniyesinde lockladığını görüyoruz. 2021-01-24 12:15:20.037 zamanında da bitmiş ve lock'ı released etmiş. Bu noktada diğer instance'daki job da bu saliseden sonra tetiklendiği için 2021-01-24 12:15:20.094 zamanında lock almış ve jobı tetiklemiş. üçüncü instance da aynı saliselerde (2021-01-24 12:15:20.094) tetiklenmiş ama ikincisi zaten locklamıştı. Dolayısıyla üçüncü instance çalışmamış oldu. Sorun benim joblarda "system out" gibi çok kısa sürede bitecek bir kod çalıştırmam ve aynı salise de çalışıp bitmesi ve diğer instance'ın bu zamandan sonra tetiklenmesi. Yukarıdaki second job tanımında ("30 * * * * *") cron ifadesi her dakikanın 30 uncu saniyesinde tetikle demek ama 30 saniyenin hangi salisesi söylenmiyor. Yukarıdaki debug da bunun kanıtı. Bu noktada kurtarıcı nokta "lockAtLeastFor" parametresi. Burda ilk instance lock'ı aldığında en az ne kadarlık bir lock konacak. Burda bir saniye veya üstü derseniz ozaman diğer instanceların job'ı çalıştırmadığınız göreceksiniz. Tabi instancelar arasındaki zamanın senkron ve aynı zaman dilimlerinde olduğu varsaydım.

Bu nedenle LockConfiguration sınıfının diğer constructorları @deprecated oldu diye düşünüyorum.

Daha detaylı bigileri ve örnekleri shedlock github hesabında bulabilirsiniz.

Teşekkürler.




 