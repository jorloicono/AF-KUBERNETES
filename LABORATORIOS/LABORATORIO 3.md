# Configuración de acceso TLS

En este ejercicio aprenderemos a acceder de forma remota al servidor API de Kubernetes (`kube-apiserver`) usando TLS, sin depender directamente de `kubectl`, sino con `curl` y los certificados adecuados. Esto nos ayuda a entender cómo funcionan las llamadas API a bajo nivel.

---

## 1. Revisar el archivo de configuración de `kubectl`

```bash
less $HOME/.kube/config
````

> **Por qué:**
> Este archivo contiene la configuración para acceder al clúster, incluyendo las direcciones del API server y los certificados TLS necesarios para autenticarse.

---

## 2. Extraer el certificado de cliente codificado en base64

```bash
export client=$(grep client-cert $HOME/.kube/config | cut -d" " -f 6)
echo $client
```

> **Por qué:**
> Necesitamos extraer el certificado para usarlo con `curl` y autenticarnos correctamente en el API server.

---

## 3. Extraer la clave privada del cliente, también codificada en base64

```bash
export key=$(grep client-key-data $HOME/.kube/config | cut -d " " -f 6)
echo $key
```

> **Por qué:**
> La clave privada es necesaria junto con el certificado para establecer la conexión segura TLS.

---

## 4. Extraer el certificado de la autoridad certificadora (CA)

```bash
export auth=$(grep certificate-authority-data $HOME/.kube/config | cut -d " " -f 6)
echo $auth
```

> **Por qué:**
> El certificado CA verifica la identidad del servidor API para evitar conexiones inseguras o a servidores no autorizados.

---

## 5. Decodificar los certificados y claves para usarlos con `curl`

```bash
echo $client | base64 -d - > ./client.pem
echo $key | base64 -d - > ./client-key.pem
echo $auth | base64 -d - > ./ca.pem
```

> **Por qué:**
> Los certificados están almacenados en base64 dentro del archivo kubeconfig; para usarlos en `curl` debemos decodificarlos en archivos `.pem`.

---

## 6. Obtener la URL del servidor API

```bash
kubectl config view | grep server
```

> **Por qué:**
> Necesitamos conocer la dirección y puerto exacto del servidor API para hacer las peticiones directas con `curl`.

---

## 7. Usar `curl` para conectarse al API server y obtener los pods

```bash
curl --cert ./client.pem \
     --key ./client-key.pem \
     --cacert ./ca.pem \
     https://k8scp:6443/api/v1/pods
```

> **Por qué:**
> Esta llamada muestra cómo hacer una petición GET autenticada y segura para obtener la lista de pods. Así confirmamos que los certificados funcionan y que podemos acceder directamente al API.

---

## 8. Crear un archivo JSON para definir un nuevo pod

```bash
cp /home/student/LFS258/SOLUTIONS/s_05/curlpod.json .
vim curlpod.json
```

Contenido ejemplo:

```json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "curlpod",
    "namespace": "default",
    "labels": {
      "name": "examplepod"
    }
  },
  "spec": {
    "containers": [
      {
        "name": "nginx",
        "image": "nginx",
        "ports": [
          {
            "containerPort": 80
          }
        ]
      }
    ]
  }
}
```

> **Por qué:**
> Este JSON describe un pod básico con un contenedor `nginx`. Lo usaremos para crear un recurso directamente con la API.

---

## 9. Usar `curl` para enviar una petición POST y crear el pod

```bash
curl --cert ./client.pem \
     --key ./client-key.pem \
     --cacert ./ca.pem \
     https://k8scp:6443/api/v1/namespaces/default/pods \
     -XPOST -H 'Content-Type: application/json' \
     -d @curlpod.json
```

> **Por qué:**
> Esta es una petición directa para crear un nuevo pod usando la API REST de Kubernetes. El encabezado `Content-Type: application/json` indica que enviamos JSON.

---

## 10. Verificar que el pod se haya creado y esté en estado Running

```bash
kubectl get pods
```

> **Por qué:**
> Confirmamos que el pod fue creado correctamente y está activo, lo que indica que nuestra petición API funcionó.

---

# Resumen

* Usamos certificados TLS extraídos de `kubectl` para autenticar llamadas directas al API server con `curl`.
* Aprendimos cómo hacer peticiones GET y POST a la API REST de Kubernetes.
* Vimos la estructura JSON necesaria para crear un pod.
* Esta técnica es útil para entender cómo funciona Kubernetes internamente y cómo interactuar con la API desde fuera de `kubectl`.

# Explorando llamadas a la API de Kubernetes

En este ejercicio vamos a profundizar en cómo `kubectl` interactúa con la API de Kubernetes, explorando los endpoints y la información que `kubectl` cachea para optimizar sus llamadas.

---

## 1. Instalar `strace` y obtener los endpoints actuales

```bash
sudo apt-get install -y strace
kubectl get endpoints
````

> **Por qué:**
> `strace` es una herramienta que nos permite ver las llamadas al sistema que hace un proceso. Aquí queremos ver qué archivos o recursos usa `kubectl` para obtener los endpoints del clúster, y el comando `kubectl get endpoints` muestra los endpoints actuales.

---

## 2. Ejecutar el comando con `strace` para ver detalles

```bash
strace kubectl get endpoints
```

> **Por qué:**
> Esto genera mucha salida que muestra cómo `kubectl` lee archivos y realiza operaciones internas. Al final, veremos que `kubectl` abre archivos dentro del directorio cacheado, que contiene información sobre el clúster.

---

## 3. Cambiar al directorio padre del cache y explorar

```bash
cd /home/student/.kube/cache/discovery/
ls
```

> **Por qué:**
> Exploramos dónde `kubectl` guarda la información cacheada sobre el descubrimiento de la API y endpoints. Esto ayuda a entender cómo `kubectl` optimiza las consultas evitando hacer demasiadas llamadas a la API.

---

## 4. Entrar al directorio del host del servidor API

```bash
cd k8scp_6443/
ls
```

> **Por qué:**
> Aquí se almacenan archivos con definiciones de recursos, versiones y otros detalles de la API del servidor Kubernetes.

---

## 5. Usar `find` para listar todos los archivos dentro

```bash
find .
```

> **Por qué:**
> Esto nos da una visión completa de todos los archivos y recursos almacenados, incluyendo información detallada de versiones, recursos y configuraciones del clúster.

---

## 6. Ver contenido JSON de los recursos API versión `v1`

```bash
python3 -m json.tool v1/serverresources.json
```

> **Por qué:**
> Este archivo JSON lista todos los recursos (objetos) disponibles en la versión API `v1`, sus nombres, si son namespaced, y las acciones (verbs) que soportan (como create, delete, get, list).

---

## 7. Buscar el `shortName` para el recurso `endpoints`

```bash
python3 -m json.tool v1/serverresources.json | less
```

> **Por qué:**
> Algunos recursos tienen nombres cortos para facilitar su uso en la CLI de `kubectl`. Aquí buscamos el shortName para `endpoints` (por ejemplo, `ep`).

---

## 8. Usar el shortName para obtener endpoints

```bash
kubectl get ep
```

> **Por qué:**
> Usar el nombre corto facilita la escritura y es equivalente a `kubectl get endpoints`.

---

## 9. Contar la cantidad de objetos en el archivo `v1/serverresources.json`

```bash
python3 -m json.tool v1/serverresources.json | grep kind
```

> **Por qué:**
> Nos muestra todos los tipos de recursos (kinds) que están disponibles en esa versión del API. Esto da una idea del alcance y las funcionalidades del clúster.

---

## 10. Ver otros recursos disponibles en otra versión API

```bash
python3 -m json.tool apps/v1/serverresources.json | grep kind
```

> **Por qué:**
> Exploramos otros grupos de recursos en la API, por ejemplo, los objetos relacionados con aplicaciones como Deployments, DaemonSets, etc.

---

## 11. Eliminar el pod creado previamente para liberar recursos

```bash
kubectl delete po curlpod
```

> **Por qué:**
> Eliminamos recursos que ya no necesitamos para mantener limpio el clúster y no consumir recursos innecesarios.

---

## 12. Explorar otros archivos y carpetas para continuar aprendiendo

> **Por qué:**
> El directorio cache contiene mucha información sobre la configuración, recursos y APIs del clúster. Explorarla ayuda a entender el funcionamiento interno de Kubernetes y la interacción de `kubectl` con la API.

---

# Resumen

* `kubectl` usa un cache local para guardar información sobre la API, mejorando la eficiencia.
* Se puede usar `strace` para investigar qué archivos y llamadas hace `kubectl`.
* Los archivos JSON en `.kube/cache/discovery` describen los recursos y versiones de la API.
* Se usan nombres cortos para recursos comunes para facilitar su uso en la CLI.
* Explorar estos archivos ayuda a entender el diseño y uso de la API de Kubernetes.





