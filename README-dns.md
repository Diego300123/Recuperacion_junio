Esto de aqui pa haiga lujos

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "256"
    vb.linked_clone = true
  end

  # Actualizar SO
  config.vm.provision "update", type: "shell", inline: <<-SHELL
    apt-get update 
  SHELL

  ######################################################################
  # Servidor 'dns1'
  ######################################################################
  config.vm.define "dns1" do |dns1|
    dns1.vm.hostname = "dns1"  # Asignamos el nombre de host
    dns1.vm.network "private_network", ip: "192.168.57.2"  # IP fija

    dns1.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end

  ######################################################################
  # Servidor 'dns2'
  ######################################################################
  config.vm.define "dns2" do |dns2|
    dns2.vm.hostname = "dns2"  # Asignamos el nombre de host
    dns2.vm.network "private_network", ip: "192.168.57.3"  # IP fija

    dns2.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end
end



















# Configuración de Servidor DNS con BIND9

## Índice
1. [Planteamiento](#planteamiento)
2. [Configuración de Servidor DNS](#configuración-de-servidor-dns)
    - [Instalar BIND9](#instalar-bind9)
    - [Configurar la Zona Directa e Inversa](#configurar-la-zona-directa-e-inversa)
    - [Configurar un Reenviador](#configurar-un-reenviador)
    - [Reiniciar BIND9](#reiniciar-bind9)
    - [Configurar Servidor Secundario](#configurar-servidor-secundario)
3. [Pruebas con `dig` y `nslookup`](#pruebas-con-dig-y-nslookup)

## 1. Planteamiento

Configura una zona directa e inversa con el software BIND9.

- **Zona directa**: `example.test`
- **Zona inversa**: `192.168.57/0`
- Servidores DNS:
  - `dns1.example.test` (IP: 192.168.57.2) - Servidor autorizado
  - `dns2.example.test` (IP: 192.168.57.3) - Servidor secundario
- La transferencia de zona estará permitida solo entre `dns1` y `dns2`.
- Configuración de un reenviador a `8.8.8.8`.
- Configuración de registros:
  - Un registro `A` para `srv.example.test` con IP `192.168.57.4`.
  - Un alias `www.example.test` a `srv.example.test`.
  - Un registro de correo con prioridad `10` que apunte a `srv.example.test`.

## 2. Configuración de Servidor DNS

### Instalar BIND9

En el servidor (tanto en `dns1` como en `dns2`), instala BIND9:

```bash
sudo apt update && sudo apt install bind9 bind9utils bind9-doc -y
```

### Configurar la Zona Directa e Inversa

#### Zona Directa

1. Abre el archivo de configuración de zonas:

```bash
sudo nano /etc/bind/named.conf.local
```

2. Añade la configuración de la zona directa:

```bash
zone "example.test" {
    type master;
    file "/etc/bind/db.example.test";
    allow-transfer { 192.168.57.3; };
};
```

3. Añade la configuración de la zona inversa:

```bash
zone "57.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192";
    allow-transfer { 192.168.57.3; };
};
```

4. Guarda y cierra el archivo (`CTRL + O`, `Enter`, `CTRL + X`).

#### Archivos de Zona

- **Zona Directa**: `/etc/bind/db.example.test`

  Crea el archivo de zona directa con:

```bash
sudo cp /etc/bind/db.empty /etc/bind/db.example.test
```

Edita el archivo y agrega:

```bash
$TTL    604800
@       IN      SOA     dns1.example.test. admin.example.test. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Servidores DNS
@       IN      NS      dns1.example.test.
@       IN      NS      dns2.example.test.

; Registros A
dns1    IN      A       192.168.57.2
dns2    IN      A       192.168.57.3
srv     IN      A       192.168.57.4
www     IN      CNAME   srv

; Registro MX
@       IN      MX 10   srv
```

- **Zona Inversa**: `/etc/bind/db.192`

  Crea el archivo de zona inversa con:

```bash
sudo cp /etc/bind/db.empty /etc/bind/db.192
```

Edita el archivo y agrega:

```bash
$TTL    604800
@       IN      SOA     dns1.example.test. admin.example.test. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Servidores DNS
@       IN      NS      dns1.example.test.
@       IN      NS      dns2.example.test.

; Registros PTR
2       IN      PTR     dns1.example.test.
3       IN      PTR     dns2.example.test.
4       IN      PTR     srv.example.test.
```

### Configurar un Reenviador

1. Abre el archivo de configuración `named.conf.options`:

```bash
sudo nano /etc/bind/named.conf.options
```

2. Añade el reenviador a `8.8.8.8`:

```bash
options {
    forwarders {
        8.8.8.8;
    };
    recursion yes;
    allow-query { any; };
};
```

### Reiniciar BIND9

Para aplicar los cambios, reinicia el servicio BIND9:

```bash
sudo systemctl restart bind9
```

Verifica que el servicio esté corriendo:

```bash
sudo systemctl status bind9
```

### Configurar Servidor Secundario (`dns2`)

1. En el servidor secundario (`dns2`), edita el archivo de zonas como `slave`:

```bash
sudo nano /etc/bind/named.conf.local
```

Añade las zonas como `slave`:

```bash
zone "example.test" {
    type slave;
    masters { 192.168.57.2; };
    file "/var/cache/bind/db.example.test";
};

zone "57.168.192.in-addr.arpa" {
    type slave;
    masters { 192.168.57.2; };
    file "/var/cache/bind/db.192";
};
```

2. Reinicia BIND9 en `dns2`:

```bash
sudo systemctl restart bind9
```

## 3. Pruebas con `dig` y `nslookup`

### Consultas Directas e Inversas

- **dns1.example.test (Directa)**:

```bash
dig @192.168.57.2 dns1.example.test
```

- **dns1.example.test (Inversa)**:

```bash
dig @192.168.57.2 -x 192.168.57.2
```

- **dns2.example.test (Directa)**:

```bash
dig @192.168.57.3 dns2.example.test
```

- **dns2.example.test (Inversa)**:

```bash
dig @192.168.57.3 -x 192.168.57.3
```

### Consultas de Alias y Correo

- **www.example.test (Alias CNAME)**:

```bash
dig @192.168.57.2 www.example.test
```

- **Registro de Correo (MX)**:

```bash
dig @192.168.57.2 example.test MX
```

### Comprobar Transferencia de Zona

Desde `dns2`:

```bash
dig @192.168.57.2 example.test AXFR
```
