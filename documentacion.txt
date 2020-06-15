Crear nodo
eksctl create nodegroup \
--cluster test-cluster \
--version auto \
--name test-workers \
--node-type t3.medium \
--node-ami auto \
--nodes 1 \
--nodes-min 1 \
--nodes-max 3 \
--asg-access


Eliminar nodo
eksctl delete nodegroup --cluster=test-cluster --name=test-workers

INGRESS CONTROLLER
https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html
	
	Se va dar permisos al service acount (se srea en el paso 4) para poder crear en aws el load balancer, por lo que se crea un IAM police, donde se define los permisos para los balanceadores y estas politicas se le esta atachando al service account

	1.
	eksctl utils associate-iam-oidc-provider \
	    --region us-east-1 \
	    --cluster test-cluster \
	    --approve

	2.    
	curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/iam-policy.json
    
    3.
	aws iam create-policy \
	    --policy-name ALBIngressControllerIAMPolicy \
	    --policy-document file://iam-policy.json
    
    4.
	kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/rbac-role.yaml    


	5. [## En la variable attach-policy-arn va el arn que devolvio como respuesta al ejecutar el paso 3, en caso no tener entrar a IAM->policies y buscar ALBIngressControllerIAMPolicy y se obtendra el valor del arn ##]

		eksctl create iamserviceaccount \
		--region us-east-1 \
		--name alb-ingress-controller \
		--namespace kube-system \
		--cluster test-cluster \
		--attach-policy-arn arn:aws:iam::111111111111:policy/ALBIngressControllerIAMPolicy
		--override-existing-serviceaccounts \
		--approve

	6. [## Se creará el ingress controller con el yaml proporcionado por aws ##]	

		kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/alb-ingress-controller.yaml

		[##  En casose necesite editar algunos valores en el deplyment del ingress controller ejecuatar kubectl edit deployment.apps/alb-ingress-controller -n kube-system  ##]

    7. [## Se necesita agregar un valor en el deployment pos lo que se necesita editarlo y agregar la sigueinte linea ##]

    	spec:
	      containers:
	      - args:
	        - --ingress-class=alb
	        - --cluster-name=test-cluster    ----> linea que se debe agregar

	8. [## para corroborar que esta ok se debe obtener los pod del namespace kube-system ##]        

		kubectl get pods -n kube-system



Ejecutar una app de pruebas

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-service.yaml


Antes de crear el ingress para el app validemos que los pods y la app esta operativa hacienndo un forward al puerto 80 del pod desde nuestro local

	- listamos los pods dentro del namespace 2048-game -> kubectl get pods -n 2048-game
	- hacemos un froward al puerto 80 de alguno de los pods creados para que se vea en el puerto 7000 de nuestro local host -> 
	  kubectl port-forward <nombre-del-pod> -n 2048-game 7000:80 
	- En el navegador ingresamos a localhost:7000  


kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.4/docs/examples/2048/2048-ingress.yaml

kubectl get ingress -n 2048-game

Ejecutamos en un navegador el address y debe cargar el app


Cambiar reglas del ingress para aceder solo desde un subdominio

- obtener la ip haciendo ping o nslookup al address
- crear un registro en el virtual host
- Editar yaml de ingress y luego remplazar en la etiqueta spec por lo que esta debajo -> kubectl edit ingress 2048-ingress -n 2048-game  

				spec:
				  rules:
				  - host: appgame.awstest.com
				    http:
				      paths:
				      - backend:
				          serviceName: service-2048
				          servicePort: 80


- Para corroborar que todo esta ok verificar los logs del ingress controller que es un pod que esta en el namespace kube-system -->
  kubectl logs -f -n kube-system <nombre-del-pod>
- Ingresar a la URL para ver el app funcionanod, con esa nueva regla solo se podrá ingresar al app con esta url y ya no non la ip o address del ingress