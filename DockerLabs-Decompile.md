# Decompile 
## 1. Desplegamos el laboratorio

$sudo bash auto_deploy.sh decompile.tar 

<img width="567" height="456" alt="image" src="https://github.com/user-attachments/assets/71b51fc1-c38d-4a53-85d3-03cd26598bd7" />


## 2. Reconocimiento:
### 2.1 Escaneo de puertos mediante nmap

$sudo nmap -p- -sS -sC -sV -T5 -n -Pn -vvv 172.17.0.2   

<img width="876" height="270" alt="image" src="https://github.com/user-attachments/assets/be07c056-d55a-464b-b54d-dc984cc4fb0a" />

<img width="1301" height="652" alt="image" src="https://github.com/user-attachments/assets/c3efa1e3-971d-42bf-91d8-281ce7e58428" />

Hay un archivo descargable y la pista interesante esta en *com/inmohogar/roomshare/BuildConfig.java*

$wget http://172.17.0.2/app-release.apk  

Decompilamos la aplicación

$apktool d app-release.apk

<img width="870" height="341" alt="image" src="https://github.com/user-attachments/assets/5bd213b9-f438-473f-ac3c-6872068c36bb" />

apktool está intentando decodificar el archivo AndroidManifest.xml esperando encontrar el formato binario de Android, pero el archivo ya está en texto plano. Como nuestra pista nos indica que debemos buscar la API Key en el archivo com/inmohogar/roomshare/BuildConfig.java solo nos interesa el código fuente.


## 3. Análisis

Dado que la pista menciona explícitamente un archivo .java, existe una herramienta en Kali Linux apropiada para recuperar el código Java original a partir de un APK: JADX.

$jadx-gui ./app-release.apk
<img width="862" height="250" alt="image" src="https://github.com/user-attachments/assets/a421f8ac-8720-4c4e-8a6b-da2e8e942650" />
<img width="1337" height="565" alt="image" src="https://github.com/user-attachments/assets/c7c03a7d-d612-4980-bdb4-078a2735cd31" />

**CTF_STAGE1 = "FLAG{stage1_buildconfig_leak_BKGVlytS}"**
La api key : BOOKING_API_KEY = "BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9
La URL apunta a booking.internal (BOOKING_BASE_URL = "https://booking.internal/v3/). Este es un dominio interno (o virtual host) que los servidores DNS de Internet no conocen. Como sabemos que la máquina del laboratorio está en la IP 172.17.0.2, necesitamos añadir una entrada a /etc/hosts.

$ echo "172.17.0.2 booking.internal" | sudo tee -a /etc/hosts

Ahora vamos a usar curl para enviar una petición con la api_key que hemos descubierto.

$ curl -i -H "X-API-Key: BKGVIytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" http://booking.internal/v3/
<img width="835" height="56" alt="image" src="https://github.com/user-attachments/assets/8f7e6e02-c75c-480f-8c6c-2f95faacb56b" />

*not here* Nos confirma que la petición llegó perfectamente al backend, que la cabecera ha sido procesada, pero que no hay ningún recurso configurado para mostrarse en la raíz de la ruta /v3/. Esto es muy habitual en las arquitecturas REST o de microservicios: las APIs casi nunca devuelven nada en su ruta base (/). 
Aquí es donde entra en juego nuestra siguiente táctica: Mapeo de Tecnologías (Fuzzing de Endpoints).

$ ffuf -w /usr/share/wordlists/dirb/common.txt -u http://booking.internal/FUZZ -fc 404
<img width="865" height="462" alt="image" src="https://github.com/user-attachments/assets/08f1b1b2-811e-4dcd-9adc-0dd4038879dc" />

Fuzzing profundo en la ruta prometedora (El objetivo real)
<img width="875" height="476" alt="image" src="https://github.com/user-attachments/assets/73a2b424-20f3-4208-b2d7-1a69e1426588" />

Vamos a usar curl para hacer exactamente la misma petición que hizo ffuf, pero esta vez queremos leer el cuerpo de la respuesta para ver si el desarrollador dejó algún mensaje de error útil:

$ curl -i -s -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" http://booking.internal/booking/properties
<img width="872" height="242" alt="image" src="https://github.com/user-attachments/assets/18edf9cc-13bc-465c-bca3-23c0e86fb783" />

Lo valioso aquí es esa nota final:
"_ctf_hint": "Auth OK. Try POST /sessions with tenant_account=INMO-MAD-PRIMARY"

Esto conecta perfectamente con nuestro Reconocimiento Pasivo. En el archivo BuildConfig.java vimos una variable llamada TENANT_ACCOUNT con ese mismo valor. Ahora el backend nos está invitando a interactuar con un nuevo endpoint (/sessions) utilizando el método HTTP POST y enviando ese dato.
<img width="1330" height="612" alt="image" src="https://github.com/user-attachments/assets/65125c26-cb1d-4889-883c-4598125c21cf" />

$curl -i -s -X POST -H "X-API-Key: BKGVlytSjDihCgXmNPm1cM6QwhRjrT86yJ2sEaf9" -H "Content-Type: application/json" -d '{"tenant_account": "INMO-MAD-PRIMARY"}' http://booking.internal/booking/sessions
<img width="871" height="307" alt="image" src="https://github.com/user-attachments/assets/963d7384-7163-4053-aff3-c97267908ff4" />

En la respuesta nos ha dejado la llave para la siguiente fase:
"_ctf_hint": "Auth accepted (Stage 2). Next: SSH user=deploy pwd=c0mpartir-p1s0s"

$ssh deploy@172.17.0.2
<img width="699" height="642" alt="image" src="https://github.com/user-attachments/assets/4f4313e1-1dca-40dd-ac57-da36f2562bd5" />

Hacemos un cat para leer la 3 flag:
<img width="426" height="37" alt="image" src="https://github.com/user-attachments/assets/a580290e-c2f8-4eb9-98a7-1d147e5ec060" />

**stage3_ssh_user_land_4ca5d8a311bb**

<img width="855" height="137" alt="image" src="https://github.com/user-attachments/assets/585d7d1f-4c7a-493d-8142-80ecea3900fa" />

El sistema está confirmando que el usuario deploy puede ejecutar dos comandos específicos con privilegios máximos (root o ALL) sin que se le pida contraseña (NOPASSWD):

  /opt/decompile/ssh/privesc/manage-svc
  /usr/local/bin/restart-booking-gw

Comprobar permisos

$ ls -l /opt/decompile/ssh/privesc/manage-svc /usr/local/bin/restart-booking-gw
<img width="813" height="151" alt="Escalada a root" src="https://github.com/user-attachments/assets/e5f918a5-c9a3-4ea6-b8f8-c42871934865" />











