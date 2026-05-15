# TeoriaKube

## Objetivo del LAB2

En este laboratorio se trabaja con dos recursos principales de Kubernetes:

- Un `Deployment`, que se encarga de crear y mantener varios Pods iguales.
- Un `Service`, que se encarga de exponer esos Pods para que puedan recibir trafico.

La idea del laboratorio es desplegar una aplicacion web basada en la imagen `maldoariel/miapp-server:v1`, crear varias replicas de esa aplicacion y luego publicar el acceso mediante un `NodePort`.

## Que son los manifiestos en Kubernetes

En Kubernetes, un manifiesto es un archivo escrito normalmente en formato YAML que describe el estado deseado de uno o varios recursos del cluster.

Un manifiesto no es un programa que se ejecuta paso a paso. En cambio, es una declaracion de intencion. Es decir, en el archivo se indica como queres que quede el sistema, y Kubernetes intenta llevar el estado real a ese estado deseado.

Por ejemplo, en un manifiesto se puede declarar:

- que exista un `Pod`
- que exista un `Deployment` con cierta cantidad de replicas
- que exista un `Service` para exponer una aplicacion
- que un contenedor use cierta imagen
- que una app escuche en determinado puerto

Cuando se ejecuta un comando como:

```powershell
kubectl apply -f archivo.yml
```

Kubernetes lee el manifiesto, interpreta lo que se quiere crear o actualizar, y lo compara con lo que ya existe en el cluster.

### Ventajas de usar manifiestos

- Permiten guardar la infraestructura como codigo.
- Facilitan repetir el mismo despliegue varias veces.
- Ayudan a versionar cambios con Git.
- Hacen mas simple auditar configuraciones.
- Permiten automatizar despliegues en pipelines CI/CD.

### Ejemplos de recursos que suelen declararse con manifiestos

- `Pod`
- `Deployment`
- `ReplicaSet`
- `Service`
- `ConfigMap`
- `Secret`
- `Ingress`
- `Namespace`
- `PersistentVolumeClaim`

## Como leer un manifiesto

En la mayoria de los manifiestos vas a ver estas partes:

`apiVersion`

- Define que version de API usa el recurso.

`kind`

- Define que tipo de recurso se esta creando.

`metadata`

- Define informacion descriptiva, como el nombre y las etiquetas.

`spec`

- Define el comportamiento deseado del recurso.

En resumen:

- `apiVersion` dice que API usar
- `kind` dice que objeto es
- `metadata` dice como se llama y como se etiqueta
- `spec` dice como debe funcionar

## Que se hizo en LAB2

El flujo del laboratorio fue este:

1. Se definio un `Deployment` en `deploy-base.yml`.
2. Ese `Deployment` crea 6 Pods iguales.
3. Cada Pod ejecuta un contenedor que escucha en el puerto `80`.
4. Todos esos Pods llevan la etiqueta `app: web`.
5. Se definio un `Service` en `service-base.yml`.
6. Ese `Service` busca todos los Pods que tengan `app: web`.
7. El `Service` expone el puerto interno `80` y lo publica hacia afuera por el `NodePort` `30088`.

En otras palabras:

- El `Deployment` administra la aplicacion.
- El `Service` le da un punto de entrada estable.
- Las etiquetas (`labels`) conectan ambos recursos.

## Esquema conceptual

```text
Usuario/Navegador
       |
       v
NodePort 30088
       |
       v
Service web-svc
       |
       v
Pods con label app=web
       |
       v
Contenedor maldoariel/miapp-server:v1 en puerto 80
```

## Explicacion detallada de `deploy-base.yml`

Archivo original:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-d
  labels:
    estado: "1"
spec:
  selector:
    matchLabels:
      app: web
  replicas: 6
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: apache-php
        image: maldoariel/miapp-server:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

### Linea por linea

`apiVersion: apps/v1`

- Indica la version de la API de Kubernetes que se usa para este recurso.
- `Deployment` pertenece al grupo `apps`, por eso se usa `apps/v1`.

`kind: Deployment`

- Define el tipo de objeto que se va a crear.
- En este caso, Kubernetes va a crear un `Deployment`.

`metadata:`

- Agrupa los datos descriptivos del recurso.
- No define el comportamiento tecnico del contenedor; define identidad y etiquetas del objeto.

`  name: web-d`

- Es el nombre del `Deployment`.
- Kubernetes va a registrar este recurso con el nombre `web-d`.
- Luego se lo puede consultar con comandos como `kubectl get deployment web-d`.

`  labels:`

- Abre la seccion de etiquetas del propio `Deployment`.
- Las `labels` son pares clave-valor usados para organizar, filtrar o relacionar recursos.

`    estado: "1"`

- Agrega una etiqueta llamada `estado` con valor `"1"`.
- Esta etiqueta queda asociada al `Deployment`.
- No participa en la seleccion del `Service`; parece una etiqueta descriptiva o de practica.

`spec:`

- A partir de aca se define el comportamiento deseado del `Deployment`.
- Es la parte mas importante del recurso.

`  selector:`

- Le dice al `Deployment` como reconocer los Pods que administra.
- Esto debe coincidir con las etiquetas del `template`.

`    matchLabels:`

- Define un criterio de seleccion exacto por etiquetas.
- Todo Pod con estas etiquetas sera considerado parte del `Deployment`.

`      app: web`

- El `Deployment` buscara Pods con la etiqueta `app=web`.
- Esta etiqueta debe existir tambien en la plantilla del Pod.

`  replicas: 6`

- Indica que Kubernetes debe mantener 6 Pods activos.
- Si se cae uno, el `Deployment` intenta recrearlo.
- Si hubiera menos o mas Pods administrados, Kubernetes corrige el estado hasta volver a 6.

`  template:`

- Define la plantilla que Kubernetes usara para crear cada Pod.
- Todo Pod nuevo generado por este `Deployment` sale de esta plantilla.

`    metadata:`

- Metadatos del Pod que va a ser creado desde la plantilla.

`      labels:`

- Etiquetas que llevara cada Pod creado por el `Deployment`.

`        app: web`

- Cada Pod creado va a tener la etiqueta `app=web`.
- Esta linea es clave porque debe coincidir con el `selector` del `Deployment` y con el `selector` del `Service`.

`    spec:`

- Define la especificacion interna del Pod.
- Aca se configura el contenedor o los contenedores que viviran dentro del Pod.

`      containers:`

- Lista de contenedores del Pod.
- Aunque en este caso hay uno solo, Kubernetes espera una lista.

`      - name: apache-php`

- Nombre interno del contenedor dentro del Pod.
- Sirve para identificarlo en eventos, logs y descripciones.

`        image: maldoariel/miapp-server:v1`

- Define la imagen de Docker que se va a descargar y ejecutar.
- `maldoariel/miapp-server` es el repositorio de imagen.
- `v1` es el tag o version concreta de la imagen.

`        ports:`

- Lista de puertos expuestos por el contenedor a nivel declarativo.
- No publica el puerto afuera por si solo, pero documenta y organiza el manifiesto.

`        - containerPort: 80`

- Indica que la aplicacion dentro del contenedor escucha en el puerto `80`.
- Esto ayuda a conectar correctamente el `Service` con la aplicacion.

`        resources:`

- Define los recursos de CPU y memoria pedidos y permitidos para este contenedor.
- Esto evita que el contenedor consuma recursos sin control.

`          requests:`

- Marca el minimo de recursos que Kubernetes intentara reservar para este contenedor.
- Se usa para decidir en que nodo ubicarlo.

`            cpu: "100m"`

- Solicita 100 millicores de CPU.
- Equivale a `0.1` CPU.

`            memory: "128Mi"`

- Solicita 128 mebibytes de memoria como minimo garantizado.

`          limits:`

- Define el maximo de recursos que el contenedor puede usar.
- Si supera memoria, puede ser finalizado.
- Si supera CPU, normalmente se limita su consumo.

`            cpu: "250m"`

- Limita el uso maximo de CPU a 250 millicores.
- Equivale a `0.25` CPU.

`            memory: "256Mi"`

- Limita el uso maximo de memoria a 256 mebibytes.

## Resumen funcional de `deploy-base.yml`

Este archivo hace que Kubernetes:

- cree un `Deployment` llamado `web-d`
- mantenga `6` replicas activas
- use la imagen `maldoariel/miapp-server:v1`
- etiquete los Pods como `app=web`
- exponga internamente la aplicacion en el puerto `80`
- controle recursos minimos y maximos del contenedor

## Explicacion detallada de `service-base.yml`

Archivo original:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  labels:
     app: web
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30088
    protocol: TCP
  selector:
     app: web
```

### Linea por linea

`apiVersion: v1`

- Define la version de API usada por este recurso.
- Los `Service` pertenecen al grupo base `v1`.

`kind: Service`

- Indica que el recurso a crear es un `Service`.
- Un `Service` da una forma estable de acceder a uno o varios Pods.

`metadata:`

- Agrupa los datos descriptivos del `Service`.

`  name: web-svc`

- Es el nombre del `Service`.
- Kubernetes lo registra con ese identificador.

`  labels:`

- Etiquetas del propio `Service`.
- Sirven para organizacion y filtrado.

`     app: web`

- Etiqueta descriptiva del `Service`.
- No es lo mismo que el `selector`, aunque casualmente usa el mismo valor.

`spec:`

- Define como funciona el `Service`.

`  type: NodePort`

- Indica que el `Service` se expone mediante un puerto del nodo.
- Permite entrar desde afuera del cluster usando `IP_del_nodo:nodePort`.

`  ports:`

- Lista de puertos que maneja el `Service`.
- Un `Service` puede mapear uno o varios puertos.

`  - port: 80`

- Es el puerto del `Service` dentro del cluster.
- Otros Pods pueden consumir este `Service` por el puerto `80`.

`    nodePort: 30088`

- Es el puerto externo publicado en cada nodo del cluster.
- Si el entorno lo permite, se accede desde afuera con `IP_del_nodo:30088`.

`    protocol: TCP`

- Define el protocolo de red usado por el puerto.
- Para trafico web HTTP se usa TCP.

`  selector:`

- Le dice al `Service` a que Pods debe enviar el trafico.

`     app: web`

- El `Service` selecciona todos los Pods que tengan la etiqueta `app=web`.
- Como los Pods del `Deployment` tienen exactamente esa etiqueta, el trafico se balancea entre ellos.

## Relacion entre `Deployment` y `Service`

La union entre ambos archivos ocurre por las etiquetas.

En `deploy-base.yml`:

- los Pods creados llevan `app: web`

En `service-base.yml`:

- el `Service` busca Pods con `app: web`

Eso significa que:

- el `Deployment` crea Pods
- el `Service` encuentra esos Pods
- el `Service` reparte el trafico hacia ellos

Si las etiquetas no coincidieran, el `Service` no encontraria ningun Pod y no habria conexion.

## Comandos usados habitualmente en este LAB

Aplicar los manifiestos:

```powershell
kubectl apply -f deploy-base.yml
kubectl apply -f service-base.yml
```

Verificar recursos:

```powershell
kubectl get deployments
kubectl get pods
kubectl get svc
kubectl get endpoints
```

Ver detalle:

```powershell
kubectl describe deployment web-d
kubectl describe service web-svc
```

Obtener URL desde Minikube:

```powershell
minikube service web-svc --url
```

Alternativa con port-forward:

```powershell
kubectl port-forward service/web-svc 8080:80
```

## Comandos mas utilizados en Kubernetes

Estos son algunos de los comandos mas usados cuando se trabaja con Kubernetes.

### 1. Crear o actualizar recursos

```powershell
kubectl apply -f archivo.yml
kubectl apply -f carpeta/
```

Se usan para crear o actualizar recursos a partir de manifiestos.

### 2. Ver recursos

```powershell
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes
kubectl get all
kubectl get pods -A
```

Sirven para listar recursos del cluster.

### 3. Ver detalles de un recurso

```powershell
kubectl describe pod NOMBRE
kubectl describe deployment NOMBRE
kubectl describe service NOMBRE
```

Muestran informacion ampliada, eventos, puertos, labels y estado.

### 4. Ver logs de contenedores

```powershell
kubectl logs NOMBRE_DEL_POD
kubectl logs -f NOMBRE_DEL_POD
```

Sirven para revisar la salida del contenedor y seguirla en tiempo real.

### 5. Ejecutar comandos dentro de un contenedor

```powershell
kubectl exec -it NOMBRE_DEL_POD -- sh
kubectl exec -it NOMBRE_DEL_POD -- bash
```

Permiten entrar al contenedor para inspeccion o pruebas.

### 6. Eliminar recursos

```powershell
kubectl delete -f archivo.yml
kubectl delete pod NOMBRE
kubectl delete service NOMBRE
kubectl delete deployment NOMBRE
```

Sirven para borrar recursos del cluster.

### 7. Escalar un Deployment

```powershell
kubectl scale deployment web-d --replicas=3
```

Permite aumentar o disminuir la cantidad de Pods administrados por un `Deployment`.

### 8. Exponer recursos localmente

```powershell
kubectl port-forward pod/NOMBRE 8080:80
kubectl port-forward service/NOMBRE 8080:80
```

Sirven para redirigir un puerto local hacia un Pod o un Service del cluster.

### 9. Consultar contexto y cluster

```powershell
kubectl config current-context
kubectl config get-contexts
kubectl cluster-info
kubectl get nodes
```

Ayudan a saber contra que cluster estas trabajando.

### 10. Ver eventos del cluster

```powershell
kubectl get events
kubectl get events --sort-by=.metadata.creationTimestamp
```

Son utiles para diagnosticar errores de despliegue o scheduling.

### 11. Trabajar con namespaces

```powershell
kubectl get namespaces
kubectl get pods -n default
kubectl get pods -A
```

Permiten ver o filtrar recursos por namespace.

## Resumen practico de comandos basicos

Para un flujo normal de trabajo, los comandos mas usados suelen ser estos:

```powershell
kubectl apply -f archivo.yml
kubectl get pods
kubectl get svc
kubectl describe pod NOMBRE
kubectl logs NOMBRE
kubectl delete -f archivo.yml
```

Con esos comandos ya se puede:

- desplegar recursos
- comprobar estado
- revisar detalles
- mirar logs
- eliminar recursos

## Conclusiones del LAB2

Este laboratorio muestra un patron muy comun de Kubernetes:

- un `Deployment` administra replicas de una aplicacion
- un `Service` publica esas replicas con una direccion estable
- las `labels` y `selectors` son el mecanismo que conecta ambos recursos

Es una base importante porque casi todos los despliegues reales en Kubernetes parten de esta misma idea.
