# LAB3 - Volúmenes Persistentes en Minikube

Este README contiene los comandos básicos para ejecutar los manifiestos de `LAB3` usando Minikube.

## Requisitos
- `minikube` instalado
- `kubectl` instalado y configurado
- Minikube corriendo

## Iniciar Minikube
```bash
minikube start
```

Si necesitas aumentar recursos para el clúster:
```bash
minikube start --memory=4096 --cpus=2
```

## Ver el estado de Minikube
```bash
minikube status
```

## Aplicar los recursos de LAB3
```bash
kubectl apply -f pv.yml
kubectl apply -f pvc.yml
kubectl apply -f nginx-persistente.yml
kubectl apply -f service-nginx-persis.yml
```

## Verificar el estado
```bash
kubectl get pv
kubectl get pvc
kubectl get pods
kubectl get svc
```

## Acceder a nginx desde el navegador
```bash
minikube service np-svc --url
```

Abre la URL que devuelve el comando en tu navegador.

## Probar persistencia de datos
1. Entra al Pod:
```bash
kubectl exec -it nginx-persitente -- /bin/sh
```
2. Crea un archivo en el volumen montado:
```sh
echo "Hola desde volumen persistente" > /usr/share/nginx/html/index.html
```
3. Sal del Pod y elimina el Pod:
```bash
kubectl delete pod nginx-persitente
kubectl apply -f nginx-persistente.yml
```
4. Revisa la página en el browser. Deberías ver el contenido creado.

## Detener Minikube
```bash
minikube stop
```

## Eliminar los recursos
```bash
kubectl delete -f service-nginx-persis.yml
kubectl delete -f nginx-persistente.yml
kubectl delete -f pvc.yml
kubectl delete -f pv.yml
```

## Notas
- En Minikube, `hostPath` funciona localmente porque el PV usa la ruta del nodo virtual.
- Si usas otro entorno, revisa el `StorageClass` y el tipo de volumen adecuado.
