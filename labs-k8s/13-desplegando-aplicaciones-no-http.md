# Desplegando aplicaciones que no utilizan tráfico HTTP

Las aplicaciones que hemos visto hasta ahora utilizan el protocolo HTTP(s), es decir son aplicaciones web.

Pero no todas las aplicaciones que queramos desplegar serán de este tipo. Puede que, en determinadas, circunstancias necesitemos dar acceso desde el exterior a una bbdd que se esté ejecutando dentro de kubernetes.

Ingress se encarga de [publicar rutas HTTP y HTTPS](https://kubernetes.io/docs/concepts/services-networking/ingress/) hacía servicios dentro de kubernetes. Por este motivo si necesitamos proporcionar acceso a aplicaciones que no hablan HTTP o HTTPS no podremos utilizar el ingress controller.

## Seleccionando una aplicación no HTTP/HTTPS

Vamos a desplegar una base de datos PostgreSQL en kubernetes y a proporcionarle acceso desde fuera de kubernetes.

Para ello buscaremos una imagen de PostgreSQL. Utilizaremos la [imagen oficial de PostgreSQL que se encuentra publicada en Dockerhub](https://hub.docker.com/_/postgres).

Será necesario leer la información facilitada de donde obtendremos la siguiente información:

* Es necesario pasar una variable de entorno **POSTGRES_PASSWORD** con el password que queramos utilizar para el acceso a la base de datos. Hemos elegido como contraseña **temporal**.
* Si no se especifica un nombre de usuario el usuario por defecto es **postgres**.

> ![HOMEWORK](../imgs/homework-icon.png) Se pueden parametrizar varios parámetros.

## Desplegando la aplicación

Para desplegar la aplicación crearemos un deployment y le pasaremos, al menos, la variable **POSTGRES_PASSWORD**:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: postgres
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: postgres
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres
        env:
        - name: POSTGRES_PASSWORD
          value: "temporal"
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
    name: postgres-service
    namespace: postgres
spec:
    type: NodePort
    selector:
      app: postgres
    ports:
    - nodePort: 31234
      port: 5432
      targetPort: 5432
```

Como ya hemos comentado no vamos a utilizar un ingress para el acceso ya que el tráfico que necesita PostgreSQL no es del tipo HTTP/HTTPS.

Por este motivo definiremos el servicio de la siguiente manera:

```yaml
---
apiVersion: v1
kind: Service
metadata:
    name: postgres-service
    namespace: postgres
spec:
    type: NodePort
    selector:
      app: postgres
    ports:
    - nodePort: 31234
      port: 5432
      targetPort: 5432
```

Ahora hemos definido el servicio de tipo **NodePort**. En los anteriores ejemplos se había dejado al valor por defecto que es **ClusterIP**. Un servicio expuesto como **ClusterIP** solo es accesible desde dentro de kubernetes ya que se publica en la red interna del cluster y por ese motivo se crea un ingress para la aplicación en el ingress controller que envia el tráfico HTTP/HTTPS al servicio.

Un servicio creado como **NodePort** expone el servicio en un puerto estático en el nodo que ejecuta el pod. Este puerto se indica con **nodePort** y en este caso se va a exponer en el puerto **31234**.

Para desplegar PostgreSQL procedemos con el [Deployment](postgres/postgres.yaml):

```console
[kubeadmin@kubemaster postgres]$ kubectl apply -f postgres.yaml 
namespace/postgres created
deployment.apps/postgres created
service/postgres-service created
[kubeadmin@kubemaster postgres]$ kubectl get pods --namespace=postgres -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP               NODE                NOMINATED NODE   READINESS GATES
postgres-6dfcf566fc-hcls6   1/1     Running   0          9s    192.169.232.28   kubenode1.acme.es   <none>           <none>
[kubeadmin@kubemaster postgres]$
```

Los servicios creados:

```console
[kubeadmin@kubemaster postgres]$ kubectl get svc -A -o wide
NAMESPACE            NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                     AGE     SELECTOR
calico-system        calico-kube-controllers-metrics              ClusterIP   10.110.61.65     <none>        9094/TCP                                    2d      k8s-app=calico-kube-controllers
calico-system        calico-typha                                 ClusterIP   10.106.50.61     <none>        5473/TCP                                    2d      k8s-app=calico-typha
default              kubernetes                                   ClusterIP   10.96.0.1        <none>        443/TCP                                     2d      <none>
haproxy-controller   haproxy-kubernetes-ingress                   NodePort    10.104.187.186   <none>        80:31826/TCP,443:30886/TCP,1024:31734/TCP   2d      run=haproxy-ingress
haproxy-controller   haproxy-kubernetes-ingress-default-backend   ClusterIP   10.103.195.4     <none>        8080/TCP                                    2d      run=ingress-default-backend
kube-system          kube-dns                                     ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP                      2d      k8s-app=kube-dns
postgres             postgres-service                             NodePort    10.97.66.252     <none>        5432:31234/TCP                              25m     app=postgres
webapp-routed        webapp-service                               ClusterIP   10.111.154.238   <none>        80/TCP                                      4h15m   app=webapp-routed
webapp-tls           webapp-tls-service                           ClusterIP   10.105.148.63    <none>        443/TCP                                     3h21m   app=webapp-tls
webapp-volumes       volumes-service                              ClusterIP   10.96.145.167    <none>        80/TCP                                      3h37m   app=webapp-volumes
[kubeadmin@kubemaster postgres]$
```

Como podemos ver PostgreSQL se está ejecutando en un container en el nodo **kubenode1.acme.es**, luego si accedemos a ese nodo al puerto **31234** podremos logearnos:

```console
[jadebustos@archimedes devopslabs]$ psql -h kubenode1.acme.es -p 31234 -U postgres
Password for user postgres: 
psql (13.4, server 14.2 (Debian 14.2-1.pgdg110+1))
WARNING: psql major version 13, server major version 14.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

postgres=# 
```

Esta configuración presenta dos problemas:

1. Necesitaremos conocer en que nodo del cluster se está ejecutando lo cual no será posible.

   Para solucionar este problema se configura un balanceador externo que balancea a todos los nodos del cluster al puerto definido como **nodePort**. De esta forma para conectarnos a PostgreSQL atacaremos el servicio balanceado.

2. Los contenedores son efímeros, por lo tanto cualquier escritura en la bbdd se perdera al parar el pod.

> ![HOMEWORK](../imgs/homework-icon.png) Se deberá mapear un volumen persistente en el directorio del pod donde se encuentran los ficheros de la bbdd para que las escrituras se persistan.

## Configurando un balanceador externo

Desplegamos una máquina CentOS 8 Stream con los siguientes requerimientos:

* Creado un usuario **ansible** con acceso a sudo sin password.
* Configurada una clave SSH para hacer ssh con el usuario **ansible**.

Con el siguiente fichero de inventario:

```ini
[all:vars]
ansible_user=ansible

[lb]
192.168.1.121
```

Se configurará un **haproxy** para acceder al PostgreSQL que hemos desplegado en kubernetes.

Para ello en [postgres/ansible/group_vars/haproxy.yaml](postgres/ansible/group_vars/haproxy.yaml) configuramos:

* **worker1** con la ip del primer worker.
* **worker2** con la ip del segundo worker.
* **nodeport** con el puerto expuesto en kubernetes para el acceso a PostgreSQL.

Una vez hayamos configurado lo anterior para configurar el balanceador:

```console
[jadebustos@archimedes ansible]$ ansible-playbook -i hosts deploy-balancer.yaml
```

Si incluimos la siguiente línea en **/etc/hosts** para tener resolución DNS:

```
192.168.1.121 postgres.acme.es
```

Podemos realizar la conexión a la base de datos vía el balanceador que previamente hemos configurado:

```console
[jadebustos@archimedes ansible]$ psql -h postgres.acme.es -p 5432 -U postgres
Password for user postgres: 
psql (13.4, server 14.2 (Debian 14.2-1.pgdg110+1))
WARNING: psql major version 13, server major version 14.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)

postgres=# 
```

> ![HOMEWORK](../imgs/homework-icon.png) Será necesario instalar un cliente de PostgreSQL en algún equipo para probar el acceso.

> ![HOMEWORK](../imgs/homework-icon.png) Comprueba en que nodo de kubernetes se está ejecutando el contenedor de PostgreSQL, fuerza a que se ejecute en otro nodo. Apagando, por ejemplo, el nodo en el que se está ejecutando. Cuando el contenedor se haya arrancado en el otro nodo prueba a realizar la conexión.