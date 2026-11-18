# -*- mode: ruby -*-
# vi: set ft=ruby :

# ==============================================================================
# Despliegue Automatizado de Infraestructura de Red - Proyecto ASIR
# ==============================================================================
# Topología del entorno:
#   * Nodo 1 (kea1): Servidor DHCPv4 Primario, DDNS, Enrutador NAT y Keepalived (MASTER)
#   * Nodo 2 (kea2): Servidor DHCPv4 Secundario (Hot-Standby), DDNS y Keepalived (BACKUP)
#   * Nodo 3 (dns1): Servidor BIND9 Maestro (asir.lan, virtual.asir.lan y delegación)
#   * Nodo 4 (dns2): Servidor BIND9 Esclavo (asir.lan) y Maestro delegado (es.asir.lan)
#   * Nodo 5 (cliente1): Equipo cliente para validación de concesiones DHCP y DDNS
# ==============================================================================

Vagrant.configure("2") do |config|
  # Imagen base del sistema operativo
  config.vm.box = "debian/bookworm64"

  # Claves TSIG para sincronización de zonas y actualizaciones dinámicas (DDNS)
  CLAVE_TRANSFERENCIA = "YDrx42/FxVoE3qowLUZE+n8Bph2aMcVbcxzlCOTg7sw="
  CLAVE_DDNS          = "sE394OMft+Ayg29f1NgbYJ+ci8grgLl7Yv1eg4Nefd0="
  
  # Interfaz física para el adaptador puente (WAN)
  ADAPTADOR_PUENTE    = "eno1"

  # ============================================================================
  # APROVISIONAMIENTO 1: Nodos del Clúster DHCP y Puerta de Enlace (kea1 / kea2)
  # ============================================================================
  script_cluster_kea = <<-SHELL
    set -e
    export DEBIAN_FRONTEND=noninteractive

    # Parámetros recibidos por argumento en el arranque
    NOMBRE_NODO="$1"
    ROL_VRRP="$2"
    PRIORIDAD_VRRP="$3"

    # Direccionamiento de red y pasarelas virtuales (VIP)
    PASARELA_WAN="172.26.0.1"
    IP_VIRTUAL_WAN="172.26.32.250"
    IP_VIRTUAL_LAN="192.168.25.1"

    # Actualización de repositorios e instalación de servicios necesarios
    apt-get update -y
    apt-get install -y kea-dhcp4-server kea-dhcp-ddns-server kea-ctrl-agent keepalived nftables curl chrony

    # Detección dinámica de las tarjetas de red según su rango IP
    NIC_WAN=$(ip -o -4 addr show | awk '$4 ~ /^172.26.30./ {print $2; exit}')
    NIC_LAN=$(ip -o -4 addr show | awk '$4 ~ /^192.168.25./ {print $2; exit}')
    [ -z "$NIC_WAN" ] || [ -z "$NIC_LAN" ] && exit 1

    # Habilitar el reenvío de paquetes IPv4 en el kernel
    echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-enrutamiento.conf
    sysctl --system

    # Configuración del cortafuegos nftables para aplicar enmascaramiento (SNAT)
    cat <<EOF > /etc/nftables.conf
#!/usr/sbin/nft -f
flush ruleset
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 192.168.25.0/24 oifname "$NIC_WAN" snat to $IP_VIRTUAL_WAN
    }
}
EOF
    systemctl enable nftables && systemctl restart nftables

    # Configuración de alta disponibilidad en puerta de enlace con Keepalived (VRRP)
    cat <<EOF > /etc/keepalived/keepalived.conf
vrrp_sync_group GRUPO_GATEWAY {
    group { INSTANCIA_LAN INSTANCIA_WAN }
}
vrrp_instance INSTANCIA_LAN {
    state $ROL_VRRP
    interface $NIC_LAN
    virtual_router_id 61
    priority $PRIORIDAD_VRRP
    advert_int 1
    track_interface { $NIC_LAN $NIC_WAN }
    authentication { auth_type PASS; auth_pass vrrpPass99; }
    virtual_ipaddress { $IP_VIRTUAL_LAN/24 }
}
vrrp_instance INSTANCIA_WAN {
    state $ROL_VRRP
    interface $NIC_WAN
    virtual_router_id 62
    priority $PRIORIDAD_VRRP
    advert_int 1
    track_interface { $NIC_LAN $NIC_WAN }
    authentication { auth_type PASS; auth_pass vrrpPass99; }
    virtual_ipaddress { $IP_VIRTUAL_WAN/16 }
}
EOF
    systemctl enable keepalived && systemctl restart keepalived

    # Configuración del demonio Kea DHCP-DDNS para actualizar BIND9
    cat <<EOF > /etc/kea/kea-dhcp-ddns.conf
{
"DhcpDdns": {
    "ip-address": "127.0.0.1",
    "port": 53001,
    "control-socket": { "socket-type": "unix", "socket-name": "/run/kea/kea-ddns-ctrl-socket" },
    "tsig-keys": [ { "name": "ddns-key", "algorithm": "HMAC-SHA256", "secret": "#{CLAVE_DDNS}" } ],
    "forward-ddns": {
        "ddns-domains": [ { "name": "asir.lan.", "key-name": "ddns-key", "dns-servers": [ { "ip-address": "192.168.25.10", "port": 53 } ] } ]
    },
    "reverse-ddns": {
        "ddns-domains": [ { "name": "25.168.192.in-addr.arpa.", "key-name": "ddns-key", "dns-servers": [ { "ip-address": "192.168.25.10", "port": 53 } ] } ]
    },
    "loggers": [ { "name": "kea-dhcp-ddns", "output_options": [ { "output": "syslog" } ], "severity": "INFO" } ]
}
}
EOF
    systemctl enable kea-dhcp-ddns-server && systemctl restart kea-dhcp-ddns-server

    # Configuración del servicio principal Kea DHCPv4 con soporte HA (Hot-Standby)
    cat <<EOF > /etc/kea/kea-dhcp4.conf
{
"Dhcp4": {
    "interfaces-config": { "interfaces": [ "$NIC_LAN" ] },
    "control-socket": { "socket-type": "unix", "socket-name": "/run/kea/kea4-ctrl-socket" },
    "lease-database": { "type": "memfile", "persist": true, "name": "/var/lib/kea/kea-leases4.csv", "lfc-interval": 3600 },
    "dhcp-ddns": { "enable-updates": true, "server-ip": "127.0.0.1", "server-port": 53001 },
    "ddns-send-updates": true,
    "ddns-qualifying-suffix": "asir.lan",
    "ddns-override-client-update": true,
    "ddns-override-no-update": true,
    "ddns-replace-client-name": "when-not-present",
    "hostname-char-set": "[^A-Za-z0-9.-]",
    "hostname-char-replacement": "",
    "valid-lifetime": 3600,
    "renew-timer": 1800,
    "rebind-timer": 3150,
    "option-data": [
        { "name": "domain-name-servers", "data": "192.168.25.10, 192.168.25.11" },
        { "name": "domain-name", "data": "asir.lan" }
    ],
    "hooks-libraries": [
        { "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_lease_cmds.so" },
        {
            "library": "/usr/lib/x86_64-linux-gnu/kea/hooks/libdhcp_ha.so",
            "parameters": {
                "high-availability": [
                    {
                        "this-server-name": "$NOMBRE_NODO",
                        "mode": "hot-standby",
                        "heartbeat-delay": 10000,
                        "max-response-delay": 30000,
                        "max-ack-delay": 5000,
                        "max-unacked-clients": 3,
                        "multi-threading": {
                            "enable-multi-threading": true,
                            "http-dedicated-listener": true,
                            "http-listener-threads": 2,
                            "http-client-threads": 2
                        },
                        "peers": [
                            { "name": "kea1", "url": "http://192.168.25.2:8001/", "role": "primary", "auto-failover": true },
                            { "name": "kea2", "url": "http://192.168.25.3:8001/", "role": "standby", "auto-failover": true }
                        ]
                    }
                ]
            }
        }
    ],
    "subnet4": [
        {
            "id": 1,
            "subnet": "192.168.25.0/24",
            "interface": "$NIC_LAN",
            "pools": [ { "pool": "192.168.25.100 - 192.168.25.200" } ],
            "option-data": [ { "name": "routers", "data": "$IP_VIRTUAL_LAN" } ],
            "reservations": [
                {
                    "hw-address": "08:00:27:aa:bb:cc",
                    "ip-address": "192.168.25.50",
                    "hostname": "impresora-dpto"
                }
            ]
        }
    ],
    "loggers": [ { "name": "kea-dhcp4", "output_options": [ { "output": "syslog" } ], "severity": "INFO" } ]
}
}
EOF
    systemctl enable kea-dhcp4-server kea-ctrl-agent
    systemctl restart kea-dhcp4-server kea-ctrl-agent

    # Creación de script auxiliar para corregir la ruta predeterminada de Vagrant
    cat <<'EOF' > /usr/local/sbin/ajustar-ruta-wan.sh
#!/bin/bash
PASARELA="__GW__"
NIC_WAN=$(ip -o -4 addr show | awk '$4 ~ /^172.26.30./ {print $2; exit}')
NIC_VAGRANT=$(ip -o -4 addr show | awk '$4 ~ /^10.0.2./ {print $2; exit}')
[ -z "$NIC_WAN" ] && exit 1
if [ -n "$NIC_VAGRANT" ]; then
  while ip route del default dev "$NIC_VAGRANT" 2>/dev/null; do :; done
fi
ip route replace default via "$PASARELA" dev "$NIC_WAN"
EOF
    sed -i "s/__GW__/$PASARELA_WAN/" /usr/local/sbin/ajustar-ruta-wan.sh
    chmod +x /usr/local/sbin/ajustar-ruta-wan.sh

    # Servicio de Systemd para garantizar que la ruta WAN persista tras reinicios
    cat <<'EOF' > /etc/systemd/system/ruta-wan.service
[Unit]
Description=Configuracion de pasarela predeterminada por interfaz WAN
After=network-online.target
Wants=network-online.target
Before=keepalived.service
[Service]
Type=oneshot
ExecStart=/usr/local/sbin/ajustar-ruta-wan.sh
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF
    systemctl daemon-reload && systemctl enable ruta-wan.service
    /usr/local/sbin/ajustar-ruta-wan.sh || true
  SHELL

  # ============================================================================
  # APROVISIONAMIENTO 2: Servidor DNS Primario / Maestro (dns1)
  # ============================================================================
  script_dns_maestro = <<-SHELL
    set -e
    export DEBIAN_FRONTEND=noninteractive
    apt-get update -y
    apt-get install -y bind9 bind9utils bind9-dnsutils chrony curl

    # Definición de claves criptográficas TSIG
    cat <<EOF > /etc/bind/transfer.key
key "transfer-key" {
    algorithm hmac-sha256;
    secret "#{CLAVE_TRANSFERENCIA}";
};
EOF

    cat <<EOF > /etc/bind/ddns.key
key "ddns-key" {
    algorithm hmac-sha256;
    secret "#{CLAVE_DDNS}";
};
EOF
    chown root:bind /etc/bind/*.key
    chmod 640 /etc/bind/*.key

    # Opciones generales del servidor BIND9
    cat <<'EOF' > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    listen-on { 127.0.0.1; 192.168.25.10; };
    listen-on-v6 { none; };
    recursion yes;
    allow-query { localhost; 192.168.25.0/24; };
    allow-recursion { localhost; 192.168.25.0/24; };
    forwarders { 1.1.1.1; 9.9.9.9; };
    forward first;
    dnssec-validation auto;
};
EOF

    # Declaración de zonas maestras (Directa e Inversa)
    cat <<'EOF' > /etc/bind/named.conf.local
include "/etc/bind/transfer.key";
include "/etc/bind/ddns.key";

zone "asir.lan" {
    type master;
    file "/var/lib/bind/db.asir.lan";
    allow-transfer { key transfer-key; };
    also-notify { 192.168.25.11; };
    update-policy { grant ddns-key zonesub ANY; };
};

zone "25.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/db.25.168.192";
    allow-transfer { key transfer-key; };
    also-notify { 192.168.25.11; };
    update-policy { grant ddns-key zonesub ANY; };
};
EOF

    # Archivo de registros de la zona directa asir.lan
    cat <<'EOF' > /var/lib/bind/db.asir.lan
$TTL 3600
@   IN SOA dns1.asir.lan. admin.asir.lan. (
        2026110401 ; Numero de serie
        3600       ; Refresco
        900        ; Reintento
        1209600    ; Expiracion
        3600 )     ; TTL negativo

$ORIGIN asir.lan.
@           IN NS   dns1.asir.lan.
@           IN NS   dns2.asir.lan.

router      IN A    192.168.25.1
kea1        IN A    192.168.25.2
kea2        IN A    192.168.25.3
dns1        IN A    192.168.25.10
dns2        IN A    192.168.25.11
impresora   IN A    192.168.25.50

; --- Subdominio virtual gestionado en este mismo archivo ---
$ORIGIN virtual.asir.lan.
www         IN A    192.168.25.60

; --- Delegacion del subdominio es.asir.lan hacia el nodo dns2 ---
$ORIGIN es.asir.lan.
@           IN NS   nssub
nssub       IN A    192.168.25.11
EOF

    # Archivo de registros de la zona inversa 25.168.192.in-addr.arpa
    cat <<'EOF' > /var/lib/bind/db.25.168.192
$TTL 3600
@   IN SOA dns1.asir.lan. admin.asir.lan. (
        2026110401 3600 900 1209600 3600 )
@   IN NS  dns1.asir.lan.
@   IN NS  dns2.asir.lan.

1   IN PTR router.asir.lan.
2   IN PTR kea1.asir.lan.
3   IN PTR kea2.asir.lan.
10  IN PTR dns1.asir.lan.
11  IN PTR dns2.asir.lan.
50  IN PTR impresora.asir.lan.
EOF

    chown -R bind:bind /var/lib/bind/
    systemctl enable named chrony
    systemctl restart named chrony

    # Ajuste de enrutamiento para salir a través de la VIP del clúster Kea
    NIC_LAN=$(ip -o -4 addr show | awk '$4 ~ /^192.168.25./ {print $2; exit}')
    NIC_VAGRANT=$(ip -o -4 addr show | awk '$4 ~ /^10.0.2./ {print $2; exit}')
    [ -n "$NIC_VAGRANT" ] && ip route del default dev "$NIC_VAGRANT" 2>/dev/null || true
    ip route replace default via 192.168.25.1 dev "$NIC_LAN"
    echo "nameserver 127.0.0.1" > /etc/resolv.conf
  SHELL

  # ============================================================================
  # APROVISIONAMIENTO 3: Servidor DNS Esclavo y Delegado (dns2)
  # ============================================================================
  script_dns_esclavo = <<-SHELL
    set -e
    export DEBIAN_FRONTEND=noninteractive
    apt-get update -y
    apt-get install -y bind9 bind9utils bind9-dnsutils chrony curl

    # Clave TSIG para autenticar las transferencias de zona con dns1
    cat <<EOF > /etc/bind/transfer.key
key "transfer-key" {
    algorithm hmac-sha256;
    secret "#{CLAVE_TRANSFERENCIA}";
};
EOF
    chown root:bind /etc/bind/transfer.key
    chmod 640 /etc/bind/transfer.key

    # Opciones de configuración de BIND9 en dns2
    cat <<'EOF' > /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    listen-on { 127.0.0.1; 192.168.25.11; };
    listen-on-v6 { none; };
    recursion yes;
    allow-query { localhost; 192.168.25.0/24; };
    allow-recursion { localhost; 192.168.25.0/24; };
    forwarders { 1.1.1.1; 9.9.9.9; };
    forward first;
    dnssec-validation auto;
};
EOF

    # Definición de zonas: Esclavo de asir.lan y Maestro de es.asir.lan
    cat <<'EOF' > /etc/bind/named.conf.local
include "/etc/bind/transfer.key";

server 192.168.25.10 {
    keys { transfer-key; };
};

zone "asir.lan" {
    type slave;
    file "/var/cache/bind/db.asir.lan";
    masters { 192.168.25.10; };
    allow-transfer { none; };
};

zone "25.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.25.168.192";
    masters { 192.168.25.10; };
    allow-transfer { none; };
};

// Autoridad directa sobre el subdominio delegado es.asir.lan
zone "es.asir.lan" {
    type master;
    file "/var/lib/bind/db.es.asir.lan";
    allow-transfer { none; };
};
EOF

    # Registros de recursos para el subdominio delegado es.asir.lan
    cat <<'EOF' > /var/lib/bind/db.es.asir.lan
$TTL 3600
@   IN SOA nssub.es.asir.lan. admin.es.asir.lan. (
        2026110401 3600 900 1209600 3600 )

$ORIGIN es.asir.lan.
@       IN NS   nssub
nssub   IN A    192.168.25.11
www     IN A    192.168.25.70
ftp     IN A    192.168.25.71
EOF

    chown -R bind:bind /var/lib/bind/
    systemctl enable named chrony
    systemctl restart named chrony

    # Configuración de puerta de enlace predeterminada hacia la VIP LAN
    NIC_LAN=$(ip -o -4 addr show | awk '$4 ~ /^192.168.25./ {print $2; exit}')
    NIC_VAGRANT=$(ip -o -4 addr show | awk '$4 ~ /^10.0.2./ {print $2; exit}')
    [ -n "$NIC_VAGRANT" ] && ip route del default dev "$NIC_VAGRANT" 2>/dev/null || true
    ip route replace default via 192.168.25.1 dev "$NIC_LAN"
    echo "nameserver 127.0.0.1" > /etc/resolv.conf
  SHELL

  # ============================================================================
  # APROVISIONAMIENTO 4: Nodo Cliente DHCP (cliente1)
  # ============================================================================
  script_nodo_cliente = <<-SHELL
    set -e
    export DEBIAN_FRONTEND=noninteractive
    hostnamectl set-hostname equipo-carlos
    apt-get update -y
    apt-get install -y bind9-dnsutils dhcpcd-base

    # Identificar la interfaz conectada a la red interna
    NIC_LAN=$(ip -o link show | awk -F': ' '$2 !~ /lo|eth0/ {print $2; exit}')

    # Solicitar concesión DHCP enviando el nombre de host para registro DDNS
    dhclient -r "$NIC_LAN" 2>/dev/null || true
    dhclient -v -H "equipo-carlos" "$NIC_LAN"

    # Eliminar la ruta por defecto de Vagrant y priorizar la pasarela del clúster
    NIC_VAGRANT=$(ip -o -4 addr show | awk '$4 ~ /^10.0.2./ {print $2; exit}')
    [ -n "$NIC_VAGRANT" ] && ip route del default dev "$NIC_VAGRANT" 2>/dev/null || true
    ip route replace default via 192.168.25.1 dev "$NIC_LAN"
  SHELL

  # ============================================================================
  # DEFINICIÓN DE LAS MÁQUINAS VIRTUALES EN VIRTUALBOX
  # ============================================================================

  # --- Máquina 1: kea1 (Primario / Master) ---
  config.vm.define "kea1" do |node|
    node.vm.hostname = "kea1"
    node.vm.network "public_network", bridge: ADAPTADOR_PUENTE, ip: "172.26.30.203", netmask: "255.255.0.0"
    node.vm.network "private_network", ip: "192.168.25.2", netmask: "255.255.255.0", virtualbox__intnet: "red_interna_asir"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ASIR_KEA1"
      vb.memory = "1024"
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end
    node.vm.provision "shell", inline: script_cluster_kea, args: ["kea1", "MASTER", "100"]
  end

  # --- Máquina 2: kea2 (Secundario / Backup) ---
  config.vm.define "kea2" do |node|
    node.vm.hostname = "kea2"
    node.vm.network "public_network", bridge: ADAPTADOR_PUENTE, ip: "172.26.30.204", netmask: "255.255.0.0"
    node.vm.network "private_network", ip: "192.168.25.3", netmask: "255.255.255.0", virtualbox__intnet: "red_interna_asir"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ASIR_KEA2"
      vb.memory = "1024"
      vb.cpus = 1
      vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end
    node.vm.provision "shell", inline: script_cluster_kea, args: ["kea2", "BACKUP", "90"]
  end

  # --- Máquina 3: dns1 (DNS Maestro) ---
  config.vm.define "dns1" do |node|
    node.vm.hostname = "dns1"
    node.vm.network "private_network", ip: "192.168.25.10", netmask: "255.255.255.0", virtualbox__intnet: "red_interna_asir"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ASIR_DNS1"
      vb.memory = "512"
      vb.cpus = 1
    end
    node.vm.provision "shell", inline: script_dns_maestro
  end

  # --- Máquina 4: dns2 (DNS Esclavo + Delegación) ---
  config.vm.define "dns2" do |node|
    node.vm.hostname = "dns2"
    node.vm.network "private_network", ip: "192.168.25.11", netmask: "255.255.255.0", virtualbox__intnet: "red_interna_asir"
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ASIR_DNS2"
      vb.memory = "512"
      vb.cpus = 1
    end
    node.vm.provision "shell", inline: script_dns_esclavo
  end

  # --- Máquina 5: cliente1 (Cliente de pruebas DHCP/DDNS) ---
  config.vm.define "cliente1" do |node|
    node.vm.hostname = "equipo-carlos"
    node.vm.network "private_network", type: "dhcp", virtualbox__intnet: "red_interna_asir", auto_config: false
    node.vm.provider "virtualbox" do |vb|
      vb.name = "ASIR_CLIENTE1"
      vb.memory = "512"
      vb.cpus = 1
    end
    node.vm.provision "shell", inline: script_nodo_cliente
  end

end