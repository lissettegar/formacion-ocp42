# Añadir nodo worker RHCOs al cluster

1. Crear la VM del nodo a añadir a partir de la plantilla de worker sin encenderla (una vez clonada se puede cambiar la cantidad de mem, cpu o disco, si se desea). Al clonarla de la plantilla ya tendra el contenido del ignitio que se necesita.

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

3. Configurar la variable `uestinfo.ignition.config.data` con el contenido del ignition file para la VM

  3.1. Obtener el contenido del ignition file que vamos a necesitar. Este fichero esta en cada uno de los nodos `hapro` y `hadev` en el directorio `/home/logicalis/Recursos_Instalacion/`. Por ejemplo para el entorno de produccion el fichero esta en `/home/logicalis/Recursos_Instalacion/ocppromad01` (siempre el directorio que no tiene fecha como parte del nombre). Este es el contenido:

        [root@hapro ocppromad01]# ll
        total 316
        -rw-r--r-- 1 root root    400 dic 11 19:25 append-bootstrap.64
        -rw-r--r-- 1 root root    299 dic 11 19:25 append-bootstrap.ign
        drwxr-xr-x 2 root root     50 dic 11 19:23 auth
        -rw-r--r-- 1 root root 292103 dic 11 19:23 bootstrap.ign
        -rw-r--r-- 1 root root   2440 dic 11 19:25 master.64
        -rw-r--r-- 1 root root   1830 dic 11 19:23 master.ign
        -rw-r--r-- 1 root root    110 dic 11 19:23 metadata.json
        -rw-r--r-- 1 root root   2440 dic 11 19:25 worker.64
        -rw-r--r-- 1 root root   1830 dic 11 19:23 worker.ign

    3.2. Siendo un worker lo que se va a añadir, copiaremos el contenido del fichero worker.64. Por ejemplo:

        [root@hapro ocppromad01]# cat worker.64
        eyJpZ25pdGlvbiI6eyJjb25maWciOnsiYXBwZW5kIjpbeyJzb3VyY2UiOiJodHRwczovL2FwaS1pbnQub2NwcHJvbWFkMDEudGljMS5pbnRyYW5ldDoyMjYyMy9jb25maWcvd29ya2VyIiwidmVyaWZpY2F0aW9uIjp7fX1dfSwic2VjdXJpdHkiOnsidGxzIjp7ImNlcnRpZmljYXRlQXV0aG9yaXRpZXMiOlt7InNvdXJjZSI6ImRhdGE6dGV4dC9wbGFpbjtjaGFyc2V0PXV0Zi04O2Jhc2U2NCxMUzB0TFMxQ1JVZEpUaUJEUlZKVVNVWkpRMEZVUlMwdExTMHRDazFKU1VSRlJFTkRRV1pwWjBGM1NVSkJaMGxKVUVoWGVFeEhNRWRMTjFsM1JGRlpTa3R2V2tsb2RtTk9RVkZGVEVKUlFYZEtha1ZUVFVKQlIwRXhWVVVLUTNoTlNtSXpRbXhpYms1dllWZGFNRTFTUVhkRVoxbEVWbEZSUkVWM1pIbGlNamt3VEZkT2FFMUNORmhFVkVVMVRWUkplRTFVUlRSTmFrMTNUakZ2V0FwRVZFazFUVlJKZDA5RVJUUk5hazEzVGpGdmQwcHFSVk5OUWtGSFFURlZSVU40VFVwaU0wSnNZbTVPYjJGWFdqQk5Va0YzUkdkWlJGWlJVVVJGZDJSNUNtSXlPVEJNVjA1b1RVbEpRa2xxUVU1Q1oydHhhR3RwUnpsM01FSkJVVVZHUVVGUFEwRlJPRUZOU1VsQ1EyZExRMEZSUlVGMVlqWmtUVnBOUzIxMk5uTUthRVZPYjFWd1lXVmtORWhYYUVWaVdXVndNRWt5TlRBdmJVeEtNVzlsT1VoMGNUZFdlRVJxY3pKalpEaG9Rbm81ZVhCQmFsUnhlSEJRVWxGSFlWZE1RUXB1YjFST2VEbGlObkJ0U2xkSlEzcDVSVWxvYVVacUsyMW1aMVJ5ZFNzeGJXcEpiVVEyYUd4S2VITndNVWRWU21OMVZuVlVXbnAyZDNnMWJXd3JZa1IyQ2xocFJUTk9lR1Y1V0M5bk1saDJhbVUxWlRFeFQzUkZiVFoyTDNGc1owODFTblJ2YVdoRU1IbDVOMnQ2V0UxMGNVeEVkRVpUYVZoS1NrZFFWRTh3YkRBS1RIaFViMmg1ZUhOQmJtSjVWVk5PV2k5V2ExRkZhblpoWlRkNFpsVnZiRXAyZGxSamJqUmljVkZQTmtSaFZtTnpRVGxYVFRkMVIwbFlSMDExUTFabVZBcEdURVF5ZHpCaWNVbE1ialppZG5OT2QzZFJUVGQ1U1Vjd05YRlhSVGxUTTNWdlNIcGlVSEp1ZWpKTlowZHFVaXRtYVZRdlNFeHFhMDlQUVRKVU5FaHJDbkl2VjNoWE4zUnliWGRKUkVGUlFVSnZNRWwzVVVSQlQwSm5UbFpJVVRoQ1FXWTRSVUpCVFVOQmNWRjNSSGRaUkZaU01GUkJVVWd2UWtGVmQwRjNSVUlLTDNwQlpFSm5UbFpJVVRSRlJtZFJWVFpMUzBWWlpsZHBPWGwzU1VrNVNVSjVlVmxNUVU5bk9URTJZM2RFVVZsS1MyOWFTV2gyWTA1QlVVVk1RbEZCUkFwblowVkNRVXAwUm01cmRHaGhhVlZpZW1ONlZHbFdTRzlEUmxWSlMxUkNiSEkwYkZveVFqUkxZalZvY1dSa2VFRjVRVUpzVEROc1pHSnNWM0ExTUZWd0NubHpiRFJQT0VvMmVFcExWRGRDYW5FeVJtZFRTakZoV2tSSUx6Tm5kVXh1Um5kd2RuUXZWVTExTURSUU5uZERSRWd5YVZBNWFIQnJkazR2TTBkaVUyMEtiRkp1TjBodGNHWlVaVXByWjI5eVVHWXZaV1pXU0RNMVpXUmxlbVEyTUdaSFRFdDFhbE54ZUZOdFFqUlpjMlZvVUVKYWNHWnBkbmx2V1ZkVWNUVkRUZ3BrWVhWRVZtMVBPR0ZUZWxGTFozVlhObkY1YWxaR1JrbDZVMHRaV0ZGcU1rVnFWSGh1U0VSM2JVWmhRa1V5YVNzelVqbEpVbmRsTjJVclVqSm5WbkpyQ25KUlVYUkZSbTVuVHpadlFsSlhiemxTUkZJcmJWQXdSMmR2YTBGUlkxa3pVRnBYUlRCWFYwRlBhMWRxU1RSS2QzcEVVR3BoWldveFdIYzFibVpLZW5ZS2IyNTBXRGRrWldFMmREUlRaM2R0YVZWdFZqTnhkM2t4Tnl0dlBRb3RMUzB0TFVWT1JDQkRSVkpVU1VaSlEwRlVSUzB0TFMwdENnPT0iLCJ2ZXJpZmljYXRpb24iOnt9fV19fSwidGltZW91dHMiOnt9LCJ2ZXJzaW9uIjoiMi4yLjAifSwibmV0d29ya2QiOnt9LCJwYXNzd2QiOnt9LCJzdG9yYWdlIjp7fSwic3lzdGVtZCI6e319

  3.3. En la Opcion `Editar Configuracion -> Opciones de Maquina Virtual -> Avanzado -> EDITAR CONFIGURACION` buscar la variable `guestinfo.ignition.config.data` y borrar el contenido.

  3.4. Copiar como contenido de la variable el contenido del fichero worker.64 que se busco en el paso 2.2.



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
