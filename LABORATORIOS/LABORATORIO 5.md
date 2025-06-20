# Usar el Proxy de Kubernetes

---

### 1. Revisar las opciones del proxy

Ejecuta el comando para ver la ayuda de `kubectl proxy`:

```bash
kubectl proxy -h
```

Te mostrará que el proxy crea un servidor local que hace de gateway hacia el API server de Kubernetes. Por defecto, escucha en `127.0.0.1:8001` y redirige las solicitudes hacia el API server, permitiéndote interactuar con la API como si estuvieras local.

---

### 2. Iniciar el proxy en segundo plano

Arranca el proxy con un prefijo para la API (usualmente `/` por defecto), y mándalo a background con `&`:

```bash
kubectl proxy --api-prefix=/ &
```

* Te devolverá un número de proceso (PID), por ejemplo `22500`.
* Anota ese PID para poder detener el proxy luego.

---

### 3. Probar acceso a la API vía proxy con `curl`

Ahora usa `curl` para hacer una petición a la API server a través del proxy local:

```bash
curl http://127.0.0.1:8001/api
```

* La respuesta debería ser similar a cuando accedes directamente al API server (sin proxy), pero el proxy hace de intermediario.

---

### 4. Obtener la lista de namespaces a través del proxy

Haz la llamada para obtener namespaces:

```bash
curl http://127.0.0.1:8001/api/v1/namespaces
```

* Esto debería funcionar correctamente a diferencia de una llamada directa, porque el proxy maneja autenticación y autorización por ti.
* La salida será un JSON con la lista de namespaces:

```json
{
  "kind": "NamespaceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces",
    "resourceVersion": "86902"
  },
  ...
}
```

---

### 5. Detener el proxy

Cuando termines, mata el proceso usando el PID que anotaste:

```bash
kill 22500
```

Si tu PID es diferente, reemplázalo por el correcto.

---

# Resumen rápido

* `kubectl proxy` crea un servidor local que hace de intermediario con el API server.
* Esto te permite hacer llamadas `curl` a la API sin preocuparte por autenticación y autorización, porque el proxy las maneja.
* Útil para probar acceso o depurar problemas de conexión.
* Recuerda detener el proxy cuando no lo uses para liberar recursos.


## Trabajando con Jobs y CronJobs — Explicación detallada

---

### Parte 1: Crear un Job simple

1. **Copiar el archivo `job.yaml`:**

```bash
cp /home/student/LFS258/SOLUTIONS/s_06/job.yaml .
```

* **Por qué:** Necesitamos un archivo con la definición del Job. El archivo `job.yaml` contiene la configuración necesaria para crear un Job en Kubernetes que ejecuta un contenedor simple.

---

2. **Definición básica del Job:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  template:
    spec:
      containers:
      - name: resting
        image: busybox
        command: ["/bin/sleep"]
        args: ["3"]
      restartPolicy: Never
```

* **Por qué:**

  * `apiVersion: batch/v1` indica que trabajamos con la API batch, versión 1, que maneja Jobs.
  * `kind: Job` identifica que este objeto es un Job.
  * `metadata.name` define un nombre único para identificar el Job.
  * `spec.template` es la plantilla del pod que Kubernetes usará para ejecutar el trabajo.
  * `containers` especifica el contenedor dentro del pod; aquí usamos `busybox` para correr el comando `sleep 3`.
  * `restartPolicy: Never` significa que si el pod falla, no se reiniciará automáticamente (importante para Jobs que deben ejecutarse una vez).

---

3. **Crear y verificar el Job:**

```bash
kubectl create -f job.yaml
kubectl get job sleepy
kubectl describe job sleepy
```

* **Por qué:**

  * `kubectl create` aplica la definición y crea el Job en el cluster.
  * `kubectl get job` muestra el estado general del Job (cuántas tareas se han completado).
  * `kubectl describe job` da detalles internos, como cuándo empezó y terminó, y el estado de los pods.

---

4. **Eliminar el Job tras su finalización:**

```bash
kubectl delete job sleepy
```

* **Por qué:** Los Jobs no se eliminan automáticamente tras completarse; es buena práctica limpiar para evitar saturar el cluster con objetos terminados.

---

### Parte 2: Configurar parámetros de Job

5. **Agregar el parámetro `completions: 5`:**

```yaml
spec:
  completions: 5
  template:
    ...
```

* **Por qué:** `completions` define cuántas veces queremos que el Job se ejecute exitosamente. Aquí le decimos que ejecute la tarea 5 veces (puede ser útil para procesamiento en batch, tareas repetitivas, etc).

---

6. **Crear el Job y observar múltiples ejecuciones:**

```bash
kubectl create -f job.yaml
kubectl get jobs
kubectl get pods
```

* **Por qué:** Queremos ver que Kubernetes crea múltiples pods para cumplir con las 5 ejecuciones. Cada pod ejecuta el comando sleep 3s y luego termina.

---

7. **Agregar `parallelism: 2`:**

```yaml
spec:
  completions: 5
  parallelism: 2
  template:
    ...
```

* **Por qué:** `parallelism` indica cuántos pods se ejecutan simultáneamente. Si no lo ponemos, Kubernetes ejecuta un pod a la vez. Al poner 2, dos pods se ejecutan al mismo tiempo, acelerando la ejecución total.

---

8. **Recrear el Job y verificar pods paralelos:**

* Veremos que dos pods corren al mismo tiempo y cuando uno termina, otro comienza hasta completar las 5 ejecuciones.

---

### Parte 3: Añadir límite de tiempo con `activeDeadlineSeconds`

9. **Agregar `activeDeadlineSeconds: 15` y modificar sleep a 5 segundos:**

```yaml
spec:
  completions: 5
  parallelism: 3
  activeDeadlineSeconds: 15
  template:
    spec:
      containers:
      - name: resting
        image: busybox
        command: ["/bin/sleep"]
        args: ["5"]
      restartPolicy: Never
```

* **Por qué:**

  * `activeDeadlineSeconds` limita el tiempo total que el Job puede estar activo.
  * Si el Job excede este límite, Kubernetes termina los pods y marca el Job como fallido.
  * Modificamos el sleep a 5 segundos para que se pueda observar el límite, ya que con 3 segundos no sería tan claro.

---

10. **Crear Job y observar que se detiene antes de completar:**

* El Job intentará correr 5 veces, pero al superar los 15 segundos totales de actividad se detendrá, dejando completados menos de 5 pods.

---

11. **Verificar el estado y razón de fallo:**

```bash
kubectl get job sleepy -o yaml
```

* En `status.conditions` verás `message: Job was active longer than specified deadline` y `reason: DeadlineExceeded`.
* **Por qué:** Esto indica que Kubernetes ha terminado el Job por superar el tiempo límite.

---

### Parte 4: Crear un CronJob

1. **Copiar y editar `cronjob.yaml`:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: sleepy
spec:
  schedule: "*/2 * * * *"  # Cada 2 minutos
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: resting
            image: busybox
            command: ["/bin/sleep"]
            args: ["5"]
          restartPolicy: Never
```

* **Por qué:**

  * `kind: CronJob` crea un recurso que ejecuta Jobs periódicamente.
  * `schedule` usa la sintaxis cron estándar para indicar que se ejecuta cada 2 minutos.
  * `jobTemplate` es la plantilla para el Job que se ejecutará en cada periodo.

---

2. **Crear el CronJob:**

```bash
kubectl create -f cronjob.yaml
```

* Esto hará que Kubernetes ejecute un Job nuevo cada 2 minutos automáticamente.

---

3. **Verificar CronJobs y Jobs:**

```bash
kubectl get cronjobs
kubectl get jobs
```

* **Por qué:** El CronJob crea un Job nuevo automáticamente; aquí puedes ver esos Jobs generados.

---

### Parte 5: Límite de tiempo en CronJob

4. **Agregar límite de tiempo y cambiar duración de sleep:**

```yaml
spec:
  jobTemplate:
    spec:
      activeDeadlineSeconds: 10
      template:
        spec:
          containers:
          - name: resting
            image: busybox
            command: ["/bin/sleep"]
            args: ["30"]
          restartPolicy: Never
```

* **Por qué:**

  * Queremos que el Job termine si dura más de 10 segundos, aunque el comando sleep intente durar 30 segundos.
  * Esto ayuda a evitar Jobs que se cuelguen o consuman recursos demasiado tiempo.

---

5. **Eliminar y recrear CronJob para aplicar cambios.**

* Los cambios en el cronjob.yaml solo se aplican si recreas el recurso.

---

### Parte 6: Limpieza final

```bash
kubectl delete job sleepy
kubectl delete cronjob sleepy
```

* **Por qué:** Mantener el cluster limpio evita acumulación innecesaria de recursos y posibles conflictos en el futuro.



