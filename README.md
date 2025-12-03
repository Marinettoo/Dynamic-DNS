📋 Escenario de Red
-------------------

-   **Red Interna:** `192.168.57.0/24`

-   **Dominio:** `micasa.es`

-   **Máquina Servidor (`server`):**

    -   IP Estática: **`192.168.57.10`**

    -   Roles: Servidor DNS Maestro y Servidor DHCP.

-   **Máquina Cliente (`c1`):**

    -   IP: Asignada por DHCP (Rango `.25` - `.50`).

    -   Rol: Cliente que recibe nombre automático (ej: `c1.micasa.es`).

* * * * *

🚀 1. Preparación del Entorno (Vagrant)
---------------------------------------

Preparamos el fichero `Vagrantfile` para levantar las dos máquinas conectadas por una red interna privada.

Ruby

```
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  # --- SERVIDOR (DNS + DHCP) ---
  config.vm.define "server" do |server|
    server.vm.hostname = "server"
    server.vm.network "private_network", ip: "192.168.57.10", virtualbox__intnet: "ddns_net"
    server.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
    end
  end

  # --- CLIENTE ---
  config.vm.define "c1" do |c1|
    c1.vm.hostname = "c1"
    # El cliente pide IP por DHCP
    c1.vm.network "private_network", type: "dhcp", virtualbox__intnet: "ddns_net"
  end
end

```

* * * * *

🔑 2. Generación de la Clave de Seguridad (TSIG)
------------------------------------------------

Para que el DHCP tenga permiso de "escribir" en el DNS, necesitan compartir una contraseña secreta.

1.  Entramos al servidor: `vagrant ssh server`

2.  Generamos la clave y la guardamos en un archivo temporal:

    Bash

    ```
    tsig-keygen -a hmac-sha256 ddns-key

    ```

    *Copia el resultado (la parte que dice `secret "..."`).*

* * * * *

🌐 3. Configuración del Servidor DNS (Bind9)
--------------------------------------------

### 3.1. Definir la Clave

Editamos el archivo de opciones para incluir la clave que acabamos de generar.

📄 **Archivo:** `/etc/bind/named.conf.options`

DNS Zone file

```
# Añadimos esto al principio del archivo (o antes del bloque options)
key "ddns-key" {
    algorithm hmac-sha256;
    secret "PEGA_AQUÍ_TU_CLAVE_GENERADA_CON_TSIG_KEYGEN";
}; [cite: 376]

options {
    directory "/var/cache/bind";
    listen-on port 53 { 192.168.57.10; }; # Escuchar en nuestra IP
    allow-query { any; };
    recursion yes;
    forwarders { 8.8.8.8; };
};

```

### 3.2. Declarar las Zonas Dinámicas

Editamos el archivo local para definir nuestras zonas y permitir que se actualicen con la clave.

📄 **Archivo:** `/etc/bind/named.conf.local`

DNS Zone file

```
# Zona Directa
zone "micasa.es" {
    type master;
    file "/var/lib/bind/db.micasa.es";
    allow-update { key "ddns-key"; }; # ¡CRUCIAL! Permite al DHCP escribir aquí [cite: 385]
};

# Zona Inversa
zone "57.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/db.192.168.57";
    allow-update { key "ddns-key"; }; # ¡CRUCIAL! [cite: 390]
};

```

### 3.3. Crear los Archivos de Zona

Creamos los archivos iniciales. Es **muy importante** crearlos en `/var/lib/bind/` porque es el único directorio donde el usuario `bind` tiene permisos de escritura para hacer cambios dinámicos.

📄 **Archivo:** `/var/lib/bind/db.micasa.es` (Zona Directa)

DNS Zone file

```
$TTL 86400
@   IN  SOA server.micasa.es. admin.micasa.es. (
        1       ; Serial
        604800  ; Refresh
        86400   ; Retry
        2419200 ; Expire
        86400 ) ; Negative Cache TTL
;
@       IN  NS  server.micasa.es.
@       IN  A   192.168.57.10
server  IN  A   192.168.57.10

```

📄 **Archivo:** `/var/lib/bind/db.192.168.57` (Zona Inversa)

DNS Zone file

```
$TTL 86400
@   IN  SOA server.micasa.es. admin.micasa.es. (
        1 ; Serial
        604800 ; Refresh
        86400 ; Retry
        2419200 ; Expire
        86400 ) ; Negative Cache TTL
;
@   IN  NS  server.micasa.es.
10  IN  PTR server.micasa.es.

```

### 3.4. Permisos y Reinicio

Damos permisos al usuario `bind` para que pueda crear los archivos temporales (`.jnl`) y reiniciamos.

Bash

```
sudo chown bind:bind /var/lib/bind/db.*
sudo systemctl restart bind9

```

* * * * *

💻 4. Configuración del Servidor DHCP
-------------------------------------

Ahora configuramos el DHCP para que use la clave y actualice las zonas que hemos creado.

📄 **Archivo:** `/etc/dhcp/dhcpd.conf`

DNS Zone file

```
# 1. Definimos la MISMA clave que en el DNS
key "ddns-key" {
    algorithm hmac-sha256;
    secret "PEGA_AQUÍ_LA_MISMA_CLAVE_QUE_EN_EL_DNS";
} [cite: 409]

# 2. Configuración Global DDNS
ddns-update-style interim;      # Estilo de actualización [cite: 424]
update-static-leases on;        # Actualizar también IPs fijas si las hay

# 3. Configuración de la Subred
subnet 192.168.57.0 netmask 255.255.255.0 {
    range 192.168.57.25 192.168.57.50;
    option routers 192.168.57.10;
    option domain-name-servers 192.168.57.10; # ¡El propio servidor!
    option domain-name "micasa.es";

    # --- ZONA DIRECTA ---
    zone micasa.es. {
        primary 192.168.57.10; # A quién hay que avisar [cite: 433]
        key "ddns-key";        # Con qué contraseña
    }

    # --- ZONA INVERSA ---
    zone 57.168.192.in-addr.arpa. {
        primary 192.168.57.10; # A quién hay que avisar [cite: 436]
        key "ddns-key";
    }
}

```

Reiniciamos el servicio DHCP:

Bash

```
sudo systemctl restart isc-dhcp-server

```

* * * * *

✅ 5. Verificación y Pruebas
---------------------------

Para comprobar que todo funciona, vamos a la máquina cliente (`c1`).

### 5.1. Forzar renovación de IP

Bash

```
sudo dhclient -r  # Soltar IP
sudo dhclient -v  # Pedir IP nueva (veremos el log de negociación)

```

### 5.2. Comprobar Resolución Directa (Ping por nombre)

El cliente debe ser capaz de contactar al servidor por su nombre.

Bash

```
ping server.micasa.es

```

### 5.3. Comprobar DDNS (La prueba de fuego) 🔥

Desde el cliente o el servidor, preguntamos al DNS por el nombre del cliente. **Si el DNS responde con la IP, el DDNS ha funcionado.**

**Prueba Directa:**

Bash

```
dig @192.168.57.10 c1.micasa.es

```

*Resultado esperado:* Debe devolver la IP (ej. `192.168.57.25`).

**Prueba Inversa:**

Bash

```
dig @192.168.57.10 -x 192.168.57.25

```

*Resultado esperado:* Debe devolver el nombre `c1.micasa.es.`.

* * * * *

### ⚠️ Solución de Problemas Comunes

-   **Error `SERVFAIL`:** Revisa que la clave `secret` sea idéntica en ambos archivos.

-   **Error `update failed: denied`:** Revisa los permisos de la carpeta `/var/lib/bind/`. El usuario `bind` necesita escribir ahí.

-   **Error `NXDOMAIN` en la inversa:** Asegúrate de haber puesto el bloque `zone 57.168.192.in-addr.arpa` dentro del archivo DHCP correctamente.# Dynamic-DNS
