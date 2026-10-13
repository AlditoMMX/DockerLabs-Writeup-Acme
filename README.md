# DockerLabs-Writeup-Acme
Laboratorio/CTF autorizado de DockerLabs. Este writeup documenta únicamente la resolución de la máquina Acme.

### En esta máquina trabajamos una cadena sencilla pero muy útil para principiantes:
  - Comprobación de conectividad.
  - Enumeración con Nmap.
  - Revisión de una pista expuesta por el servicio web.
  - Enumeración del banner de SSH.
  - Obtención de credenciales temporales.
  - Acceso inicial mediante SSH.
  - Enumeración de binarios con bit SUID.
  - Escalada de privilegios abusando de Bash SUID.

## 1. Reconocimiento inicial
Comenzamos comprobando que la máquina objetivo responde en la red:

ping -c 1 172.17.0.2
