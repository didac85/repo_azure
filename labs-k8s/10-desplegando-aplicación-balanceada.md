# Desplegando una aplicación balanceada

Desplegaremos una aplicación balanceada.

También entraremos en más detalle sobre como exponer aplicaciones al exterior.

## Namespace

Crearemos el namespace en el que se desplegará la aplicación:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: webapp-balanced
  labels:
    name: webapp-balanced
```

## Creación de volúmenes

En el servidor NFS crearemos un share:

```console
[root@kubemaster ~]# cat /etc/exports
/srv/balanced 192.168.1.110(rw,sync)
/srv/balanced 192.168.1.111(rw,sync)
/srv/balanced 192.168.1.112(rw,sync)
[root@kubemaster ~]# exportfs -r
[root@kubemaster ~]#
```

Copiamos el fichero [webapp-balanced](index.php) al share que hemos creado en el servidor NFS. Este fichero será servido por la aplicación balanceada.

## Almacenamiento

Creamos el persistent volume y el claim:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-balanced
  namespace: webapp-balanced
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /srv/balanced
    server: 192.168.1.110
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-balanced
  namespace: webapp-balanced
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

## Deployment

Creamos el deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-balanced
  namespace: webapp-balanced
  labels:
    app: webapp-balanced
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp-balanced
  template:
    metadata:
      labels:
        app: webapp-balanced
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - webapp-balanced
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: webapp-balanced
        image: quay.io/rhte_2019/webapp:v1
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
        volumeMounts:
        - name: site-pvc-balanced
          mountPath: /var/www/public
      volumes:
      - name: site-pvc-balanced
        persistentVolumeClaim:
          claimName: nfs-pvc-balanced
```

> ![TIP](../imgs/tip-icon.png) Incluimos una regla de antiafinidad ya que queremos que cada pod se esté ejecutando en un nodo diferente para evitar que si un nodo cae nos quedemos sin servicio.

Inicialmente crearemos una única replica.

## Creamos el servicio balanceado

Creamos el servicio y el ingress tal y como hemos hecho anteriormente. En este caso en el servicio indicaremos que es del tipo **LoadBalancer**:

```yaml
apiVersion: v1
kind: Service
metadata:
    name: balanced-service
    namespace: webapp-balanced
spec:
    selector:
      app: webapp-balanced
    ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
    type: LoadBalancer
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: balanced-ingress
  namespace: webapp-balanced
  labels:
    app: webapp-balanced
  annotations:
      haproxy.org/path-rewrite: "/"
spec:
  rules:
  - host: foo-balanced.bar
    http:
      paths:
      - path: /balanced
        pathType: "Prefix"
        backend:
          service:
            name: balanced-service
            port:
              number: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-configmap
  namespace: webapp-balanced
data:
  servers-increment: "42"
  ssl-redirect: "OFF"
```

> ![INFORMATION](../imgs/information-icon.png) [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)

## Probando la aplicación balanceada

Listamos los pods:

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl get pods --namespace webapp-balanced -o wide
NAME                               READY   STATUS    RESTARTS   AGE   IP               NODE                NOMINATED NODE   READINESS GATES
webapp-balanced-5897c4887c-zdjvv   1/1     Running   0          84s   192.169.232.24   kubenode1.acme.es   <none>           <none>
[kubeadmin@kubemaster webapp-balanced]$
```

La ip expuesta para conexiones:

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc --namespace haproxy-controller
NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                     AGE
haproxy-kubernetes-ingress                   NodePort    10.104.187.186   <none>        80:31826/TCP,443:30886/TCP,1024:31734/TCP   3d1h
haproxy-kubernetes-ingress-default-backend   ClusterIP   10.103.195.4     <none>        8080/TCP 
[kubeadmin@kubemaster webapp-balanced]$ 
```

Si nos conectamos a la aplicación:

![IMG](../imgs/webapp-balanced-1.png)

Si recargamos la página podremos ver que la ip no cambia. Tenemos un único pod sirviendo la aplicación.

Escalamos el deployment para tener dos pods:

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl scale --replicas=2 deployment/webapp-balanced --namespace=webapp-balanced
deployment.apps/webapp-balanced scaled
[kubeadmin@kubemaster webapp-balanced]$ kubectl get pods --namespace webapp-balanced -o wide
NAME                               READY   STATUS    RESTARTS   AGE    IP               NODE                NOMINATED NODE   READINESS GATES
webapp-balanced-5897c4887c-89s4n   1/1     Running   0          11s    192.169.49.92    kubenode2.acme.es   <none>           <none>
webapp-balanced-5897c4887c-zdjvv   1/1     Running   0          6m7s   192.169.232.24   kubenode1.acme.es   <none>           <none>
[kubeadmin@kubemaster webapp-balanced]$
```

Vemos que los pods se están ejecutando en nodos diferentes, tal y como configuramos el deployment con las reglas de antiafinidad. Si recargamos la página del navegador:

![IMG](../imgs/webapp-balanced-2.png)

Podemos ir recargando y veremos que cada vez que recargamos se muestra una ip diferente. Cada petición la está sirviendo un pod diferente.

## Escalando la aplicación

Escalamos la aplicación para tener tres pods:

```console
[kubeadmin@kubemaster devopslabs]$ kubectl scale --replicas=3 deployment/webapp-balanced --namespace=webapp-balanced
deployment.apps/webapp-balanced scaled
[kubeadmin@kubemaster devopslabs]$ kubectl get pods --namespace webapp-balanced -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE                NOMINATED NODE   READINESS GATES
webapp-balanced-5897c4887c-89s4n   1/1     Running   0          2m48s   192.169.49.92    kubenode2.acme.es   <none>           <none>
webapp-balanced-5897c4887c-x7xch   0/1     Pending   0          27s     <none>           <none>              <none>           <none>
webapp-balanced-5897c4887c-zdjvv   1/1     Running   0          8m44s   192.169.232.24   kubenode1.acme.es   <none>           <none>
[kubeadmin@kubemaster devopslabs]$
```

Vemos que el tercer pod no se ha desplegado. En el clúster tenemos dos nodos y hemos incluido una regla de antiafinidad, por lo tanto no existe ningún nodo en el clúster en el que desplegar el pod:

```console
[kubeadmin@kubemaster devopslabs]$ kubectl get nodes
NAME                 STATUS   ROLES                  AGE    VERSION
kubemaster.acme.es   Ready    control-plane,master   3d1h   v1.23.3
kubenode1.acme.es    Ready    <none>                 3d1h   v1.23.3
kubenode2.acme.es    Ready    <none>                 3d1h   v1.23.3
[kubeadmin@kubemaster devopslabs]$ 
```

Podemos ver los eventos del namespace en el que vemos que no se cumplen las reglas de antiafinidad del deployment:

```console
[kubeadmin@kubemaster devopslabs]$ kubectl get events --namespace webapp-balanced
...
59s         Warning   FailedScheduling         pod/webapp-balanced-5897c4887c-x7xch    0/3 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate, 2 node(s) didn't match pod anti-affinity rules.
...
[kubeadmin@kubemaster devopslabs]$ 
```

## Explorando el servicio con tipo LoadBalancer

Veamos los datos del servicio que hemos creado:

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl describe svc balanced-service --namespace webapp-balanced
Name:                     balanced-service
Namespace:                webapp-balanced
Labels:                   <none>
Annotations:              <none>
Selector:                 app=webapp-balanced
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.110.197.47
IPs:                      10.110.197.47
Port:                     http  80/TCP
TargetPort:               80/TCP
NodePort:                 http  30339/TCP
Endpoints:                192.169.232.24:80,192.169.49.92:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc balanced-service --namespace webapp-balanced -o wide
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
balanced-service   LoadBalancer   10.110.197.47   <pending>     80:30339/TCP   5h5m   app=webapp-balanced
[kubeadmin@kubemaster network-policies]$
```

Obtengamos la configuración del servicio:

```yaml
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc balanced-service --namespace webapp-balanced -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"balanced-service","namespace":"webapp-balanced"},"spec":{"ports":[{"name":"http","port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"webapp-balanced"},"type":"LoadBalancer"}}
  creationTimestamp: "2022-02-16T18:59:01Z"
  name: balanced-service
  namespace: webapp-balanced
  resourceVersion: "171850"
  uid: 8479a356-276b-4918-bc62-c7289bb33e62
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.110.197.47
  clusterIPs:
  - 10.110.197.47
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    nodePort: 30339
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webapp-balanced
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}
[kubeadmin@kubemaster webapp-balanced]$
```

Prestemos atención a:

```yaml
  ports:
  - name: http
    nodePort: 30339
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webapp-balanced
```

+ **nodePort** indica el puerto externo por el que la aplicación se encontrará disponible.
+ **port** indica el puerto expuesto internamente.
+ **targetPort** indica el puerto en el que los contenedores están escuchando.

Se ha expuesto la aplicación por un puerto (nodeport) en el nodo master, 30339:

```console
[kubeadmin@kubemaster webapp-balanced]$ netstat -ln | grep 30339
tcp        0      0 0.0.0.0:30339           0.0.0.0:*               LISTEN    
[kubeadmin@kubemaster webapp-balanced]$
```

> ![HOMEWORK](../imgs/homework-icon.png) También en los workers.

Si accedemos a la aplicación por ese puerto:

![IMG](../imgs/webapp-balanced-nodeport.png)

> ![HOMEWORK](../imgs/homework-icon.png) Prueba a acceder a los workers y al puerto **30339**.

Si recargamos la página con el navegador veremos que la dirección IP no cambia. Si lo hacemos varias veces terminará cambiando la dirección IP (balanceará al otro contenedor).

Cuando accedemos al servicio por **http://foo-balanced.bar:31826/balanced** estamos accediendo por el ingress, que reenvia la petición al servicio y desde el servicio se balancea a los pods que se encuentren levantados. Sin embargo, cuando accedemos utilizando el nodePort creado al indicar que el servicio es de tipo **LoadBalancer**, es decir por la url **http://foo-balanced.bar:30339**, estamos accediendo por el balanceador creado.

Luego con esta configuración tenemos dos accesos posibles a la aplicación.

En los siguientes apartados vamos a clarificar esto.

## Acceso por un servicio de tipo LoadBalancer

Hemos especificado que el servicio es de tipo [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer), este tipo requiere de un balanceador externo proporcionado por un proveedor cloud (**AWS**, **Azure**, **GCP**, ...). Pero no estamos utilizando ningún balanceador externo. ¿Que está pasando?

Un balanceador externo recibirá las peticiones y serán balanceadas a **http://foo-balanced.bar:30339**. Como no tenemos ninguno desplegado estamos accediendo directamente al puerto. Cuando despleguemos en un proveedor cloud al crear un servicio de tipo **LoadBalancer** se creará un balanceador que balanceará el tráfico a los pods.

Si utilizamos un servicio de tipo **LoadBalancer** no necesitaremos crear el ingress. En este caso como no hemos desplegado en un cloud provider:

![IMG](../imgs/SVC-LoadBalancer.png)

Desplegar en un cloud provider implica integrar kubernetes con el cloud provider. Para está integración se desplegará el [Cloud Controller Manager de Kubernetes](https://kubernetes.io/docs/concepts/architecture/cloud-controller/) que permitirá la integración de Kubernetes con el proveedor cloud.

En este caso el desplegar con un servicio de tipo **LoadBalancer** desplegaría un balanceador de carga en el cloud provider y tendríamos algo similar a:

![IMG](../imgs/SVC-LoadBalancer-Cloud-Provider.png)

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc balanced-service --namespace webapp-balanced -o wide
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE     SELECTOR
balanced-service   LoadBalancer   10.110.197.47   <pending>     80:30339/TCP   5h35m   app=webapp-balanced
[kubeadmin@kubemaster webapp-balanced]$
```

Podemos observar que la **EXTERNAL-IP** se queda en **\<pending>**. Ello es debido a que no se despliega un balanceador al no estar desplegado kubernetes en un proveedor cloud, con integración en el mismo.

En este caso para evitar que se cree un NodePort y que se acceda por el ingress será necesario especificar en el servicio:

```yaml
spec:
  type: LoadBalancer
  allocateLoadBalancerNodePorts: False
  ...
```

Si desplegamos añandiendo lo anterior tendremos:

```yaml
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc balanced-service --namespace webapp-balanced -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"balanced-service","namespace":"webapp-balanced"},"spec":{"allocateLoadBalancerNodePorts":false,"ports":[{"name":"http","port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"webapp-balanced"},"type":"LoadBalancer"}}
  creationTimestamp: "2022-02-17T00:47:02Z"
  name: balanced-service
  namespace: webapp-balanced
  resourceVersion: "194377"
  uid: 044c2f94-774a-48ad-854c-4afa5539b8ce
spec:
  allocateLoadBalancerNodePorts: false
  clusterIP: 10.109.158.179
  clusterIPs:
  - 10.109.158.179
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webapp-balanced
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}
[kubeadmin@kubemaster webapp-balanced]$
```

Podemos observar que el nodeport ya no se encuentra presente.

## Acceso por un servicio de tipo NodePort

Con un servicio de tipo [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) no es necesario un ingress. 

![IMG](../imgs/SVC-NodePort.png)

> ![HOMEWORK](../imgs/homework-icon.png) Redespliega la aplicación eliminando el ingress, configurando el servicio como **NodePort** y que no hay acceso por el ingress.

Este tipo de acceso expone un puerto en el master a través del cual será accesible la aplicación:

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl describe svc balanced-service --namespace webapp-balanced
Name:                     balanced-service
Namespace:                webapp-balanced
Labels:                   <none>
Annotations:              <none>
Selector:                 app=webapp-balanced
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.97.148.96
IPs:                      10.97.148.96
Port:                     http  80/TCP
TargetPort:               80/TCP
NodePort:                 http  30928/TCP
Endpoints:                192.169.232.47:80,192.169.49.93:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc balanced-service --namespace webapp-balanced -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"balanced-service","namespace":"webapp-balanced"},"spec":{"ports":[{"name":"http","port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"webapp-balanced"},"type":"LoadBalancer"}}
  creationTimestamp: "2022-02-17T00:51:07Z"
  name: balanced-service
  namespace: webapp-balanced
  resourceVersion: "195132"
  uid: f35fad18-dc2b-4410-a8df-17686d7c37bb
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.97.148.96
  clusterIPs:
  - 10.97.148.96
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    nodePort: 30928
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webapp-balanced
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}
[kubeadmin@kubemaster webapp-balanced]$
```

Si observamos las salidas anteriores y las comparamos con las del servicio de tipo **LoadBalancer** veremos que son iguales. Es debido a que en ambos casos se está utilizando un **NodePort**. Cuando no utilizamos un balanceador externo, desactivando **NodePort** ambos servicios son equivalentes. Al desactivar **NodePort** y utilizar un balanceador externo se balanceará directamente a los pods ya que en los workers se encuentra creado el **NodePort**.

## Acceso por un servicio de tipo ClusterIP

Cuando hemos creado aplicaciones accesibles desde el exterior hemos utilizado un ingress. Esto es debido a que el tipo por defecto de servicio es de tipo **ClusterIP**, este tipo de servicio es solo accesible desde dentro del cluster y por ese motivo utilizabamos un ingress para enrutar las peticiones hacía el servicio interno.

![IMG](../imgs/SVC-ClusterIP.png)

> ![HOMEWORK](../imgs/homework-icon.png) Redespliega la aplicación configurando el servicio como **ClusterIP**. Dado que es el tipo por defecto para el objeto **Service** bastaría con eliminar:
>
>  ```yaml
>    type: NodePort
>  ```

> ![HOMEWORK](../imgs/homework-icon.png) Después de desplegar la aplicación y antes de escalarla para tener dos pods conectarese al pod del ingress y revisar la configuración del haproxy para ver como está definido el balanceo al único pod.

> ![HOMEWORK](../imgs/homework-icon.png) Escalar la aplicación a dos pods y revisa la configuración del haproxy para ver como está definido el balanceo a los dos pods.

> ![TIP](../imgs/tip-icon.png) En [actividades anteriores](03-desplegando-routed-application.md) ya hemos visto como conectarse al pod del ingress y revisar la configuración del haproxy.

Con este tipo de servicio es necesario un ingress:

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl describe svc balanced-service --namespace webapp-balanced
Name:              balanced-service
Namespace:         webapp-balanced
Labels:            <none>
Annotations:       <none>
Selector:          app=webapp-balanced
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.0.241
IPs:               10.96.0.241
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         192.169.232.54:80,192.169.49.94:80
Session Affinity:  None
Events:            <none>
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc balanced-service --namespace webapp-balanced -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"name":"balanced-service","namespace":"webapp-balanced"},"spec":{"ports":[{"name":"http","port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"webapp-balanced"}}}
  creationTimestamp: "2022-02-17T01:11:14Z"
  name: balanced-service
  namespace: webapp-balanced
  resourceVersion: "198233"
  uid: 67e13cbc-d4ca-4161-99d2-6b77a9216f5d
spec:
  clusterIP: 10.96.0.241
  clusterIPs:
  - 10.96.0.241
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: webapp-balanced
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
[kubeadmin@kubemaster webapp-balanced]$
```

Para acceder a la aplicación:

```console
[kubeadmin@kubemaster webapp-balanced]$ kubectl describe ingress balanced-ingress --namespace webapp-balanced
Name:             balanced-ingress
Labels:           app=webapp-balanced
Namespace:        webapp-balanced
Address:          
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host              Path  Backends
  ----              ----  --------
  foo-balanced.bar  
                    /balanced   balanced-service:80 (192.169.232.54:80,192.169.49.94:80)
Annotations:        haproxy.org/path-rewrite: /
Events:             <none>
[kubeadmin@kubemaster webapp-balanced]$ kubectl get svc --namespace haproxy-controller
NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                     AGE
haproxy-kubernetes-ingress                   NodePort    10.104.187.186   <none>        80:31826/TCP,443:30886/TCP,1024:31734/TCP   3d2h
haproxy-kubernetes-ingress-default-backend   ClusterIP   10.103.195.4     <none>        8080/TCP                                    3d2h
[kubeadmin@kubemaster webapp-balanced]$
```

lo haremos a través **http://foo-balanced.bar:31826/balanced** y si recargamos la página cada carga será servida por un pod.

Por lo tanto para balancear utilizando un ingress a varios pods no es necesario nada más.

> ![NOTE](../imgs/note-icon.png) Este funcionamiento depende del ingress, con lo cual si utilizamos otro ingress diferente de haproxy el comportamiento podría no ser el mismo.

> ![HOMEWORK](../imgs/homework-icon.png) Modificar el fichero [webapp-balanced/webapp-balanced.yaml](webapp-balanced/webapp-balanced.yaml) para que la aplicación sólo se pueda acceder utilizando [TLS](10-desplegando-aplicación-balanceada.md).