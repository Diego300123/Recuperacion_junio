Pa que haiga lujo

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"

  config.vm.network "private_network", ip: "192.168.57.4"
  config.vm.hostname = "www.example.test"

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y nginx openssl apache2-utils
  SHELL
end




# Guía de instalación y configuración de Nginx con SSL y autenticación básica

Este documento describe paso a paso cómo instalar y configurar un servidor web Nginx en una máquina virtual Linux, incluyendo la configuración de HTTPS con un certificado autofirmado y autenticación básica en una ruta protegida.

---

## Paso 1: Actualizar y preparar el sistema

Abre una terminal y ejecuta los siguientes comandos para actualizar el sistema:

```bash
sudo apt update
sudo apt upgrade
```

---

## Paso 2: Instalar Nginx

Instala Nginx con:

```bash
sudo apt install nginx
```

Asegúrate de que Nginx se inicie automáticamente:

```bash
sudo systemctl enable nginx
```

Inicia el servicio:

```bash
sudo systemctl start nginx
```

---

## Paso 3: Verificar la instalación de Nginx

Accede a la IP de tu máquina virtual (por ejemplo, `http://192.168.57.4`) desde un navegador. Deberías ver la página de bienvenida de Nginx.

---

## Paso 4: Configuración de la IP y el nombre del servidor

Edita el archivo de configuración predeterminado de Nginx:

```bash
sudo nano /etc/nginx/sites-available/default
```

Reemplaza el contenido del bloque `server` con lo siguiente:

```nginx
server {
    listen 80;
    server_name www.example.test;
    root /var/www/html;

    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Guarda y cierra el archivo (Ctrl+X, luego Y y Enter).

Reinicia Nginx:

```bash
sudo systemctl restart nginx
```

---

## Paso 5: Configuración del certificado SSL autofirmado

### Instalar OpenSSL

```bash
sudo apt install openssl
```

### Crear directorio para los certificados

```bash
sudo mkdir /etc/nginx/ssl
```

### Generar certificado autofirmado

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/nginx/ssl/example.test.key \
-out /etc/nginx/ssl/example.test.crt
```

**Nota:** Cuando se te pida el `Common Name`, escribe: `www.example.test`.

### Configurar HTTPS en Nginx

Edita de nuevo el archivo de configuración:

```bash
sudo nano /etc/nginx/sites-available/default
```

Reemplaza el contenido por:

```nginx
server {
    listen 80;
    server_name www.example.test;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name www.example.test;

    ssl_certificate /etc/nginx/ssl/example.test.crt;
    ssl_certificate_key /etc/nginx/ssl/example.test.key;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Reinicia Nginx:

```bash
sudo systemctl restart nginx
```

---

## Paso 6: Verificar el certificado SSL

Ejecuta el siguiente comando para ver los detalles del certificado:

```bash
openssl x509 -in /etc/nginx/ssl/example.test.crt -noout -text
```

---

## Paso 7: Crear la página principal

Edita el archivo `index.html`:

```bash
sudo nano /var/www/html/index.html
```

Agrega el siguiente contenido:

```html
<h1>www.example.test</h1>
```

Guarda y cierra el archivo.

---

## Paso 8: Configurar autenticación básica en `/admin`

### Instalar herramientas necesarias

```bash
sudo apt install apache2-utils
```

### Crear archivo de contraseñas

```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

> Usa la contraseña `1234` cuando se te solicite.

### Editar configuración de Nginx

```bash
sudo nano /etc/nginx/sites-available/default
```

Modifica el segundo bloque `server` (HTTPS) para incluir lo siguiente:

```nginx
server {
    listen 443 ssl;
    server_name www.example.test;

    ssl_certificate /etc/nginx/ssl/example.test.crt;
    ssl_certificate_key /etc/nginx/ssl/example.test.key;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /admin {
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
        return 200 "<h1>Admin panel</h1>";
    }
}
```

Reinicia Nginx:

```bash
sudo systemctl restart nginx
```

---

## Paso 9: Verificación

1. Accede a: [https://www.example.test](https://www.example.test)
   - Verás:  
     ```html
     <h1>www.example.test</h1>
     ```

2. Accede a: [https://www.example.test/admin](https://www.example.test/admin)
   - Inicia sesión con:
     - Usuario: `admin`
     - Contraseña: `1234`
   - Verás:
     ```html
     <h1>Admin panel</h1>
     ```

3. Si usas credenciales incorrectas, verás un error 401.

---

¡Configuración completa!

¡IMPORTANTE!
Ruta para editar los hosts: C:\Windows\System32\drivers\etc\hosts
Por ejemplo:
  192.168.57.4   www.example.test
