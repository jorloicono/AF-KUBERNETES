## Trabajando con ReplicaSets

---

### Concepto general

Un **ReplicaSet** es un objeto en Kubernetes que asegura que un número determinado de pods idénticos estén corriendo en un momento dado. ReplicaSets reemplazaron a los antiguos ReplicationControllers y son la base para los Deployments (que gestionan ReplicaSets de forma automática para actualizar y escalar aplicaciones).

---

### Paso 1: Ver ReplicaSets actuales

```bash
kubectl get rs
```

* **Por qué:**
  Para verificar si ya hay ReplicaSets corriendo en el namespace actual (normalmente `default`).
  Si no hay recursos, indica que el entorno está limpio y listo para trabajar.

---

### Paso 2: Crear archivo YAML para un ReplicaSet simple

```bash
cp /home/student/LFS258/SOLUTIONS/s_07/rs.yaml .
vim rs.yaml
```

Archivo contiene:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-one
spec:
  replicas: 2
  selector:
    matchLabels:
      system: ReplicaOne
  template:
    metadata:
      labels:
        system: ReplicaOne
    spec:
      containers:
      - name: nginx
        image: nginx:1.22.1
        ports:
        - containerPort: 80
```

* **Por qué:**
  Definimos un ReplicaSet para crear **2 pods** con la imagen `nginx:1.22.1`.
  La clave es el **selector** `matchLabels` que debe coincidir con las etiquetas de los pods (plantilla) para que el ReplicaSet los gestione correctamente.

---

### Paso 3: Crear el ReplicaSet

```bash
kubectl create -f rs.yaml
```

* **Por qué:**
  Aplicamos la definición para que Kubernetes cree el ReplicaSet y los pods asociados.

---

### Paso 4: Verificar ReplicaSet creado

```bash
kubectl describe rs rs-one
```

* **Por qué:**
  Confirmar que el ReplicaSet fue creado correctamente, que tiene el número deseado de réplicas (`replicas: 2`), y que los pods están corriendo.

---

### Paso 5: Ver pods creados por el ReplicaSet

```bash
kubectl get pods
```

* **Por qué:**
  Ver los pods que fueron creados para cumplir la especificación del ReplicaSet.
  Deberías ver 2 pods corriendo con etiquetas que coinciden con el selector.

---

### Paso 6: Eliminar el ReplicaSet pero no los pods (`--cascade=orphan`)

```bash
kubectl delete rs rs-one --cascade=orphan
```

* **Por qué:**
  El flag `--cascade=orphan` elimina solo el ReplicaSet pero **no elimina los pods** que controla, dejándolos "huérfanos".
  Esto es útil para entender cómo los pods sobreviven aunque el ReplicaSet desaparezca.

---

### Paso 7: Ver que ReplicaSet fue eliminado pero pods siguen

```bash
kubectl describe rs rs-one  # Error (No encontrado)
kubectl get pods            # Pods siguen corriendo
```

* **Por qué:**
  El ReplicaSet ya no existe, pero los pods siguen activos. Esto muestra que los pods no dependen exclusivamente del ReplicaSet para existir, pero no serán gestionados ni reemplazados si fallan.

---

### Paso 8: Crear el ReplicaSet nuevamente

```bash
kubectl create -f rs.yaml
```

* **Por qué:**
  Al volver a crear el ReplicaSet con el mismo selector, este **"reconoce"** los pods huérfanos existentes y vuelve a gestionarlos (los "adopta").
  Esto evita que Kubernetes cree pods duplicados.

---

### Paso 9: Ver la edad del ReplicaSet y de los pods

```bash
kubectl get rs
kubectl get pods
```

* **Por qué:**
  Para observar que el ReplicaSet y los pods están activos y funcionando, y que la edad de los pods no cambia (porque no se recrearon).

---

### Paso 10: Cambiar etiqueta de un pod para aislarlo

```bash
kubectl edit pod rs-one-3c6pb
```

Cambiar:

```yaml
labels:
  system: IsolatedPod
```

* **Por qué:**
  Al cambiar la etiqueta `system` de uno de los pods para que no coincida con el selector del ReplicaSet, el pod queda "aislado" y deja de estar gestionado por el ReplicaSet.

---

### Paso 11: Verificar que el ReplicaSet mantiene el número de pods deseado

```bash
kubectl get rs
```

* **Por qué:**
  El ReplicaSet detecta que tiene menos pods con la etiqueta `system=ReplicaOne` y crea un nuevo pod para mantener el número deseado (`replicas: 2`).

---

### Paso 12: Ver pods con etiqueta `system`

```bash
kubectl get pods -L system
```

* **Por qué:**
  Ahora hay 3 pods:

  * 2 con `system=ReplicaOne` (gestionados por ReplicaSet)
  * 1 con `system=IsolatedPod` (aislado, no gestionado)

---

### Paso 13: Eliminar el ReplicaSet

```bash
kubectl delete rs rs-one
```

* **Por qué:**
  Borramos el ReplicaSet para observar qué pasa con los pods.

---

### Paso 14: Verificar estado de los pods después de eliminar ReplicaSet

```bash
kubectl get pods
```

* **Por qué:**
  Algunos pods quedan corriendo (los huérfanos), pero otros pueden estar terminándose (si no tienen dueño, pueden ser eliminados eventualmente).

---

### Paso 15: Eliminar manualmente el pod aislado

```bash
kubectl delete pod -l system=IsolatedPod
```

* **Por qué:**
  Como el pod aislado no está gestionado, debe ser eliminado manualmente para limpiar el entorno.

---

# Trabajando con Deployments

---

### ¿Qué es un Deployment?

* Un **Deployment** es un recurso de Kubernetes que administra la creación y actualización de un conjunto de pods idénticos.
* Proporciona una forma **declarativa** de definir el estado deseado de una aplicación (número de réplicas, imagen del contenedor, etc.).
* Se encarga automáticamente de crear y gestionar **ReplicaSets**, que a su vez crean y mantienen los pods.
* Soporta **rolling updates**, es decir, actualizaciones sin tiempo de inactividad, reemplazando los pods antiguos de forma gradual.

---

### Paso 1: Crear un archivo YAML de Deployment usando el comando imperativo

```bash
kubectl create deploy webserver --image nginx:1.22.1 --replicas=2 --dry-run=client -o yaml | tee dep.yaml
```

* **Por qué:**

  * `kubectl create deploy` crea un Deployment llamado `webserver`.
  * `--image nginx:1.22.1` especifica la imagen del contenedor.
  * `--replicas=2` indica que queremos 2 pods idénticos.
  * `--dry-run=client` genera la configuración sin aplicarla en el cluster (solo local).
  * `-o yaml` formatea la salida en YAML.
  * `tee dep.yaml` guarda la configuración en el archivo `dep.yaml`.

Esto nos da un archivo declarativo que podemos usar para crear el Deployment de manera reproducible.

---

### Contenido del archivo `dep.yaml` (simplificado)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - name: nginx
        image: nginx:1.22.1
```

* **Por qué:**

  * `kind: Deployment` define que el recurso es un Deployment.
  * `metadata.name` es el nombre del Deployment.
  * `spec.replicas` define cuántos pods queremos.
  * `spec.selector` selecciona qué pods pertenecen a este Deployment, usando etiquetas.
  * `spec.template` define cómo deben ser los pods (etiquetas, contenedores, imágenes).

---

### Paso 2: Crear el Deployment y verificar

```bash
kubectl create -f dep.yaml
kubectl get deploy
kubectl get pods
```

* **Por qué:**

  * `kubectl create -f dep.yaml` aplica la configuración y crea el Deployment en el cluster.
  * `kubectl get deploy` lista los deployments y muestra cuántos pods están listos, disponibles y actualizados.
  * `kubectl get pods` muestra los pods creados por el Deployment, con sus estados.

En el ejemplo, ves:

```
webserver   2/2     2            2           14s
```

Esto significa que las 2 réplicas están listas y disponibles.

---

### Paso 3: Verificar la imagen que está corriendo dentro de los pods

```bash
kubectl describe pod webserver-6cbc654ddc-lssbm | grep Image:
```

* **Por qué:**

  * Esto permite confirmar que los pods creados usan la imagen `nginx:1.22.1`, como definimos en el Deployment.
  * Es importante para asegurar que el Deployment está corriendo la versión correcta de la aplicación.

---

### Conceptos clave que se aplican aquí

* **Declarativo vs Imperativo:**
  Usar `kubectl create` con `--dry-run` y `-o yaml` permite generar archivos declarativos que luego se pueden versionar y reaplicar, mejor práctica en infraestructuras modernas.

* **ReplicaSets y Pods:**
  El Deployment crea un ReplicaSet para gestionar el número de pods que definimos. ReplicaSets a su vez crean los pods.

* **Alta disponibilidad:**
  Al pedir 2 réplicas, si un pod falla, Kubernetes crea otro para mantener el estado deseado.

* **Rolling updates:**
  Más adelante, puedes actualizar la imagen en el Deployment y Kubernetes hará una actualización sin downtime.

---

## Rollout y Rollback usando Deployment

---

### Introducción general

El objetivo principal de este ejercicio es entender cómo actualizar una aplicación **sin interrupciones (rolling updates)** y cómo revertir (rollback) en caso de que la actualización cause problemas. Esto es clave en microservicios donde las aplicaciones deben estar siempre disponibles mientras se actualizan.

---

### Paso 1: Verificar la estrategia de actualización actual del Deployment

```bash
kubectl get deploy webserver -o yaml | grep -A 4 strategy
```

Salida esperada:

```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
  type: RollingUpdate
```

* **Por qué:**
  El Deployment tiene un campo `strategy` que define cómo Kubernetes actualiza los pods.

  * `type: RollingUpdate` indica que la actualización será gradual, reemplazando pods antiguos con nuevos sin downtime.
  * `maxSurge` y `maxUnavailable` controlan cuántos pods adicionales se crean y cuántos pueden estar indisponibles durante la actualización.

---

### Paso 2: Cambiar la estrategia a Recreate

```bash
kubectl edit deploy webserver
```

Modificar:

```yaml
strategy:
  type: Recreate
```

Eliminar líneas de `rollingUpdate` (maxSurge, maxUnavailable).

* **Por qué:**
  `Recreate` es una estrategia de actualización más simple donde Kubernetes elimina todos los pods antiguos antes de crear los nuevos. Esto puede causar downtime pero es útil cuando la actualización necesita reiniciar completamente la aplicación o no puede convivir con versiones antiguas simultáneamente.

---

### Paso 3: Actualizar la imagen del Deployment a una versión más nueva usando el comando `set image`

```bash
kubectl set image deploy webserver nginx=nginx:1.23.1-alpine --record
```

* **Por qué:**
  Cambiar la imagen a `nginx:1.23.1-alpine` para simular una actualización real del software.
  Usar `kubectl set image` es una forma rápida e imperativa para actualizar el contenedor sin editar el YAML manualmente.
  `--record` sirve para registrar esta acción en el historial de rollout (aunque ahora está deprecado).

---

### Paso 4: Verificar que los pods se están ejecutando con la nueva imagen

```bash
kubectl get pods
kubectl describe pod <pod-name> | grep Image:
```

* **Por qué:**
  Confirmar que la actualización realmente afectó los pods, y que los nuevos pods están usando la imagen `nginx:1.23.1-alpine`.

---

### Paso 5: Ver historial de despliegues

```bash
kubectl rollout history deploy webserver
```

Salida:

```
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deploy webserver nginx=nginx:1.23.1-alpine --record=true
```

* **Por qué:**
  Kubernetes mantiene un historial de versiones del Deployment para poder hacer rollback.
  Aquí vemos que hay dos revisiones: la original (1) y la actualización (2).

---

### Paso 6: Ver detalles de versiones específicas

```bash
kubectl rollout history deploy webserver --revision=1
kubectl rollout history deploy webserver --revision=2
```

* **Por qué:**
  Comparar la configuración de cada versión, donde lo principal que cambia es la imagen del contenedor.
  Esto ayuda a entender qué cambios ocurrieron en cada rollout.

---

### Paso 7: Revertir a la versión anterior (rollback)

```bash
kubectl rollout undo deploy webserver
```

* **Por qué:**
  Si la nueva versión tiene problemas, Kubernetes permite revertir el Deployment a una versión anterior estable, restaurando la configuración previa (por ejemplo, la imagen antigua). Esto es crucial para minimizar impactos en producción.

---

### Paso 8: Probar la estrategia RollingUpdate nuevamente

* Abre el YAML del Deployment y vuelve a cambiar la estrategia:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

* Cambia la imagen a una versión más nueva, por ejemplo:

```bash
kubectl set image deploy webserver nginx=nginx:1.26-alpine
```

* Aplica el cambio y observa que los pods se actualizan gradualmente sin downtime.

* **Por qué:**
  La estrategia `RollingUpdate` es preferible para producción porque garantiza alta disponibilidad.
  Kubernetes reemplaza pods antiguos de forma controlada, sin que todos estén caídos simultáneamente.

---

### Paso 9: Limpiar el entorno

```bash
kubectl delete deploy webserver
```

* **Por qué:**
  Siempre es buena práctica limpiar recursos que ya no usas para liberar espacio y evitar confusiones en el cluster.

---
Claro, aquí tienes un desglose paso a paso y con explicaciones del **Ejercicio 7.4: Working with DaemonSets** que acabas de compartir:

---

## Trabajando con DaemonSets

---

### Concepto general

Un **DaemonSet** es un objeto en Kubernetes que asegura que una copia de un pod esté corriendo en **cada nodo** del clúster (o en un conjunto de nodos definidos). Esto es diferente de un Deployment, que mantiene un número total de pods sin preocuparse de la distribución exacta por nodo.

Los DaemonSets son especialmente útiles para desplegar servicios que deben correr en todos los nodos, por ejemplo:

* Recolectores de logs (fluentd, logstash)
* Agentes de monitoreo (Prometheus node exporter)
* Servicios de red o seguridad (firewalls, proxies)
* Cualquier demonio que debe estar presente en cada nodo

---

### Paso 1: Crear el archivo YAML para un DaemonSet

Se parte de un archivo `rs.yaml` (ReplicaSet) y se copia a `ds.yaml` para editarlo.

```bash
cp rs.yaml ds.yaml
vim ds.yaml
```

Cambios principales:

* Cambiar `kind: ReplicaSet` por `kind: DaemonSet`
* Eliminar la línea `replicas: 2` porque DaemonSet no usa replicas, sino que asegura que haya un pod por nodo.
* Cambiar las etiquetas y selectores de `system: ReplicaOne` a `system: DaemonSetOne` (o algo que diferencie este DaemonSet).

Ejemplo del cambio:

```yaml
kind: DaemonSet
metadata:
  name: ds-one
spec:
  selector:
    matchLabels:
      system: DaemonSetOne
  template:
    metadata:
      labels:
        system: DaemonSetOne
    spec:
      containers:
      - name: nginx
        image: nginx:1.22.1
        ports:
        - containerPort: 80
```

---

### Paso 2: Crear y verificar el DaemonSet

Crear:

```bash
kubectl create -f ds.yaml
```

Verificar:

```bash
kubectl get ds
```

Salida esperada:

| NAME   | DESIRED | CURRENT | READY | UP-TO-DATE | AVAILABLE | NODE-SELECTOR | AGE |
| ------ | ------- | ------- | ----- | ---------- | --------- | ------------- | --- |
| ds-one | 2       | 2       | 2     | 2          | 2         | <none>        | 1m  |

* **Por qué:**
  `DESIRED` indica la cantidad de nodos donde debe correr un pod (en este caso 2 nodos en el clúster).
  `CURRENT`, `READY`, `UP-TO-DATE` y `AVAILABLE` indican el estado de los pods creados por el DaemonSet.

Luego:

```bash
kubectl get pods
```

Verás pods con nombres como `ds-one-b1dcv`, uno por nodo.

---

### Paso 3: Verificar la imagen del pod

```bash
kubectl describe pod ds-one-b1dcv | grep Image:
```

Salida esperada:

```
Image: nginx:1.22.1
```

* **Por qué:**
  Confirmamos que los pods del DaemonSet están usando la imagen nginx que especificamos.

---

## Rollout y Rollback usando DaemonSet (Detalle ampliado)

---

### 1. Ver la estrategia de actualización actual del DaemonSet

```bash
kubectl get ds ds-one -o yaml | grep -A 4 updateStrategy:
```

**¿Qué hace?**
Este comando muestra la configuración YAML del DaemonSet `ds-one`, filtrando la sección `updateStrategy`, que indica cómo se actualizan los pods gestionados.

**Salida esperada:**

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

**Significado:**

* `type: RollingUpdate` indica que la actualización se hará de forma progresiva, reemplazando pods poco a poco.
* `maxUnavailable: 1` limita que solo un pod pueda estar indisponible durante la actualización.

---

### 2. Cambiar la estrategia a `OnDelete`

```bash
kubectl edit ds ds-one
```

Busca la sección `updateStrategy` y cambia a:

```yaml
updateStrategy:
  type: OnDelete
```

**¿Qué implica?**

* Cuando uses `OnDelete`, **los pods no se actualizarán automáticamente** al cambiar la imagen.
* Los pods solo se recrearán con la nueva imagen **cuando los elimines manualmente**. Esto da más control sobre el momento exacto de la actualización.

---

### 3. Actualizar la imagen del DaemonSet a una nueva versión

```bash
kubectl set image ds ds-one nginx=nginx:1.26-alpine --record
```

**¿Qué hace?**
Actualiza la imagen del contenedor `nginx` dentro del DaemonSet `ds-one` a la versión `1.26-alpine`.

**Nota:**

* El flag `--record` guarda un historial del comando para usarlo en `rollout history`.
* Aunque actualices la imagen aquí, con `OnDelete` los pods existentes no cambiarán hasta que los borres.

---

### 4. Verificar que la imagen del pod actual no cambió (actualización OnDelete)

```bash
kubectl describe pod ds-one-b1dcv | grep Image:
```

**Salida esperada:**

```
Image: nginx:1.22.1
```

**¿Por qué?**
El pod todavía está corriendo con la imagen anterior porque el DaemonSet no actualiza automáticamente los pods con `OnDelete`.

---

### 5. Eliminar manualmente un pod para crear uno nuevo con la imagen actualizada

```bash
kubectl delete pod ds-one-b1dcv
kubectl get pod
```

Espera que Kubernetes cree un nuevo pod para reemplazar el eliminado.

```bash
kubectl describe pod ds-one-xxxxx | grep Image:
```

**Salida esperada:**

```
Image: nginx:1.26-alpine
```

**¿Qué ocurrió?**
Al eliminar el pod manualmente, el DaemonSet crea un nuevo pod con la imagen actualizada `1.26-alpine`.

---

### 6. Ver la imagen de un pod que no ha sido eliminado

```bash
kubectl describe pod ds-one-z31r4 | grep Image:
```

**Salida esperada:**

```
Image: nginx:1.22.1
```

**Significado:**
Los pods no eliminados siguen corriendo la versión antigua.

---

### 7. Consultar el historial de revisiones del DaemonSet

```bash
kubectl rollout history ds ds-one
```

**Salida típica:**

```
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image ds ds-one nginx=nginx:1.26-alpine --record=true
```

**¿Qué muestra?**
El historial de cambios que se han aplicado al DaemonSet, incluyendo la actualización de imagen.

---

### 8. Ver detalles específicos de cada revisión

```bash
kubectl rollout history ds ds-one --revision=1
kubectl rollout history ds ds-one --revision=2
```

**Salida relevante:**

```yaml
Image: nginx:1.22.1
```

y

```yaml
Image: nginx:1.26-alpine
```

**¿Qué indica esto?**
Cada revisión contiene la plantilla del pod (pod template) usada, mostrando la imagen que corre en esa versión.

---

### 9. Hacer rollback a una versión anterior

```bash
kubectl rollout undo ds ds-one --to-revision=1
```

**¿Qué hace?**
Vuelve a la configuración previa del DaemonSet (imagen antigua `1.22.1`).

**Importante:**

* Con `OnDelete` los pods existentes **no cambian automáticamente**; seguirán con la versión 1.26-alpine hasta que se borren manualmente.

---

### 10. Eliminar un pod para que se cree con la versión antigua tras rollback

```bash
kubectl delete pod ds-one-xxxxx
kubectl describe pod ds-one-yyyyy | grep Image:
```

**Salida esperada:**

```
Image: nginx:1.22.1
```

---

### 11. Ver la configuración actual del DaemonSet para confirmar `updateStrategy`

```bash
kubectl get ds ds-one -o yaml | grep -A 5 updateStrategy:
```

**Salida esperada:**

```yaml
updateStrategy:
  type: OnDelete
```

---

### 12. Crear un nuevo DaemonSet con estrategia `RollingUpdate`

```bash
kubectl get ds ds-one -o yaml > ds2.yaml
vim ds2.yaml
```

* Cambia el nombre a `ds-two` para no crear conflictos.
* Cambia `updateStrategy` a:

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```

---

### 13. Crear el nuevo DaemonSet

```bash
kubectl create -f ds2.yaml
```

---

### 14. Verificar los pods y versión de nginx en el nuevo DaemonSet

```bash
kubectl get pods
kubectl describe pod ds-two-xxxxx | grep Image:
```

**Ejemplo:**

```
Image: nginx:1.15.1
```

---

### 15. Actualizar imagen del DaemonSet `ds-two`

```bash
kubectl edit ds ds-two
```

Edita la imagen a:

```yaml
image: nginx:1.26-alpine
```

---

### 16. Verificar edad y estado del DaemonSet y sus pods

```bash
kubectl get ds ds-two
kubectl get pods
```

* La edad del DaemonSet será mayor que la edad de los pods nuevos.
* Los pods más jóvenes estarán corriendo la nueva imagen.

---

### 17. Ver el estado del rollout

```bash
kubectl rollout status ds ds-two
```

**Salida esperada:**

```
daemonset "ds-two" successfully rolled out
```

---

### 18. Limpieza: eliminar ambos DaemonSets

```bash
kubectl delete ds ds-one ds-two
```




