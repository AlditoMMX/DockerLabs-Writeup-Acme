# DockerLabs-Writeup-Acme
Laboratorio/CTF autorizado de DockerLabs. Este writeup documenta únicamente la resolución de la máquina Acme.

### En esta máquina trabajamos una cadena sencilla pero muy útil para principiantes:
  - Comprobación de conectividad.
  - Enumeración con **Nmap**.
  - Revisión de una pista expuesta por el servicio web.
  - Enumeración del banner de **SSH**.
  - Obtención de credenciales temporales.
  - Acceso inicial mediante SSH.
  - Enumeración de binarios con bit **SUID**.
  - Escalada de privilegios abusando de Bash **SUID**.

## 1. Reconocimiento inicial
Comenzamos comprobando que la máquina objetivo responde en la red:

```
ping -c 1 172.17.0.2
```

La respuesta confirma que el host se encuentra activo y accesible desde nuestra máquina atacante.

<img width="450" height="177" alt="image" src="https://github.com/user-attachments/assets/eac6da14-431a-4e85-ac3c-b7e9fd357979" />


## 2. Enumeración de puertos y servicios

Realizamos un escaneo completo de puertos TCP con Nmap:

```
nmap -sS -sCV -Pn -n 172.17.0.2
```

Las opciones utilizadas nos permiten realizar un SYN Scan, ejecutar scripts básicos de NSE e identificar versiones de los servicios detectados.

  - `-sS`: Escaneo TCP SYN, también conocido como SYN scan o escaneo semiabierto.
  - `-sC`: Ejecuta los scripts predeterminados de Nmap para recopilar información adicional.
  - `-sV`: Intenta identificar el servicio y su versión en los puertos abiertos.
  - `-Pn`: Omite el descubrimiento inicial del host y lo trata como activo.
  - `-n`: No realiza resolución DNS para el objetivo.

Los puertos relevantes encontrados son:

```
22/tcp open  ssh   OpenSSH 8.9p1
80/tcp open  http  Apache httpd 2.4.52
```

Además, Nmap detecta una entrada interesante en robots.txt:

```
/_migration_notes.txt
```

También identifica el título de la aplicación web:

```
ACME Corporation - Portal en Mantenimiento
```

<img width="482" height="284" alt="image" src="https://github.com/user-attachments/assets/699c65f5-f9d3-4575-94bf-6789cd4e389e" />


## 3. Enumeración del servicio web
Al acceder al servidor web mediante:

`
http://172.17.0.2
`

nos encontramos con el portal de infraestructura de ACME en modo mantenimiento.

La propia aplicación proporciona una pista muy clara: el acceso a las consolas de gestión se realiza mediante SSH y recomienda iniciar una conexión con cualquier usuario para consultar el aviso del sistema.

<img width="775" height="577" alt="image" src="https://github.com/user-attachments/assets/b8c00d00-c7d9-4e1f-b47e-f2beae407f27" />

Durante el reconocimiento también revisamos el archivo encontrado por Nmap:

```
http://172.17.0.2/migration_notes.txt
```

El memorando confirma que el acceso al servidor se canaliza por el puerto 22 y que debemos consultar el banner de conexión SSH.

<img width="880" height="282" alt="image" src="https://github.com/user-attachments/assets/f96342a9-c1f0-4681-9898-e04b2b7c01c7" />

Este paso demuestra por qué es importante revisar archivos expuestos por el servidor web y prestar atención a cualquier información obtenida durante la enumeración.

## 4. Enumeración del banner SSH
Siguiendo la pista anterior, intentamos iniciar una conexión SSH utilizando un usuario cualquiera:

```
ssh test@172.17.0.2
```

Antes de completar la autenticación, el servidor muestra un banner de mantenimiento que expone credenciales temporales:

```
Usuario: usuario
Password: P@ssw0rd2026_CTF!
```

<img width="527" height="171" alt="image" src="https://github.com/user-attachments/assets/cf9fd037-94f8-4693-afc1-5e0d9184aae0" />

Esto supone una divulgación de información sensible, ya que un usuario no autenticado puede obtener credenciales válidas simplemente iniciando una conexión al servicio SSH.

## 5. Acceso inicial mediante SSH
Utilizamos las credenciales obtenidas en el banner:

```
ssh usuario@172.17.0.2
```

Tras autenticarnos correctamente conseguimos una shell como el usuario:

`
usuario
`

Comprobamos nuestra identidad:

```
whoami
```

Después enumeramos el directorio personal y encontramos la flag de usuario:

```
ls
cat user.txt
```

Resultado:

`
FLAG{nmap_recon_ssh_foothold_7a9f24e1}
`

<img width="550" height="355" alt="image" src="https://github.com/user-attachments/assets/7e830b2f-7b01-4aaa-be68-f61b7cdc5456" />

Con esto obtenemos nuestro foothold o acceso inicial al sistema.

## 6. Enumeración local y búsqueda de SUID
Una vez dentro del sistema comenzamos la enumeración local.

Primero verificamos el UID y los grupos del usuario actual:

```
id
```

A continuación buscamos archivos que tengan activado el bit SUID:

```
find / -perm -4000 2>/dev/null
```

El significado del comando es el siguiente:
  - `find /`: busca desde la raíz del sistema.
  - `-perm -4000`: localiza archivos con el bit SUID activado.
  - `2>/dev/null`: oculta los errores de permisos enviados a stderr.

Entre los resultados aparece:

```
/usr/bin/bash
```

<img width="418" height="247" alt="image" src="https://github.com/user-attachments/assets/4648d2b2-f1dd-4b09-8144-46352b21824b" />

Que `/usr/bin/bash` tenga SUID activado es una configuración muy peligrosa si el binario pertenece a `root`.

El bit SUID permite que un programa se ejecute utilizando el UID efectivo del propietario del archivo en lugar del UID del usuario que lo ejecuta.

## 7. Escalada de privilegios con Bash SUID
Aprovechamos la configuración insegura ejecutando Bash en modo privilegiado:

```
/usr/bin/bash -p
```

La opción -p hace que Bash conserve los privilegios efectivos en lugar de descartarlos.

Comprobamos nuestra identidad:

```
whoami
```

El resultado es:

`
root
`

Ya con privilegios máximos navegamos al directorio de root:

```
cd /
cd root
ls
cat root.txt
```

Y obtenemos la flag final:

`
FLAG{wpShell_cve_2026_63030_core_rce_root_99d10c8b}
`

<img width="381" height="155" alt="image" src="https://github.com/user-attachments/assets/ed3c09fe-c50d-4b12-aa7b-78e3dac73558" />

## 8. Resumen de la cadena de ataque

  1. Reconocimiento
  2. Ping
  3. Nmap
  4. Puerto 80 + robots.txt
  5. /migration_notes.txt
  6. Pista para revisar SSH
  7. Banner SSH
  8. Credenciales temporales expuestas
  9. SSH como usuario
  10. Enumeración SUID
  11. /usr/bin/bash con SUID
  12. /usr/bin/bash -p
  13. ROOT

## 9. Vulnerabilidades encontradas

|       Vulnerabilidad / mala configuración        |                 Impacto                  |
|--------------------------------------------------|------------------------------------------|
| Información sensible expuesta en el servicio web | Facilita la enumeración del servicio SSH |
| Credenciales expuestas en banner SSH             | Acceso inicial con usuario válido        |
| Bash con bit SUID activado                       | Escalada local directa a `root`          |

## 10. Conclusión
Acme es una máquina muy sencilla, pero resulta útil para practicar una metodología básica de pentesting:

  1. Enumerar
  2. analizar pistas
  3. obtener acceso
  4. enumerar localmente
  5. escalar privilegios

La máquina también deja una lección importante: no siempre es necesario utilizar un exploit complejo o un CVE. Una mala configuración aparentemente pequeña, como exponer credenciales en un banner o asignar SUID a Bash, puede terminar provocando la comprometición total del sistema.
