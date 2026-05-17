# Volúmenes Persistentes en Kubernetes — Guía para principiantes

Esta guía explica los manifiestos de `LAB3` paso a paso: qué hace cada archivo, por qué existe, y cómo ejecutar y verificar la persistencia de datos en un clúster Kubernetes. Está escrita para una persona que se inicia en Kubernetes.

**Archivos en la carpeta `LAB3`**
- `pv.yml` — Define un PersistentVolume (PV) local que provee almacenamiento físico al clúster.
- `pvc.yml` — Define un PersistentVolumeClaim (PVC) que solicita almacenamiento del clúster.
- `nginx-persistente.yml` — Manifiesto de un `Pod` que monta el `PVC` para servir contenido persistente con `nginx`.
- `service-nginx-persis.yml` — Servicio que expone el `Pod` (tipo `NodePort`).

---

## 1) `pv.yml` — ¿Qué es y por qué?
Contenido (resumen):
- `kind: PersistentVolume` — Recurso que representa almacenamiento real disponible en el clúster.
- `hostPath.path: "/mnt/persistent-volume"` — Ruta del host (solo para pruebas locales o minikube). No usar hostPath en producción.
- `capacity.storage: 1Gi` y `accessModes: [ReadWriteOnce]` — Capacidad y modo de acceso.

Por qué: el PV es la pieza que realmente tiene el espacio en disco. El `PVC` se enlaza a un PV compatible.

Notas para principiantes:
- `hostPath` monta una carpeta del nodo. En entornos gestionados (GKE/AKS/EKS) se usa un provisionador dinámico o un `StorageClass` adecuado.
- `storageClassName: manual` aquí indica que no hay provisionamiento dinámico; el PV debe existir antes de que el PVC lo reclame.

---

## 2) `pvc.yml` — ¿Qué es y para qué sirve?
Contenido (resumen):
- `kind: PersistentVolumeClaim` — Solicita almacenamiento al clúster.
- `resources.requests.storage: 1Gi` — Cantidad solicitada.
- `accessModes: [ReadWriteOnce]` y `storageClassName: manual` — Debe coincidir con el PV disponible.

Por qué: los Pods no piden PVs directamente; piden PVCs. Kubernetes busca un PV que satisfaga la petición y los enlaza.

---

## 3) `nginx-persistente.yml` — El Pod con volumen montado
Contenido (resumen):
- `volumes` referencia el `persistentVolumeClaim.claimName: persistent-volume-claim`.
- En `containers` el `volumeMounts` monta ese volumen en `/usr/share/nginx/html` (la carpeta donde nginx sirve contenido estático).
- Incluye `resources` (requests/limits) — buena práctica para evitar "noisy neighbor".

Por qué: al montar el PVC dentro del contenedor, cualquier fichero escrito en `/usr/share/nginx/html` permanece disponible aún si el Pod se reinicia o se recrea (siempre que el mismo PVC/PV exista y sea compatible).

---

## 4) `service-nginx-persis.yml` — Exponer el nginx
Contenido (resumen):
- `kind: Service`, `type: NodePort` y `ports.port: 80`.
- Selecciona por label `app: ng-persistente` para dirigir tráfico al Pod.

Por qué: permite acceder a nginx desde fuera del nodo (por ejemplo, mediante `curl http://<node-ip>:<nodePort>`). Para entornos locales puede ser conveniente para comprobar el contenido persistente.

---

## 5) ¿Por qué usar Deployment en vez de Pod?

### Diferencias clave
- `Pod`:
  - Es una sola unidad de ejecución.
  - Si se elimina, no se vuelve a crear automáticamente.
  - No gestiona escalado ni actualizaciones.
- `Deployment`:
  - Crea un `ReplicaSet` automáticamente.
  - Mantiene el número de réplicas deseado.
  - Facilita actualizaciones continuas (`rolling update`) y escalado.

### ¿Qué sucede con el ReplicaSet?
Al aplicar un `Deployment`, Kubernetes crea y gestiona un `ReplicaSet` internamente. El `ReplicaSet` controla los Pods y garantiza que el número de réplicas sea el correcto.

### Mismo volumen, misma forma de montar
Un `Deployment` usa el mismo `volumeMounts` y `volumes` que un `Pod`. Para persistencia, el `PVC` se monta de la misma manera, incluso dentro de un `Deployment`.

### Ejemplo de `Deployment` equivalente
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-persistente-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ng-persistente
  template:
    metadata:
      labels:
        app: ng-persistente
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: http-server
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nginx-storage
      volumes:
      - name: nginx-storage
        persistentVolumeClaim:
          claimName: persistent-volume-claim
```

### Recomendación
Usa `Deployment` cuando quieras:
- que el Pod se recupere solo si falla,
- actualizar la aplicación sin downtime,
- escalar el número de réplicas.

---

## Flujo recomendado (orden de despliegue paso a paso)
1. Revisa los archivos de manifiesto para comprender cada recurso.

2. Aplicar el PV (si no existe):

```bash
kubectl apply -f pv.yml
```

Salida esperada:
```text
persistentvolume/persistent-volume created
```

3. Crear el PVC:

```bash
kubectl apply -f pvc.yml
```

Salida esperada:
```text
persistentvolumeclaim/persistent-volume-claim created
```

4. Verificar que el PVC quedó enlazado al PV:

```bash
kubectl get pv
kubectl get pvc
kubectl describe pvc persistent-volume-claim
```

Salida esperada de `kubectl get pv`:
```text
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                          STORAGECLASS   REASON   AGE
persistent-volume    1Gi        RWO            Retain           Bound    default/persistent-volume-claim   manual                    1m
```

Salida esperada de `kubectl get pvc`:
```text
NAME                       STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistent-volume-claim     Bound    persistent-volume   1Gi        RWO            manual         1m
```

En `kubectl describe pvc persistent-volume-claim` verás líneas como:
```text
Status: Bound
Volume: persistent-volume
```

5. Crear el Pod (nginx) y el Service:

```bash
kubectl apply -f nginx-persistente.yml
kubectl apply -f service-nginx-persis.yml
```

Salida esperada:
```text
pod/nginx-persitente created
service/np-svc created
```

6. Verifica el Pod y el Service:

```bash
kubectl get pods -o wide
kubectl describe pod nginx-persitente
kubectl get svc np-svc
```

Salida esperada de `kubectl get pods -o wide`:
```text
NAME              READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx-persitente  1/1     Running   0          20s   10.0.0.12    minikube   <none>           <none>
```

Salida esperada de `kubectl get svc np-svc`:
```text
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
np-svc  NodePort   10.96.0.1      <none>        80:XXXXX/TCP   20s
```

7. Acceder a nginx:

- Con Minikube:

```bash
minikube service np-svc --url
```

Salida esperada:
```text
http://192.168.49.2:XXXXX
```

- Con NodePort, desde otra terminal:

```bash
curl http://<node-ip>:<nodePort>
```

Si todo está bien, verás la página por defecto de nginx o el contenido persistente que escribiste en `/usr/share/nginx/html`.

---

## Probar la persistencia (ejemplo sencillo)
1. Obtén un shell dentro del Pod:

```bash
kubectl exec -it nginx-persitente -- /bin/sh
```

2. Dentro del contenedor, crea un archivo index.html:

```sh
echo "Hola desde volumen persistente" > /usr/share/nginx/html/index.html
```

3. Sal del Pod y elimina el Pod (simular reinicio):

```bash
kubectl delete pod nginx-persitente
kubectl apply -f nginx-persistente.yml
```

4. Cuando el Pod se vuelva a crear, visita la URL del servicio. Deberías seguir viendo el `index.html` creado, lo que demuestra persistencia.

Notas: si usas `hostPath`, el archivo también estará en la ruta del nodo (`/mnt/persistent-volume`) que está apuntada por el PV.

---

## Comandos útiles para depuración
- `kubectl get pv,pvc,pods,svc` — visión rápida.
- `kubectl describe pvc persistent-volume-claim` — ver por qué no se enlaza.
- `kubectl describe pv persistent-volume` — ver `claimRef` y estado.
- `kubectl logs <pod>` — revisar errores de contenedor.
- `kubectl exec -it <pod> -- ls -la /usr/share/nginx/html` — comprobar archivos montados.

Problemas comunes:
- `PVC` en `Pending`: generalmente no existe PV compatible o `storageClass` no coincide.
- `mountPath` inválido: revisar que no tenga `:` y que la ruta sea correcta.
- `hostPath` no accesible: en algunos sistemas la ruta del host debe existir y tener permisos adecuados.

---

## Limpieza
```bash
kubectl delete -f service-nginx-persis.yml
kubectl delete -f nginx-persistente.yml
kubectl delete -f pvc.yml
kubectl delete -f pv.yml
```

---

## Recomendaciones finales y recursos
- Para producción, evita `hostPath`. Usa un `StorageClass` con provisionamiento dinámico (ej. `gp2`, `standard` según proveedor).
- Aprende sobre `StatefulSet` cuando gestionas aplicaciones con almacenamiento persistente que requieren identidad estable.
- Documentación oficial: https://kubernetes.io/docs/concepts/storage/

---

Si quieres, puedo:
- Añadir capturas de comandos y salidas esperadas.
- Adaptar los manifiestos para `Deployment` en vez de `Pod`.
- Añadir un `README.md` con comandos de `minikube` si usas `minikube`.


