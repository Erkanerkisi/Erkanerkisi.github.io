---
layout: post
title: Docker - k8s - Microservices part 1
---

Merhaba,

k8s ile playground yaptığım notlarımı aşağıda bulabilirsiniz. İki mikroservis içeren ve birbiriyle konuşan bu yapıyı docker ile cantainerized ederek k8s de yönetmeye çalışacağım.

Yüklü olmasını beklediğim toollar: Docker, docker içindeki k8s veya minikube etc., k8s dashboard (detayları daha güzel görmek için).

Elimde iki tane springboot projesi var. 

	1. person-service
	2. person-title-service

Person-service projesinden bir person id vererek kişinin bilgilerini ve title bilgisini alabileceğimiz bir rest servisimiz var. person-title-service e rest call atıp title bilgisini alıyoruz.

Projeleri ayrı ayrı dockerize ediyoruz. Her biri için Dockerfile oluşturuyoruz.


FROM openjdk:8-jdk-alpine
ARG JAR_FILE=target/*.jar
EXPOSE 8080
COPY ${JAR_FILE} /project/person-service.jar
ENTRYPOINT ["java","-jar","project/person-service.jar"]

FROM openjdk:8-jdk-alpine
ARG JAR_FILE=target/*.jar
EXPOSE 8080
COPY ${JAR_FILE} /project/person-title-service.jar
ENTRYPOINT ["java","-jar","project/person-title-service.jar"]

Yukarıda expose demesekte springboot tomcat üzerinde default 8080 den kalkıyor. İçerde properties de server.port vermiş isek ordan expose etmemiz gerekir.

imageları lokalimizde oluşturuyoruz.

	docker build --tag person-service .

	docker build --tag person-title-service .
	
Şimdi sıra deployment ve service yaml dosyalarını oluşturmaya. Deployment yaml dosyası sizin podunuzu nasıl yöneteceğinizi içeren declarative bir rehber. 

k8s dökümantasyonundan detaylarına bakabilirsiniz. 

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: person-service-deploy
	spec:
	  replicas: 1
	  selector:
		matchLabels:
		  app: person-service
	  template:
		metadata:
		  labels: 
			app: person-service
		spec: 
		  containers:
		  - name: person-service
			image: person-service:latest
			imagePullPolicy: Never
			ports:
			- containerPort: 8080

Yukarıda image ismini belirtiyoruz(image:) ve port kısmında containerPort aslında sizin oluşturduğunuz image ile birlikte projeniz hangi porttan expose olacak ise onu yazıyoruz. Container içindeki app inizin dinlenilen portu diyebiliriz.

Aynısını person-title-service içinde oluşturuyoruz.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: person-title-service-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: person-title-service
  template:
    metadata:
      labels: 
        app: person-title-service
    spec: 
      containers:
      - name: person-title-service
        image: person-title-service:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8080

Her ikisinde de imagePullPolicy never a çektim. çekmeyince image'ı docker hubda aramaya çalışıyor ve bulamıyor. Burasını netleştirmem lazım.

Tasarım olarak her pod da tek conteiner olacak şekilde yapıyoruz. Genel anlayış bu şekilde ilerliyor. Eğer tek poda birden fazla container koymak isterseniz bunun gerçekten iyi bir nedeni olmalı. Yoksa tek podda iki veya daha fazla container olması durumunda birbirlerine bağlamış olacaksınız. birindeki bir restart diğerini de etkileyecektir.Tek pod tek container ile her servisi 8080 den sunabiliriz. 

Tek podun içerisinde birden fazla container olduğu durumda, her iki container da aynı porttan hizmet veremiyor. Çünkü podun içindeki containerlar aynı network yapısını paylaşıyorlar. Bu nedenle farklı portlardan expose etmek gerekir.

Yukarıdaki deployment yaml dosyalarını oluşturduktan sonra kubectl ile çalıştıralım.

	kubectl apply -f dosyanın-locationı\deploy-person-service.yml
	kubectl apply -f dosyanın-locationı\deploy-person-title-service.yml

Çalıştıktan sonra podlara ve deploymentları inceleyebiliriz.

	kubectl decribe pods
	kubectl describe deployments
	
Dashboard kısmını ayarlayalım. k8s dökümantasyon sitesinde dashboard için recomended.yaml dosyası mevcut. Bunu aldıktan sonra içinde security kısmını skip etmek için aşağıdakini koyuyoruz. (image: kubernetesui/dashboard:v2.0.0 kısmına)

	- --enable-skip-login
    - --disable-settings-authorizer

Bunun sebebi dashboard açıldığında bir token vs istiyor. Dökümantasyonda user açılması ve token alınması gibi işlemlerden kaçmak için yaptık. Şimdilik öğrenme amacıyla uğraştığızdandolayı security kısmını discard ettik.

şimdi dışarı expose edelim.

	kubectl proxy

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=default

istenen dashboard a yukarıdaki linkle giriş yapıp inceleyebiliriz.

Deploymentlardan sonra service yaml dosyalarını oluşturmamız lazım. Service yaml dosyaları bizim podlarımızın iplerini ve portlarını yöneten ve bir kere oluşturulması yeterli olan rehberdir. Podlar ölebilir ve yerine yenileri gelir. her podun bir ipsi vardır ve podlar arasında iletişimde bu ipleri kullanmak sakıncalıdır. Sebebi pod öldüğünde yerine yenisi geldiğinde yeni bir ip alır. her defasında projenin içine bu pod iplerini yazmamız mantıksız olacağından imdadımıza servis katmanı geliyor.

Servis katmanı static bir ip alır ve label ile podlarımıza loadbalancer görevi görür. Bu sayede farklı bir poddan başka bir poda bir istek atacak isek service ipsini ve expose ettiğimiz portu vermemiz yetecektir. Bizim podumuzun ölmesi veya 1 yerine 10 tane pod ayağa kaldırdığımızda hepsi farklı ip alsa da servis katmanı bunu yönetiyor olacak.

person-service yaml:

	apiVersion: v1
	kind: Service
	metadata:
	  name: person-service-svc
	  labels:
		app: person-service
	spec:
	  type: NodePort
	  ports:
	  - port: 8080
		nodePort: 30002
		protocol: TCP
	  selector:
		app: person-service

perosn-title-service yaml:

	apiVersion: v1
	kind: Service
	metadata:
	  name: person-title-service-svc
	  labels:
		app: person-title-service
	spec:
	  type: NodePort
	  ports:
	  - port: 8089
		targetPort: 8080
		nodePort: 30001
		protocol: TCP
	  selector:
		app: person-title-service

Port dediğimiz cluster içinde erişilebilecek servisin portu.
Nodeport dediğimiz cluster dışı servisimize erişilebilecek servisin portu.
targetPort dediğimiz poddaki dinlenilen port.Servis bu porta request atacaktır. 


Genel Hatlarıyla:
![_config.yml]({{ site.baseurl }}/images/overall.png)


Sonrasında test için ilgili nodeportlardan lokalimizde test edelim.

localhost:30002/swagger-ui.html




