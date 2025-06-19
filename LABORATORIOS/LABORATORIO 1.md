## 1. Creación de máquinas virtuales en Google Cloud Platform (GCP)

### 1.1. Acceder a Google Cloud Console

* Entra en [https://console.cloud.google.com/](https://console.cloud.google.com/)
* Selecciona o crea un proyecto nuevo (por ejemplo, `k8s-lab`).

---

### 1.2. Crear las instancias de máquina virtual (VM)

Para este laboratorio necesitas al menos **2 máquinas**:

* 1 Nodo de Control (control plane)
* 1 Nodo Trabajador (worker)

---

### 1.3. Configuración de cada VM

* Ve a **Compute Engine > Instancias de VM > Crear instancia**.

* Pon nombre para la VM: por ejemplo, `k8s-control` y `k8s-worker1`.

* Región: escoge la que esté más cerca o donde quieras.

* Tipo de máquina: selecciona al menos **2 vCPU y 7.5 GB RAM** (por ejemplo, e2-standard-2).

  > Esto asegura que Kubernetes pueda correr sin problemas. Más CPU y RAM ayudan a evitar errores por recursos limitados.

* En **Disco de arranque** selecciona:

  * Imagen: Ubuntu 24.04 LTS
  * Tamaño del disco: mínimo 20 GB (para tener espacio suficiente)

* En **Firewall**:

  * Marca las casillas para permitir tráfico HTTP y HTTPS (esto puede simplificar pruebas de red).
  * Más adelante abriremos el firewall para puertos Kubernetes.

* Opcional: En la pestaña "Administración, seguridad, disco, red, sola ubicación":

  * Bajo **Redes** asegúrate que la VM está en la misma red o VPC que otras máquinas.

* **Crea** la instancia.

---

### 1.4. Crear una clave SSH para conectarse

* Si tienes Windows y vas a usar **PuTTY**, primero crea un par de claves SSH:

1. Descarga **PuTTYgen**: [https://www.puttygen.com/](https://www.puttygen.com/)
2. Abre PuTTYgen y haz clic en **Generate** para crear un par de claves.
3. Guarda la clave privada (.ppk) en tu PC.
4. Copia la clave pública (el texto que sale en PuTTYgen, que empieza por `ssh-rsa ...`).

---

### 1.5. Agregar clave pública SSH a las VM de GCP

* En Google Cloud Console, ve a **Compute Engine > Instancias de VM**.
* Haz clic en la VM `k8s-control`.
* Ve a la pestaña **Editar**.
* En la sección **Claves SSH**, pega tu clave pública generada con PuTTYgen.
* Guarda los cambios.
* Repite para la VM `k8s-worker1`.

---

## 2. Configuración de Firewall en GCP para Kubernetes

Para que los nodos puedan comunicarse y Kubernetes funcione:

* Ve a **VPC Network > Reglas de firewall > Crear regla de firewall**.
* Pon un nombre, por ejemplo `allow-k8s`.
* Red: selecciona la red donde están tus VM.
* Objetivos: "Todas las instancias en la red" (o solo las instancias específicas si quieres seguridad).
* Filtros de fuente: deja en 0.0.0.0/0 para abrir desde cualquier IP (no recomendado para producción, solo para laboratorio).
* Protocolos y puertos:

  * Marca TCP y escribe: `22,6443,2379-2380,10250,10251,10252,10255`
  * Esto abre los puertos básicos que Kubernetes usa para comunicación entre nodos.
* Crea la regla.

---

## 3. Conexión a las máquinas usando PuTTY (Windows)

### 3.1. Configurar PuTTY para conectarse

* Abre PuTTY.
* En "Host Name (or IP address)", escribe la IP pública de la VM `k8s-control` (la puedes ver en Google Cloud Console).
* En **Connection > SSH > Auth**, busca la clave privada `.ppk` que guardaste con PuTTYgen.
* Guarda la sesión con un nombre (ejemplo: `k8s-control`).
* Haz clic en **Open**.

> La primera vez te preguntará si confías en la clave: acepta.

### 3.2. Usuario para login

* El usuario suele ser `ubuntu` en las imágenes oficiales de Ubuntu.
* Si pediste un nombre distinto o configuraste otro usuario, úsalo.
* Ejemplo de login:

```bash
login as: ubuntu
```

---

## 4. Instalación Kubernetes con kubeadm

---

### 4.1. Actualización del sistema operativo

```bash
sudo -i
apt update && apt upgrade -y
```

**¿Por qué?**

* Para asegurarnos de que el sistema operativo tiene los últimos parches de seguridad y actualizaciones, lo que evita problemas por bugs o vulnerabilidades.

---

### 4.2. Instalación de dependencias básicas

```bash
apt install -y apt-transport-https ca-certificates software-properties-common socat curl
```

**¿Por qué?**

* Estas herramientas son necesarias para poder instalar software desde repositorios HTTPS, manejar certificados y descargar archivos (curl).

---

### 4.3. Desactivar swap

```bash
swapoff -a
```

**¿Por qué?**

* Kubernetes no funciona bien con swap activado porque puede confundir el manejo de memoria y provocar que el kubelet marque errores o el scheduler no funcione correctamente.

---

### 4.4. Activar módulos del kernel y parámetros de red

```bash
modprobe overlay
modprobe br_netfilter
```

Configura sysctl:

```bash
cat <<EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
EOF

sysctl --system
```

**¿Por qué?**

* Kubernetes necesita que las reglas de iptables puedan pasar el tráfico entre interfaces de red en la capa de puente.
* También habilitamos el reenvío de paquetes IP para la comunicación entre pods.

---

### 4.5. Instalación y configuración de `containerd` (runtime de contenedores)

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list
apt update
apt install -y containerd.io
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

**¿Por qué?**

* Kubernetes necesita un runtime de contenedores para ejecutar los pods.
* `containerd` es un runtime popular, liviano y recomendado por Kubernetes y GCP.
* Configuramos `systemd` para que el cgroup manager funcione correctamente con Kubernetes.

---

### 4.6. Instalación de Kubernetes (kubeadm, kubelet, kubectl)

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
apt update
apt install -y kubeadm=1.32.1-1.1 kubelet=1.32.1-1.1 kubectl=1.32.1-1.1
apt-mark hold kubeadm kubelet kubectl
```

**¿Por qué?**

* Instalamos versiones específicas para asegurar compatibilidad.
* `kubeadm` permite configurar el clúster.
* `kubelet` es el agente que corre en cada nodo.
* `kubectl` es la herramienta cliente para interactuar con el clúster.
* Bloqueamos las versiones para evitar actualizaciones automáticas que rompan el clúster.

---

### 4.7. Configurar hostname e IP en `/etc/hosts`

```bash
hostname -i
vim /etc/hosts
```

Agrega líneas, por ejemplo:

```
10.128.0.3 k8scp cp
```

**¿Por qué?**

* Para que los nodos puedan resolverse por nombre dentro del clúster, facilitando la comunicación y configuración.

---

### 4.8. Inicializar el nodo de control con kubeadm

```bash
kubeadm init --config=kubeadm-config.yaml --upload-certs --node-name=cp
```

**¿Por qué?**

* `kubeadm init` configura el nodo de control.
* El archivo `kubeadm-config.yaml` contiene parámetros de configuración como versión de Kubernetes, red de pods, etc.
* `--upload-certs` permite compartir certificados con nodos trabajadores que se unan.
* `--node-name` asigna un nombre al nodo.

---

### 4.9. Configurar kubectl para el usuario normal

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**¿Por qué?**

* Esto configura la herramienta `kubectl` para que puedas administrar el clúster sin ser root.

---

### 4.10. Instalar el plugin de red Cilium

```bash
kubectl apply -f cilium-cni.yaml
```

**¿Por qué?**

* Kubernetes requiere un plugin de red para que los pods puedan comunicarse entre nodos.
* Cilium es un plugin avanzado con soporte para políticas de red basadas en eBPF.

---

### 4.11. Configurar autocompletado de kubectl

```bash
sudo apt install bash-completion -y
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

**¿Por qué?**

* Facilita la escritura de comandos con sugerencias automáticas.

---


Con esto tienes:

* Las máquinas VM en GCP configuradas con Ubuntu 24.04.
* Conexión segura desde Windows con PuTTY.
* Firewall preparado para Kubernetes.
* Instalación completa de Kubernetes usando kubeadm.
* Nodo control listo para unir trabajadores y desplegar aplicaciones.

---

## 5. Agregar un Nodo Worker al Clúster de Kubernetes

En este apartado conectaremos una segunda instancia (nodo worker), instalaremos el software necesario y lo uniremos al clúster Kubernetes creado en el nodo `control plane`.

---

### 5.1. Conectar al Nodo Worker

Conéctate a tu segundo nodo (por ejemplo, `k8s-worker1`) usando **PuTTY** (Windows) o SSH (Linux/Mac), tal como hiciste con el nodo `k8s-control`.


```bash
student@worker:~$ sudo -i
````

---

### 5.2. Actualizar el sistema

```bash
apt-get update && apt-get upgrade -y
```

> Permite aplicar actualizaciones del sistema. Si te pregunta, permite reiniciar servicios y conservar la configuración local.

---

### 5.3. Instalar containerd y dependencias

```bash
apt install apt-transport-https software-properties-common ca-certificates tree socat -y
swapoff -a
modprobe overlay
modprobe br_netfilter
```

---

### 5.4. Configurar parámetros de red para Kubernetes

```bash
cat <<EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

---

### 5.5. Instalar containerd (runtime de contenedores)

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
| tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install containerd.io -y

containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
```

---

### 5.6. Instalar Kubernetes (kubeadm, kubelet, kubectl)

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /" \
| tee /etc/apt/sources.list.d/kubernetes.list

apt-get update

apt-get install -y kubeadm=1.32.1-1.1 kubelet=1.32.1-1.1 kubectl=1.32.1-1.1
apt-mark hold kubeadm kubelet kubectl
```

> 💡 Es importante usar las **mismas versiones** que en el nodo de control.

---

### 5.7. Configurar DNS local para el nodo de control

Edita el archivo `/etc/hosts`:

```bash
vim /etc/hosts
```

Agrega estas líneas con la IP privada del nodo `cp`:

```
10.128.0.3 k8scp
10.128.0.3 cp
```

> Esto permite que el nodo worker resuelva el nombre del nodo de control.

---

### 5.8. Obtener el comando de unión desde el nodo `cp`

En el nodo de control, ejecuta:

```bash
sudo kubeadm token create --print-join-command
```

Ejemplo de salida:

```bash
kubeadm join k8scp:6443 --token abcdef.0123456789abcdef \
--discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

---

### 5.9. Ejecutar el comando para unir el nodo al clúster

En el nodo worker, ejecuta el comando anterior (con tu propio token y hash):

```bash
kubeadm join k8scp:6443 --token abcdef.0123456789abcdef \
--discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX \
--node-name=worker
```

> 🔒 Este comando configura al nodo como parte del clúster y arranca el `kubelet`.

---

### 5.10. Verificar que el nodo se ha unido correctamente

En el nodo de control, ejecuta:

```bash
kubectl get nodes
```

> El nuevo nodo `worker` debe aparecer con estado `Ready`.

---

### 5.11: Probar `kubectl` en el nodo worker 

```bash
kubectl get nodes
```

> Esto fallará porque no tienes el archivo `.kube/config` en este nodo. Es normal.

Error esperado:

```bash
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

¡Has unido exitosamente un nodo worker al clúster Kubernetes usando `kubeadm`!

---

## 6. Finalizar la configuración del clúster Kubernetes

En este apartado verificaremos el estado del clúster, eliminaremos las restricciones del nodo control plane para permitir pods no infraestructurales, y aseguraremos que los componentes esenciales como DNS y Cilium estén funcionando correctamente.

---

### 6.1. Ver nodos disponibles y su estado

Ejecuta en el nodo control plane:

```bash
kubectl get nodes
````

Ejemplo de salida:

```
NAME     STATUS   ROLES                  AGE     VERSION
cp       Ready    control-plane          28m     v1.32.1
worker   Ready    <none>                 50s     v1.32.1
```

> Nota: El estado puede tardar uno o dos minutos en pasar de `NotReady` a `Ready`.

---

### 6.2. Ver detalles de un nodo específico

Para ver detalles y estado del nodo `cp`:

```bash
kubectl describe node cp
```

Revisa la sección `Taints`. El nodo `cp` tiene por defecto un taint que impide correr pods que no sean de infraestructura:

```
Taints:
  node-role.kubernetes.io/control-plane:NoSchedule
```

Esto es por seguridad y para evitar saturar el nodo con pods que no sean críticos.

---

## Paso 3: Remover taints para permitir pods normales en el nodo control plane

Para permitir que el nodo `cp` ejecute pods normales (solo en ambientes de entrenamiento):

1. Verifica los taints:

   ```bash
   kubectl describe node cp | grep -i taint
   ```

2. Elimina el taint con:

   ```bash
   kubectl taint nodes cp node-role.kubernetes.io/control-plane:NoSchedule-
   ```

3. Verifica que ya no existan taints:

   ```bash
   kubectl describe node cp | grep -i taint
   ```

> Nota: El signo `-` al final del taint indica que se remueve.

---

### 6.4. Verificar que los pods DNS y Cilium están corriendo

Ejecuta:

```bash
kubectl get pods --all-namespaces
```

Ejemplo de salida esperada:

```
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   cilium-operator-788c7d7585-tnsph   1/1     Running   0          95m
kube-system   cilium-swjsj                       1/1     Running   0          95m
kube-system   coredns-5d78c9869d-dwds8           1/1     Running   0          100m
kube-system   coredns-5d78c9869d-t24p5           1/1     Running   0          100m
```

> Todos deben mostrar `STATUS: Running`.

---

### 6.5. Si los pods `coredns` están en estado `ContainerCreating`

1. Elimina los pods `coredns` para que Kubernetes los recree automáticamente:

   ```bash
   kubectl -n kube-system delete pod <nombre-pod-coredns-1> <nombre-pod-coredns-2>
   ```

2. Verifica nuevamente que estén en `Running`:

   ```bash
   kubectl get pods --all-namespaces
   ```

---

### 6.6. Revisar interfaces de red creadas por Cilium

En el nodo `cp` ejecuta:

```bash
ip a
```

Verás interfaces con nombres como `cilium_net` y `cilium_vxlan` activas, que Cilium utiliza para la comunicación en el clúster.

---

### 6.7: Configurar `crictl` para usar containerd correctamente

1. Configura el endpoint de runtime e imagen para `crictl` en ambos nodos (`cp` y `worker`):

```bash
sudo crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock \
--set image-endpoint=unix:///run/containerd/containerd.sock
```

2. Puedes verificar la configuración con:

```bash
cat /etc/crictl.yaml
```

Salida esperada:

```yaml
runtime-endpoint: "unix:///run/containerd/containerd.sock"
image-endpoint: "unix:///run/containerd/containerd.sock"
timeout: 0
debug: false
pull-image-on-create: false
disable-pull-on-run: false
```

---

Con estos pasos, el clúster Kubernetes está finalizado, los nodos preparados y los servicios básicos como DNS y red funcionando correctamente.

---

Claro, aquí tienes una explicación detallada del propósito de cada paso del instructivo para lanzar una aplicación simple con **nginx** en Kubernetes. El formato es Markdown, por lo que puedes copiarlo directamente en tu documentación.

---

## 7. Lanzar una aplicación simple (nginx) – Explicación paso a paso

---

### 7.1. Crear un deployment con nginx y verificar que esté corriendo

```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments
```

**Propósito**: Se crea un *Deployment* que administra un *Pod* corriendo la imagen `nginx`. Esto asegura alta disponibilidad y manejo automático del ciclo de vida del contenedor.

---

### 7.2. Ver detalles del deployment

```bash
kubectl describe deployment nginx
```

**Propósito**: Proporciona detalles extensos sobre el Deployment, como eventos recientes, estrategia de actualización, etiquetas, y definición de los Pods que maneja.

---

### 7.3. Ver eventos del cluster para ver el despliegue del pod

```bash
kubectl get events
```

**Propósito**: Permite revisar eventos del clúster relacionados con la creación y el estado de los recursos, útil para diagnosticar errores o confirmar que todo va bien.

---

### 7.4. Ver el despliegue en formato YAML (útil para ver configuración detallada)

```bash
kubectl get deployment nginx -o yaml
```

**Propósito**: Obtener la configuración completa del Deployment en formato declarativo. Este formato es útil para edición, auditoría o recreación del recurso.

---

### 7.5. Guardar el YAML en un archivo, luego editarlo para eliminar las líneas automáticas

```bash
kubectl get deployment nginx -o yaml > first.yaml
vim first.yaml
```

**Editar**:
Eliminar metadatos generados automáticamente:

* `creationTimestamp`
* `resourceVersion`
* `uid`
* Todo desde `status:` en adelante

**Propósito**: Al limpiar el YAML, este se puede reutilizar para crear nuevos recursos sin conflictos con información generada por el sistema.

---

### 7.6. Borrar el deployment actual

```bash
kubectl delete deployment nginx
```

**Propósito**: Elimina el Deployment original para probar la creación a partir del archivo limpio (`first.yaml`).

---

### 7.7. Crear el deployment usando el archivo editado

```bash
kubectl create -f first.yaml
```

**Propósito**: Recrea el Deployment a partir del archivo YAML limpio, asegurando control completo sobre su configuración.

---

### 7.8. Comparar el YAML nuevo con el anterior

```bash
kubectl get deployment nginx -o yaml > second.yaml
diff first.yaml second.yaml
```

**Propósito**: Verifica qué cambios ocurrieron automáticamente al aplicar el recurso (nuevos UIDs, timestamps, etc.).

---

### 7.9. Generar YAML sin crear objeto usando --dry-run

```bash
kubectl create deployment two --image=nginx --dry-run=client -o yaml
```

**Propósito**: Permite generar un manifiesto YAML para revisión o versionado sin aplicarlo directamente en el clúster.

---

### 7.10. Ver el deployment nginx en YAML

```bash
kubectl get deployments nginx -o yaml
```

**Propósito**: Similar al paso 4, permite revisar o respaldar la configuración actual.

---

### 7.11. Ver el deployment nginx en JSON

```bash
kubectl get deployment nginx -o json
```

**Propósito**: Alternativa al YAML, útil cuando se integran herramientas de automatización que prefieren formato JSON.

---

### 7.12. Consultar ayuda para exponer servicios

```bash
kubectl expose -h
```

**Propósito**: Muestra todas las opciones disponibles del comando `kubectl expose`, que convierte un recurso en un servicio accesible.

---

### 7.13. Intentar exponer el deployment sin especificar puerto

```bash
kubectl expose deployment/nginx
```

**Resultado esperado**: Error indicando que no se puede encontrar un puerto.

**Propósito**: Ilustra que para exponer un Deployment como servicio, se debe especificar un puerto explícitamente o definirlo previamente en el contenedor.

---

### 7.14. Editar el archivo YAML para añadir el puerto 80

```yaml
ports:
- containerPort: 80
  protocol: TCP
```

**Propósito**: Se añade la especificación de puerto necesario para que `kubectl expose` funcione correctamente.

---

### 7.15. Reemplazar el deployment con el archivo modificado (forzado)

```bash
kubectl replace -f first.yaml --force
```

**Propósito**: Elimina y vuelve a crear el recurso, útil cuando se necesita aplicar cambios estructurales profundos.

---

### 7.16. Ver pods y deployment

```bash
kubectl get deploy,pod
```

**Propósito**: Confirma que el nuevo Pod se está ejecutando y está vinculado al Deployment correctamente.

---

### 7.17. Ahora exponer el deployment

```bash
kubectl expose deployment/nginx
```

**Propósito**: Crea un *Service* tipo `ClusterIP` que permite acceso interno al Deployment por medio de un nombre DNS.

---

### 7.18. Ver configuración del servicio y endpoints

```bash
kubectl get svc nginx
kubectl get ep nginx
```

**Propósito**: Verifica que el *Service* esté creado correctamente y que esté enlazado a uno o más Pods disponibles.

---

### 7.19. Probar acceso al servicio usando curl

```bash
curl 10.100.61.122:80
curl 192.168.1.5:80
```

**Propósito**: Comprueba que el servicio y el pod están respondiendo correctamente. Debe devolver la página de bienvenida de nginx.

---

### 7.20. Escalar el deployment a 3 réplicas

```bash
kubectl scale deployment nginx --replicas=3
kubectl get deployment nginx
```

**Propósito**: Demuestra cómo escalar una aplicación para soportar mayor carga o disponibilidad.

---

### 7.21. Ver endpoints (deberían mostrar 3 IPs)

```bash
kubectl get ep nginx
```

**Propósito**: Confirma que los nuevos Pods se han agregado al Service y están accesibles.

---

### 7.22. Eliminar el pod más antiguo

```bash
kubectl get pods -o wide
kubectl delete pod <nombre-del-pod-antiguo>
```

**Propósito**: Simula una falla o reinicio manual de un Pod, útil para observar cómo el Deployment mantiene el estado deseado.

---

### 7.23. Esperar a que se cree un nuevo pod

```bash
kubectl get pods
```

**Propósito**: Verifica que Kubernetes reemplaza automáticamente Pods eliminados para mantener la cantidad especificada.

---

### 7.24. Ver endpoints otra vez

```bash
kubectl get ep nginx
```

**Propósito**: Confirma que el nuevo Pod reemplazó al anterior y que el Service sigue funcionando correctamente.

---

## 8. Acceder desde el exterior Cluster

Acceso a un Servicio desde fuera del clúster usando variables de entorno y LoadBalancer

---

### 8.1. Listar los pods actuales

```bash
kubectl get pods
```

**¿Por qué?**
Para conocer qué pods están corriendo actualmente, su estado y edad. Necesitamos saber qué pods están disponibles para interactuar con ellos o para probar el acceso.

---

### 8.2. Ejecutar un comando dentro de un pod para ver variables de entorno relacionadas con Kubernetes

```bash
kubectl exec nginx-1423793266-13p69 -- printenv | grep KUBERNETES
```

**¿Por qué?**
Las variables de entorno como `KUBERNETES_SERVICE_HOST` y `KUBERNETES_SERVICE_PORT` son inyectadas automáticamente dentro de los pods para que las aplicaciones puedan descubrir y comunicarse con el API Server de Kubernetes. Verlas confirma que el pod está configurado correctamente para comunicarse con el clúster.

---

### 8.3. Ver el servicio (Service) actual de nginx

```bash
kubectl get svc
```

**¿Por qué?**
Un Service en Kubernetes es el objeto que expone los pods para que puedan ser accedidos. Aquí vemos qué servicios están disponibles y cómo están configurados (tipo, IP, puertos).

---

### 8.4. Eliminar el servicio nginx actual

```bash
kubectl delete svc nginx
```

**¿Por qué?**
Vamos a eliminar el servicio actual para recrearlo con una configuración diferente (tipo `LoadBalancer`) que nos permitirá acceder a la aplicación desde fuera del clúster.

---

### 8.5. Crear el servicio nuevamente, pero esta vez con tipo LoadBalancer

```bash
kubectl expose deployment nginx --type=LoadBalancer
```

**¿Por qué?**
El tipo `LoadBalancer` crea un servicio que intenta asignar una IP externa accesible fuera del clúster, ideal para exponer aplicaciones al mundo exterior (por ejemplo, Internet). Si estás en una nube pública, el proveedor asignará automáticamente la IP pública.

---

### 8.6. Verificar el servicio para ver la IP externa y puertos

```bash
kubectl get svc
```

**¿Por qué?**
Para confirmar que el servicio `LoadBalancer` fue creado y revisar la IP externa (o el estado `pending` si no está asignada todavía) y el puerto expuesto. También muestra el puerto interno y el puerto NodePort asignado para acceso externo.

---

### 8.7. Acceder al servicio desde fuera del clúster

* Obtener IP pública (por ejemplo, con `curl ifconfig.io`)
* Usar la IP pública y el puerto NodePort para acceder vía navegador o `curl`.

**¿Por qué?**
El objetivo es probar el acceso real desde fuera del entorno de Kubernetes al servicio desplegado, confirmando que la configuración LoadBalancer y NodePort funciona correctamente.

---

### 8.8. Escalar el deployment a 0 réplicas (apagar nginx)

```bash
kubectl scale deployment nginx --replicas=0
kubectl get pods
```

**¿Por qué?**
Al escalar a 0 se terminan todos los pods, lo que debería hacer que el servicio ya no tenga backend disponible y, por ende, no responda a las peticiones. Esto verifica que la conexión depende realmente de los pods activos.

---

### 8.9. Escalar el deployment a 2 réplicas (encender nginx)

```bash
kubectl scale deployment nginx --replicas=2
kubectl get pods
```

**¿Por qué?**
Volver a levantar 2 pods permite restablecer el servicio y confirmar que el escalado dinámico funciona y que el servicio vuelve a responder con múltiples réplicas.

---

### 8.10. Eliminar deployment y servicio para liberar recursos

```bash
kubectl delete deployment nginx
kubectl delete svc nginx
```

**¿Por qué?**
Para limpiar el entorno y liberar los recursos asignados (CPU, memoria, IPs), evitando dejar objetos o servicios activos innecesarios.

