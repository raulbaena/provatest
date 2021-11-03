# Prova realitzada amb minikube
## Instalació de Minikube
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

## Arrencar minikube
```
raultest@ubuntu:/var/tmp/provatest/provakube$ minikube start
😄  minikube v1.23.2 on Ubuntu 20.04
✨  Using the docker driver based on existing profile
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔄  Restarting existing docker container for "minikube" ...
🐳  Preparing Kubernetes v1.22.2 on Docker 20.10.8 ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```
## Parar minikube
```
raultest@ubuntu:/var/tmp/provatest/provakube$ minikube stop
✋  Stopping node "minikube"  ...
🛑  Powering off "minikube" via SSH ...
🛑  1 nodes stopped.
```
## Explicació dels fitxers realitzats

### wordpress-deployment.yaml
Aixó es un volum persistent es un troç de disc dur on es guarden les dades del pod i no es borren quan aquest es eliminat
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
Aqui es defineix com vols que sigui la app, quina imatge ha d'usar, versió, etc.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:4.8-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim

```
Una manera abstracta d'exposar una aplicació que s'executa en un conjunt de pods com a servei de xarxa.
```
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
```
### mysql-deployment.yaml
Aixó es un volum persistent es un troç de disc dur on es guarden les dades del pod i no es borren quan aquest es eliminat
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
```
Aqui es defineix com vols que sigui la app, quina imatge ha d'usar, versió, etc.
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
Una manera abstracta d'exposar una aplicació que s'executa en un conjunt de pods com a servei de xarxa.
```
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```
## kustomization.yaml
Fitxers amb contraseñas sensibles
```
secretGenerator:
- name: mysql-pass
  literals:
  - password=test
resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml
```
# Fem el deploy de les apps
Per aplicar tots els fitxers i que s'instalin els secrets per la comunicació entre wordpress i mysql fem lo següent.
```
raultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl apply -k ./
secret/mysql-pass-9gtcfc4f5k created
service/wordpress created
service/wordpress-mysql created
persistentvolumeclaim/mysql-pv-claim created
persistentvolumeclaim/wp-pv-claim created
deployment.apps/wordpress created
deployment.apps/wordpress-mysql created
```
Comprovem que s'hagi creat el secret.
```
raultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl get secrets
NAME                    TYPE                                  DATA   AGE
default-token-qzljm     kubernetes.io/service-account-token   3      63s
mysql-pass-9gtcfc4f5k   Opaque                                1      47s
```
Comprovem que s'hagin creat els volums persistents
```
raultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pv-claim   Bound    pvc-246205a4-f023-416d-93f5-a287ad1cc5ca   2Gi        RWO            standard       111s
wp-pv-claim      Bound    pvc-3dceef64-7313-44ae-b236-d2ed39fc3a38   2Gi        RWO            standard       111s
```
Comprovem que s'hagin creat els pods
```
raultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl get po
NAME                               READY   STATUS    RESTARTS   AGE
wordpress-648d9b88c-qljq5          1/1     Running   0          2m23s
wordpress-mysql-6bf69cb55f-7dtbh   1/1     Running   0          2m23s
```
Comprovem que el service estigui en funcionament
```
raultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl get svc wordpress
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
wordpress   LoadBalancer   10.108.50.143   <pending>     80:31872/TCP   5m1s
```
Agafem la informació del pod

```
raultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl describe pod wordpress-648d9b88c-qljq5
Name:         wordpress-648d9b88c-qljq5
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Wed, 03 Nov 2021 03:07:17 -0700
Labels:       app=wordpress
              pod-template-hash=648d9b88c
              tier=frontend
Annotations:  <none>
Status:       Running
IP:           172.17.0.4
IPs:
  IP:           172.17.0.4
Controlled By:  ReplicaSet/wordpress-648d9b88c
Containers:
  wordpress:
    Container ID:   docker://cb820184c095c96587175f83c73dd972aa59c69980266211bc0727aee34584a9
    Image:          wordpress:4.8-apache
    Image ID:       docker-pullable://wordpress@sha256:6216f64ab88fc51d311e38c7f69ca3f9aaba621492b4f1fa93ddf63093768845
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 03 Nov 2021 03:08:54 -0700
    Ready:          True
    Restart Count:  0
    Environment:
      WORDPRESS_DB_HOST:      wordpress-mysql
      WORDPRESS_DB_PASSWORD:  <set to the key 'password' in secret 'mysql-pass-9gtcfc4f5k'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-h8smd (ro)
      /var/www/html from wordpress-persistent-storage (rw)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  wordpress-persistent-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  wp-pv-claim
    ReadOnly:   false
  kube-api-access-h8smd:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  6m23s (x2 over 6m31s)  default-scheduler  0/1 nodes are available: 1 pod has unbound immediate PersistentVolumeClaims.
  Normal   Scheduled         6m21s                  default-scheduler  Successfully assigned default/wordpress-648d9b88c-qljq5 to minikube
  Normal   Pulling           6m20s                  kubelet            Pulling image "wordpress:4.8-apache"
  Normal   Pulled            4m45s                  kubelet            Successfully pulled image "wordpress:4.8-apache" in 1m34.762554841s
  Normal   Created           4m45s                  kubelet            Created container wordpress
  Normal   Started           4m44s                  kubelet            Started container wordpress
```
Com es pot observar en la descripció del pod es tracta de la versió de wordpress 4.8, en aquest caso anem a actualizarlo. Editem l'arxiu de configuració de deployment de wordpress
i posem qe sigui l'utlima versió. En la part on posa *image* posem *wordpress:latest*
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:latest
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim
----
```
Apliquem l'actualització.
```
raultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl apply -f wordpress-deployment.yaml 
service/wordpress unchanged
persistentvolumeclaim/wp-pv-claim unchanged
deployment.apps/wordpress configured
```
Mirem l'informació i com podem observar es pot veure que es l'ultima versió
```
aultest@ubuntu:/var/tmp/provatest/provakube/app$ kubectl describe pod wordpress-7d8887678-ggpzx 
Name:         wordpress-7d8887678-ggpzx
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Wed, 03 Nov 2021 03:18:56 -0700
Labels:       app=wordpress
              pod-template-hash=7d8887678
              tier=frontend
Annotations:  <none>
Status:       Pending
IP:           172.17.0.4
IPs:
  IP:           172.17.0.4
Controlled By:  ReplicaSet/wordpress-7d8887678
Containers:
  wordpress:
    Container ID:   
    Image:          wordpress:latest
    Image ID:       
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       CreateContainerConfigError
    Ready:          False
    Restart Count:  0
    Environment:
      WORDPRESS_DB_HOST:      wordpress-mysql
      WORDPRESS_DB_PASSWORD:  <set to the key 'password' in secret 'mysql-pass'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2xbhb (ro)
      /var/www/html from wordpress-persistent-storage (rw)
```
