### Persisencia y Volúmenes

##### Persistencia

En esta sección vamos a utilizar tanto persistencia en _mongodb_ así mismo trabajaremos con volúmenes.<br/>

Una de las razones para realizar esta reingenieria es que en la lección pasada todos los cambios se perdían cuando se reiniciaban las instancias, así que debemos atacar esa deficiencia persistiendo los datos. La reingenieria consiste en agregar un microservicio que interactue directamente con _Position Tracker_ para que persista los datos en una base de datos no relacional: _MongoDB_.<br/>

Empezaremos por actualizar las imagenes _docker_ ubicadas dentro de la sección _template_ en los _deployments_ utilizados en la sección anterior, actualizaremos de la versión 1 a la versión 2. Posteriormente vamos a correr todo para ver reflejados los cambios.<br/>

Una vez que los cambios se hayan reflejado procederemos a realizar la adaptación para persistir, dado que el servicio _Position Tracker_ ahora se conectará con una base de datos tenemos que actualizar la imagen _Docker_ a la versión 3. La nueva versión esta hecha como las anteriores en _Spring Boot_ y en el caso del _properties_ donde define la _url_ para conectarse a _Mongo_ en vez de establecer una dirección _ip_ se reemplaza por el nombre que se usa internamente en _Kubernetes_.<br/>

Nota: cuando se trabaja con Kubernetes es muy importante tener en consideración el orden en que se realizan los despliegues, por ejemplo, si se levanta el _Position Tracker_ antes que _Mongo_ evidentemente existirá un error.<br/>   

Antes de desplegar los cambios realizados anteriormente vamos a crear un _deploymet_ para Mongo, y dado que se trata de Mongo existe la versión oficial, así que utilizaremos esa imagen en nuestros _pods_:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:3.6.5-jessie
  replicas: 1
``` 
Y no olvidemos su respectivo servicio, que a final de cuentas es a donde se el _Position Tracker_ almacenará la información:<br/>

```yaml
kind: Service
apiVersion: v1
metadata:
  name: fleetman-mongodb
spec:
  selector:
    app: mongodb
  ports:
    - name: mongoport
      port: 27017
  type: ClusterIP
``` 

Nótese que es un _ClusterIP_ ya que no necesita ser accedido desde el exterior.<br/>

Ahora bien, después de que hemos creado un _deployment_ basado en _mongo_ junto a su respectivo servicio, entonces estamos listos para actualizar a la versión 3 el _Position Tracker_ para que así pueda enviar la información a guardar en el _deployment_ de _mongo_:<br/>

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
          image: richardchesterwood/k8s-fleetman-position-tracker:release3
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production-microservice
  replicas: 1
```

##### Volúmenes

Nuestra arquitectura tiene un defecto y es que cuando el microservicio encargado de correr la base de datos muere, toda la información se pierde, al reiniciarlo empieza a almacenar la información de cero. Una forma de solventar este defecto es montar un volumen externo y una excelente alternativa es montar un volumen de Amazon, sin embargo vamos a empezar montando un volumen perteneciente al _minikube_ en los pods. Como recordaremos el _minikube_ no es otra cosa más que una distribución Linux, así que la nomenclatura empleada para especificar los puntos de montaje es 100% de Linux.<br/>

Ahora procederemos a modificar el _deployment_ para que refleje el montaje del volumen:<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:3.6.5-jessie
          volumeMounts:
            - mountPath: /data/db
              name: mongo-persistent-storage
      volumes:
        - name: mongo-persistent-storage
          hostPath:
            path: /mnt/some/directory/structure
  replicas: 1
```

Aunque el ejemplo anterior funciona bien no es la mejor forma de trabajar con volúmenes, ya que implica definir las rutas en cada _pod_ haciendo que sea complicado el mantenimiento, lo mejor es crear archivos de configuraciones que a su vez se encargarán de la administración de los volúmenes, empezaremos por modificar el deployment anterior con el valor de -persistentVolumeClaim_:<br/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:3.6.5-jessie
          volumeMounts:
            - mountPath: /data/db
              name: mongo-persistent-storage
      volumes:
        - name: mongo-persistent-storage
          # apunta a la configuración de COMO queremos que se lleve a cabo el montaje
          persistentVolumeClaim:
            claimName: mongo-pvc
  replicas: 1
``` 

A continuación necesitamos dos archivos más de configuración, el primero específica QUE queremos y el segundo específica el COMO lo queremos, así pues tenemmos en el primer archivo:<br/>

```yaml
# Este documento específica QUE es lo que queremos
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  # Nombre que definimos en el template del deployment
  name: mongo-pvc
spec:
  storageClassName: mylocalstorage
  resources:
    requests:
      storage: 20Gi
  accessModes:
    - ReadWriteOnce
```

Y en el segundo específicamos COMO lo queremos hacer:<br/>
```yaml
# Este documento específica el COMO lo queremos
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pvc
spec:
  storageClassName: mylocalstorage
  capacity:
    storage: 20Gi
  hostPath:
    path: "/mnt/some new/directory/structure"
    type: DirectoryOrCreate
  accessModes:
    - ReadWriteOnce
```

Nótese la correlacion entre el _mongo-pvc_, _storageClassName_ y _claimName_.<br/>

Después de realizar estos cambios es posible acceder al minikube con el usuario _docker_ y la contraseña _tcuser_ y validar que se haya creado la ruta que definimos anteriormente.<br/> 

