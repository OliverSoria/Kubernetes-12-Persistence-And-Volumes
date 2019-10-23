### Arquitectura de Microservicios

En esta sección vamos a crear una arquitectura de la siguiente manera:<br/>

* **Position Simulator** simula la posición de los vehículos y envía esta posición a una cola
* **Active Mq** es una cola que retiene las posiciones de los vehículos
* **Position Tracker**  es un tercer microservicio que va y obtiene la posición de los vehículos en la cola
* **API Gateway** es un cuarto servicio por que es accedido por medio de una url; obtiene la posición de los vehículos del servicio anterior
* **Front** en un extremo del flujo tenemos la vista desarrollada en Angular y que se utiliza en un navegador web

##### Active Mq

Habiendo descrito la arquitectura de nuestro sistema empezamos por desplegar el corazón del sistema, que en este caso es la cola (Active Mq), tenemos entonces el siguiente archivo _mq-deploy.yml_:<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: queue
spec:
  selector:
    matchLabels:
      app: queue
  template:
    metadata:
      labels:
        app: queue
    spec:
      containers:
        - name: whatever
          image: richardchesterwood/k8s-fleetman-queue:release1
  replicas: 1
```

Y su respectivo servicio _mq-service.yml_:<br/>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-queue
spec:
  selector:
    app: queue
  ports:
    - name: http
      port: 8161
      nodePort: 30010
    - name: endpoint
      port: 61616
  type: NodePort
```
**Nota:** es muy importante tener en cuenta que el nombre del servicio, ya que de lo contrario el simulador de posición no lo va a encontrar.<br/>

##### Position Simulator

El siguiente servicio a desplegar es el simulador de posición, y lo haremos por medio de un _Deployment_:<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-simulator
spec:
  selector:
    matchLabels:
      app: position-simulator
  template:
    metadata:
      labels:
        app: position-simulator
    spec:
      containers:
        - name: whatever
          image: richardchesterwood/k8s-fleetman-position-simulator:release1
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production-microservice
  replicas: 1
```

Para efectos prácticos, hemos establecido que exista una sola réplica y además hace su aparición la etiqueta _env_, que nos permite definir variables de entorno, en este caso el perfil activo.<br/>

##### Tracker Simulator

El tercer servicio tiene la función de procesar los mensajes que se han depositado en la cola por el simulador de posociones, y lo haremos por medio de un _deployment_:<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: position-tracker
spec:
  selector:
    matchLabels:
      app: position-tracker
  template:
    metadata:
      labels:
        app: position-tracker
    spec:
      containers:
        - name: position-tracker
          image: richardchesterwood/k8s-fleetman-position-tracker:release1
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production-microservice
  replicas: 1
```

En esta ocasión vamos a utilizar un servicio para el simulador de rastreo, que nos permitirá obtener la posición de los vehículos en formato XML vía web:<br/>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-position-tracker
spec:
  selector:
    app: position-tracker
  ports:
    - name: http
      port: 8080
      nodePort: 30020
  type: NodePort
```

Sin embargo, la realidad es que no es necesario tener acceso desde el exterior al servicio, por lo tanto en lugar de declararlo como un _NodePort_ lo podemos declarar como un _ClusterIP_, cabe mencionar que si se hace esto, tambien se debe eliminar la línea que define el _nodePort_, ya que de lo contrario no funcionará. Con estos cambios el cluster tendra cominicación con otros clusters, pero no hacía el exterior.<br/>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-position-tracker
spec:
  selector:
    app: position-tracker
  ports:
    - name: http
      port: 8080
  type: ClusterIP
```
##### API Gateway

No hay mucho que decir al respecto de este componente, sigue la misma línea que el anterior, tenemos el _deployment_:<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: richardchesterwood/k8s-fleetman-api-gateway:release1
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production-microservice
  replicas: 1
```

Y respecto al servicio, se usará también un _ClusterIP_  similar al del paso anterior:<br/>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fleetman-api-gateway
spec:
  selector:
    app: api-gateway
  ports:
    - name: http
      port: 8080
  type: ClusterIP
```
##### Webapp

Esta sección finaliza con el despliegue del _front_, el cual está hecho en _Angular_, así que empezamos con el _deployment_:<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: richardchesterwood/k8s-fleetman-webapp-angular:release1
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production-microservice
  replicas: 1
```

Y dado que es un componente con el cual el usuario interactuará directamente procedemos a agregar su respectivo servicio:<br/>




