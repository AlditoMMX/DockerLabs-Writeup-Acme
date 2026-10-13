# DockerLabs-Writeup-Acme

## 1. Descripción general
Este documento presenta el análisis técnico y la resolución de la máquina Acme, perteneciente a la plataforma DockerLabs. El objetivo es documentar de forma estructurada las etapas del proceso de evaluación de seguridad, desde el reconocimiento inicial hasta la obtención de privilegios de administrador (root).

Durante el laboratorio se identificaron mecanismos de divulgación de información en los servicios HTTP y SSH, así como una configuración insegura del binario Bash mediante el bit SUID, que permitió la escalada local de privilegios.

El procedimiento se desarrolla siguiendo una metodología de pentesting basada en la recopilación de información, la enumeración de servicios, la validación de hallazgos y el análisis de configuraciones del sistema.

**Aviso:** Este write-up tiene fines educativos y documenta actividades realizadas en un entorno de laboratorio autorizado.

## 2. Objetivos

  - Identificar los servicios expuestos por el sistema objetivo.
  - Analizar la información disponible en el servicio web.
  - Evaluar la información divulgada durante el establecimiento de una conexión SSH.
  - Obtener acceso inicial mediante credenciales identificadas durante la enumeración.
  - Examinar configuraciones locales que puedan permitir una escalada de privilegios.
  - Documentar los hallazgos y sus posibles implicaciones de seguridad.

## 3. Reconocimiento inicial

La primera fase consiste en verificar la conectividad con el objetivo y determinar si se encuentra accesible desde la máquina de trabajo.

```
ping -c 1 172.17.0.2
```

La respuesta confirma que el host se encuentra activo y accesible desde nuestra máquina atacante.

<img width="450" height="177" alt="image" src="https://github.com/user-attachments/assets/eac6da14-431a-4e85-ac3c-b7e9fd357979" />


La respuesta obtenida permite comprobar la conectividad con la dirección IP del objetivo, siempre que la red y las reglas de filtrado permitan el tráfico ICMP.

### 3.1 Enumeración de puertos y servicios

A continuación, se ejecuta un escaneo mediante Nmap para identificar los puertos TCP abiertos y recopilar información sobre los servicios disponibles.

```
nmap -sS -sCV -Pn -n 172.17.0.2
```

Las opciones utilizadas nos permiten realizar un SYN Scan, ejecutar scripts básicos de NSE e identificar versiones de los servicios detectados.

  - `-sS`: realiza un escaneo TCP SYN.
  - `-sC`: ejecuta el conjunto predeterminado de scripts de Nmap.
  - `-sV`: intenta identificar los servicios y sus versiones.
  - `-Pn`: omite el descubrimiento inicial del host.
  - `-n`: desactiva la resolución DNS.

Entre los resultados relevantes se identificaron los siguientes servicios:

```
22/tcp open  ssh   OpenSSH 8.9p1
80/tcp open  http  Apache httpd 2.4.52
```

El reconocimiento también permitió identificar una referencia a un archivo de notas de migración en `robots.txt`:

```
/_migration_notes.txt
```

Asimismo, el servidor web presenta el título:

```
ACME Corporation - Portal en Mantenimiento
```

<img width="482" height="284" alt="image" src="https://github.com/user-attachments/assets/699c65f5-f9d3-4575-94bf-6789cd4e389e" />

Estos elementos proporcionan puntos de partida para profundizar en la enumeración de los servicios expuestos.

## 4. Enumeración del servicio HTTP

Se accede al servicio web mediante el navegador o una herramienta de línea de comandos:

`
http://172.17.0.2
`

El servidor muestra un portal de infraestructura de ACME en estado de mantenimiento. El contenido incluye información que orienta la investigación hacia el servicio SSH y las consolas de gestión del sistema.

<img width="775" height="577" alt="image" src="https://github.com/user-attachments/assets/b8c00d00-c7d9-4e1f-b47e-f2beae407f27" />

### 4.1 Análisis de archivos expuestos

A partir de la referencia identificada durante el reconocimiento, se consulta el archivo de notas de migración:

```
http://172.17.0.2/migration_notes.txt
```

El documento aporta información sobre el mecanismo de acceso al servidor y señala la conveniencia de revisar el banner de conexión SSH.

<img width="880" height="282" alt="image" src="https://github.com/user-attachments/assets/f96342a9-c1f0-4681-9898-e04b2b7c01c7" />

Este hallazgo demuestra la importancia de examinar los recursos accesibles desde un servicio web, incluidos archivos auxiliares que pueden revelar información sobre la infraestructura o los mecanismos de autenticación.

Resultado de la fase: se obtiene una pista que permite orientar la siguiente etapa de enumeración hacia el servicio SSH.

## 5. Enumeración del servicio SSH

Con base en la información recopilada, se inicia una conexión SSH utilizando un nombre de usuario de prueba:

```
ssh test@172.17.0.2
```

Antes de completar el proceso de autenticación, el servidor presenta un banner de mantenimiento que expone credenciales temporales del laboratorio.

```
Usuario: usuario
Password: P@ssw0rd2026_CTF!
```

<img width="527" height="171" alt="image" src="https://github.com/user-attachments/assets/cf9fd037-94f8-4693-afc1-5e0d9184aae0" />

Este comportamiento representa una divulgación de información sensible, ya que los datos quedan disponibles antes de que el usuario complete una autenticación válida.

## Hallazgo 01 — Exposición de credenciales

  - **Descripción:** el servicio SSH revela información de autenticación mediante su mensaje de conexión.
  - **Impacto potencial:** un tercero con acceso al servicio podría obtener credenciales utilizables para intentar acceder al sistema.
  - **Recomendación:** eliminar las credenciales de los banners, evitar la divulgación de información operativa y gestionar las credenciales temporales mediante mecanismos seguros.

## 6. Acceso inicial al sistema

Con las credenciales identificadas en el entorno del laboratorio, se establece una conexión autenticada:

```
ssh usuario@172.17.0.2
```

Tras completar la autenticación, se obtiene una sesión de shell con los permisos de la cuenta usuario.

Se verifica la identidad de la sesión:

`
usuario
`

Comprobamos nuestra identidad:

```
whoami
```

A continuación, se inspecciona el directorio de trabajo para localizar la evidencia asociada al usuario:

```
ls
cat user.txt
```

La lectura del archivo permite recuperar la flag de usuario definida por el laboratorio.

**Resultado de la fase:** se confirma la obtención de acceso inicial al sistema objetivo.
`
FLAG{nmap_recon_ssh_foothold_7a9f24e1}
`

<img width="550" height="355" alt="image" src="https://github.com/user-attachments/assets/7e830b2f-7b01-4aaa-be68-f61b7cdc5456" />

Con esto obtenemos nuestro foothold o acceso inicial al sistema.

## 7. Enumeración local y análisis de permisos SUID

Una vez obtenida la sesión inicial, se inicia la enumeración local con el propósito de identificar configuraciones que puedan facilitar una escalada de privilegios.

En primer lugar, se consultan la identidad, los identificadores y los grupos asociados a la sesión:

```
id
```

Posteriormente, se buscan archivos que tengan habilitado el bit SUID:

```
find / -perm -4000 2>/dev/null
```

El comando realiza las siguientes operaciones:
  - `find /`: busca desde la raíz del sistema.
  - `-perm -4000`: localiza archivos con el bit SUID activado.
  - `2>/dev/null`: oculta los errores de permisos enviados a stderr.

Entre los resultados aparece el siguiente binario:

```
/usr/bin/bash
```

<img width="418" height="247" alt="image" src="https://github.com/user-attachments/assets/4648d2b2-f1dd-4b09-8144-46352b21824b" />

Que `/usr/bin/bash` tenga SUID activado es una configuración muy peligrosa si el binario pertenece a `root`.

## Análisis técnico

El bit SUID permite que un ejecutable se inicie con el identificador efectivo del propietario del archivo, sujeto al comportamiento del sistema operativo y del propio programa.

Cuando un intérprete de comandos como Bash tiene este permiso habilitado y pertenece a root, la configuración puede permitir que un usuario local obtenga privilegios superiores.

Por tanto, es necesario verificar tanto los permisos del archivo como su propietario y las condiciones de ejecución.

**Resultado de la fase:** se identifica una configuración potencialmente insegura que permite investigar una posible escalada local de privilegios.

## 8. Escalada de privilegios

Durante la enumeración se determina que Bash dispone de una configuración SUID que puede aprovecharse en este entorno.

Se ejecuta el siguiente comando:

```
/usr/bin/bash -p
```

La opción `-p` solicita que Bash mantenga sus privilegios efectivos en lugar de descartarlos bajo las condiciones habituales de ejecución.

Para verificar el resultado, se consulta nuevamente la identidad de la sesión:

```
whoami
```

El resultado obtenido es:

`
root
`

Esto confirma que la sesión ha alcanzado privilegios de administrador en el entorno de laboratorio.

A continuación, se accede al directorio de root y se consulta el archivo correspondiente a la flag final:

```
cd /
cd root
ls
cat root.txt
```

**Resultado de la fase:** se confirma una escalada local de privilegios desde la cuenta inicial hasta root.

`
FLAG{wpShell_cve_2026_63030_core_rce_root_99d10c8b}
`

<img width="381" height="155" alt="image" src="https://github.com/user-attachments/assets/ed3c09fe-c50d-4b12-aa7b-78e3dac73558" />

## 9. Resumen de la cadena de ataque

La resolución de la máquina puede resumirse en las siguientes etapas:
  1. R**econocimiento:** verificación de conectividad con el objetivo.
  2. **Enumeración de red:** identificación de los servicios SSH y HTTP mediante Nmap.
  3. **Análisis web:** revisión del portal de mantenimiento y de los archivos expuestos.
  4. **Enumeración SSH:** identificación de información sensible en el banner de conexión.
  5. **Acceso inicial:** autenticación mediante las credenciales obtenidas en el laboratorio.
  6. **Enumeración local:** búsqueda de archivos con permisos SUID.
  7. **Identificación del vector:** detección de Bash con una configuración SUID insegura.
  8. **Escalada de privilegios:** ejecución de Bash con la opción -p.
  9. **Verificación:** confirmación de privilegios root y recuperación de la flag final.

## 10. Hallazgos de seguridad

|          Hallazgo            |            Impacto potencial           |             Medida correctiva              |
|------------------------------|----------------------------------------|--------------------------------------------|
| Información operativa expuesta en el servicio web | Facilita la enumeración de la infraestructura | Limitar la información pública y revisar los archivos publicados |
| Credenciales expuestas en el banner SSH  | Posibilita el acceso no autorizado si las credenciales son válidas | Eliminar secretos de los banners y rotar las credenciales expuestas   |
| Bash configurado con permisos SUID         | Puede permitir la escalada local hasta root | Retirar el bit SUID y revisar los permisos de los ejecutables privilegiados |


## 11. Recomendaciones de seguridad

A partir de los hallazgos identificados, se proponen las siguientes medidas:
  - **Protección de información sensible:** evitar la publicación de credenciales, tokens, contraseñas y otros secretos en archivos web o mensajes de conexión.
  - **Endurecimiento de SSH:** revisar los banners de autenticación y aplicar políticas adecuadas de gestión de credenciales y acceso.
  - **Revisión de permisos SUID:** mantener únicamente los permisos privilegiados estrictamente necesarios y auditar periódicamente los binarios del sistema.
  - **Principio de mínimo privilegio:** limitar los permisos de las cuentas y los procesos a las funciones que realmente necesitan.
  - **Validación posterior a la corrección:** repetir las comprobaciones para verificar que los hallazgos hayan sido mitigados.

## 12. Conclusión

La máquina Acme permite analizar una cadena de ataque compuesta por varias debilidades de configuración que, en conjunto, conducen al compromiso total del sistema.

El proceso comienza con la enumeración de los servicios expuestos y el análisis de la información disponible en el servidor web. Posteriormente, la divulgación de credenciales facilita el acceso inicial mediante SSH. Finalmente, la identificación de Bash con permisos SUID permite demostrar una escalada local de privilegios hasta root.

El principal aprendizaje es que la seguridad de un sistema no depende exclusivamente de la presencia o ausencia de vulnerabilidades de software. La exposición de información sensible y los permisos incorrectamente configurados también pueden generar riesgos críticos.

Este laboratorio refuerza la importancia de aplicar una metodología sistemática de reconocimiento, enumeración, validación de hallazgos y análisis de privilegios, así como de documentar cada etapa con evidencias y recomendaciones correctivas.
