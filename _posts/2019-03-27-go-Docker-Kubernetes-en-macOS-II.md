
---
layout: post
title: Go, Docker y Kubernetes en macOS (II)
---

## Elegir versión de Kubernetes
Al momento de intentar usar Kubernetes en nuestro Mac descubrimos que solo corre en Linux, por lo que deberemos usar una máquina virtual.

Dado que mi máquina (Macbook Air de 11″) no es una oda a la abundancia de recursos (aunque excelente para quien tiene que viajar frecuentemente y necesita un equipo liviano), debía buscar una opción liviana tanto de VM como de Kubernetes. Luego de leer y buscar por la red, encontré que la combinación «perfecta» (si es que algo así existe) es MicroK8s corriendo sobre Multipass.
<!--end_excerpt-->
Multipass es una versión de Linux virtualizada creada por Canonical que puede ser descargada desde aqui:  https://github.com/CanonicalLtd/multipass/releases

La instalación es simple, solo hay que seguir las instrucciones del pkg.
Una vez instalada, ejecutaremos multipass indicándole que instale MicroK8s.

```bash
multipass launch --name microk8s-vm --mem 4G --disk 40G
```

Luego instalamos microK8s

```bash
multipass exec microk8s-vm -- sudo snap install microk8s --classic
```

Luego definimos un forwarding para poder comunicarnos desde el exterior de la vm con todo lo que haya dentro.

```bash
multipass exec microk8s-vm -- sudo iptables -P FORWARD ACCEPT
```

Luego ejecutamos el comando list para verificar que la vm este corriendo.

```bash
Javiers-MacBook-Air:src jleyba$ multipass list
Name                    State             IPv4             Release
microk8s-vm             RUNNING           192.168.64.2     Ubuntu 18.04 LTS
Javiers-MacBook-Air:src jleyba$
```

En este punto es saludable tomar nota de la ip que aparece en la lista ya que es la que usaremos para nuestras pruebas.

Luego ejecutaremos el siguiente comando para habilitar el servicio de dns, registry y de dashboard.

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.enable dns dashboard registry
```

Un comando útil es el que nos da las direcciones de los servicios del cluster con sus ip:

```bash
Javiers-MacBook-Air:src jleyba$ multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl cluster-info
Kubernetes master is running at http://127.0.0.1:8080
Heapster is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Grafana is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
InfluxDB is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/monitoring-influxdb:http/proxy
```

En caso de necesidad, podemos abrir un proxy para acceder a nuestra aplicación.

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl proxy --address='0.0.0.0' --accept-hosts='.*'
Starting to serve on [::]:8001
```

En nuestro caso no es necesario ya que, para nuestra aplicación, habilitaremos un «service» del tipo load balancer que se encargará de balancear sobre todas las instancias de nuestra aplicación.

Ahora podremos usar el browser para, por ejemplo, acceder a Grafana y asi verificar que nuestra instalación de multipass + MicroK8s esta funcionando y accesible desde macOS. Para ello le pasaré a mi browser la siguiente url: http://192.168.64.2:8080/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy/?orgId=1


Como se puede ver en la imagen, hemos podido acceder a Grafana sin problemas.

Para desplegar y generar el servicio hago:

```bash
# gotest.yaml
# gotest.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gotest
  labels:
    app: gotest
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gotest
  template:
    metadata:
      labels:
        app: gotest
    spec:
      containers:
      - name: gotest
        image: antonof/gotest
        ports:
        - containerPort: 8081
        imagePullPolicy: Always
##
---
kind: Service
apiVersion: v1
metadata:
  name: gotest-service
  labels:
    app: gotest
spec:
  selector:
    app: gotest
  ports:
  - name: http
    port: 8081
    protocol: TCP
  type: LoadBalancer
```

Como se puede ver en ese fichero hay dos partes. La primera se refiere al deployment y la otra al servicio.
Es importante destaca los labels ya que Kubernetes los usa para identificar a nuestra aplicación. Si miran bien, nuestra aplicacion tiene el label «gotest» y abajo el servicio tiene un selector con el label «app=gotest» eso quiere decir que el servicio balanceará todas las instancias de los pods que tengan el label «app=gotest».

También hay que destacar que el servicio de tipo «LoadBalancer» utiliza el protocolo SCTP por lo que no funcionará en soluciones Cloud que no soporten a dicho protocolo.

Antes de comenzar con el despliegue, podemos verificar que todo esta funcionando entrando al dashboard de Kubernetes en
la url http://ip de la vm/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

Si queremos ver todos los servicios desplegados y sus urls podemos hacer

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl get all --all-namespaces
```

Ahora que ya tenemos todo listo podemos desplegar nuestra aplicación. Se puede hacer de forma manual, con el siguiente comando

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl run gotest --image=antonof/gotest --replicas=1 --port=8081
```

Y luego crear el servicio también de forma manual con

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl expose deployment.apps/gotest -n default --type=LoadBalancer
```

O también se puede crear todo junto utilizando el fichero yaml que hemos creado.
El problema aquí es que kube8s esta dentro de la vm por lo que hay que poner el fichero allí. Para hacer eso podemos abrir una consola dentro de la vm asi:

```bash
multipass shell microk8s-vm
```

Y una vez dentro podemos abrir vim y copiar allí el contenido del yaml salvandolo como gotest.yaml

Luego entonces podemos ejecutar:

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl create -f gotest.yaml
```

Para ver si el servicio se esta ejecutando, podemos usar la consola de Kubernetes descripta más arriba o el siguiente comando

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl get pods
```

Y para ver el servicio

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE

gotest       LoadBalancer   10.152.183.80        8081:30988/TCP   49s

kubernetes   ClusterIP      10.152.183.1            443/TCP          24m
```

Nuestra aplicación estará disponible en el puerto del servicio que se muestra arriba (y este balanceará entre las instancias).
probar en el browser (sin haber creado proxy).

http://192.168.64.2:30988/

Y para detener tanto servicio como pods

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl delete -f gotest.yaml
```

Para escalar los pods usamos el nombre que aparece cuando ejecutamos get all –all-namespaces

```bash
multipass exec microk8s-vm -- /snap/bin/microk8s.kubectl scale --replicas=1 deployment.apps/gotest
```

