# Añadir nodo worker RHCOs al cluster

1. Crear la VM del nodo a añadir copiando el fichero ignitio de los workers (fichero que se uso para crear el cluster) y encenderla.

2. Una vez el nodo este arrancado aprobar los certificados para que de esta forma entre en el cluster:

        $ oc get csr -o name | xargs oc adm certificate approve
        certificatesigningrequest.certificates.k8s.io/csr-hlp79 approved
        certificatesigningrequest.certificates.k8s.io/csr-tmfz2 approved

3. En este punto ya deberia aparecer dentro del cluster:

        $ oc get nodes
        NAME                                STATUS   ROLES    AGE     VERSION
        master1dev.ocpdevmad01.tic1.intranet   Ready    master   4d17h   v1.14.6+6ac6aa4b0
        master2dev.ocpdevmad01.tic1.intranet   Ready    master   4d17h   v1.14.6+6ac6aa4b0
        master3dev.ocpdevmad01.tic1.intranet   Ready    master   4d17h   v1.14.6+6ac6aa4b0
        mgmt1dev.ocpdevmad01.tic1.intranet     Ready    worker   4d17h   v1.14.6+6ac6aa4b0
        worker1dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0
        worker2dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0
        worker3dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0
        worker4dev.ocpdevmad01.tic1.intranet   Ready    worker   4d17h   v1.14.6+6ac6aa4b0