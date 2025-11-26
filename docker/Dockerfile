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