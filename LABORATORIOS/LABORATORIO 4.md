# Configuración de acceso TLS a la API de Kubernetes

Este ejercicio muestra cómo hacer llamadas directas al API server de Kubernetes usando TLS, con `curl` y los certificados que `kubectl` usa internamente. Así puedes entender mejor cómo funciona la API y no depender solo de `kubectl`.

---

### 1. Revisar el archivo de configuración de kubectl

```bash
less $HOME/.kube/config
```

* **Por qué:** Aquí está la configuración para acceder al clúster, incluyendo las direcciones del servidor API y los certificados codificados en base64 que permiten autenticación segura.

---

### 2. Extraer el certificado de cliente

```bash
export client=$(grep client-cert $HOME/.kube/config | cut -d" " -f 6)
echo $client
```

* **Por qué:** Necesitamos el certificado de cliente para autenticarnos en el servidor API usando TLS.

---

### 3. Extraer la clave privada del cliente

```bash
export key=$(grep client-key-data $HOME/.kube/config | cut -d" " -f 6)
echo $key
```

* **Por qué:** El certificado solo no es suficiente, también se requiere la clave privada correspondiente.

---

### 4. Extraer el certificado de la autoridad certificadora (CA)

```bash
export auth=$(grep certificate-authority-data $HOME/.kube/config | cut -d" " -f 6)
echo $auth
```

* **Por qué:** El certificado CA verifica la identidad del servidor para evitar conexiones inseguras.

---

### 5. Decodificar los certificados y la clave en archivos

```bash
echo $client | base64 -d - > ./client.pem
echo $key | base64 -d - > ./client-key.pem
echo $auth | base64 -d - > ./ca.pem
```

* **Por qué:** Los datos están codificados en base64, deben decodificarse para ser usados con `curl`.

---

### 6. Obtener la URL del servidor API

```bash
kubectl config view | grep server
```

* **Por qué:** Para saber la dirección y puerto del API server, que puede variar según la instalación.

---

### 7. Usar curl para consultar la API y obtener la lista de pods

```bash
curl --cert ./client.pem \
     --key ./client-key.pem \
     --cacert ./ca.pem \
     https://k8scp:6443/api/v1/pods
```

* **Por qué:** Se hace una petición GET autenticada al endpoint `/api/v1/pods`, obteniendo la lista de pods en formato JSON.

---

### 8. Crear un archivo JSON para definir un nuevo pod

```bash
cp /home/student/LFS258/SOLUTIONS/s_05/curlpod.json .
vim curlpod.json
```

Contenido del JSON (ejemplo):

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

* **Por qué:** Este JSON define el objeto Pod que queremos crear vía API.

---

### 9. Crear el pod con una petición POST usando curl

```bash
curl --cert ./client.pem \
     --key ./client-key.pem \
     --cacert ./ca.pem \
     https://k8scp:6443/api/v1/namespaces/default/pods \
     -XPOST -H 'Content-Type: application/json' \
     -d @curlpod.json
```

* **Por qué:** Esta es una llamada POST al API server que crea un nuevo pod en el namespace `default`.

---

### 10. Verificar que el pod se haya creado y esté corriendo

```bash
kubectl get pods
```

* **Por qué:** Confirmar que el pod `curlpod` existe y su estado es `Running`.

---

# Resumen

* Extraemos los certificados TLS desde el archivo de configuración de `kubectl`.
* Usamos `curl` con estos certificados para hacer llamadas directas a la API de Kubernetes.
* Creamos y verificamos un nuevo pod sin usar directamente `kubectl create`, sino con una petición REST.
* Esto ayuda a entender cómo Kubernetes maneja su estado deseado a través de su API.

# Explorar llamadas a la API (Detalles completos)

---

### 1. Instalar `strace` y obtener endpoints

Primero, instala la herramienta `strace` si no está presente, para poder inspeccionar las llamadas al sistema de un comando:

```bash
sudo apt-get install -y strace
```

Luego, ejecuta el siguiente comando para listar los endpoints del clúster:

```bash
kubectl get endpoints
```

Salida esperada (ejemplo):

```
NAME         ENDPOINTS          AGE
kubernetes   10.128.0.3:6443    3h
```

---

### 2. Ejecutar `kubectl get endpoints` con `strace`

Ejecuta el mismo comando pero precedido con `strace` para ver todas las llamadas al sistema que hace:

```bash
strace kubectl get endpoints
```

* La salida será muy larga.
* Cerca del final, busca líneas con llamadas a `openat()` hacia un directorio local como:

```
/home/student/.kube/cache/discovery/k8scp_6443
```

* Esto indica que `kubectl` cachea información de los recursos disponibles en el API server.
* Puede ser útil redireccionar la salida a un archivo y buscar dentro con `grep` si no lo ves en pantalla.

---

### 3. Cambiar al directorio de cacheo discovery

Ve al directorio cacheado que encontraste en la salida de `strace`. Por ejemplo:

```bash
cd /home/student/.kube/cache/discovery
ls
```

Verás algo así:

```
k8scp_6443
```

Cambia a ese directorio:

```bash
cd k8scp_6443/
```

---

### 4. Explorar los contenidos del directorio discovery

Haz un listado del contenido:

```bash
ls -l
```

Verás carpetas y archivos JSON que contienen información sobre distintos grupos de API y versiones. Ejemplos:

```
admissionregistration.k8s.io
certificates.k8s.io
node.k8s.io
apiextensions.k8s.io
policy
rbac.authorization.k8s.io
apps
batch
networking.k8s.io
...
servergroups.json
```

---

### 5. Listar todos los archivos dentro del directorio

Para ver todos los archivos dentro de este árbol, usa:

```bash
find .
```

La salida listará subdirectorios y archivos, por ejemplo:

```
./storage.k8s.io
./storage.k8s.io/v1beta1
./storage.k8s.io/v1beta1/serverresources.json
./storage.k8s.io/v1
./storage.k8s.io/v1/serverresources.json
./rbac.authorization.k8s.io
...
```

---

### 6. Ver objetos disponibles en la versión `v1` del API

Visualiza el contenido del archivo `v1/serverresources.json` usando el formateador JSON de Python:

```bash
python3 -m json.tool v1/serverresources.json
```

Este archivo describe:

* La versión de la API (`apiVersion` y `groupVersion`)
* Los recursos disponibles (`resources`), que incluyen para cada objeto:

  * `kind` (tipo del objeto, por ejemplo `Pod`, `Service`, `Endpoints`)
  * `name` (nombre plural)
  * `namespaced` (si el objeto está en un namespace o es global)
  * `verbs` (acciones que soporta, ej. `create`, `get`, `delete`, `watch`)
  * `shortNames` (abreviaturas para uso en la CLI)

Ejemplo parcial:

```json
{
  "apiVersion": "v1",
  "groupVersion": "v1",
  "kind": "APIResourceList",
  "resources": [
    {
      "kind": "Binding",
      "name": "bindings",
      "namespaced": true,
      "verbs": ["create"]
    },
    ...
  ]
}
```

---

### 7. Buscar el shortName para `endpoints`

Para encontrar el shortName de `endpoints`, puedes buscar en el JSON:

```bash
python3 -m json.tool v1/serverresources.json | less
```

Y buscar la sección correspondiente a `Endpoints`. Por ejemplo:

```json
{
  "kind": "Endpoints",
  "name": "endpoints",
  "namespaced": true,
  "shortNames": ["ep"],
  "singularName": "",
  "verbs": ["create", "delete", "get", "list", "patch", "update", "watch"]
}
```

Esto indica que puedes usar:

```bash
kubectl get ep
```

como abreviación de

```bash
kubectl get endpoints
```

---

### 8. Usar el shortName para listar endpoints

Ejecuta:

```bash
kubectl get ep
```

Esto debe mostrar la misma lista de endpoints que el comando largo.

---

### 9. Ver cuántos objetos hay en `v1`

Para contar todos los recursos listados en `v1/serverresources.json`:

```bash
python3 -m json.tool v1/serverresources.json | grep kind
```

Esto mostrará algo como:

```
"kind": "APIResourceList",
"kind": "Binding",
"kind": "ComponentStatus",
"kind": "ConfigMap",
"kind": "Endpoints",
"kind": "Event",
...
```

Se indica que hay **37 objetos** en esta versión.

---

### 10. Explorar otra API, `apps/v1`

Para ver los recursos en el grupo `apps/v1` (más enfocado a workloads):

```bash
python3 -m json.tool apps/v1/serverresources.json | grep kind
```

Algunos objetos pueden ser:

```
"kind": "APIResourceList",
"kind": "ControllerRevision",
"kind": "DaemonSet",
"kind": "Deployment",
"kind": "ReplicaSet",
"kind": "StatefulSet",
...
```

Se indica que hay **9 objetos** en este grupo.

---

### 11. Eliminar el pod creado `curlpod`

Para limpiar el clúster y liberar recursos:

```bash
kubectl delete po curlpod
```

Salida esperada:

```
pod "curlpod" deleted
```

---

### 12. Explorar otros archivos en el directorio cacheado

Finalmente, se recomienda que navegues y explores otros archivos dentro del directorio `.kube/cache/discovery/` para familiarizarte con la estructura y la información sobre la API Kubernetes.

---

# Resumen

Este ejercicio te muestra:

* Cómo usar `strace` para inspeccionar internamente qué hace `kubectl`.
* Dónde `kubectl` guarda cache de recursos API.
* Cómo explorar esos archivos JSON para entender qué recursos y acciones están disponibles.
* Cómo encontrar abreviaciones (shortNames) para facilitar el uso de `kubectl`.
* Cómo verificar y eliminar recursos creados.


