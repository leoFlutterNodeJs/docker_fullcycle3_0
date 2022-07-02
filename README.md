# Curso Docker Full Cycle 3.0

# Hello World Docker
  ### Roda e executa uma imagem
    docker run hello-world
  ### Ver container rodando
    docker ps
  ### Container criados mas sem rodar
    docker ps -a
  ### Apagando um container
    docker rm CONTAINER_ID
  ### Mostrando images
    docker images
  ### Apagando images 
    docker rmi IMAGE_ID
  ### Apagando todos os container
    docker rm $(docker ps -a -q) -f

# Gerenciamento básico de container
  ### Instalando o nginx lastet
    docker run nginx
  ### Rodando processo em background
    docker run -d nginx
  ### Parando um container
    docker stop CONTAINER_ID
  ### Iniciando um container
    docker start CONTAINER_ID
  ### Removendo um container rodando 
    docker rm CONTAINER_ID -f

# Expondo Portas e Nomeando Containers
  ### Nomeando um container my_nginx
    docker run -d --name my_nginx nginx:alpine
  ### Parando um container usando o nome
    docker stop my_nginx
  ### Iniciando um container usando o nome 
    docker start my_nginx
  ### Expondo porta (meu_pc_port:meu_container_port = 8080:80)
    docker run -d --name expond_port -p 8080:80 nginx:alpine
      # => Acesse http://localhost:8080
  ### Conectando a porta 80 do meu_pc com a port 80 do container
    docker run -d --name expond_port_80 -p 80:80 nginx:alpine
      # => Acesse http://localhost/

# Executando comandos no container
  ### Exibindo o container
    docker exec my_nginx uname -a
  ### Executando o bash
    docker exec my_nginx bash
  ### Interagindo com o bash
    docker exec -it my_nginx bash

# Iniciando com volumes
  ### Básico - Criando um volumes
    docker run -d --name my_nginx -p 8080:80 -v $(pwd):/usr/share/nginx/html nginx
  ### Listando volume
    docker volume ls
  ### Criando um volume
    docker volume create vol_test
  ### Descobrindo comandos
    docker volume --help
  ### Inspecionando volume
    docker volume inspect vol_test
  ### Criando volume apontando o diretório
    docker volume create --driver local --opt type=none --opt device=$(pwd) --opt o=bind volume_name
  ### Usando o volume criado em um novo container
    docker run -d --name nginx2 -p 8081:80 -v volume_name:/usr/share/nginx/html nginx
  ### Matando volume desatachado
    docker volume prune

# Iniciando Networks
  ### Listando Network
    docker network ls
  ## Crie 2 containers e tenta dar um ping
  ### Inspecionando o network - mostra os containers da rede
    docker network inspect bridge
  ### Criando uma network
    docker network create -d bridge my_network
  ## Fazendo o ping entre serviços com o nome
  ### Criando container linkando com a network
    docker run -d --name rede1 --net=my_network nginx
    docker run -d --name rede2 --net=my_network nginx

# Docker Commit
    docker commit CONTAINER_ID NOME_DA_IMAGE(leocardosodev/nginx-image)
  ## Docker push
  ### Primeiro docker commit, depois docker login
    docker push NOME_DA_IMAGE(exemplo: leocardosodev/nginx-image)

# Trabalhando com Dockerfile
  ## Criando Dockerfile
    FROM php:7.3.6-fpm-alpine3.9
    RUN apk add --no-cache shadow
    WORKDIR /var/www
    RUN rm -rf /var/www/html 
    COPY . /var/www
    RUN ln -s public html
    RUN usermod -u 1000 www-data
    USER www-data
    EXPOSE 9000
    ENTRYPOINT ["php-fpm"]
### Fazendo o build usando o Dockerfile
    docker build -t NOME_CONTAINER . (local do Dockefile)
### Criando um container baseado na image criada no Dockerfile
    docker run -d --name swoole -p 9501:9501 test_swoole
### Tag para push no Docker Hub
    docker build -t leocardosodev/php-swoole:latest .
### subindo a imagem para o DockerHub
    docker push leocardosodev/php-swoole:latest

# Dockerfile criando image laravel
  ## Criando Dockerfile com laravel
    FROM php:7.3.6-fpm-alpine3.9
    WORKDIR /var/www
    RUN rm -rf /var/www/html
    COPY . /var/www 
    RUN ln -s public html
    EXPOSE 9000
    ENTRYPOINT ["php-fpm"]
  ### Depois de criar o Dockerfile gerar ou atualizar(build) a imagem
    docker build -t leocardosodev/laravel .
  ### Criando container espelhado com volumes
    docker run -d --name laravel -v $(pwd):/var/www -p 9000:9000 leocardosodev/laravel
  ### Ñ precisa mais do -v pois já foi copiado dentro do Dockfile
    docker run -d --name laravel -p 8000:8000 leocardosodev/laravel
  ### Instalando bash (alpine ñ tem bash - )
    docker exec -u root -it laravel apk add bash
  ### Acessando o container recém criado
    docker exec -u root -it laravel bash
    cd /var/www php artisan serve --host=0.0.0.0
  ### Subindo a imagem com laravel para o DockerHub
    docker push leocardosodev/laravel

# Iniciando com docker-compose
  ## Criando um projeto Laravel
    composer create-project --prefer-dist laravel/laravel laravel "5.8.*"
    cd laravel
  ### => Cria a pasta .docker/ para fazer o Dockerfile de cada serviço
  #### .docker/nginx Dockerfile
    FROM nginx:1.15.0-alpine
    RUN rm /etc/nginx/conf.d/default.conf
    COPY ./nginx.conf /etc/nginx/conf.d
  ### .docker/mysql Dockerfile
    FROM mysql:5.7
    RUN usermod -u 1000 mysql
  #### Dockerfile na raiz
    FROM php:7.3.6-fpm-alpine3.9
    RUN apk add --no-cache shadow \ 
        && apk add bash mysql-client \ 
        && docker-php-ext-install pdo pdo_mysql
    WORKDIR /var/www
    RUN rm -rf /var/www/html 
    RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    # RUN composer install && \
    #   cp .env.exemple .env && \ 
    #   php artisan key:generate && \ 
    #   php artisan config:cache
    # COPY . /var/www
    RUN ln -s public html
    RUN usermod -u 1000 www-data
    USER www-data
    EXPOSE 9000
    ENTRYPOINT ["php-fpm"]
  #### docker-compose - criando os serviços
    version: '3'

    services:
      app:
        build: .
        container_name: app
        volumes: 
          - .:/var/www
        networks: 
          - app-network
      
      nginx:
        build: .docker/nginx
        container_name: nginx
        restart: always
        tty: true
        ports:
          - "8000:80"
        volumes:
          - .:/var/www
        networks: 
          - app-network

      db:
        build: .docker/mysql
        command: --innodb-use-native-aio=0
        container_name: db
        restart: always
        tty: true
        ports: 
          - "3306:3306"
        volumes:
          - ./.docker/mysql/dbdata:/var/lib/mysql
        environment: 
          - MYSQL_DATABASE=laravel
          - MYSQL_ROOT_PASSWORD=root
          - MYSQL_USER=root
        networks: 
          - app-network

      redis:
        image: redis:alpine
        expose:
          - 6379
        networks: 
          - app-network

    networks: 
      app-network:
        driver: bridge


  #### Executando o docker-compose
    docker-compose up -d
  #### Derrubando os serviços do docker-compose
    docker-compose down
  #### Rebuild o docker-compose
    docker-compose up -d --build

# Trabalhando como Dockerize
      https://github.com/jwilder/dockerize
  ##### Adicionando no Dockerfile
    FROM php:7.3.6-fpm-alpine3.9
    RUN apk add --no-cache shadow openssl \ 
        && apk add bash mysql-client \ 
        && docker-php-ext-install pdo pdo_mysql
    ENV DOCKERIZE_VERSION v0.6.1
    RUN wget https://github.com/jwilder/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-alpine-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
        && tar -C /usr/local/bin -xzvf dockerize-alpine-linux-amd64-$DOCKERIZE_VERSION.tar.gz \
        && rm dockerize-alpine-linux-amd64-$DOCKERIZE_VERSION.tar.gz
    WORKDIR /var/www
    RUN rm -rf /var/www/html 
    RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    RUN ln -s public html
    RUN usermod -u 1000 www-data
    USER www-data
    EXPOSE 9000
    ENTRYPOINT ["php-fpm"]
  ###### Rebuild container
      docker-compose up -d --build
  ###### Entrando no container para executar p dockerize
      docker exec -u root -it app bash
      dockerize -wait tcp://db:3306 -timeout 20s
  ###### Comando docker para exibir logs de um serviço
      docker logs app
  ### Criando o entrypoint.sh