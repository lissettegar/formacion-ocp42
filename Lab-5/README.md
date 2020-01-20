## 1. Instalar y configurar Nexus

1.1. Exportar variable de entorno:

    $ export GUID=<iniciales en minuscula>

1.2. Cree un nuevo proyecto donde instalar Nexus:

    $ oc new-project ${GUID}-nexus --display-name "${GUID} Shared Nexus"

1.3. Desplegar Nexus y crear el route para exponer el servicio:

    $ oc new-app sonatype/nexus3:latest
    $ oc expose svc nexus3

1.4. Pausar el despliegue para hacer cambios en el deployment config:

    $ oc rollout pause dc nexus3

1.5. Cambiar la estrategia de deployment de `Rolling` a `Recreate` y configurar `requests` y `limits` para la `memoria`:

    $ oc patch dc nexus3 --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'
    $ oc set resources dc nexus3 --limits=memory=2Gi,cpu=2 --requests=memory=1Gi,cpu=500m

1.6. Configurar los valores para el `liveness` y `readiness` probe para asegurar el correcto funcionamiento del Nexus:

    $ oc set probe dc/nexus3 --liveness --failure-threshold 3 --initial-delay-seconds 60 -- echo ok
    $ oc set probe dc/nexus3 --readiness --failure-threshold 3 --initial-delay-seconds 60 --get-url=http://:8081/

1.7. Reanudar el deployment del Nexus deployment configuration para que actualice todos los cambios:

    $ oc rollout resume dc nexus3

1.8. Una vez que Nexus este desplegado, configurar el repositorio de Nexus usando el script proporcionado (setup_nexus3.sh). Usar el usuario por defecto (admin). La password por defecto de `admin` se guarda en el pod de Nexus en el fichero `/nexus-data/admin.password`. Usar el comando `oc rsh` para obtener dicha password:

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

1.9. Este script crea:

* Algunos repositorios proxy de Maven para almacenar en caché las dependencias de Red Hat y JBoss.
* Un repositorio de grupo maven-all-public que contiene los repositorios proxy para todos los artefactos necesarios.
* Un repositorio proxy NPM para almacenar en caché los artefactos de compilación Node.JS.
* Un registro privado de Docker.
* Un repositorio de versiones para los archivos WAR producidos por su canalización.
* Un registro de Docker en Nexus escuchando en el puerto 5000. OpenShift no conoce este endpoint adicional, por lo que debe crear una ruta adicional que exponga el registro docker de Nexus para su uso.
* Asegúrese de que la secuencia de comandos se realizó correctamente comprobando los códigos de retorno HTTP. Debería ver HTTP / 1.1 200 OK.


1.10. Crear un servicio para exponer el puerto 5000 del registro docker de Nexus y crear el route para acceder desde el exterior:

    $ oc expose dc nexus3 --port=5000 --name=nexus-registry
    $ oc create route edge nexus-registry --service=nexus-registry --port=5000

1.11. Confirmar que se ha creado:

    $ oc get routes -n ${GUID}-nexus
    NAME             HOST/PORT                                                 PATH   SERVICES         PORT       TERMINATION   WILDCARD
    nexus-registry   nexus-registry-lgp-nexus.apps.ocpdevmad01.tic1.intranet          nexus-registry   5000       edge          None
    nexus3           nexus3-lgp-nexus.apps.ocpdevmad01.tic1.intranet                  nexus3           8081-tcp                 None

1.12. Acceder al Nexus y seguir las indicaciones para cambiar la password. Desde un navegador acceder a la URL del Nexus que muestra el comando anterior, en este ejemplo es:

*nexus3-lgp-nexus.apps.ocpdevmad01.tic1.intranet*

## 2. Instalar y configurar Sonarqube

2.1. Crear un nuevo proyecto para el Sonarqube:

    $ oc new-project ${GUID}-sonarqube --display-name "${GUID} Shared Sonarqube"

2.2. Desplegar el un PostgreSQL Database para el Sonarqube:

    $ oc new-app --template=postgresql-ephemeral --param POSTGRESQL_USER=sonar --param POSTGRESQL_PASSWORD=sonar --param POSTGRESQL_DATABASE=sonar --labels=app=sonarqube_db

2.3. Asegurarse que la ddbb esta desplegada antes de continuar con el siguiente paso:

    $ oc get all
    NAME                      READY   STATUS      RESTARTS   AGE
    pod/postgresql-1-7r7mk    1/1     Running     0          77s
    pod/postgresql-1-deploy   0/1     Completed   0          85s

    NAME                                 DESIRED   CURRENT   READY   AGE
    replicationcontroller/postgresql-1   1         1         1       85s

    NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
    service/postgresql   ClusterIP   172.30.104.252   <none>        5432/TCP   86s

    NAME                                            REVISION   DESIRED   CURRENT   TRIGGERED BY
    deploymentconfig.apps.openshift.io/postgresql   1          1         1         config,image(postgresql:10)

2.4. Desplegar el Sonarqube desde la imagen `wkulhanek/sonarqube:6.7.6` disponible en DockerHub. Esta imagen espera las siguientes variables de entorno: SONARQUBE_JDBC_USERNAME, SONARQUBE_JDBC_PASSWORD, y SONARQUBE_JDBC_URL. Crear un route:

    $ oc new-app --docker-image=wkulhanek/sonarqube:6.7.6 --env=SONARQUBE_JDBC_USERNAME=sonar --env=SONARQUBE_JDBC_PASSWORD=sonar --env=SONARQUBE_JDBC_URL=jdbc:postgresql://postgresql/sonar --labels=app=sonarqube
    $ oc expose service sonarqube

2.5. Pausar el despliegue para hacer cambios en el deployment config:

    $ oc rollout pause dc sonarqube

2.6. Cambiar la estrategia de deployment de `Rolling` a `Recreate` y configurar `requests` y `limits` para la `memoria`:

    $ oc set resources dc/sonarqube --limits=memory=3Gi,cpu=2 --requests=memory=2Gi,cpu=1
    $ oc patch dc sonarqube --patch='{ "spec": { "strategy": { "type": "Recreate" }}}'

2.7. Configurar los valores para el `liveness` y `readiness` probe para asegurar el correcto funcionamiento del Sonarqube:

    $ oc set probe dc/sonarqube --liveness --failure-threshold 3 --initial-delay-seconds 40 -- echo ok
    $ oc set probe dc/sonarqube --readiness --failure-threshold 3 --initial-delay-seconds 20 --get-url=http://:9000/about

2.8. Reanudar el deployment del Sonarqube deployment configuration para que actualice todos los cambios:

    $ oc rollout resume dc sonarqube

2.9. Una vez que el SonarQube haya arrancado completamente, inicie sesión a través de la ruta expuesta. Credenciales `admin` y la contraseña es `admin`.


## 3. Crear repo en Git para la aplicacion "Tasks"

3.1. Crear un Repo en Github donde se alojara la aplicacion **Tasks**. En este ejemplo el repo que he creado es `https://github.com/lissettegar/openshift-tasks`.

3.2. Desde el directorio de trabajo:

    $ cd /home/<directorio de trabajo>

3.3. Clonarse el repo donde esta la app **Tasks** y añadir nuestro repo como remote:

    $ git clone https://github.com/redhat-gpte-devopsautomation/openshift-tasks.git
    $ cd /home/<directorio de trabajo>/openshift-tasks

    $ git remote add mirepo https://github.com/lissettegar/openshift-tasks.git
    $ git push -u mirepo master

3.4. Configurar el fichero `nexus_settings.xml` para builds locales, asegurandp que la  <url> apunta a la URL externa del Nexus (deberia quedar como este ejemplo cambiando `lgp` por el GUID definido). Reemplazar el usuario y la password del Nexus por la que se hubiera definido en el punto 1.12:

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

3.5. Cambiar tambien el contenido del fichero `nexus_openshift_settings.xml` para que apunte a la URL interna del Nexus (deberia quedar como este ejemplo cambiando `lgp` por el GUID definido). Reemplazar el usuario y la password del Nexus por la que se hubiera definido en el punto 1.12:

    <?xml version="1.0"?>
    <settings>
      <mirrors>
        <mirror>
          <id>Nexus</id>
          <name>Nexus Public Mirror</name>
          <url>http://nexus3.lgp-nexus.svc.cluster.local:8081/repository/maven-all-public/</url>
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

3.6. Subir los cambios al repo que os habeis creado en el paso 3.1:

    $ git commit -m "Updated Settings" nexus_settings.xml nexus_openshift_settings.xml
    $ git push mirepo  master

## 4. Instalar y configurar Jenkins

4.1. Cree un nuevo proyecto donde instalar Jenkins:

    $ oc new-project ${GUID}-jenkins --display-name "${GUID} Shared Jenkins"

4.2. Desplegar Jenkins y configurar el `request` y `limit` para la memoria y cpu:

    $ oc new-app jenkins-ephemeral --param ENABLE_OAUTH=true --param MEMORY_LIMIT=2Gi --param DISABLE_ADMINISTRATIVE_MONITORS=true
    $ oc set resources dc jenkins --limits=memory=2Gi,cpu=2 --requests=memory=1Gi,cpu=500m

4.3. Para crear un pod de agente de Jenkins personalizado, se usara un build con la estrategia Docker. Es mucho más fácil que construir la imagen del contenedor localmente y luego subirla al registro de OpenShift:

    $ oc new-build --strategy=docker -D $'FROM quay.io/openshift/origin-jenkins-agent-maven:4.1.0\n
       USER root\n
       RUN curl https://copr.fedorainfracloud.org/coprs/alsadi/dumb-init/repo/epel-7/alsadi-dumb-init-epel-7.repo -o /etc/yum.repos.d/alsadi-dumb-init-epel-7.repo && \ \n
       curl https://raw.githubusercontent.com/cloudrouter/centos-repo/master/CentOS-Base.repo -o /etc/yum.repos.d/CentOS-Base.repo && \ \n
       curl http://mirror.centos.org/centos-7/7/os/x86_64/RPM-GPG-KEY-CentOS-7 -o /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7 && \ \n
       DISABLES="--disablerepo=rhel-server-extras --disablerepo=rhel-server --disablerepo=rhel-fast-datapath --disablerepo=rhel-server-optional --disablerepo=rhel-server-ose --disablerepo=rhel-server-rhscl" && \ \n
       yum $DISABLES -y --setopt=tsflags=nodocs install skopeo && yum clean all\n
       USER 1001' --name=jenkins-agent-appdev -n ${GUID}-jenkins

4.4. Este comando creará la imagen y la cargará automáticamente en el registro interno del clúster. También creará un ImageStream. Comprobarlo:

    $ oc get is
    NAME                         IMAGE REPOSITORY                                                                                               TAGS     UPDATED
    jenkins-agent-appdev         default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-jenkins/jenkins-agent-appdev         latest   About a minute ago
    origin-jenkins-agent-maven   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-jenkins/origin-jenkins-agent-maven   4.1.0    2 minutes ago


## 5. Crear el proyecto de desarrollo para la aplicacion

5.1. Crear el proyecto que contendra la versión de desarrollo de la aplicación openshift-tasks:

    $ oc new-project ${GUID}-tasks-dev --display-name "${GUID} Tasks Development"

5.2. Configurar los permisos para que Jenkins pueda manipular objetos en el proyecto ${GUID}-tasks-dev:

    $ oc policy add-role-to-user edit system:serviceaccount:${GUID}-jenkins:jenkins -n ${GUID}-tasks-dev

5.3. Crear un BuildConfig con fuente de tipo `binary` utilizando la imagen `jboss-eap71-openshift:1.4`:

    $ oc new-build --binary=true --name="tasks" jboss-eap71-openshift:1.4 -n ${GUID}-tasks-dev

5.4. Crear un deploymentconfig que apunte a la imagen `tasks:0.0-0`.

    $ oc new-app ${GUID}-tasks-dev/tasks:0.0-0 --name=tasks --allow-missing-imagestream-tags=true -n ${GUID}-tasks-dev

5.5. Comprobar que se ha creado el BuildConfig y el imagestream:

    $ oc get all
    NAME                                   TYPE     FROM     LATEST
    buildconfig.build.openshift.io/tasks   Source   Binary   0

    NAME                                   IMAGE REPOSITORY                                                                            TAGS   UPDATED
    imagestream.image.openshift.io/tasks   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-tasks-dev/tasks          


5.6. Deshabilitar la construcción y el despliegue automáticos del deploymentconfig:

    $ oc set triggers dc/tasks --remove-all -n ${GUID}-tasks-dev

5.7. Exponer el deploymentconfig como un service (en el puerto 8080) y el service como un route:

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

## 6. Crear el proyecto de produccion para la aplicacion

6.1. Crear el proyecto que contendra la versión de produccion de la aplicación openshift-tasks:

    $ oc new-project ${GUID}-tasks-prod --display-name "${GUID} Tasks Production"

6.2. Configurar los permisos para que Jenkins pueda manipular objetos en el proyecto ${GUID}-tasks-prod:

    $ oc policy add-role-to-group system:image-puller system:serviceaccounts:${GUID}-tasks-prod -n ${GUID}-tasks-dev
    $ oc policy add-role-to-user edit system:serviceaccount:${GUID}-jenkins:jenkins -n ${GUID}-tasks-prod

6.3. Crear un deploymentconfig tasks en el proyecto production a partir de la imagen del proyecto de desarrollo:

    $ oc new-app ${GUID}-tasks-dev/tasks:0.0 --name=tasks --allow-missing-imagestream-tags=true -n ${GUID}-tasks-prod

6.4. Comprobar que se han creado el deploymentconfig y el imagestream:

    $ oc get all
    NAME                                       REVISION   DESIRED   CURRENT   TRIGGERED BY
    deploymentconfig.apps.openshift.io/tasks   0          1         0         config,image(tasks:0.0)

    NAME                                   IMAGE REPOSITORY                                                                             TAGS   UPDATED
    imagestream.image.openshift.io/tasks   default-route-openshift-image-registry.apps.ocpdevmad01.tic1.intranet/lgp-tasks-prod/tasks   0.0    

6.5. Deshabilitar la construcción y el despliegue automáticos del deploymentconfig:

    $ oc set triggers dc/tasks --remove-all -n ${GUID}-tasks-prod

6.6.. Exponer el deploymentconfig como un service (en el puerto 8080) y el service como un route:

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

## 7. Preparar el Jenkinsfile y copiarlo en el Repo de Git de la aplicacion

7.1. Crear el directorio
7.1. En el repo de la formacion esta el Jenkinsfile que se usa en este lab, path **<directorio de trabajo>/formacion-ocp-4.2/Lab-5**. Copiar el fichero **Jenkinsfile** al repo de la aplicacion que creamos en el paso 3 (ejemplo):

$ cp <directorio de trabajo>/formacion-ocp-4.2/Lab-5/




Crear un Build Config que apunte al pipeline en el repositorio de codigo, definiendo como `contextDir openshift-tasks`, y las siguientes variables:

GUID: el valor de la variable que se este usando para definir los proyectos.

REPO: La URL al repositorio de codigo donde esta el Jenkinsfile

CLUSTER: La base de la URL del cluster


    kind: BuildConfig
    apiVersion: build.openshift.io/v1
    metadata:
      name: openShift-tasks
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
        contextDir: openshift-tasks
      triggers:
        - type: GitHub
        - type: Generic
        - type: ConfigChange
