# Añadir nodo worker RHCOs al cluster

1. Crear la VM del nodo a añadir a partir de la plantilla o clonandola de otro worker sin encenderla (una vez clonada se puede cambiar la cantidad de mem, cpu o disco, si se desea).

2. Añadir entrada en el dhcpd.conf para la maquina a añanadir. El servidor dhcp es el nodo hadev para desarrollo y el hapro para produccion. Ejemplo de añadir el nodo `mcm1pro`:

        subnet 10.1.5.0 netmask 255.255.255.0 {
                option subnet-mask      255.255.255.0;
                option domain-name-servers      10.1.5.247, 10.1.5.249;
                option routers  10.1.5.2;
                range 10.1.5.55 10.1.5.55;
                range 10.1.5.59 10.1.5.59;
                range 10.1.5.70 10.1.5.70;
                range 10.1.5.80 10.1.5.81;
                range 10.1.5.115 10.1.5.115;
                range 10.1.5.144 10.1.5.144;
                range 10.1.5.150 10.1.5.150;
                range 10.1.5.180 10.1.5.180;
                range 10.1.5.220 10.1.5.221;
                range 10.1.5.224 10.1.5.224;
                range 10.1.5.246 10.1.5.246;
                range 10.1.5.88 10.1.5.88;
        }

        # entradas estaticas
        host bootstrappro { option host-name "bootstrappro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:89:a5; fixed-address 10.1.5.59; }
        host master1pro { option host-name "master1pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:22:3f; fixed-address 10.1.5.70; }
        host master2pro { option host-name "master2pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:2b:7b; fixed-address 10.1.5.80; }
        host master3pro { option host-name "master3pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:8e:46; fixed-address 10.1.5.81; }
        host worker1pro { option host-name "worker1pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:5d:40; fixed-address 10.1.5.144; }
        host worker2pro { option host-name "worker2pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:7a:39; fixed-address 10.1.5.150; }
        host worker3pro { option host-name "worker3pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:b7:6f; fixed-address 10.1.5.180; }
        host worker4pro { option host-name "worker4pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:97:11; fixed-address 10.1.5.220; }
        host worker5pro { option host-name "worker5pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:ff:5b; fixed-address 10.1.5.221; }
        host worker6pro { option host-name "worker6pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:37:2a; fixed-address 10.1.5.224; }
        host mgmt1pro { option host-name "mgmt1pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:f0:33; fixed-address 10.1.5.115; }
        host mgmt2pro { option host-name "mgmt2pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:dd:dd; fixed-address 10.1.5.246; }
        host mcm1pro { option host-name "mcm1pro.ocppromad01.tic1.intranet"; hardware ethernet 00:50:56:8c:14:51`; fixed-address 10.1.5.88; }
`

3. Configurar la variable `guestinfo.ignition.config.data` con el contenido del ignition file para la VM

  3.1. Para obtener el fichero ignition seguir las indicaciones del doc:

  https://access.redhat.com/solutions/4799921

  3.2. Copiar el fichero ignition creado en el nodo bastion.


  3.3. En la Opcion `Editar Configuracion -> Opciones de Maquina Virtual -> Avanzado -> EDITAR CONFIGURACION` buscar la variable `guestinfo.ignition.config.data` y borrar el contenido.

  3.4. Copiar como contenido de la variable el contenido del fichero worker.64 del punto 3.2.



4. Encender la VM. Una vez el nodo este arrancado aprobar los certificados para que de esta forma entre en el cluster:

        $ oc get csr -o name | xargs oc adm certificate approve
        certificatesigningrequest.certificates.k8s.io/csr-hlp79 approved
        certificatesigningrequest.certificates.k8s.io/csr-tmfz2 approved

5. En este punto ya deberia aparecer dentro del cluster:

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
