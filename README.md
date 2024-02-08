### MAILHEBIAU Damien - 4IRC

# TP1 - Docker

## 1-1 Document your database container essentials: commands and Dockerfile.

## Architecture finale de la 1ere partie
```sh
╰─ tree                                                   
.
├── compose.override.yml
├── compose.yml
├── database
│   ├── CreateScheme.sql
│   ├── Dockerfile
│   └── InsertData.sql
├── httpd
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
├── java
│   ├── Dockerfile
│   ├── Main.class
│   └── Main.java
├── readme.md
└── spring
    └── simple-api-student
        ├── Dockerfile
        ├── HELP.md
        ├── pom.xml
        ├── README.md
        └── src
            ├── main
            │   ├── java
            │   │   └── fr
			...
            │   └── resources
            │       └── application.yml
            └── test
             ...

28 directories, 32 files

```

### Dockerfile
```dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY ./*.sql /docker-entrypoint-initdb.d
 ```

Préférer mettre les scripts sql dans un dossier à part. Cela permet de rendre l'architecture plus propre et plus lisible.

Aussi, on peut *ne pas mettre* les variables d'environnement ici (particulièrement le password), et le(s) définir dans la commande `docker run` lorsqu'on démarre le conteneur.
### Commande pour lancer la base Postgres

Le port 5432 est mappé sur le 5433 de ma machine car j'ai deja un postgres qui tourne.

```sh
docker run -d -p 5433:5432 -v ./data:/var/lib/postgresql/data --name postgres --network tp1-network tp1/postgres
```
OU avec l'option -e pour définir les variables d'environnement
```sh
docker run -d -p 5433:5432 -v ./data:/var/lib/postgresql/data --name postgres --network tp1-network -e POSTGRES_PASSWORD=pwd tp1/postgres
```
### Commande pour lancer Adminer

```sh
docker run -d -p 9000:9000 --network tp1-network --name adminer adminer
```

Pour se connecter à la base via Adminer, il faut récupérer l'IP de notre postgres, car nous sommes bien sur 2 'machines' différentes dans le même sous réseau. On fait un `docker inspect postgres` et on trouve la ligne : `"IPAddress": "172.18.0.3",...`.
On utilise donc celle ci avec le port 5432 pour se connecter a la BDD.
Ou alors on peut utiliser le nom du container directement, car ils sont dans le même réseau docker.

![capture adminer](assets/adminer.png)

## 1-2 Why do we need a multistage build? And explain each step of this dockerfile.

On utilise un build multistage afin de découper notre build et notre run. Cela permet de faire des blocs réutilisables. Aussi, pour l'etape de build on n'a besoin que de maven pour compiler l'application, qui est plus leger que l'image complete amazon corretto.

```Dockerfile
# Build
# On utilise une image maven 3.8.6 que l'on nomme myapp-build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
# On définit la var d'environnement MYAPP_HOME afin de pouvoir la réutiliser plus tard
ENV MYAPP_HOME /opt/myapp
# On se place dans le repertoire définit précédemment
WORKDIR $MYAPP_HOME
# On copie notre pom.xml dans le repertoire courant
COPY pom.xml .
# Idem avec le src
COPY src ./src
# On run la commande mvn package pour créer notre package en skippant les tests
RUN mvn package -DskipTests

# Run
# On utilise une image amazon corretto version 17
FROM amazoncorretto:17
# On redéfinit la var d'environnement MYAPP_HOME afin de pouvoir la réutiliser plus tard (redéfini car on utilise une image différente de la premiere)
ENV MYAPP_HOME /opt/myapp
# On se place dans le repertoire défini précédemment
WORKDIR $MYAPP_HOME
# On copie notre app compilée, depuis le resultat du build précédent, vers la destination choisie
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

# On définit l'entrypoint pour lancer l'application au lancement du container.
ENTRYPOINT java -jar myapp.jar
```

## 1-3 Document docker-compose most important commands.
### docker compose 

`up` : crée et lance les instances du fichier compose.yml

`down` : eteint les containers du fichier compose.yml
- `-v or --volumes` : supprime les volumes associés

`logs` : affiche tous les logs des conteneurs du compose.yml
- `--tails 50` : affiche les 50 dernières lignes
- `--follow` : continue a s'éxécuter pour suivre le fil
- `--timestamps` : affiche les timestamps

`ls` : affiche les différentes stacks en cours d'éxécution : cette commande peut etre éxécutée depuis n'importe où, pas seulement aux endroits où il y a un fichier compose.yml

`-d` : lance en arrière plan

`--force-recreate` : force le fait de recréer les conteneurs meme s'ils sont déjà en route

`--build` : force le build s'il y a un build à effectuer


## 1-4 Document your docker-compose file.
Ajout d'un fichier `.env` pour garder toute les données sensibles dedans

On utilise un volume nommé pour la base de données, cela permet de ne pas perdre les données si le container est supprimé, mais également on laisse docker s'occuper de l'endroit où est stocké ce volume.

```yaml
version: '3.8'

services:
    api:
        container_name: api
        build: 
            context: spring/simple-api-student
            dockerfile: Dockerfile
        networks: 
          - tp1-network
        depends_on:
          - database
        env_file:
          - .env

    database:
        container_name: database
        build:
            context: database
            dockerfile: Dockerfile
        networks:
         - tp1-network
        env_file:
          - .env
        volumes:
          - data:/var/lib/postgresql/data

    web:
        container_name: web
        build:
            context: httpd
            dockerfile: Dockerfile
        ports: 
          - "8080:80"
        networks:
          - tp1-network
        depends_on:
          - api

networks:
    tp1-network:

volumes:
    data:
```


```sh
# .env file
POSTGRES_USER=usr
POSTGRES_PASSWORD=pwd
POSTGRES_DB=database
```

On peut du coup aussi utiliser les variables d'environnement dans le application.yml de l'api comme ceci :
```yaml
...
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://database:5432/${POSTGRES_DB:database}
    username: ${POSTGRES_USER:usr}
    password: ${POSTGRES_PASSWORD:pwd}
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops
```

PS : comme nous sommes dans un docker network, on peut utiliser le nom du container (**database**) au lieu de l'IP, cela evite de mettre une addresse en dur et de devoir la changer si docker change l'IP lors de la creation du conteneur

![capture docker ps](assets/dockerps.png)

## 1-5 Document your publication commands and published images in dockerhub.


```sh
# Build de l'image docker
docker build -t daddyornot/devops-web .                 
[+] Building 1.0s (9/9) FINISHED                                                                         docker:default
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 151B                                                                               0.0s
 => [internal] load metadata for docker.io/library/httpd:2.4                                                       0.8s
 => [auth] library/httpd:pull token for registry-1.docker.io                                                       0.0s
 => [1/3] FROM docker.io/library/httpd:2.4@sha256:bf3df534d25718ac5b206f6705ebd157f9ed5d62687766aa058556ed4b76002  0.0s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 62B                                                                                   0.0s
 => CACHED [2/3] COPY ./index.html /usr/local/apache2/htdocs/                                                      0.0s
 => CACHED [3/3] COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf                                              0.0s
 => exporting to image                                                                                             0.0s
 => => exporting layers                                                                                            0.0s
 => => writing image sha256:61c229d9492254751f475c4fe3cad02e397f06a811af65bc8e5716d5e88a864b                       0.0s
 => => naming to docker.io/library/devops-web                                                                      0.0s
```

```shell
# On tag l'image avec un nom de repository et une version
docker tag devops-web daddyornot/devops-web:1.0
# On push l'image sur docker hub
docker push daddyornot/devops-web:1.0          
The push refers to repository [docker.io/daddyornot/devops-web]
38a33e45fd7a: Pushed 
2b2e7cac014b: Pushed 
ab3a0403a0d9: Mounted from library/httpd 
40a428a249db: Mounted from library/httpd 
24bd64e09119: Mounted from library/httpd 
5f70bf18a086: Mounted from library/httpd 
c3147eaa9536: Mounted from library/httpd 
fb1bd2fc5282: Mounted from library/httpd 
1.0: digest: sha256:bc2d3674a80010b0c5f0990a4215c060937b144f2e8b39a5e41bb6baea1df482 size: 1987
```
Cela nous permettra à l'avenir de pouvoir pull l'image depuis n'importe quelle machine, et de pouvoir la lancer.

![capture docker hub](assets/hub.png)

--- 

# TP2 - GitHub Actions

## 2-1 What are testcontainers?

Testcontainers sont des librairies Java qui permettent de lancer des conteneurs Docker pour les tests. Cela permet de tester des applications qui ont besoin de dépendances externes (comme une base de données) sans avoir à les installer sur la machine de développement.

## 2-2 Document your Github Actions configurations.

![capture action success](assets/action-succeed.png)
![capture action success](assets/1-pipeline-success.png)

## Configuration

Ne pas oublier de renseigner les secrets dans les settings du repository sur GitHub : `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `SONAR_TOKEN`

![capture secrets](assets/docker-secrets.png)

### main.yml
```yaml
name: CI devops 2023
on:
  # triggered on push on main or develop branches
  push:
    branches: ["main", "develop"]
  pull_request:

jobs:
  # Job to build and test the backend
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Set up JDK 17
        uses: actions/setup-java@v3 
        with:
          java-version: '17'
          distribution: 'adopt'

      - name: Build and test with Maven
        # place where the code is located
        working-directory: ./simple-api-student
        # run the following commands to build, test, and analyze the code with SonarCloud
        run: |
          mvn clean install
          mvn -B verify sonar:sonar -Dsonar.projectKey=tp-devops-cpe-2024_simple-api -Dsonar.organization=tp-devops-cpe-2024 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./simple-api-student/pom.xml

  # Job to build and publish docker image
  build-and-push-docker-image:
   # run only when code is compiling and tests are passing
   needs: test-backend
   runs-on: ubuntu-22.04

   # steps to perform in job
   steps:
    - name: Checkout code
      uses: actions/checkout@v2.5.0

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build image and push backend
      uses: docker/build-push-action@v3
      with:
        # relative path to the place where source code with Dockerfile is located
        context: ./simple-api-student
        tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
        # This line allows to deploy the image only when the code is pushed to the main branch
        push: ${{ github.ref == 'refs/heads/main' }}

    - name: Build image and push database
      uses: docker/build-push-action@v3
      with:
        context: ./database
        tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-database:latest
        push: ${{ github.ref == 'refs/heads/main' }}

    - name: Build image and push httpd
      uses: docker/build-push-action@v3
      with:
        context: ./httpd
        tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-httpd:latest
        push: ${{ github.ref == 'refs/heads/main' }}
```

![full pipeline](assets/full-pipeline.png)

## 2-3 SonarCloud - Document your quality gate configuration. 

- Une fois le compte créé, il faut créer une organisation ainsi qu'un projet.
- Il nous faudra également créer un token afin de pouvoir accéder à l'API de SonarCloud.
- Il faut aussi renommer la branche master en main pour que cela soit bien intégré avec SonarCloud.
- Modifier les valeurs de `PROJECT_KEY` et `ORGANIZATION_KEY` par les réels.
Il faut ajouter cette commande lors du build and test afin de lancer l'analyse de SonarCloud :
```sh
mvn -B verify sonar:sonar -Dsonar.projectKey=PROJECT_KEY -Dsonar.organization=ORGANIZATION_KEY -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
```
![capture sonar](assets/quality-gate.png)

## Bonus - Splitted pipeline

On créé 2 fichiers yaml pour séparer les jobs de la pipeline, cela permet de mieux organiser le code et de le rendre plus lisible.

### test-backend.yml
Ce job sera lancé à chaque push sur les branches main et develop, ainsi qu'à chaque pull request. Il conditionnera le lancement du job suivant `build-deploy-docker-image`, s'il fail, il ne lancera pas le job `build-deploy-docker-image` mais `on-failure-echo`.

```yaml
name: Test Backend and Sonar Analysis
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: ["main", "develop"]
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - name: Checkout code
        uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3 
        with:
          java-version: '17'
          distribution: 'adopt'

     #finally build your app with the latest command
      - name: Build and test with Maven
        working-directory: ./simple-api-student
        run: |
              mvn clean install 
              mvn -B verify sonar:sonar -Dsonar.projectKey=tp-devops-cpe-2024_simple-api -Dsonar.organization=tp-devops-cpe-2024 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
```

### build-and-push-docker-image.yml
Ce job sera lancé à chaque fois que le job précédent (test-backend) se termine, grace à l'option workflow_run:completed. Aussi, grace au résultat de `github.event.workflow_run.conclusion == 'success'`, on peut lancer des jobs différents en fonction du résultat du job précédent.
```yaml
name: Build and Push to DockerHub
on:
  workflow_run:
    workflows: ["Test Backend and Sonar Analysis"]
    types: [completed]
    branches:
      - 'main'

jobs:
  build-deploy-docker-image:
    runs-on: ubuntu-22.04
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          context: ./simple-api-student
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-database:latest
          push: ${{ github.ref == 'refs/heads/main' }}

      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./httpd
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-httpd:latest
          push: ${{ github.ref == 'refs/heads/main' }}
  on-failure-echo:
    runs-on: ubuntu-22.04
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    steps:
      - run: echo 'Test Backend and Sonar Analysis has failed'
```

---
### Failed Pipeline

Ainsi, on peut voir sur un failed de test-backend, le job suivant se lance, mais ne fait pas les étapes liées a docker, il effectue seulement celle du on-failure
(Les noms des jobs ne sont pas les memes que dans mon fichier plus haut car j'ai mis leur nom à jour entre temps, mais rien n'a changé à part ce nom) :
![capture failed](assets/failed-test.png)
![capture failed](assets/on-failure.png)
![capture failed](assets/detail-on-failure.png)

--- 
### Success Pipeline

Et sur une pipeline réussie, on voit bien que le job on-success se lance, et effectue les étapes liées à docker (Les noms des jobs ne sont pas les memes que dans mon fichier plus haut car j'ai mis leur nom à jour entre temps, mais rien n'a changé à part ce nom) :

![capture success](assets/pipeline-success.png)
![capture success](assets/on-success.png)
![capture success](assets/detail-on-success.png)


# TP3 - Ansible 

## 3-1 Document your inventory and base commands
## Commandes principale ansible :
Sans le spécifier, ansible va chercher les hosts dans le fichier `/etc/ansible/hosts`
- `ansible all -m yum -a "name=httpd state=present" --private-key=../../../.ssh/id_rsa -u centos --become`

`all` : vise tous les hosts définis dans l'inventaire

`-m` : module à utiliser

- `name=httpd` : nom du package à installer

- `state=present` : permet de s'assurer que le package est installé, sinon l'installe, peut prendre la valeur `absent` pour le désinstaller

`-a` : arguments à passer au module

`--private-key` : clé privée à utiliser pour se connecter

`-u` : utilisateur à utiliser pour se connecter

`--become` : permet de passer en root pour lancer la commande

- `ansible all -m shell -a 'echo "<html><h1>Hello World</h1></html>" >> /var/www/html/index.html' --private-key=../../../.ssh/id_rsa -u centos --become`

`-m` : module à utiliser

  `shell` : permet de lancer une commande shell

`-a` : arguments à passer au module

  - `'echo "<html><h1>Hello World</h1></html>" >> /var/www/html/index.html'` : commande à lancer

## Inventory
```yaml
all:
  vars:
    # user to use for ssh
    ansible_user: centos
    # ssh key to use
    ansible_ssh_private_key_file: /home/damien/Documents/Ecole/s8/Devops/.ssh/id_rsa
  # hosts
  children:
    # hosts in 'prod' group
    prod:
      hosts: damien.mailhebiau.takima.cloud
```

Maintenant on peut spécifier l'inventaire à utiliser avec l'option `-i` :
```sh
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```
Grâce au module `setup`, on peut récupérer des informations sur les machines, comme par exemple ici, la distribution de l'OS.

```sh
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
damien.mailhebiau.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "7",
        "ansible_distribution_release": "Core",
        "ansible_distribution_version": "7.9",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```

## 3-2 Document your playbook

On se retrouve avec cette architecture :
```sh
.
├── inventories
│   └── setup.yml
├── playbook.yml
└── roles
    └── docker
        ├── handlers
        │   └── main.yml
        └── tasks
            ├── main.yml
```

### playbook.yml
Avec l'ajout d'un role docker, notre playbook s'est allégé, et est plus lisible, il ne fait plus que lancer le role docker.

```yaml
- hosts: all
  gather_facts: false
  become: true

  roles:
    - docker
```

Et c'est maintenant le role docker qui contient les tâches à effectuer : 

### roles/docker/tasks/main.yml
```yaml
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Install python3
  yum:
    name: python3
    state: present

- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

## Document your docker_container tasks configuration.

### playbook.yml

On lance notre playbook avec la commande : `ansible-playbook -i inventories/setup.yml playbook.yml --ask-vault-pass` en rentrant le mot de passe du vault précédemment défini.

Pour rappel, on encrypte nos variables sensibles avec la commande `ansible-vault encrypt_string 'valeur_a_encrypt' --name 'nom_de_variable'`

Sinon, pour encrypt un fichier entier, on utilise `ansible-vault encrypt file.yml`. Ou on peut aussi créer un fichier vaulté directement avec `ansible-vault create file.yml`. Que l'on place dans group_vars/all/ pour etre detecté par ansible.
On les rentre ensuite sous la forme suivante dans l'editeur : 
```yaml
mot_de_passe_bdd: "mot_de_passe_securise"
utilisateur_bdd: "utilisateur_secure"
```
Et on peut l'utiliser ainsi dans notre playbook sous la forme `{{ mot_de_passe_bdd }}`

```yaml
---
- hosts: all
  gather_facts: false
  become: true
  vars: 
    ansible_user: "centos"
    docker_network_name: "api-network"
    db_container_name: "database"
    api_container_name: "api"
    proxy_container_name: "web"
    front_container_name: "front"
  
  roles:
    - docker
    - network
    - database
    - app
    - proxy
    - front
```
### roles/network/tasks/main.yml
```yaml
---
- name: Create docker network
  docker_network:
    name: "{{ docker_network_name }}"
```

### roles/copys-env/tasks/main.yml
Ce role n'est plus utilisé, mais il permettait de copier un fichier .env sur la machine distante, et de le rendre accessible seulement par l'utilisateur root. Il était utilisé avant d'utiliser les variables d'environnement vaultée.
```yaml
---
- name: ensures {{ path_env_file }} dir exists
  file: 
    path: "{{ path_env_file }}"
    state: directory

- name: Copy env file to dest
  copy:
    src: .env
    dest: "{{ path_env_file }}" 
    mode: '400'
```
### roles/database/tasks/main.yml
```yaml
---
- name: Launch database
  docker_container:
    name: "{{ db_container_name }}"
    image: daddyornot/tp-devops-database
    state: started
    recreate: true
    pull: true
    networks:
      - name: "{{ docker_network_name }}"
    volumes:
      - data:/var/lib/postgresql/data
    env:
      POSTGRES_USER: "{{ POSTGRES_USER }}" 
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
      POSTGRES_DB: "{{ POSTGRES_DB }}"
```
### roles/app/tasks/main.yml
```yaml
---
- name: Launch api
  docker_container:
    name: "{{ api_container_name }}"
    image: daddyornot/tp-devops-simple-api
    state: started
    recreate: true
    pull: true
    networks:
      - name: "{{ docker_network_name }}"
    env:
      POSTGRES_USER: "{{ POSTGRES_USER }}" 
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
      POSTGRES_DB: "{{ POSTGRES_DB }}"
```
### roles/proxy/tasks/main.yml
```yaml
---
- name: Launch proxy
  docker_container:
    name: "{{ proxy_container_name }}"
    image: daddyornot/tp-devops-httpd
    state: started
    recreate: true
    pull: true
    networks:
      - name: "{{ docker_network_name }}"
    ports: 
      - "80:80"
      - "8080:8080"
```
### roles/front/tasks/main.yml
```yaml
---
- name: Launch front
  docker_container:
    name: "{{ front_container_name }}"
    image: daddyornot/tp-devops-front
    state: started
    recreate: true
    pull: true
    networks:
      - name: "{{ docker_network_name }}"
```

### httpd.conf
En n'ouvrant que des ports sur le conteneur httpd, on peut rediriger les requêtes vers les autres conteneurs, et ainsi ne pas exposer directement les autres conteneurs sur le réseau.

```apache
# toutes les requetes arrivant sur le port 80 redirigeront vers le port 80 du container front
<VirtualHost *:80>
  ProxyPreserveHost On
  ProxyPass / http://front:80/
  ProxyPassReverse / http://front:80/
</VirtualHost>

# toutes les requetes arrivant sur le port 8080 redirigeront vers le port 8080 du container api
<VirtualHost *:8080>
  ProxyPreserveHost On
  ProxyPass / http://api:8080/
  ProxyPassReverse / http://api:8080/
</VirtualHost>

# On rajoute également Listen sur le port 8080 afin d'écouter les requetes sur ce port en plus du 80 défini plus haut
Listen 8080
```

### Workflow Github Action
Ce workflow permet de lancer le playbook ansible à chaque fois que le workflow précédent se termine, et que le résultat est un succès. 
Il est necessaire de créer 3 nouveaux secrets dans les settings du repository : `SSH_PRIVATE_KEY`, `ANSIBLE_INVENTORY`, `VAULT_PASSWORD`

```yaml
name: Deploy with Ansible to aws
on:
  workflow_run:
    workflows: ["Build and Push to DockerHub"]
    types: [completed]
    branches:
      - 'main'

jobs:
  ansible-deploy:
    runs-on: ubuntu-22.04
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Run deploy playbook
        uses: dawidd6/action-ansible-playbook@v2
        with:
        # Required, playbook filepath
          playbook: playbook.yml
          # Optional, directory where playbooks live
          directory: ./ansible
          # Optional, ansible configuration file content (ansible.cfg)
          # Sans cette ligne, cela ne fonctionnait pas meme avec le ansible_user défini dans le playbook...
          configuration: |
            [defaults]
            ansible_user=centos
          # Optional, SSH private key
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          # Optional, literal inventory file contents
          # inventory: |
          #   [all]
          #   damien.mailhebiau.takima.cloud
          inventory: ${{ secrets.ANSIBLE_INVENTORY }}
          # Optional, encrypted vault password
          vault_password: ${{ secrets.VAULT_PASSWORD }} 
```


### Finally
![capture website](assets/working-server.png)

