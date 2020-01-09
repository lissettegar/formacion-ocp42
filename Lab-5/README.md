# Instrucciones Laboratorio 5

## Monitorizacion con Prometheus

1. Acceder a Prometheus. En la consola de OpenShift opcion **Monitoring** -> **Metrics** -> **Prometheus UI**

![alt Prometheus][imagen1]

[imagen1]: images/lab-prometheus1.png

2. Una vez en la consola de Prometheus seleccionar **Status** -> **Targets**

3. Confirmar que todos los Targets estan en estado **UP**

![alt Prometheus][imagen2]

[imagen2]: images/lab-prometheus2.png

4. Hacer clic en **Unhealthy** para comprobar que la pagina esta vacia. Esto indica que todo esta bien en el cluster.

5. Seleccionar la Opcion **Graph** para probar varias queries de Prometheus.

6. En la entrada **Expresion** escribir la siguiente lo siguiente y pulsar enter:

        kubelet_running_pod_count

7. Espere ver el numero de pods corriendo en cada nodo:

![alt Prometheus][imagen3]

[imagen3]: images/lab-prometheus3.png

8. Probar mas queries:

* node_memory_MemTotal_bytes / 1024 /1024
* node_memory_MemFree_bytes /1024 /1024
* round(node_memory_MemFree_bytes/1024/1024)

9. Borrar la Expresion y empezar a escribir **node**. Ver la lista de expresiones que se despliega y experimente con otras queries de nodo.

10. Explore con otras queries viendo la salida en modo grafico, en la pestaña **Graph** en lugar de **Condole**:

* namespace_pod_container:container_cpu_usage_seconds_total:sum_rate

![alt Prometheus][imagen4]

[imagen4]: images/lab-prometheus4.png

11. Todas las queries se pueden hacer tambien desde la consola de OpenShift **Monitoring** -> **Metrics**. Explorar por ejemplo **node_filesystem_size_bytes**:

![alt Prometheus][imagen8]

[imagen8]: images/lab-prometheus5.png

## Dashboards de Grafana

1. Acceder a Grafana. En la consola de OpenShift opcion **Monitoring** -> **Dashboards**

![alt Grafana][imagen5]

[imagen5]: images/lab-grafana1.png

2. Buscar la grafica **Kubernetes / Compute Resources / Cluster**. Esta grafica muestra un overview del cluster incluida la utilización de cpu, memoria:

![alt Grafana][imagen6]

[imagen6]: images/lab-grafana2.png

3. Explore los gráficos haciendo lo siguiente:

* Encuentra diferentes proyectos.
* Ver los gráficos para un solo proyecto.
* Cambiando los rangos de tiempo (parte superior derecha de la ventana)

4. Explorar los graficos de utilización de los nodos **USE Method / Node**:

![alt Grafana][imagen7]

[imagen7]: images/lab-grafana3.png
