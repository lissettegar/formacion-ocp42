# Añadir un nodo de tipo Infraestructura


Puede crear un nodos de infrastructura para alojar solo componentes de infraestructura. Aplique etiquetas Kubernetes específicas a estas máquinas y luego actualice los componentes de infraestructura para que se ejecuten solo en esas máquinas. Estos nodos de infraestructura no se cuentan para el número total de suscripciones del cluster.

### Componentes de infraestructura de OpenShift Container Platform

Para calificar como nodo de infraestructura y usar el derecho incluido, solo los siguientes componentes incluidos en OpenShift pueden ejecutarse en estos nodos:

  * Kubernetes and OpenShift Container Platform control plane services that run on masters
  * The default router
  * The container image registry
  * The cluster metrics collection, or monitoring service
  * Cluster aggregated logging
  * Service brokers

Cualquier nodo que corra cualquier otro contenedor, pod o componente es un worker que necesita subscription.

## Procedimiento

1. [Añadir un nodo worker](Node/Adding-worker-node)

1. Etiquetar los nodos de "infra":

        $ oc label node esdc1svdla042.emea.segur.test node-role.kubernetes.io/infra=""

2. Comprobar que muestra el role correctamente:

        $ oc get nodes
        $ oc get node --show-labels

3. First, create an infra-mcp.yaml file with the following content:

        $ mkdir /opt/ocp4/infra
        vi infra-mcp.yaml
        apiVersion: machineconfiguration.openshift.io/v1
        kind: MachineConfigPool
        metadata:
          name: infra
        spec:
          machineConfigSelector:
            matchExpressions:
              - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,infra]}
          nodeSelector:
            matchLabels:
              node-role.kubernetes.io/infra: ""

        $ oc create -f infra-mcp.yaml

4. To verify the creation check for a rendered-infra config by running the following to view all Machine Configs:

        oc get mc
        oc get mcp

5. When it’s completed, the status for the infra MCP will show: True for UPDATED, False for UPDATING and False for DEGRADED.

    Note: Since the nodes are rebooted in order to apply this new machine config, this takes several minutes to complete.

6. Añadir taint a los nodos de infra para que no se ejecuten pods del cliente:

        oc adm taint nodes -l node-role.kubernetes.io/infra infra=reserved:NoSchedule infra=reserved:NoExecute

Referencias:

https://access.redhat.com/solutions/5034771
