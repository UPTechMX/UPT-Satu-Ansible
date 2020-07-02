# ansible-onedataportal
[Ansible](https://docs.ansible.com) playbooks para el despliegue automático del Portal One Data (Portal Satu Data) que comprende [CKAN](https://ckan.org) version 2.8.3 y [Oskari](https://www.oskari.org/) version 1.55.2.

## Requirements
### Target server
* Un recientemente instalado [Ubuntu Server 18.04](https://ubuntu.com/download/server) (mínimo de 4 GB RAM y 20 GB de espacio libre) con un usuario que is parte del grupo `sudo` (IMPORTANTE: usar el super usuario no es recomendable ya que, por default, el servidor que provee el playbook `0_provision_server.yml` desactivara el super usuario para mejorar la seguridad). Abajo se encuentra un ejemplo de un usuario llamado `username` que fue creado y agregado al grupo `sudo`. 
  ```
  adduser username
  usermod -aG sudo username
  ```
* El servidor esta totalmente actualizado y configurado con una dirección IP estática (abajo se presenta un ejemplo de configuración de la dirección IP del servidor a 192.168.1.200 sobre la interfaz de red `enp0s3`, revisa el nombre de interfaz disponible con `ifconfig`)
  ```
  sudo apt update && sudo apt upgrade
  sudo nano /etc/netplan/50-cloud-init.yaml
  ```
  ```
  network:
     ethernets:
         enp0s3:
             dhcp4: no
             addresses: [192.168.1.200/24]
             gateway4: 192.168.1.1
             nameservers:
                 addresses: [8.8.8.8,8.8.4.4]
     version: 2
  ```
  ```
  sudo netplan apply
  sudo reboot
  ```
* Ansible instalado en el servidor objetivo
  ```
  sudo apt update
  sudo apt install -y software-properties-common
  sudo apt-add-repository --yes --update ppa:ansible/ansible
  sudo apt install -y ansible
  ansible --version
    ansible 2.9.6
      config file = /etc/ansible/ansible.cfg
      configured module search path = [u'/home/user/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
      ansible python module location = /usr/lib/python2.7/dist-packages/ansible
      executable location = /usr/bin/ansible
      python version = 2.7.17 (default, Nov  7 2019, 10:07:09) [GCC 7.4.0]
  ```
  
## Como ejecutarlo
Utiliza SSH para el servidor objetivo y realiza los siguientes pasos:
* Clona este repositorio (instala el paquete `git` si es necesario)
  ```
  cd ~
  git clone https://github.com/geoenvo/ansible-onedataportal.git
  cd ansible-onedataportal
  ```
* Configura las variables del playbook correspondientes
  * ```nano 0_provision_server.yml```
    ```
    server_timezone: set this to the server's timezone (default is Asia/Jakarta)
    ```
  * ```nano vars_postgresql.yml```
    ```
    postgres_user_password: the password for the default postgres role
    ```
  * ```nano vars_ckan.yml```
    ```
    ckan_site_url: set this to the IP address or domain name of the server where CKAN will be accessed (default is http://192.168.1.200)
    ckan_admin.username: set the username for the CKAN admin user (default is admin)
    ckan_admin.password: set the default password for the CKAN admin user
    ```
  * ```nano vars_oskari.yml```
    ```
    oskari_domain: set this to the IP address or domain name of the server where Oskari will be accessed (default is http://192.168.1.200:8080)
    ```
* Ejecuta los playbooks
    * Uno por uno
      * Se deben ejecutar en el orden del nombre de los archivos
        ```
        ansible-playbook -K -i inventory 0_provision_server.yml
          BECOME password: enter the user's sudo password
        ansible-playbook -K -i inventory 1_install_postgresql.yml
        ansible-playbook -K -i inventory 2_install_solr.yml
        ansible-playbook -K -i inventory 3_install_ckan.yml
        ansible-playbook -K -i inventory 4_install_ckan_extras.yml
        ansible-playbook -K -i inventory 5_install_oskari.yml
        ```
    * O ejecutar la instalación de todos los playbook para instalar todo de manera ordenada
      * Esto ejecutará los playbooks anteriores en el orden correcto de 1 al 5
        ```
        ansible-playbook -K -i inventory install_all.yml
          BECOME password: enter the user's sudo password
        ```
    * Opcional: instalar las herramientas de desarrollo para personzalizar/construir los componentes de Oskari
      * Refiere a https://oskari.org/documentation/backend/setup-development
      * PostgreSQL y PostGIS deben ser instalados primero si aún no han sido instalados previamente
        ```
        ansible-playbook -K -i inventory 1_install_postgresql.yml
          BECOME password: enter the user's sudo password
        ansible-playbook -K -i inventory 6_setup_oskari_development.yml
        ```
* Algunas configuraciones manuales pueden ser necesarias para lograr lo siguiente (esto es opcional):
  * Configurar el firewall para mejorar la seguridad del servidor
  * Hacer un proxi inverso a un (sub)dominio con certificado SSL
  * Cambiar las contraseñas predefinidas para CKAN/Oskari/GeoServer
  * Modificar configuraciones avanzadas de CKAN y Oskari
      * Configuración predefinida del trayecto del archivo de CKAN es `/etc/ckan/ckan/production.ini`
      * Configuración predefinida del trayecto del archivo de Oskari es `/opt/oskari/oskari-server/resources/oskari-ext.properties`
  * Usa y actualizar una localización no-inglesa
  * Construir el frontend de Oskari y los componentes del servidor
  * Revisar este repositorio de wiki para más detalles
* Reiniciar el servidor
  ```
  sudo reboot
  ```
* Acceso al Porta One Data en la dirección IP del servidor o dominio en un navegador web:
  * De manera predefinida el Portal Data de CKAN es accesible en http://the.server.ip.address (Proxy inverso por NGINX de http://the.server.ip.address:8090)
  * servicio de CKAN DataPusher en http://the.server.ip.address:8800/
  * servicio pycsw en http://the.server.ip.address:8000/?service=CSW&version=2.0.2&request=GetCapabilities
  * Solr admin en http://the.server.ip.address:8983/solr/
  * Oskari Geoportal en http://the.server.ip.address:8080
  * GeoServer admin en http://the.server.ip.address:8082/geoserver/web
* ¡Listo!