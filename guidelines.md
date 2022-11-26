# Guideline from scratch

I) Install tool:
---
- native-image

  sudo bash <(curl -sL https://get.graalvm.org/jdk) graalvm-ee-java17-22.3.0 (upgrade all graalvm platform)

  export PATH=$PATH:/home/opc/soft/graalvm-ee-java17-22.3.0/bin

- bat

  wget -O bat.zip https://github.com/sharkdp/bat/releases/download/v0.8.0/bat-v0.8.0-x86_64-unknown-linux-musl.tar.gz

  tar -xvzf bat.zip -C $HOME

  sudo mv bat-v0.8.0-x86_64-unknown-linux-musl/ /usr/local/bat

  sudo alias bat='/usr/local/bat/bat'

- docker-compose

  sudo curl -SL https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
  
  sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
  
  sudo chmod ugo+x /usr/bin/docker-compose 
  
- termgraph

  scl enable rh-python38 bash
  
  python3 -m pip install termgraph

II) Step0 (container + jar)
---
(Slides 1-4)
1. JAR File

    git checkout step0

    bat src/.....

    mvn clean package

    java -jar target/jibber-0.0.1-SNAPSHOT-exec.jar

    curlie http://localhost:8080/jibber 

2. Container 

    bat Dockerfiles/Dockerfile.jdk

    docker build -t localhost/jibber:jdk.0.1 -f ./Dockerfiles/Dockerfile.jdk --build-arg JAR_FILE=jibber-0.0.1-SNAPSHOT-exec.jar .

    docker-compose up -d(deattached)

    ./sizes.sh

    ./startups.sh (faststartup ms)

    ./stress.sh (2) (latency, tps, throuput)

    docker-compose stop

II) Step1 (Native image) 
---
1. native-image

    git checkout step1

    bat pom.xml (check profile=<native> mvn plugins)

    mvn package -DskipTests -Pnative

    ./target/jibber

    curlie http://localhost:8080/jibber 

    ldd ./target/jibber (ldd (List Dynamic Dependencies) is a *nix utility that prints the shared libraries required by each program or shared library specified on the command line)
    objdump -d target/jibber (miremos adentro de este ejecutable que objetos tiene, machine code, wow instrucciones de assembler, bien)

    (Slides 5-13)

2. Container
  
    bat Dockerfiles/Dockerfile.native (OL slim version)

    docker build -t localhost/jibber:native.0.1 -f ./Dockerfiles/Dockerfile.native --build-arg APP_FILE=jibber .

    docker-compose up -d

    ./sizes.sh 

    ./startups.sh (significativamente mas veloz en el arranque)

    ./stress.sh (2..3 veces) (el JVM container hace profiling y puede mejorar hasta cierto punto, mientras que la NI ya es un ejecutable inmutable y nos dara las mismas metricas)

    pero podemos mejorar la performance del native image?

III) Step2 (native image GC1)
---
G1 feature solo esta disponible en la EE. Es un GC multi-thread que está optimizado para reducir los thread es estado waiting, por lo tanto, mejora la latencia y al mismo tiempo logra un alto rendimiento rendimiento.

    git checkout step2

    bat pom.xml

    mvn package -DskipTests -Pnativeg1

    docker build -t localhost/jibber:nativeg1.0.1 -f ./Dockerfiles/Dockerfile.native --build-arg APP_FILE=jibber-g1 .

    docker-compose up -d

    ./sizes.sh

    ./startups.sh

    ./stress.sh (2 o 3 veces)

    docker-compose stop

IV) Step3 (base container distroiless + +Static param)
---
    (Slide 14)

    git checkout step3

    bat pom.xml

    bat Dockerfiles/Dockerfile.distroless

    mvn package -DskipTests -Pdistroless

    docker build -t localhost/jibber:distroless.0.1 -f ./Dockerfiles/Dockerfile.distroless --build-arg APP_FILE=jibber-distroless .

    docker images | grep localhost

    docker-compose up -d

    ./sizes.sh

    ./startups.sh

    (con el distroiless solo hemos optimizado el footprint y arranque, la imagen native sigue con la misma configuracin de built que el anterior)

V) PGO (feature) - Profile-guided optimizations, que nos permite agregar mayor perf aun y aumentar el throuput. Graalvm EE only
---
(Slide 15)
  
  git checkout pgo

  bat pom.xml

  jibber-pgoinst (hace un profile instrumentado del posible rendimiento de la app, y luego de ello genera una NI tuneado o instrumentado)
  jibber-pgo (despues de hacer todo el profile y la instrumentacion el punto mas alto de rendimiento, y cuando corra la aplicacion anterior se le indica una ruta por default
   donde guardar el profile information)

  mvn package -DskipTests -Ppgoinst

  docker build -t localhost/jibber:pgoinst.0.1 -f ./Dockerfiles/Dockerfile.distroless --build-arg APP_FILE=jibber-pgoinst .

  ./target/jibber-pgoinst

  curlie http://localhost:8080/jibber

  (debes parar el servicio para que genere el default.iprof)
 
  mvn package -DskipTests -Ppgo

  docker build -t localhost/jibber:pgo.0.1 -f ./Dockerfiles/Dockerfile.distroless --build-arg APP_FILE=jibber-pgo . 


--JLink--You can use the jlink tool to assemble and optimize a set of modules and their dependencies into a custom runtime image.

  rm -rf ./target/min-jre

  jlink --add-modules java.base,java.logging,java.desktop,jdk.httpserver,java.management,java.naming,java.security.jgss,java.instrument \
          --output ./target/min-jre

  mvn package -DskipTests

  docker build -t localhost/jibber:jlink.0.1 -f ./Dockerfiles/Dockerfile.jlink --build-arg JAR_FILE=jibber-0.0.1-SNAPSHOT-exec.jar .

  docker-compose up -d

  ./sizes.sh

  ./startups.sh

  ./stress.sh

  ./prom.sh (analizar lo que sale en prometheus consumo de memoria)

  (Slides 16-17)

















