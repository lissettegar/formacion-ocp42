# Instrucciones Laboratorio 5

## Desplegar aplicacion en OCP usando un pipeline complejo

## 1. Instalar y configurar Nexus

1.1. Exportar la variable de entorno:

    $ export GUID=<iniciales en minuscula>

1.2. Cree un nuevo proyecto donde instalar Nexus:

    $ oc new-project ${GUID}-nexus --display-name "${GUID} Shared Nexus"

1.3. Desplegar Nexus y crear el route para exponer el servicio:

    $ oc new-app sonatype/nexus3:latest
    $ oc expose svc nexus3

1.4. Pausar el despliegue para hacer cambios en el deployment config (hacemos un pause para que no redespliegue un nuevo pod con cada unos de los cambios):

    $ oc rollout pause dc nexus3

1.5. Cambiar la estrategia de deployment de `Rolling` a `Recreate` y configurar `requests` y `limits` para la `memoria`:

    $ oc patch dc nexus3 --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'
    $ oc set resources dc nexus3 --limits=memory=2Gi,cpu=2 --requests=memory=1Gi,cpu=500m

1.6. Configurar los valores para el `liveness` y `readiness` probe para comprobar el correcto funcionamiento del Nexus:

    $ oc set probe dc/nexus3 --liveness --failure-threshold 3 --initial-delay-seconds 60 -- echo ok
    $ oc set probe dc/nexus3 --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8081/

1.7. Reanudar el deployment del Nexus para que actualice todos los cambios:

    $ oc rollout resume dc nexus3

1.8. Una vez que Nexus este desplegado, configurar el repositorio de Nexus usando el script `setup_nexus3.sh` (se descargara con un curl). Usar el usuario por defecto (admin). La password por defecto de `admin` se guarda en el pod de Nexus en el fichero `/nexus-data/admin.password`. Para obtener dicha password usaremos el comando `oc rsh`:

     $ cd /home/<directorio de trabajo>
     $ oc get pods
     NAME              READY   STATUS      RESTARTS   AGE
     nexus3-1-deploy   0/1     Completed   0          14m
     nexus3-2-4ndw4    1/1     Running     0          7m44s
     nexus3-2-deploy   0/1     Completed   0          8m3s
     $ export NEXUS_PASSWORD=$(oc rsh nexus3-2-4ndw4 cat /nexus-data/admin.password)
     $ echo $NEXUS_PASSWORD
     6727c68c-e6d1-457b-9b31-be65b0c2fd0a
     $ curl -o setup_nexus3.sh -s https://raw.githubusercontent.com/redhat-gpte-devopsautomation/ocp_advanced_development_resources/master/nexus/setup_nexus3.sh
     $ chmod +x setup_nexus3.sh
     $ ./setup_nexus3.sh admin $NEXUS_PASSWORD http://$(oc get route nexus3 --template='{{ .spec.host }}')

1.9. Este script `setup_nexus3.sh` crea lo siguiente:

* Algunos repositorios proxy de Maven para almacenar en caché las dependencias de Red Hat y JBoss.
* Un repositorio de grupo maven-all-public que contiene los repositorios proxy para todos los artefactos necesarios.
* Un repositorio proxy NPM para almacenar en caché los artefactos de compilación Node.JS.
* Un registro privado de Docker.
* Un repositorio de versiones para los archivos WAR producidos por su canalización.
* Un registro de Docker en Nexus escuchando en el puerto 5000. OpenShift no conoce este endpoint adicional, por lo que debe crear una ruta adicional que exponga el registro docker de Nexus para su uso.
* Asegúrese de que la secuencia de comandos se realizó correctamente comprobando los códigos de retorno HTTP. Debería ver HTTP / 1.1 200 OK.


1.10. Crear un servicio para exponer el puerto 5000 del registro docker de `Nexus` y crear el route para acceder desde el exterior:

    $ oc expose dc nexus3 --port=5000 --name=nexus-registry
    $ oc create route edge nexus-registry --service=nexus-registry --port=5000

1.11. Confirmar que se ha creado el route:

    $ oc get routes -n ${GUID}-nexus
    NAME             HOST/PORT                                                 PATH   SERVICES         PORT       TERMINATION   WILDCARD
    nexus-registry   nexus-registry-lgp-nexus.apps.ocpdevmad01.tic1.intranet          nexus-registry   5000       edge          None
    nexus3           nexus3-lgp-nexus.apps.ocpdevmad01.tic1.intranet                  nexus3           8081-tcp                 None

1.12. Acceder a la consola de Nexus, logarse con el usuario "admin" y la contraseña obtenida en el paso `1.8`. Seguir las indicaciones para cambiar la password del usuario `admin` a `admin` y no activar acceso `Anonimo`.


## 2. Instalar y configurar Sonarqube

2.1. Crear un nuevo proyecto para el Sonarqube:

    $ oc new-project ${GUID}-sonarqube --display-name "${GUID} Shared Sonarqube"

2.2. Desplegar el sonarqube desde el `template` sonarqube-ephemeral:

    $ oc new-app sonarqube-ephemeral

2.3. Para ver el yaml del template (el template creará todos los objetos necesarios para la aplicacion Sonarqube, incluido el route):

    $ oc get template sonarqube-ephemeral -nopenshift -oyaml

2.4. Comprobar todos los objetos creados:

    $ oc get all

2.5. Pausar el despliegue para hacer cambios en el deployment config:

    $ oc rollout pause dc sonarqube

2.6. Configurar `requests` y `limits` de `cpu` y cambiar la estrategia de despliegue:

    $ oc set resources dc/sonarqube --limits=memory=3Gi,cpu=2 --requests=memory=2Gi,cpu=1
    $ oc patch dc sonarqube --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'

2.7. Configurar los valores para el `liveness` y `readiness` probe para asegurar el correcto funcionamiento del Sonarqube:

    $ oc set probe dc/sonarqube --liveness --failure-threshold 3 --initial-delay-seconds 40 -- echo ok
    $ oc set probe dc/sonarqube --readiness --failure-threshold 3 --initial-delay-seconds 20 --get-url=http://:9000/about

2.8. Reanudar el deployment del Sonarqube para que actualice todos los cambios:

    $ oc rollout resume dc sonarqube

2.9. Una vez que el SonarQube haya arrancado, iniciar sesión a través de la ruta expuesta. Credenciales `admin`/`admin`.


## 3. Crear repo en Git para la aplicacion "Tasks"

3.1. Crear un Repo en Github donde se alojara la aplicacion **Tasks**. En este ejemplo el repo que he creado es `https://github.com/lissettegar/openshift-tasks`.

3.2. Desde el directorio de trabajo:

    $ cd /home/<directorio de trabajo>

3.3. Clonarse el repo donde esta la app **Tasks** y añadir nuestro repo como remote (sustituir https://github.com/lissettegar/openshift-tasks.git por vuestro repo y `mirepo` por el nombre que querais asignarle):

    $ git clone https://github.com/redhat-gpte-devopsautomation/openshift-tasks.git
    $ cd /home/<directorio de trabajo>/openshift-tasks

    $ git remote add <mirepo> <https://github.com/lissettegar/openshift-tasks.git>
    $ git push -u <mirepo> master

3.4. En el repo que acabais de clonar hay 2 ficheros que hay que modificar (`nexus_settings.xml` y `nexus_openshift_settings.xml`). Cambiar la entrada `url` para que apunte a la URL externa del Nexus (deberia quedar como este ejemplo cambiando `lgp` por el GUID definido). Reemplazar el usuario y la password del Nexus por la que se hubiera definido en el punto `1.12` (admin/admin):

    <?xml version="1.0"?>
    <settings>
      <mirrors>
        <mirror>
          <id>Nexus</id>
          <name>Nexus Public Mirror</name>
          <url>http://nexus3-lgp-nexus.apps.ocpdevmad01.tic1.intranet/repository/maven-all-public/</url>
          <mirrorOf>*</mirrorOf>
        </mirror>
      </mirrors>
      <servers>
      <server>
        <id>nexus</id>
        <username>admin</username>
        <password>admin</password>
      </server>
    </servers>
    </settings>


3.5. Subir los cambios al repo que os habeis creado en el paso `3.1`:

    $ git commit -m "Updated Settings" nexus_settings.xml nexus_openshift_settings.xml
    $ git push <mirepo>  master

## 4. Instalar y configurar Jenkins

4.1. Cree un nuevo proyecto donde instalar `Jenkins`:

    $ oc new-project ${GUID}-jenkins --display-name "${GUID} Shared Jenkins"

4.2. Desplegar Jenkins y configurar el `request` y `limit` para la memoria y cpu:

    $ oc new-app jenkins-ephemeral --param ENABLE_OAUTH=true --param MEMORY_LIMIT=2Gi --param DISABLE_ADMINISTRATIVE_MONITORS=true
    $ oc set resources dc jenkins --limits=memory=2Gi,cpu=2 --requests=memory=1Gi,cpu=500m

4.3. Para crear un pod de agente de Jenkins personalizado, se creará un BuildConfig con estrategia `Docker` y fuente `Dockerfile`. Es mucho más fácil que construir la imagen del contenedor localmente y luego subirla al registro de OpenShift:

    $ oc new-build --strategy=docker -D $'FROM quay.io/openshift/origin-jenkins-agent-maven:4.1.0\n
       USER root\n
       RUN curl https://copr.fedorainfracloud.org/coprs/alsadi/dumb-init/repo/epel-7/alsadi-dumb-init-epel-7.repo -o /etc/yum.repos.d/alsadi-dumb-init-epel-7.repo && \ \n
       curl https://raw.githubusercontent.com/cloudrouter/centos-repo/master/CentOS-Base.repo -o /etc/yum.repos.d/CentOS-Base.repo && \ \n
       curl http://mirror.centos.org/centos-7/7/os/x86_64/RPM-GPG-KEY-CentOS-7 -o /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 && \ \n
       DISABLES="--disablerepo=rhel-server-extras --disablerepo=rhel-server --disablerepo=rhel-fast-datapath --disablerepo=rhel-server-optional --disablerepo=rhel-server-ose --disablerepo=rhel-server-rhscl" && \ \n
       yum $DISABLES -y --setopt=tsflags=nodocs install skopeo && yum clean all\n
       USER 1001' --name=jenkins-agent-appdev -n ${GUID}-jenkins

4.4. El BuildConfig creará la imagen para el agente de Jenkins donde correra nuestro pipeline:

    $ oc get is
    NAME                         IMAGE REPOSITORY                                                                                               TAGS     UPDATED
    jenkins-agent-appdev         default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-jenkins/jenkins-agent-appdev         latest   About a minute ago
    origin-jenkins-agent-maven   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-jenkins/origin-jenkins-agent-maven   4.1.0    2 minutes ago


## 5. Crear el proyecto de Desarrollo para la aplicacion Tasks

5.1. Crear el proyecto que contendra la versión de desarrollo de la aplicación openshift-tasks:

    $ oc new-project ${GUID}-tasks-dev --display-name "${GUID} Tasks Development"

5.2. Configurar los permisos para que Jenkins pueda manipular objetos en el proyecto ${GUID}-tasks-dev:

    $ oc policy add-role-to-user edit system:serviceaccount:${GUID}-jenkins:jenkins -n ${GUID}-tasks-dev

5.3. Crear un BuildConfig con fuente de tipo `binary` utilizando la imagen `jboss-eap71-openshift:1.4`:

    $ oc new-build --binary=true --name="tasks" jboss-eap71-openshift:1.4 -n ${GUID}-tasks-dev

5.4. Crear un deploymentconfig que apunte a la imagen `tasks:0.0-0`.

    $ oc new-app ${GUID}-tasks-dev/tasks:0.0-0 --name=tasks --allow-missing-imagestream-tags=true -n ${GUID}-tasks-dev

5.5. Comprobar que se han creado el deploymentconfig, el buildconfig y el imagestream:

    $ oc get all
    NAME                                       REVISION   DESIRED   CURRENT   TRIGGERED BY
    deploymentconfig.apps.openshift.io/tasks   0          1         0         config,image(tasks:0.0-0)

    NAME                                   TYPE     FROM     LATEST
    buildconfig.build.openshift.io/tasks   Source   Binary   0

    NAME                                   IMAGE REPOSITORY                                                                            TAGS    UPDATED
    imagestream.image.openshift.io/tasks   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-tasks-dev/tasks   0.0-0   

5.6. Deshabilitar la construcción y el despliegue automáticos del deploymentconfig:

    $ oc set triggers dc/tasks --remove-all -n ${GUID}-tasks-dev

5.7. Crear el `service` y el `route`:

    $ oc expose dc tasks --port 8080 -n ${GUID}-tasks-dev
    $ oc expose svc tasks -n ${GUID}-tasks-dev

5.8. Configurar probe readiness para la aplicacion:

    $ oc set probe dc/tasks -n ${GUID}-tasks-dev --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/

5.9. Crear un ConfigMap (que sera actualizado por el pipeline) y añadirlo al deploymentconfig:

    $ oc create configmap tasks-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n ${GUID}-tasks-dev
    $ oc set volume dc/tasks --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-config -n ${GUID}-tasks-dev
    $ oc set volume dc/tasks --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-config -n ${GUID}-tasks-dev

5.10. Comprobar todos los recursos que se han creado:

    $ oc get all
    NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    service/tasks   ClusterIP   172.30.127.26   <none>        8080/TCP   13m

    NAME                                       REVISION   DESIRED   CURRENT   TRIGGERED BY
    deploymentconfig.apps.openshift.io/tasks   0          1         0         

    NAME                                   TYPE     FROM     LATEST
    buildconfig.build.openshift.io/tasks   Source   Binary   0

    NAME                                   IMAGE REPOSITORY                                                                            TAGS    UPDATED
    imagestream.image.openshift.io/tasks   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-tasks-dev/tasks   0.0-0   

    NAME                             HOST/PORT                                            PATH   SERVICES   PORT   TERMINATION   WILDCARD
    route.route.openshift.io/tasks   tasks-lgp-tasks-dev.apps.ocpdevmad01.tic1.intranet          tasks      8080                 None

## 6. Crear el proyecto de produccion para la aplicacion Tasks

6.1. Crear el proyecto que contendra la versión de produccion de la aplicación openshift-tasks:

    $ oc new-project ${GUID}-tasks-prod --display-name "${GUID} Tasks Production"

6.2. Configurar los permisos para que Jenkins pueda manipular objetos en el proyecto ${GUID}-tasks-prod:

    $ oc policy add-role-to-group system:image-puller system:serviceaccounts:${GUID}-tasks-prod -n ${GUID}-tasks-dev
    $ oc policy add-role-to-user edit system:serviceaccount:${GUID}-jenkins:jenkins -n ${GUID}-tasks-prod

6.3. Crear un deploymentconfig tasks en el proyecto `production` a partir de la imagen del proyecto de `desarrollo`:

    $ oc new-app ${GUID}-tasks-dev/tasks:0.0 --name=tasks --allow-missing-imagestream-tags=true -n ${GUID}-tasks-prod

6.4. Comprobar que se han creado el deploymentconfig y el imagestream:

    $ oc get all
    NAME                                       REVISION   DESIRED   CURRENT   TRIGGERED BY
    deploymentconfig.apps.openshift.io/tasks   0          1         0         config,image(tasks:0.0)

    NAME                                   IMAGE REPOSITORY                                                                             TAGS   UPDATED
    imagestream.image.openshift.io/tasks   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-tasks-prod/tasks   0.0    

6.5. Deshabilitar la construcción y el despliegue automáticos del deploymentconfig:

    $ oc set triggers dc/tasks --remove-all -n ${GUID}-tasks-prod

6.6. Crear el `service` y el `route` para la aplicacion de produccion:

    $ oc expose dc tasks --port 8080 -n ${GUID}-tasks-prod
    $ oc expose svc tasks -n ${GUID}-tasks-prod

6.7. Configurar probe readiness para la aplicacion:

    $ oc set probe dc/tasks -n ${GUID}-tasks-prod --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8080/

6.8. Crear un ConfigMap (que sera actualizado por el pipeline) y añadirlo al deploymentconfig:

    $ oc create configmap tasks-config --from-literal="application-users.properties=Placeholder" --from-literal="application-roles.properties=Placeholder" -n ${GUID}-tasks-prod
    $ oc set volume dc/tasks --add --name=jboss-config --mount-path=/opt/eap/standalone/configuration/application-users.properties --sub-path=application-users.properties --configmap-name=tasks-config -n ${GUID}-tasks-prod
    $ oc set volume dc/tasks --add --name=jboss-config1 --mount-path=/opt/eap/standalone/configuration/application-roles.properties --sub-path=application-roles.properties --configmap-name=tasks-config -n ${GUID}-tasks-prod

6.9. Comprobar todos los recursos que se han creado:

    $ oc get all
    NAME            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
    service/tasks   ClusterIP   172.30.62.6   <none>        8080/TCP   65s

    NAME                                       REVISION   DESIRED   CURRENT   TRIGGERED BY
    deploymentconfig.apps.openshift.io/tasks   0          1         0         

    NAME                                   IMAGE REPOSITORY                                                                             TAGS   UPDATED
    imagestream.image.openshift.io/tasks   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-tasks-prod/tasks   0.0    

    NAME                             HOST/PORT                                             PATH   SERVICES   PORT   TERMINATION   WILDCARD
    route.route.openshift.io/tasks   tasks-lgp-tasks-prod.apps.ocpdevmad01.tic1.intranet          tasks      8080                 None

## 7. Preparar el Jenkinsfile y subirlo al Repo de Git de la aplicacion

7.1. En el repo de la formacion esta el fichero Jenkinsfile que se usara en este lab, path **<directorio de trabajo>/formacion-ocp-4.2/Lab-5**. Copiar el fichero **Jenkinsfile** al repo de la aplicacion que se creo en el paso `3` (ejemplo):

    $ cp <directorio de trabajo>/formacion-ocp-4.2/Lab-5/
    $ cp Jenkinsfile <directorio de trabajo>/opeshift-tasks/

7.2. Subir los cambios al repo:

    $ git add .;git commit -m "v`date '+%y%m%d'`";git push <mirepo> master

## 8. Crear un Build Config de tipo Pipeline

8.1. Modificar el fichero **buidconfig-task.yaml** (path <directorio de trabajo>/formacion-ocp-4.2/Lab-5) cambiando el namespace por el nombre del namespace de Jenkins de cada uno, cambiar el GUID y el nombre del repo de Git que se este usando (repo de la aplicacion Tasks):

* GUID: el valor de la variable que se este usando para definir los proyectos.
* REPO: La URL al repositorio de codigo donde esta el Jenkinsfile
* CLUSTER: La base de la URL del cluster (ya esta configurada para el entorno de desarrollo)


      kind: BuildConfig
      apiVersion: build.openshift.io/v1
      metadata:
        name: openshift-tasks
        namespace: lgp-jenkins
        labels:
          build: openshift-tasks
      spec:
        nodeSelector: {}
        output: {}
        resources: {}
        successfulBuildsHistoryLimit: 5
        failedBuildsHistoryLimit: 5
        strategy:
          type: JenkinsPipeline
          jenkinsPipelineStrategy:
            jenkinsfilePath: Jenkinsfile
            env:
              - name: CLUSTER
                value: api.ocpdevmad01.tic1.intranet
              - name: GUID
                value: lgp
              - name: REPO
                value: 'https://github.com/lissettegar/openshift-tasks.git'
        postCommit: {}
        source:
          type: Git
          git:
            uri: 'https://github.com/lissettegar/openshift-tasks.git'
            ref: master


8.2. Crear el buidconfig:

    $ oc create -f buidconfig-task.yaml

8.3. Comprobar en la consola de OpenShift que se ha creado el **BuildConfig** de tipo **Pipeline**:

![alt BuildConfig][imagen1]

[imagen1]: images/buildconfig.png

8.4. Acceder a Jenkins y comprobar que se ha creado el **pipeline**:

![alt Pipeline][imagen2]

[imagen2]: images/jenkins.png

8.5. Desde la consola de OpenShift hacer un Start Build:

![alt Startbuild][imagen3]

[imagen3]: images/startbuild.png

8.6. El progreso del pipeline se puede seguir desde varios sitios:

* Desde la consola de OpenShift Build -> Build details:

![alt Builddetail][imagen4]

[imagen4]: images/builddetail.png

* Desde el Jenkins:

![alt Jenkins-stages][imagen5]

[imagen5]: images/jenkins-stages.png

* Desde Open Blue Ocean:

![alt Blue-Ocean][imagen6]

[imagen6]: images/blue-ocean.png

* Desde la consola del pipeline en Jenkins (opcion **Console Output**):

![alt jenkins-console][imagen7]

[imagen7]: images/jenkins-console.png


8.7. Iran ejecutandose todos los stages hasta llegar al punto en que se quede esperando por la accion de alguien que promueba el cambio a produccion (ejemplo):

![alt promote][imagen8]

[imagen8]: images/promote.png

8.8. Antes de promover el cambio a produccion comprobar que la aplicacion `Tasks` esta corriendo en el proyecto de `desarrollo`. Comprobarlo por linea de comando y en el GUI. En este ejemplo es el pod **tasks-1-kf9mc**:

    $ oc get pods -n $GUID-tasks-dev
    NAME             READY   STATUS      RESTARTS   AGE
    tasks-1-build    0/1     Completed   0          23m
    tasks-1-deploy   0/1     Completed   0          22m
    tasks-1-kf9mc    1/1     Running     0          22m

8.9. Comprobar que se tiene acceso a la aplicacion del proyecto de desarrollo, en este ejemplo el route es http://tasks-lgp-tasks-dev.apps.ocpdevmad01.tic1.intranet/:

![alt Route1][imagen13]

[imagen13]: images/route1.png

8.10. Promover el cambio y comprobar que el pipeline termina correctamente:

* Desde la consola de OpenShift Build -> Build details:

![alt Buid-Finish][imagen9]

[imagen9]: images/finish1.png

* Desde el Jenkins:finish2

![alt Jenkins-Finish][imagen10]

[imagen10]: images/finish2.png

* Desde Open Blue Ocean:

![alt Blue-Ocean-Finish][imagen11]

[imagen11]: images/finish3.png

* Desde la consola del pipeline en Jenkins (opcion **Console Output**):

![alt Console-finish][imagen12]

[imagen12]: images/finish4.png

8.11. Comprobar que se tiene acceso a la aplicacion del proyecto de `produccion`, en este ejemplo el route es http://tasks-lgp-tasks-prod.apps.ocpdevmad01.tic1.intranet/:

![alt Route1][imagen14]

[imagen14]: images/route2.png

8.12. Acceder a la consola de `Nexus` para ver los ficheros que ha subido y de `Sonarqube` para ver el resultado de los tests.

## 9. Limpiar el entorno

    oc delete project ${GUID}-tasks-dev
    oc delete project ${GUID}-tasks-prod
    oc delete project ${GUID}-nexus
    oc delete project ${GUID}-sonarqube
    oc delete project ${GUID}-jenkins
