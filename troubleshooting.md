# Master Troubleshooting Document - OpenShift Container Platform 4.x

Documento basado en:
 - Master Troubleshooting Article OpenShift Container Platform 4.x (https://access.redhat.com/articles/4217411)
 - OpenShift docs - Troubleshooting (https://docs.openshift.com/container-platform/4.5/support/troubleshooting/verifying-node-health.html)

## Indice
[[_TOC_]]

## Recoleccion de logs para soporte
Cuando se presenta algun problema con el cluster, el primer paso deberia ser abrir un caso de soporte en el portal del cliente: https://access.redhat.com/support/cases/

Para OpenShift Container Platform en general lo primero que se pide en los casos de soporte es un *must gather* que se toma de la siguiente manera:
```
$ oc adm must-gather
```

Luego comprimir los logs para subir al caso:
```
$ tar cvf must-gather_$(date +%Y%m%d%H%M).tar.gz must-gather.local.5421342344627712289/ 
```

Para mas informacion sobre la herramienta y otros modos de toma de datos ver: https://docs.openshift.com/container-platform/4.5/support/gathering-cluster-data.html

## Instalacion
> Troubleshooting OpenShift Container Platform 4.x: Installation (https://access.redhat.com/solutions/3780961)

Proceso de instalacion y troubleshooting de cada paso:
 - Bootstrap machine boots and starts hosting the remote resources required for master machines to boot.
 - La VM de Bootstrap bootea, durante el proceso de booteo el CoreOS obtiene su direccion IP a traves de DHCP asi como la direccion de los servidores DNS. Durante este proceso puede fallar lo siguiente:
	- La obtencion de la direccion IP desde el servidor DHCP.
	- La obtencion de los ignition files desde el servidor HTTP (generalmente el bastion)

 - Una vez que la VM de Bootstrap haya booteado, la misma levanta un control plane temporal para hostear los recursos necesarios para que boteen los masters. Durante este proceso puede fallar lo siguiente:
	- La obtencion de los recursos desde internet, es decir que el Bootstrap no se pueda bajar las imagenes que necesita de internet hay que revisar la conectividad contra quay.io

 - Los Master bootean y obtienen los recursos remotos desde el bootstrap. Durante este proceso puede fallar lo siguiente:
	- La obtencion de la direccion IP desde el servidor DHCP.
	- La obtencion de los ignition files desde el servidor HTTP (generalmente el bastion)
	- La conexion de los masters hacia el bootstrap mediante la Machine Config API (en este caso hay que revisar los DNS y el load balancer, puede suceder que los masters no lleguen al servidor de DNS o que la configuracion del load balancer este mal y no puedan bajar la configuracion de la URL: https://api-int.ocp.stg.cvattv.com.ar:22623/config/master
	- La obtencion de los recursos desde internet, es decir que los masters no se pueda bajar las imagenes que necesita de internet hay que revisar la conectividad contra quay.io

 - Los Master machines usan el bootstrap para formar un cluster etcd. Durante este proceso puede fallar lo siguiente:
	- La conexion de los masters hacia el bootstrap o entre ellos a traves de la api (https://api-int.ocp.stg.cvattv.com.ar:6443), revisar el load balancer y los DNS.

 - El nodo de Bootstrap node inicia un control plane temporal de Kubernetes utilizando el etcd cluster recien creado.
	- Revisar los logs del bootstrap:
```
$ ssh -l core bootstrap.ocp.stg.cvattv.com.ar
$ journalctl -b -f -u bootkube.service
```

 - El control plane temporal levanta el control plane productivo en los masters.
 - El control plane temporal se da de baja y le entrega el control al control plane productivo en los masters.
 - El nodo de Bootstrap node injecta componentes especificos de OpenShift al nuevo control plane.
 - En una instalacion IPI, el instalador destruye el nodo de bootstrap. En una instalacion UPI esto lo hace un administrador.
	- Durante estos pasos el troubleshooting se realiza conectandose a los masters y verificando los logs y el estado de los containers:
```
$ ssh -l core master01.ocp.stg.cvattv.com.ar
$ journalctl -xef
```

Para ver el estado de los containers pasarse al usuario root:
```
$ sudo -i
# crictl ps
```

Revisar los containers que tengan muchos reinicios (ATTEMPTS), hay que usar el container ID
```
# crictl logs 84d33630e4157
```

## Estado general
Se puede revisar el estado general del cluster en el "Overview" de la Web UI.

Desde el command line no hay un overview general, el equivalente son estos comandos:

Revision de los Cluster Operator
```
$ oc get clusteroperator
```

Revision del Cluster Version
```
$ oc get clusterversion
```

Revision de los eventos
```
$ oc get events -A
```

Revision del estado de los nodos
```
$ oc get nodes
```

Revision de la carga de CPU y Memoria de los nodos
```
$ oc adm top nodes
```


## Networking
Para diagnosticar problemas problemas con la SDN de OpenShift, consultar los documentos:
 - Troubleshooting the OpenShift Container Platform 4.x: cluster-network-operator (https://access.redhat.com/solutions/3787121)
 - Troubleshooting OpenShift Container Platform 4.x: openshift-sdn (https://access.redhat.com/solutions/3787161)

Para depurar problemas de red dentro de un Pod:
 - How to use tcpdump inside OpenShift v4 Pod (with ssh) (https://access.redhat.com/articles/4365651)

### DNS
En el siguiente link se podra informacion para validar el estado general el servicio de DNS dentro de la plataforma OpenShift:
 - Troubleshooting OpenShift Container Platform 4: DNS (https://access.redhat.com/solutions/3804501)

Fuera del OpenShift es necesario validar que los registros directos y reversos de cada nodo coincidan en su resultado y que existan.

Una cosa que no debe suceder y en principio no parece un problema pero puede generar que falle la instalacion es que en los registros reversos de los masters tambien esten los etcd.
Si sucede eso, luego cuando un componente de la instalacion hace una consulta sobre la IP del master puede llegar a recibir una respuesta que apunta al etcd en lugar de FQDN del nodo.

Validaciones por cluster
```
# LAB
$ DOMAIN="ocp.stg.cvattv.com.ar"
$ NODE_NAMES="infra1 infra2 infra3 master1 master2 master3 worker1 worker2"

# DEV+STAGING
$ DOMAIN="socp.stg.int.cvattv.com.ar"
$ NODE_NAMES="infra01 infra02 infra03 master01 master02 master03 worker01 worker02 worker03 worker04 worker05 worker06"

# PREPROD
$ DOMAIN="pocp.prepro.int.cvattv.com.ar"
$ NODE_NAMES="master01 master02 master03 worker01 worker02 worker03"

# PROD
$ DOMAIN="ocp01.int.cvattv.com.ar"
$ NODE_NAMES="app-worker01 app-worker02 app-worker03 app-worker04 app-worker05 app-worker06 infra01 infra02 infra03 master01 master02 master03 ocs-worker01 ocs-worker02 ocs-worker03"

# Validacion de registros directos
$ for i in ; do echo "$j.$DOMAIN <-> $(dig +short $j.$DOMAIN)"; done

# Validacion de registros reversos
$ for i in master01 master02 master03 infra01 infra02 infra03 worker01 worker02 worker03; do echo "$(dig +short $i.$DOMAIN) <-> $(dig +short -x $(dig +short $i.$DOMAIN))"; done

# Validacion de registros etcd
$ dig +short srv _etcd-server-ssl._tcp.$DOMAIN
$ for i in $(dig +short srv _etcd-server-ssl._tcp.$DOMAIN | awk '{ print $4 }'); do echo "$i <-> $(dig +short $i)"; done
$ for i in $(dig +short srv _etcd-server-ssl._tcp.$DOMAIN | awk '{ print $4 }'); do echo "$(dig +short $i) <-> $(dig +short -x $(dig +short $i))"; done

# Validacion de registros api
$ dig +short api.$DOMAIN
$ dig +short -x $(dig +short api.$DOMAIN)

# Validacion de registros api-int
$ dig +short api-int.$DOMAIN
$ dig +short -x $(dig +short api-int.$DOMAIN)

# Validacion de registros *.apps
$ dig +short *.apps.$DOMAIN
$ dig +short -x $(dig +short *.apps.$DOMAIN)
```

### Load Balancer
Revisar las reglas del Load Balancer para verificar que esten correctos los puertos, direcciones IP y el metodo de Scheduling

### LAB
SERVICE|FRONTEND|Virtual IP|PORT|BACKEND POOL|Scheduler|
|------|--------|----------|----|------------|---------|
|API Service|api.ocp.stg.cvattv.com.ar|10.254.245.51|6443|Bootstrap Server / Master Servers|Source IP|
|API Service (int)|api-int.ocp.stg.cvattv.com.ar|10.254.245.57|6443|Bootstrap Server / Master Servers|Source IP|
|Machine Server Config|api-int.ocp.stg.cvattv.com.ar|10.254.245.57|22623|Bootstrap Server / Master Servers|Source IP|
|Ingress HTTP|\*.apps.ocp.stg.cvattv.com.ar|10.254.245.125|80|Worker Servers|Round-Robin|
|Ingress HTTPS|\*.apps.ocp.stg.cvattv.com.ar|10.254.245.125|443|Worker Servers|Round-Robin|

### DEV+STAGING
SERVICE|FRONTEND|Virtual IP|PORT|BACKEND POOL|Scheduler|
|------|--------|----------|----|------------|---------|
|API Service|api.socp.stg.int.cvattv.com.ar|10.254.245.52|6443|Bootstrap Server / Master Servers|Source IP|
|API Service (int)|api-int.socp.stg.int.cvattv.com.ar|10.254.245.53|6443|Bootstrap Server / Master Servers|Source IP|
|Machine Server Config|api-int.socp.stg.int.cvattv.com.ar|10.254.245.53|22623|Bootstrap Server / Master Servers|Source IP|
|Ingress HTTP|\*.apps.socp.stg.int.cvattv.com.ar|10.254.244.217|80|Worker Servers|Round-Robin|
|Ingress HTTPS|\*.apps.socp.stg.int.cvattv.com.ar|10.254.244.217|443|Worker Servers|Round-Robin|

### PREPROD
SERVICE|FRONTEND|Virtual IP|PORT|BACKEND POOL|Scheduler|
|------|--------|----------|----|------------|---------|
|API Service|api.pocp.prepro.int.cvattv.com.ar|10.200.190.241|6443|Bootstrap Server / Master Servers|Source IP|
|API Service (int)|api-int.pocp.prepro.int.cvattv.com.ar|10.200.190.241|6443|Bootstrap Server / Master Servers|Source IP|
|Machine Server Config|api-int.pocp.prepro.int.cvattv.com.ar|10.200.190.241|22623|Bootstrap Server / Master Servers|Source IP|
|Ingress HTTP|\*.apps.pocp.prepro.int.cvattv.com.ar|10.200.189.241|80|Worker Servers|Round-Robin|
|Ingress HTTPS|\*.apps.pocp.prepro.int.cvattv.com.ar|10.200.189.241|443|Worker Servers|Round-Robin|

### PROD
SERVICE|FRONTEND|Virtual IP|PORT|BACKEND POOL|Scheduler|
|------|--------|----------|----|------------|---------|
|API Service|api.ocp01.int.cvattv.com.ar|10.200.203.208|6443|Bootstrap Server / Master Servers|Source IP|
|API Service (int)|api-int.ocp01.int.cvattv.com.ar|10.200.203.209|6443|Bootstrap Server / Master Servers|Source IP|
|Machine Server Config|api-int.ocp01.int.cvattv.com.ar|10.200.203.209|22623|Bootstrap Server / Master Servers|Source IP|
|Ingress HTTP|\*.apps.ocp01.int.cvattv.com.ar|10.200.173.9|80|Worker Servers|Round-Robin|
|Ingress HTTPS|\*.apps.ocp01.int.cvattv.com.ar|10.200.173.9|443|Worker Servers|Round-Robin|


## ETCD
ETCD es el key-value store para la plataforma OpenShift, el mismo persiste el estado de todos los objetos en el cluster. (es donde se almacena las configuraciones realizadas en el cluster).

### Backup
> Backing up etcd (https://docs.openshift.com/container-platform/4.5/backup_and_restore/backing-up-etcd.html)

En el bastion de cada ambiente (STAGING / PRERPOD / PROD) se encuentra un backup del etcd, el mismo esta en el cron del usuario openshift y corre todas las noches
```
[openshift@viptvcvaocbastion01 backups]$ crontab -l
# Creacion de un snapshot etcd y backup de los recursos estaticos de servicio de la API de Kubernetes
00 23 * * * /home/openshift/etcd-backup.sh

[openshift@viptvcvaocbastion01 backups]$ ls /home/openshift/backups/ocp/etcd/
[openshift@viptvcvaocbastion01 backups]$ ls /home/openshift/backups/socp/etcd/
```

### Status
> Replacing an unhealthy etcd member (https://docs.openshift.com/container-platform/4.5/backup_and_restore/replacing-unhealthy-etcd-member.html)
Ingresar al uno de los podd del etcd
```
$ POD=`oc get pods -n openshift-etcd -o=jsonpath='{.items[0].metadata.name}'`
$ oc rsh -n openshift-etcd -c etcdctl $POD
```

Verificar la lista de miembros y su estado
```
$ etcdctl member list -w table
$ etcdctl endpoint status -w table
$ etcdctl endpoint health
```

Lista de alarmas activadas
```
$ etcdctl alarm list
```

La performance de etcd es fundamental para el correcto funcionamiento del cluster, con estos comandos analizamos la performance
```
$ etcdctl check perf
$ etcdctl check perf --load='s'
$ etcdctl check perf --load='m'
```

Cada tanto puede ayudar a mejorar la performance una defragmentacion del etcd, aunque este proceso se realiza automaticamente en forma periodica.
```
$ etcdctl defrag
```

## Monitoring
Componentes del stack de Monitoring
 - Cluster Monitoring Operator: Controla el deployment centralizado de los componentes y recursos.
 - Prometheus Operator: Crea, configura y administra el Prometheus y el Alertmanager.
 - Prometheus: software de monitoreo y alerting que registra en tiempo real en una base de datos de tipo time-series.
 - Alertmanager: servicio que administra las alertas enviadas por Prometheus
 - node-exporter: agente que corre sobre cada nodo para recolectar metricas sobre el mismo
 - Grafana: plataforma de analytics que provee dashboards para visualizar y analizar las metricas.


## Logging
Componentes del stack de Logging:
 - Kibana: UI para visualizar los logs almacenados en Elasticsearch. Realiza queries hacia Elasticsearch utilizando la REST API.
 - Elasticsearch: donde se almacenan los logs enviados por Fluentd para persistencia.
 - Fluentd: aplicacion que se deploya en todos los nodos mediante un DaemonSet. Recolecta los logs de los containers que corren en cada nodo y los envia a Elasticsearch
 - Curator: programa que se configura para limpiar periodicamente los logs a fin de liberar espacio en el Elasticsearch. Se comunica con Elasticsearch mediante la REST API.

Cambiar al projecto de OpenShift Logging
```
$ oc project openshift-logging
```

El detalle de la instancia de Cluster Logging en el apartado status -> nodeConditions tiene un resumen del estado del cluster
```
$ oc get clusterlogging instance -o yaml
```

Para revisar el estado del Elasticsearch, primero vemos como se llama la instancia (en general es: elasticsearch)
```
$ oc get Elasticsearch
```

Podemos ver el estado de la instancia viendo el detalle status -> clusterHealth
```
$ oc get Elasticsearch elasticsearch -n openshift-logging -o yaml
```

Verificar el estado de los indices
```
$ POD=`oc get pods --selector component=elasticsearch -n openshift-logging -o=jsonpath='{.items[0].metadata.name}'`
$ oc exec -n openshift-logging $POD -- indices
```

Verificar el estado general del cluster
```
$ POD=`oc get pods --selector component=elasticsearch -n openshift-logging -o=jsonpath='{.items[0].metadata.name}'`
$ oc exec -n openshift-logging $POD -- es_cluster_health
```

Verificar el espacio libre en el elasticsearch
```
$ POD=`oc get pods --selector component=elasticsearch -n openshift-logging -o=jsonpath='{.items[0].metadata.name}'`
$ oc exec -c elasticsearch -n openshift-logging $POD -- df -h /elasticsearch/persistent
```

Si hay algun componente del cluster o alguna aplicacion que esta logeando mucho el tiempo de retencion de los logs en combinacion con el espacio aprovisionado podria ser insuficiente.
Como solucion inmediata, para liberar espacio se puede modificar el curator para bajarle el tiempo de retencion
```
$ oc edit -n openshift-logging configmap/curator
```

Luego se hace una pasada manual del curator para que se aplique el nuevo tiempo de retencion de logs
```
$ oc create job --from=cronjob/curator curator-$(date +%Y%m%d%H%M)
```


## OCS
Documentacion: https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.4/

### Arquitectura
Componentes del OCS:
 - Rook-Ceph operator: automatiza el deployment, management, upgrade y escalamiento del OCS.
 - NooBaa operator: Provee el Multicloud Object Gateway, un object store compatible con S3.
 - Ceph CSI plugins: implementan una interface CSI (Container Storage Interface) entre Kubernetes y el Cluster de Ceph. Permite el aprovisionamiento dinamico de volumenes Ceph, adjuntandolos a los workloads.
	- csi-cephfsplugin-<ID>: Este plugin corre en todos los nodos y permite adjuntar volumenes de tipo CephFS (Ceph Filesystem: similar a NFS)
	- csi-rbdplugin-<ID>: Este plugin corre en todos los nodos y permite adjuntar volumenes de tipo CephRBD (Ceph RADOS Block Device: volumen de bloques)
 - Monitors: Los monitores mantienen un mapa del estado del cluster de OCS y ayudan en la toma de decisiones. No son parte del data-path y por ende no sirven requests de IO hacia clientes
 - MGRs: Estan estrechamente ligados con los Monitors y recolectan estadisticas dentro del cluster de Ceph. 
 - MDSs (Meta Data Servers): Manejan los metadatos de los filesystem POSIX (como permisos de archivos, la jerarquia de directorios, etc.). Solo son deployados si se configura un shared filesystem dentro del cluster Ceph (algun volumen de tipo CephFS)
 - OSDs: Se deploya un OSD para cada block device configurado en el cluster. Sirven IO requests y garantizan la proteccion de los datos, el rebalanceo en caso de falla y la coherencia de los datos
 - RADOS: El corazon de Ceph es un object store conocido como RADOS (Reliable Autonomic Distributed Object Store). Esta capa provee a Ceph un software defined storage con la capacidad de almacenar datos, servir pedidos de IO, proteger los datos y validar la consistencia e integridad a traves de mecanismos integrados.
 - CRUSH: Se trata de un algoritmo de emplazamiento pseudo-aleatorio cuyo objetivo es distribuir los objetos a lo largo de grupos de emplazamiento (PG o Placement Groups) y utiliza reglas para determinar el mapeo de PGs hacia los OSDd. En escencia los PGs representan una capa de abstraccion entre los objetos (capa de aplicacion) y los OSD (capa fisica). En el caso de falla los PGs seran remapeados hacia dispositivos fisicos distintos (OSDs).

Diagrama de los componentes de Rook-Ceph
![Rook-Ceph Diagram](img/Rook-Ceph.png "Rook-Ceph Diagram")

### Status
En el overview de la Web UI hay un apartado para OCS en el cluster de PROD (que es el que tiene Ceph instalado)
https://console-openshift-console.apps.ocp01.int.cvattv.com.ar/dashboards/persistent-storage
![Persistent Storage Overview](img/OCS-Overview.png "Persistent Storage Overview")

### CephTools
Habilitar CephTools (Solo es necesario hacerlo una vez)
```
$ oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
```

Verificar que haya levantado el pod (rook-ceph-tools)
```
$ oc get pods -n openshift-storage
```

Conectarse al toolbox
```
$ TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
$ oc rsh -n openshift-storage $TOOLS_POD
```

Verificar el estado del Ceph Cluster
```
$ ceph status
$ ceph osd status
$ ceph osd tree
$ ceph osd df
$ ceph health detail
$ ceph df
```


## OpenShift nodes
Este documento es muy extenso y tiene todo tipo de formas de obtener metricas para realizar troubleshooting de los nodos
> How to collect worker metrics do troubleshoot CPU load, memory pressure and interrupt issues and networking on worker nodes in Red Hat OpenShift Container Platform 4.x (https://access.redhat.com/solutions/5343671)

Se puede entrar por SSH a los nodos con el usuario 'core' para revisar el estado de los mismos (luego se pude usar sudo si es necesario)
```
$ ssh -l core <node>
```

Revisar los logs de la consola para ver que no hayan errores a nivel hardware o algun otro mensaje que pueda indicar problemas con la plataforma subyacente.
```
$ dmesg
```

### OSTree
Para revisar la version del OSTree
```
$ grep OSTREE_VERSION /etc/os-release
$ rpm-ostree version
```

Con el status vemos que imagen va a ser booteada en el proximo
```
$ rpm-ostree status
State: idle
Deployments:
● pivot://quay.io/openshift/okd-content@sha256:8cf7e06dd4095f2cd54e13fdb6fd313abbeb6e03d568f17956d97433623093c2
              CustomOrigin: Managed by machine-config-operator
                 Timestamp: 2020-07-27T13:06:27Z

  ostree://fedora:fedora/x86_64/coreos/stable
                   Version: 32.20200907.3.0 (2020-09-23T08:16:31Z)
                    Commit: b53de8b03134c5e6b683b5ea471888e9e1b193781794f01b9ed5865b57f35d57
              GPGSignature: Valid signature by 97A1AE57C3A2372CCA3A4ABA6C13026D12C944D0
```

En algunos casos puede ser necesario hacer rollback de la imagen si fallo el upgrade el CoreOS
```
$ rpm-ostree rollback
```

Los servicios mas importantes a revisar en el nodo son el de Kubelet y CRI-O (container runtime)
 - kubelet es el servicio que cumple con los pedidos de levantar y detener containers de parte de los masters. Tambien registra el nodo con el apiserver.
 - CRI-O container engine (crio) es el servicio que corre y administra los containers
```
$ systemctl status kubelet.service
$ systemctl status crio.service
$ journalctl -b -f -u kubelet.service -u crio.service
$ journalctl -b -f -p err -u kubelet.service
$ journalctl -b -f -p err -u crio.service
```

Tambien es importante revisar el estado de los pods y los containers en el nodo, mirar la columna ATTEMPTS para tener una idea de con que frecuencia fue reiniciado un container.
Si un container fue reiniciado muchas veces significa que esta teniendo algun problema y no puede llegar al estado 'ready'
```
$ sudo crictl ps
$ sudo crictl logs <Container ID> 
$ sudo crictl pods
$ sudo crictl logs <Pod ID> 
$ sudo tail -f /var/log/containers/*
```

Revisar tambien si no hay algun pod consumiendo mucha memoria/cpu
```
$ sudo podman ps
$ sudo podman stats
```

### Machine Config
> Troubleshooting OpenShift Container Platform 4.x: machine-config operator (https://access.redhat.com/articles/4550741)

Revisar el Machine Config Pool que no haya ningun nodo en UPDATING o DEGRADED
```
[openshift@viptvcvaocbastion01 ~]$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
infra    rendered-infra-94b2fc70ad42e53049c138aaf26041c7    True      False      False      3              3                   3                     0                      45d
master   rendered-master-1b51797d1c53cc456d33350d281ed698   True      False      False      3              3                   3                     0                      245d
worker   rendered-worker-2aa1193009dca1bca71ffe68e1d2bbe8   True      False      False      6              6                   6                     0                      245d
```

Si hay algun problema, entonces revisar los logs del Machine Config Daemon del nodo en cuestion
```
$ oc get pods -n openshift-machine-config-operator -o wide
[openshift@viptvcvaocbastion01 ~]$ oc get pods -n openshift-machine-config-operator -o wide
NAME                                         READY   STATUS    RESTARTS   AGE   IP              NODE                                  NOMINATED NODE   READINESS GATES
etcd-quorum-guard-6d647d9cf-dzfhr            1/1     Running   0          47h   10.254.245.61   master02.socp.stg.int.cvattv.com.ar   <none>           <none>
etcd-quorum-guard-6d647d9cf-kq9qb            1/1     Running   0          47h   10.254.245.62   master03.socp.stg.int.cvattv.com.ar   <none>           <none>
etcd-quorum-guard-6d647d9cf-ms4fg            1/1     Running   0          47h   10.254.245.60   master01.socp.stg.int.cvattv.com.ar   <none>           <none>
machine-config-controller-56555657d8-mtndd   1/1     Running   3          47h   10.32.0.25      master02.socp.stg.int.cvattv.com.ar   <none>           <none>
machine-config-daemon-5jdtr                  2/2     Running   1          2d    10.254.245.60   master01.socp.stg.int.cvattv.com.ar   <none>           <none>
machine-config-daemon-f4t6s                  2/2     Running   1          2d    10.254.245.66   worker01.socp.stg.int.cvattv.com.ar   <none>           <none>
<...>

$ oc logs machine-config-daemon-f4t6s -c machine-config-daemon -n openshift-machine-config-operator
```

Para revisar los logs de los Machine Config Daemon en todos los Nodos
```
for i in $(oc get pods -n openshift-machine-config-operator -o wide | grep 'machine-config-daemon' | awk '{ print $1 }'); do oc logs $i -c machine-config-daemon -n openshift-machine-config-operator; read; done
```


## Updating
Se puede abrir un caso proactivo para tener al equipo de soporte atento a cualquier inconveniente que pueda surgir durante una actualizacion
> How to open a PROACTIVE case for patching or upgrading Red Hat OpenShift Container Platform (https://access.redhat.com/solutions/3521621)

Hay un documento que explica como visualizar los posibles caminos de upgrade entre versiones:
> Upgrade Paths explained: https://access.redhat.com/solutions/4583231

OpenShift Troubleshooting Upgrade Issues: https://access.redhat.com/articles/4394111

Si se desea ejecutar el upgrade desde el command line, antes hay que ejecutar estos comandos para revisar el estado del cluster:
```
$ oc get clusteroperator
$ oc get clusterversion
$ oc get clusterversion -o yaml
$ oc logs $(oc get pod -n openshift-cluster-version -l k8s-app=cluster-version-operator -oname) -n openshift-cluster-version
```

Verificar el channel seleccionado para el upgrade, para entornos productivos se recomienda el channel 'stable'
```
$ oc get clusterversion -o yaml
```

Para modificar el channel, ejecutar el siguiente comando
```
$ oc patch clusterversion/version -p '{"spec":{"channel":"stable-4.4"}}' --type=merge
```

Si se ejecuta el upgrade sin indicar una version especifica entonces el oc adm presenta una lista de posibles versiones
```
$ oc adm upgrade
```

Para actualizar a la ultima version dentro del channel seleccionado:
```
$ oc adm upgrade --to-latest
```

Una vez iniciado el upgrade, se debe monitorear el estado de los Cluster Operator y el Cluster Version
```
$ oc get clusteroperator
$ oc get clusterversion
$ oc get clusterversion -o yaml
```

### Verificaciones post-upgrade
Uno de los ultimos pasos de la instalacion es actualizar el sistema operativo de los nodos, este paso se puede monitorear mirando el Machine Config Pool.
> Verificar que los nodos hayan quedado en estado UPDATED: True, UPDATING: false.
```
$ oc get nodes

$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
infra    rendered-infra-c9bbc41a8559701f78595734aa05df61    True      False      False      3              3                   3                     0                      125d
master   rendered-master-d2d8ce559c3acaa92442e799522792ff   True      False      False      3              3                   3                     0                      125d
worker   rendered-worker-73a3d6b2fc3c74f76817472ab3bcbbd4   True      False      False      6              6                   6                     0                      125d
```

Operators: Los operators que se instalan desde el Operator Hub no se actualizan automaticamente ya que se define el channel de actualizacion al instalarlos y eso no cambia.
Estos operators se deben actualizar por separado
 - Cluster Logging: https://docs.openshift.com/container-platform/4.5/logging/cluster-logging-upgrading.html
 - OpenShift Container Storage: https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.4/html/managing_openshift_container_storage/upgrading-your-cluster_aws-vmware

Una vez actualizado el cluster tambien hay que actualizar a la version de oc que corresponda al update porque sino puden fallar algunos comandos oc debido a los updates en las APIs
```
OCP_VERSION=4.5.11
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux-${OCP_VERSION}.tar.gz
tar xzvf openshift-client-linux-${OCP_VERSION}.tar.gz
cp oc /usr/local/bin
```


## Borrar Proyecto en estado "Terminating"
A veces puede pasar que al borrar un Proyecto, algunos recursos del mismo no lleguen a borrarse automaticamente.
Lo primero que hay que hacer es depurar el problema diligentemente revisando los recursos asociados al proyecto y revisar porque no se pueden borrar.
Una vez realizado eso, si ya no queda nada mas por revisar entonces seguir los pasos de la solution:
> Unable to Delete a Project or Namespace (https://access.redhat.com/solutions/4165791)
