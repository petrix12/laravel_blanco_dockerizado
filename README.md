# Entorno para proyectos de Laravel

## Prompt para IA
```md
Necesito crear un entorno Docker para generar proyectos Laravel en blanco. Los requisitos del contenedor son:
+ Ubuntu (22.04 o 24.04).
+ PHP.
+ Composer.
+ Git.
+ Node y npm (para proyectos que lo necesiten).
Quiero que me proporciones:
1. Un Dockerfile.
2. Un docker-compose.yml.
3. Instrucciones paso a paso para levantar el entorno y crear el proyecto Laravel en blanco como plantilla.
Notas importantes:
+ Trabajo en Windows 11 con WSL2 y Docker Desktop.
+ No necesito servicio de base de datos dentro de Docker; usaré el MySQL local que corre en mi laptop en el puerto 5306.
+ Indícame también qué debo añadir en C:\Windows\System32\drivers\etc\hosts para asignar un dominio local, por ejemplo: blanco.test
+ Si falta algún detalle para evitar conflictos entre múltiples proyectos Laravel locales (cada uno con su propio dominio), tenlo en cuenta y menciónalo.
+ Como estoy en WSL con Ubuntu 24.04, quizás no sea conveniente la distribución de Ubuntu en la imagen Docker, sino un php:8.3-fpm o php:8.4-fpm. Tú ve qué es más práctico y conveniente.
+ Por favor razona tu respuesta lo mejor posible.
```

## Construcción del proyecto en blanco
1. Estructura de archivos
```bash
/laravel-docker
├── docker/
│   ├── nginx/
│   │   └── default.conf
│   └── Dockerfile
└── docker-compose.yml
```

2. Crear los archivos:
+ **docker/Dockerfile**:
```Dockerfile
FROM php:8.3-fpm

# Argumentos definidos en docker-compose.yml
ARG user
ARG uid

# 1. Instalar dependencias del sistema
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    # Limpiar caché para reducir tamaño de imagen
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# 2. Instalar extensiones de PHP requeridas por Laravel
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# 3. Obtener el último Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# 4. Instalar Node.js y npm (Versión LTS 20)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# 5. Crear usuario del sistema para ejecutar comandos Composer y Artisan
# Esto es CRÍTICO en Linux/WSL para evitar problemas de permisos de root
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

# 6. Establecer directorio de trabajo
WORKDIR /var/www

USER $user
```

+ **docker/nginx/default.conf**:
```conf
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    root /var/www/public;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

+ **docker-compose.yml (Opción 1)**:
```yml
services:
  # 1. SERVICIO APP (PHP-FPM)
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
      args:
        user: wsl_user
        uid: 1000
    image: blanco-app
    container_name: blanco-app
    restart: unless-stopped
    working_dir: /var/www
    volumes:
      - ./:/var/www
    networks:
      - webproxy # Se une a la red del proxy

  # 2. SERVICIO NGINX (WEB SERVER)
  nginx:
    image: nginx:alpine
    container_name: blanco-nginx
    restart: unless-stopped
    volumes:
      - ./:/var/www
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app
    networks:
      - webproxy # Crucial: se une a la red del proxy para ser descubierto
      
    # ETIQUETAS (LABELS) DE CONFIGURACIÓN PARA TRAEFIK
    labels:
      - "traefik.enable=true"
      # Nuevo router (debe ser único por proyecto)
      - "traefik.http.routers.blanco.rule=Host(`blanco.test`)"
      - "traefik.http.routers.blanco.entrypoints=web"
      - "traefik.http.services.blanco.loadbalancer.server.port=80"
      - "traefik.docker.network=webproxy" # Especificar la red interna de Traefik

  # 3. SERVICIO TRAEFIK (PROXY INVERSO LOCAL)
  traefik:
    image: traefik:v2.10
    container_name: blanco-traefik-proxy # Nombre único
    restart: always
    command:
      # Habilita Docker Provider para leer los labels de APP y NGINX
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      # Define el puerto de escucha (80 en el host)
      - --entrypoints.web.address=:80 
    ports:
      # Expone Traefik al puerto 80 del HOST
      - "80:80" 
      # Puedes exponer el dashboard en un puerto diferente para cada proyecto si lo necesitas:
      #- "8081:8080"
    volumes:
      # Montar el socket de Docker para que Traefik pueda leer otros contenedores
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - webproxy

networks:
  # La red ahora es INTERNA, se crea con el proyecto y se destruye con él.
  webproxy:
    name: webproxy-${COMPOSE_PROJECT_NAME} # Añadimos el nombre del proyecto para que sea única
    driver: bridge
```

+ **docker-compose.yml (Opción 2)**:
```yml
services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
      args:
        user: wsl_user
        uid: 1000 # Generalmente el primer usuario en WSL es 1000
    image: laravel-app
    container_name: laravel-app
    restart: unless-stopped
    working_dir: /var/www
    volumes:
      - ./:/var/www
    networks:
      - laravel

  nginx:
    image: nginx:alpine
    container_name: laravel-nginx
    restart: unless-stopped
    ports:
      # Mapeamos puerto 80 del contenedor al 8000 del host para evitar conflictos
      - "8000:80"
    volumes:
      - ./:/var/www
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app
    networks:
      - laravel

networks:
  laravel:
    driver: bridge
```

3. Levantar entorno:
    ```bash
    docker compose up -d --build
    ```

4. Entra en la terminal del contenedor:
    ```bash
    docker exec -u root -it app bash
    # o
    docker compose exec -u root -it app bash
    ```

5. Crea el proyecto en una carpeta temporal:
    ```bash
    composer create-project laravel/laravel temp_app
    ```

6. Mover los archivos a la raíz y borra la carpeta temporal:
    ```bash
    mv temp_app/* .
    mv temp_app/.env .
    mv temp_app/.editorconfig .
    mv temp_app/.git* .
    rm -rf temp_app
    exit
    ```

7. Estructura final del proyecto:
    ```
    /laravel-docker
    ├── app/
    ├── bootstrap/
    ├── config/
    ├── docker/           <-- Tu carpeta original de configuración
    ├── public/
    ├── vendor/
    ├── .env              <-- Archivo importante movido
    ├── docker-compose.yml <-- Tu archivo original
    ├── artisan
    ├── composer.json
    └── ... (resto de archivos de Laravel)    
    ```

8. Asegurar permisos de escritura:
    ```bash
    docker compose exec -u root app chmod -R 777 storage bootstrap/cache
    docker compose exec -u root app chown -R wsl_user:www-data storage bootstrap/cache
    docker compose exec -u root app chmod -R ug+w storage bootstrap/cache
    sudo chown -R petrix:petrix .    
    ```

9. Configurar el .env:
    ```env
    DB_CONNECTION=mysql    
    DB_HOST=host.docker.internal    # host.docker.internal apunta a tu máquina Windows/WSL desde dentro de Docker    
    DB_PORT=5306                    # El puerto de tu bd
    DB_DATABASE=nombre_de_tu_bd_local
    DB_USERNAME=tu_usuario_local
    DB_PASSWORD=tu_password_local    
    ```

10. Configuración de Hosts (blanco.test):
    ``` title="C:\Windows\System32\drivers\etc\hosts"
    127.0.0.1 blanco.test
    ```
    + **Nota**: Abrir Bloc de notas como Administrador.
    + http://blanco.test

11. Instalar autenticación (login, registro, logout):
    ```bash
    composer require laravel/breeze --dev
    php artisan breeze:install
    npm install
    npm run build
    php artisan migrate
    ```