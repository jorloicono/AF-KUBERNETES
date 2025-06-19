
# Mantenimiento Básico de Nodos

En esta sección realizaremos un respaldo de la base de datos **etcd**, luego actualizaremos la versión de Kubernetes usada en los nodos del plano de control y nodos trabajadores.

---

## Respaldo de la base de datos etcd

Aunque el proceso de actualización es estable, siempre es recomendable **hacer un respaldo del estado del clúster antes de actualizar** para evitar pérdidas de datos ante cualquier falla.

Existen muchas herramientas para respaldar y gestionar etcd, pero usaremos el comando `snapshot` incluido. Ten en cuenta que la restauración depende de la herramienta, versión y tipo de desastre.

### Pasos para hacer el respaldo:

1. **Encontrar el directorio de datos del daemon etcd:**

   ```bash
   sudo grep data-dir /etc/kubernetes/manifests/etcd.yaml
   ```

   * Esto nos indica dónde está almacenada la base de datos, importante para saber dónde guardar el respaldo.

2. **Entrar al contenedor de etcd para conocer opciones del cliente `etcdctl`:**

   ```bash
   kubectl -n kube-system exec -it etcd-<TAB> -- sh
   etcdctl -h
   ```

   * Se inspeccionan las opciones para entender cómo interactuar con la base de datos, especialmente cómo usar TLS (seguridad).

3. **Identificar los archivos necesarios para TLS:**

   ```bash
   cd /etc/kubernetes/pki/etcd
   echo *
   ```

   * Estos archivos garantizan la conexión segura con la base de datos. Es fundamental porque etcd requiere comunicación segura para evitar accesos no autorizados.

4. **Verificar la salud de la base de datos con TLS:**

   ```bash
   kubectl -n kube-system exec -it etcd-cp -- sh -c "ETCDCTL_API=3 \
   ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
   ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
   ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
   etcdctl endpoint health"
   ```

   * Esto confirma que la base de datos está funcionando correctamente antes de realizar cualquier cambio o respaldo.

5. **Listar los miembros del clúster etcd (bases de datos):**

   ```bash
   kubectl -n kube-system exec -it etcd-cp -- sh -c "ETCDCTL_API=3 \
   ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
   ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
   ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
   etcdctl --endpoints=https://127.0.0.1:2379 member list"
   ```

   * Es importante saber cuántas bases de datos conforman el clúster para garantizar la consistencia y que el respaldo incluya toda la información necesaria.

6. **Guardar el snapshot (respaldo) en el directorio de datos:**

   ```bash
   kubectl -n kube-system exec -it etcd-cp -- sh -c "ETCDCTL_API=3 \
   ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
   ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
   ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
   etcdctl --endpoints=https://127.0.0.1:2379 snapshot save /var/lib/etcd/snapshot.db"
   ```

   * Se crea un archivo que contiene todo el estado actual de la base de datos, para poder restaurar en caso de falla.

7. **Verificar que el snapshot exista y se haya guardado correctamente:**

   ```bash
   sudo ls -l /var/lib/etcd/
   ```

   * Confirmar que el archivo realmente se creó evita sorpresas si se necesita restaurar.

8. **Copiar el snapshot y archivos necesarios a un lugar seguro:**

   ```bash
   mkdir $HOME/backup
   sudo cp /var/lib/etcd/snapshot.db $HOME/backup/snapshot.db-$(date +%m-%d-%y)
   sudo cp /root/kubeadm-config.yaml $HOME/backup/
   sudo cp -r /etc/kubernetes/pki/etcd $HOME/backup/
   ```

   * Guardar esta información fuera del nodo permite recuperar el clúster aún si el nodo falla por completo.

9. **Nota importante:**
   Restaurar la base de datos puede dejar el clúster inutilizable si no se hace correctamente. Es recomendable practicar restauraciones en un entorno de pruebas.

---

## Actualización del clúster

1. **Actualizar la metadata de paquetes para apt:**

   ```bash
   sudo apt update
   ```

   * Esto garantiza que los paquetes que instalemos estén actualizados y disponibles en los repositorios.

2. **Actualizar el repositorio de Kubernetes a la nueva ubicación:**

   ```bash
   sudo sed -i's/32/33/g' /etc/apt/sources.list.d/kubernetes.list
   sudo apt-get update
   ```

   * Kubernetes migró sus repositorios, por lo que es necesario apuntar a la nueva fuente para obtener las últimas versiones.

3. **Consultar las versiones disponibles de kubeadm:**

   ```bash
   sudo apt-cache madison kubeadm
   ```

   * Para verificar qué versiones podemos instalar y elegir la correcta.

4. **Eliminar la restricción (hold) en kubeadm y actualizar:**

   ```bash
   sudo apt-mark unhold kubeadm
   sudo apt-get install -y kubeadm=1.33.1-1.1
   ```

   * Se libera el paquete para actualizarlo a la nueva versión.

5. **Volver a colocar la restricción para evitar actualizaciones automáticas:**

   ```bash
   sudo apt-mark hold kubeadm
   ```

   * Así evitamos que actualizaciones inesperadas rompan la compatibilidad o introduzcan errores.

6. **Verificar que kubeadm se actualizó correctamente:**

   ```bash
   sudo kubeadm version
   ```

7. **Preparar el nodo de control para la actualización evacuando pods:**

   ```bash
   kubectl drain cp --ignore-daemonsets
   ```

   * Se evacúan pods para evitar interrupciones en el servicio durante la actualización. Se ignoran DaemonSets porque deben correr siempre.

8. **Planificar la actualización con `kubeadm upgrade plan`:**

   ```bash
   sudo kubeadm upgrade plan
   ```

   * Esto muestra qué cambios ocurrirán, qué versiones están disponibles y valida la salud del clúster antes de actualizar.

9. **Aplicar la actualización:**

   ```bash
   sudo kubeadm upgrade apply v1.33.1
   ```

   * Se actualiza el clúster a la versión deseada.

10. **Verificar estado del nodo después de la actualización:**

    ```bash
    kubectl get node
    ```

    * Para comprobar que el nodo está listo pero con programación deshabilitada.

11. **Liberar hold de kubelet y kubectl y actualizarlos:**

    ```bash
    sudo apt-mark unhold kubelet kubectl
    sudo apt-get install -y kubelet=1.33.1-1.1 kubectl=1.33.1-1.1
    sudo apt-mark hold kubelet kubectl
    ```

12. **Reiniciar kubelet para aplicar los cambios:**

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart kubelet
    ```

13. **Verificar versión actualizada del nodo de control y actualizar otros nodos si existen:**

    ```bash
    kubectl get node
    sudo kubeadm upgrade node
    ```

14. **Habilitar nuevamente el scheduling para el nodo de control:**

    ```bash
    kubectl uncordon cp
    kubectl get node
    ```

15. **Actualizar nodos trabajadores siguiendo un proceso similar (desbloquear paquetes, actualizar kubeadm, drenar nodo, etc.):**

    ```bash
    # En el nodo trabajador
    sudo apt-mark unhold kubeadm
    sudo sed -i's/32/33/g' /etc/apt/sources.list.d/kubernetes.list
    sudo apt-get update
    sudo apt-get install -y kubeadm=1.33.1-1.1
    sudo apt-mark hold kubeadm

    # En el nodo de control
    kubectl drain worker --ignore-daemonsets
    ```

---

Cada paso tiene el propósito de garantizar que el clúster se mantenga en un estado seguro, estable y funcional durante la actualización, evitando pérdidas de datos y minimizando el tiempo de inactividad.

---

# Trabajando con Restricciones de CPU y Memoria

En este ejercicio seguimos trabajando con nuestro clúster Kubernetes construido en el laboratorio anterior. El objetivo es aprender a aplicar **límites de recursos** (CPU y memoria) a un contenedor para controlar cuánto puede usar. También veremos cómo funcionan los **namespaces** y un despliegue más complejo para entender mejor la arquitectura y las relaciones.

---

## Pasos y explicación

### 1. Desplegar el contenedor `stress` para generar carga

```bash
kubectl create deployment hog --image vish/stress
kubectl get deployments
````

> **Por qué:** Desplegamos un contenedor que simula carga en la CPU y memoria para probar los límites. El comando `kubectl get deployments` verifica que el despliegue esté listo y funcionando.

---

### 2. Ver detalles del despliegue y confirmar que no hay límites de recursos

```bash
kubectl describe deployment hog
kubectl get deployment hog -o yaml
```

> **Por qué:** Es importante ver la configuración actual para confirmar que no hay restricciones de recursos aplicadas, lo que se refleja con `resources: {}` vacío. Esto indica que el contenedor puede usar recursos sin límites.

---

### 3. Exportar la configuración YAML del despliegue

```bash
kubectl get deployment hog -o yaml > hog.yaml
```

> **Por qué:** Exportamos el YAML para editarlo y agregar los límites de recursos manualmente.

---

### 4. Editar el archivo YAML para añadir límites de memoria

```yaml
resources:
  limits:
    memory: "4Gi"       # Limite máximo de memoria asignada
  requests:
    memory: "2500Mi"    # Memoria solicitada (mínima garantizada)
```

> **Por qué:**

* `requests` indica la cantidad mínima de memoria que Kubernetes reservará para el contenedor.
* `limits` es el máximo que el contenedor puede usar, evitando que consuma más y afecte a otros procesos.

También se eliminan campos irrelevantes como `status` y `creationTimestamp` para evitar conflictos en la actualización.

---

### 5. Reemplazar el despliegue con el nuevo archivo YAML

```bash
kubectl replace -f hog.yaml
```

> **Por qué:** Aplicamos la configuración editada para que Kubernetes establezca los límites de recursos en el contenedor.

---

### 6. Verificar que los límites fueron aplicados

```bash
kubectl get deployment hog -o yaml
```

> **Por qué:** Confirmamos que la configuración ahora incluye los límites y solicitudes de recursos que agregamos.

---

### 7. Revisar los logs del pod para ver la asignación de memoria

```bash
kubectl get pods
kubectl logs <nombre-del-pod>
```

> **Por qué:** Observamos que el contenedor `stress` reporta la cantidad de memoria que está usando, verificando que los límites se están aplicando en ejecución.

---

### 8. Monitorear el uso real de recursos en los nodos

* Abrir varias terminales para los nodos del clúster y ejecutar `top`.
* Verificar el uso de CPU y memoria de los procesos y contenedores.
* Usar `kubectl get events` para detectar si algún pod fue expulsado por falta de recursos.

> **Por qué:** Esto permite ver el impacto real de los límites y detectar problemas de asignación o sobrecarga.

---

### 9. Editar el archivo YAML para que `stress` consuma CPU y memoria, usando argumentos

```yaml
resources:
  limits:
    cpu: "1"
    memory: "4Gi"
  requests:
    cpu: "0.5"
    memory: "500Mi"
args:
  - --cpus
  - "2"
  - --mem-total
  - "950Mi"
  - --mem-alloc-size
  - "100Mi"
  - --mem-alloc-sleep
  - "1s"
```

> **Por qué:** Aquí configuramos el contenedor para que realmente consuma recursos de CPU y memoria, usando los argumentos que el programa `stress` acepta para simular la carga.

---

### 10. Eliminar y recrear el despliegue con los nuevos parámetros

```bash
kubectl delete deployment hog
kubectl create -f hog.yaml
```

> **Por qué:** El contenedor debe reiniciarse con la configuración actualizada para reflejar el nuevo consumo de recursos. Esto debería generar un aumento visible en uso de CPU y memoria.

---

### Nota sobre posibles errores

Si no ves aumento en el uso, o el pod entra en estado de error, revisa los logs con:

```bash
kubectl logs <nombre-del-pod>
```

> **Por qué:** Puede haber errores en los parámetros (como unidades mal escritas). Por ejemplo, `150mi` en vez de `150Mi` causará un error y el contenedor no consumirá recursos aunque esté corriendo.

---

Este ejemplo muestra cómo Kubernetes permite controlar los recursos de un contenedor para evitar que consuma más CPU o memoria de la que debería, asegurando estabilidad y eficiencia en el clúster. Aprender a configurar `requests` y `limits` es clave para manejar aplicaciones en producción sin impactar negativamente otros servicios.


# Límites de Recursos para un Namespace

En los pasos anteriores configuramos límites para un despliegue en particular. En este ejercicio aprenderemos a establecer límites para **todo un namespace**. Esto es útil para controlar los recursos máximos que pueden usar todos los pods dentro de un namespace específico.

---

## Pasos y explicación

### 1. Crear un nuevo namespace llamado `low-usage-limit` y verificar que existe

```bash
kubectl create namespace low-usage-limit
kubectl get namespace
````

> **Por qué:**
> Creamos un namespace nuevo para aislar el entorno donde aplicaremos límites. Kubernetes usa namespaces para separar recursos y configuraciones, lo que permite administrar recursos y políticas por grupos lógicos.

---

### 2. Crear un archivo YAML para definir los límites de CPU y memoria con un recurso tipo `LimitRange`

```bash
cp /home/student/LFS258/SOLUTIONS/s_04/low-resource-range.yaml .
vim low-resource-range.yaml
```

Contenido típico del archivo:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: low-resource-range
spec:
  limits:
  - default:
      cpu: "1"
      memory: 500Mi
    defaultRequest:
      cpu: 0.5
      memory: 100Mi
    type: Container
```

> **Por qué:**
> El objeto `LimitRange` permite definir límites y solicitudes predeterminadas para todos los contenedores dentro del namespace, asegurando que ningún contenedor pueda usar más recursos de los especificados.

* `default` es el límite máximo que se asignará si no se especifica otro.
* `defaultRequest` es la cantidad mínima garantizada.
* `type: Container` indica que los límites aplican a los contenedores.

---

### 3. Crear el objeto `LimitRange` en el namespace `low-usage-limit`

```bash
kubectl create -f low-resource-range.yaml -n low-usage-limit
```

> **Por qué:**
> Asignamos las reglas de límite al namespace recién creado, para que afecte a todos los pods que se ejecuten dentro.

---

### 4. Verificar que el `LimitRange` existe y está aplicado

```bash
kubectl get LimitRange             # En el namespace default (debería no haber ninguno)
kubectl get LimitRange --all-namespaces
```

> **Por qué:**
> Comprobamos que la configuración está activa y vinculada al namespace correcto.

---

### 5. Crear un nuevo despliegue dentro del namespace `low-usage-limit`

```bash
kubectl -n low-usage-limit create deployment limited-hog --image vish/stress
```

> **Por qué:**
> Desplegamos el mismo contenedor de estrés dentro del namespace con límites aplicados, para comparar su comportamiento con el despliegue en el namespace default.

---

### 6. Listar todos los despliegues en todos los namespaces

```bash
kubectl get deployments --all-namespaces
```

> **Por qué:**
> Podemos ver que el despliegue `hog` sigue en el namespace default y que `limited-hog` está en `low-usage-limit`. Esto confirma la separación de ambientes y la aplicación de límites según namespace.

---

### 7. Ver los pods dentro del namespace `low-usage-limit`

```bash
kubectl -n low-usage-limit get pods
```

> **Por qué:**
> Para confirmar que el pod se está ejecutando en el namespace con límites activos.

---

### 8. Ver los detalles del pod para confirmar que hereda los límites del namespace

```bash
kubectl -n low-usage-limit get pod <nombre-pod> -o yaml
```

> **Por qué:**
> Al inspeccionar el YAML del pod, veremos que automáticamente Kubernetes ha aplicado los límites de CPU y memoria definidos en el `LimitRange` del namespace.
> Esto es importante para entender cómo Kubernetes aplica políticas a nivel de namespace sin que el usuario tenga que configurar límites manualmente en cada despliegue.

---
En este apartado aprendimos a usar **LimitRanges** para definir límites de recursos en todo un namespace, asegurando que todos los contenedores que se creen dentro respeten ciertos límites de CPU y memoria. Esto es fundamental para administrar recursos en clústeres multiusuario o multiaplicación, previniendo que un contenedor consuma recursos excesivos y afecte a los demás.





