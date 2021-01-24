---
layout: post
title: k8s - Configmaps - Secrets - pv - pvc
featured_img: /images/view1.jpg
---

Merhaba,


k8s ile playground yaptığım notlarımı aşağıda bulabilirsiniz. Mikroservis örneğinden devam ederek örnek config map ve secret eklemeye çalışalım.

configmap: cluster içinde insensitive configleri key/value şeklinde tutmaya yarayan bir yapı.

secrets: üstteki tanım ile aynı olmak ile birlikte burda daha çok sensitive dataların tutulması sağlanır. Passwordler vs. tutulabilir. Yine key/value şeklinde ama encoded olarak saklanır.

persistent volume(pv): k8s dökümantasyonunda pv yi api resources olarak tanımlıyor. Aslında birer storage tanımı diyebiliriz. Dinamik veya static storage olarak ayarlanabiliyor.

persistent volume claim(pvc): PV resourcelarını kullanmak için yapılan istektir. deploy veya replica setlerde örneğin bu pv storage yi kullanmak istiyorum yada bu storageClass üzerinden otomatik olarak pv oluştur gibi yapılan isteklere pvc diyoruz.


Şimdi bir tane config map oluşturup person title servisimize koyalım. Ben komut ile oluşturdum ama yaml ile de oluşturulabiliyor.


	kubectl create configmap app-config --from-file="C:\Program Files\k8s\app-config.properties" 
	

İsmine app-config verdim ve windowstaki dosyanın yolunu from file ile vermiş oldum.


	kubectl describe configmap appconfig
	
	
Burdan detaylarına bakabiliriz. Şimdi oluşturduğumuz configmap i kullanalım.



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
			volumeMounts:
			- name: app-config-volume 
			  mountPath: /project/config
		  volumes:
		  - name: app-config-volume
			configMap:
			  name: app-config


person title service içerisinde spec altında volumes tanımlıyoruz. burda config map name ini veriyoruz. Sonrasında bu valume tanımını ilgili container'ımıza(zaten bir tane var) maunt ediyoruz. bu config map deki properties dosyasını project/config altına maunt etmiş olduk.


	kubectl apply -f C:\erkan\projects\springboot-microservices\deploy-person-title-service.yml 


Podu tekrar ayağa kaldırdığımızda config mapin geldiğini kontrol edelim.

	kubectl exec -it person-title-service-deploy-fbd9f45b9-jljrb /bin/sh
	
	cd project/config

Dosya container a mount olmuş durumda. Bundan sonra artık bu dosyayı kullanmamız kalıyor. Nasıl kullanacağımız bize kalmış.

Secretlarda yukarıdaki gibi kullanılabiliyorlar. Volume ve Volume mount ile import edilebildikleri gibi podda environment variables olarak da tanımlanabiliyorlar.

Poda Environment olarak tanımlamak istersek;

	kubectl create secret generic my-secret --from-file="C:\erkan\projects\springboot-microservices\password.txt" --from-file="C:\erkan\projects\springboot-microservices\username.txt" 

	kubectl get secret my-secret -o yaml

	apiVersion: v1
	data:
	  password.txt: bXlwYXNzaXNlYXN5
	  username.txt: amFuZQ==
	kind: Secret
	metadata:
	  creationTimestamp: "2020-12-18T20:38:20Z"
	  managedFields:
	  - apiVersion: v1
		fieldsType: FieldsV1
		fieldsV1:
		  f:data:
			.: {}
			f:password.txt: {}
			f:username.txt: {}
		  f:type: {}
		manager: kubectl.exe
		operation: Update
		time: "2020-12-18T20:38:20Z"
	  name: my-secret
	  namespace: default
	  resourceVersion: "1200933"
	  selfLink: /api/v1/namespaces/default/secrets/my-secret
	  uid: 1a599225-5a1e-4723-9d2a-fd44f2a0745a
	type: Opaque


Yukarıdaki yaml de gözüktüğü üzere secret değerleri base64 encoded halde. bunları şimdi podu ayağa kaldırdıktan sonra environment değişkeni olarak görebilecek miyiz bakalım.

Podun içine bakalım.
	
	kubectl exec -it person-title-service-deploy-69dfb7694b-gb58h /bin/sh

	echo $SECRET_USERNAME
	echo $SECRET_PASSWORD

ulaşabiliyoruz hem de decode halinde.

![_config.yml]({{ site.baseurl }}/images/password.PNG)



