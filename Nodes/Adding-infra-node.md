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

2. Etiquetar el nodo para que solo sea de `Infra`:

        $ oc label node mgmt1dev.ocpdevmad01.tic1.intranet node-role.kubernetes.io/infra=""
        node/mgmt1dev.ocpdevmad01.tic1.intranet labeled
        $ oc label node mgmt1dev.ocpdevmad01.tic1.intranet node-role.kubernetes.io/worker-
        node/mgmt1dev.ocpdevmad01.tic1.intranet labeled

3. Comprobar que muestra la etiqueta correctamente:

        $ oc get nodes
        NAME                                STATUS   ROLES    AGE     VERSION
        master1dev.ocpdevmad01.tic1.intranet   Ready    master   4d17h   v1.14.6+6ac6aa4b0
        master2dev.ocpdevmad01.tic1.intranet   Ready    master   4d17h   v1.14.6+6ac6aa4b0
        master3dev.ocpdevmad01.tic1.intranet   Ready    master   4d17h   v1.14.6+6ac6aa4b0
        mgmt1dev.ocpdevmad01.tic1.intranet     Ready    infra    4d17h   v1.14.6+6ac6aa4b0
        worker1dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0
        worker2dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0
        worker3dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0
        worker4dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0

4. Si es el primer nodo de `infra` que se crea, crear un MachineConfigPool para el nodo de infraestructura:

        $ vi infra-machineconfigpool.yaml

        apiVersion: machineconfiguration.openshift.io/v1
        kind: MachineConfigPool
        metadata:
          name: infra
        spec:
          configuration:
            source:
            - apiVersion: machineconfiguration.openshift.io/v1
              kind: MachineConfig
              name: 00-worker
            - apiVersion: machineconfiguration.openshift.io/v1
              kind: MachineConfig
              name: 01-worker-container-runtime
            - apiVersion: machineconfiguration.openshift.io/v1
              kind: MachineConfig
              name: 01-worker-kubelet
            - apiVersion: machineconfiguration.openshift.io/v1
              kind: MachineConfig
              name: 99-worker-1ae64748-a115-11e9-8f14-005056899d54-registries
            - apiVersion: machineconfiguration.openshift.io/v1
              kind: MachineConfig
              name: 99-worker-ssh
          machineConfigSelector:
            matchLabels:
              machineconfiguration.openshift.io/role: worker
          maxUnavailable: null
          nodeSelector:
            matchLabels:
              node-role.kubernetes.io/infra: ""
          paused: false

        $ oc create -f infra-machineconfigpool.yaml

        $ oc get machineconfig
        NAME                                                        GENERATEDBYCONTROLLER                      IGNITIONVERSION   CREATED
        00-master                                                   55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        00-worker                                                   55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        01-master-container-runtime                                 55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        01-master-kubelet                                           55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        01-worker-container-runtime                                 55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        01-worker-kubelet                                           55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        99-master-cfeb4b91-12c0-11ea-8599-0050568c1451-registries   55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        99-master-ssh                                                                                          2.2.0             2d23h
        99-worker-cfec3d3e-12c0-11ea-8599-0050568c1451-registries   55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        99-worker-ssh                                                                                          2.2.0             2d23h
        rendered-infra-286b825c7893dd1ed0567c2b30fac014            55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             4m30s
        rendered-master-5cbef496fab483c10f891e9b3d9a7ca1            55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h
        rendered-worker-286b825c7893dd1ed0567c2b30fac014            55bb5fc17da0c3d76e4ee6a55732f0cba93e8520   2.2.0             2d23h

Referencias:

  https://access.redhat.com/solutions/4246261

  https://access.redhat.com/solutions/4287111

  https://docs.openshift.com/container-platform/4.2/machine_management/creating-infrastructure-machinesets.html
