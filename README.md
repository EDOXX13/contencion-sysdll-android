Contencion defensiva de `co.sitic.pp` / SysDLL en Android mediante ADB sin root
Guia reproducible y anonimizada basada en pruebas reales sobre dos implementaciones de `co.sitic.pp`: Xiaomi y Nubia.
El objetivo es documentar por separado que funciono, que fallo y que persiste despues de reinicios o actualizaciones del sistema. No se asume que un procedimiento valido para una marca funcione en otra.
> **Alcance defensivo:** esta guia esta destinada a propietarios, administradores e investigadores autorizados. No contiene IMEI, numeros de serie, PINes, numeros telefonicos ni mensajes capaces de activar funciones de bloqueo/desbloqueo.
---
Indice
Resumen comparativo
Dispositivo probado 1: Xiaomi 24090RA29G
Dispositivo probado 2: Nubia Z2468N
Comparacion de resultados
Procedimiento generico para otros dispositivos
Analisis del APK con JADX
Verificacion con Logcat
Comandos de reversion
Advertencias y limitaciones
Registro de cambios
---
1. Resumen comparativo
Caracteristica	Xiaomi 24090RA29G	Nubia Z2468N
Paquete	`co.sitic.pp`	`co.sitic.pp`
Nombre observado	SysDLL	Sysdll_CLA_CL
Android observado	Android 16 / HyperOS	Android 15, API 35
Device	No conservado en esta guia	`P780F02`
Ruta APK	`/product/priv-app/SysDLL/`	`/system/priv-app/Sysdll_CLA_CL/`
Tipo	`SYSTEM + PRIVILEGED + PRODUCT`	`SYSTEM + PRIVILEGED`
UID observado	`10168`	`10288`
Servicio	`.ServicioRegistro`	`.ServicioRegistro`
Suspender paquete	Funciono	Funciono
Servicio siguio ejecutandose	Si	Si
Force-stop	Inefectivo	Prohibido por firmware
Desinstalar por ADB	Fallo	Prohibido por firmware
Revocar `RECEIVE_SMS`	No fue posible	Si fue posible
`RECEIVE_SMS granted=false`	No	Si
`RUN_IN_BACKGROUND=ignore`	Funciono	Funciono y persistio
`RUN_ANY_IN_BACKGROUND=ignore`	El firmware llego a restaurarlo	Funciono y persistio
`WAKE_LOCK=ignore`	No fue la prueba principal	Funciono y persistio
Blacklist NetPolicy	Probada	Probada, no persistio reinicio
Analisis JADX de SMS	Si	Pendiente
Prueba SMS real / Logcat	Si	Pendiente, equipo sin SIM durante la prueba
OTA auditada	No	Si
Leyenda:
Funciona: comprobado en el equipo indicado.
Falla/prohibido: comprobado que el firmware lo impide o no logra el efecto deseado.
Pendiente: no probado en ese dispositivo.
---
2. Dispositivo probado 1: Xiaomi 24090RA29G
2.1 Caracteristicas observadas
Fabricante: Xiaomi.
Modelo observado: `24090RA29G`.
Sistema durante las pruebas: Android 16 / HyperOS.
Paquete: `co.sitic.pp`.
Nombre/ruta observada: SysDLL.
Ruta base:
```text
/product/priv-app/SysDLL
```
Tipo de aplicacion:
```text
SYSTEM
PRIVILEGED
PRODUCT
```
UID observado durante las pruebas:
```text
10168
```
2.2 Identificacion
```bat
adb shell pm path co.sitic.pp
adb shell dumpsys package co.sitic.pp | findstr /i "codePath system privileged enabled installed suspended"
adb shell pm list packages -U | findstr /i "co.sitic.pp"
adb shell pidof co.sitic.pp
```
La aplicacion se encontraba en la particion `/product` como aplicacion privilegiada.
2.3 Intento de desinstalacion
```bat
adb shell pm uninstall --user 0 co.sitic.pp
```
Resultado observado:
```text
Failure [-1000]
```
Conclusion: Package Manager no permitio retirar la aplicacion para el usuario 0.
2.4 Intento de deshabilitacion
```bat
adb shell pm disable-user --user 0 co.sitic.pp
```
Resultado relevante:
```text
java.lang.SecurityException: Cannot change carrier core packages component state.
```
Tambien se intento modificar componentes individuales como:
```text
co.sitic.pp/.CellBroadcastReceiver
co.sitic.pp/.ServicioRegistro
```
El firmware rechazo el cambio del estado de los componentes desde ADB shell.
2.5 Suspension
```bat
adb shell cmd package suspend --user 0 co.sitic.pp
```
Se obtuvo:
```text
suspended=true
```
Sin embargo:
```bat
adb shell pidof co.sitic.pp
```
continuo devolviendo un PID.
Conclusion: `suspended=true` no neutralizo por si solo el servicio privilegiado.
2.6 Servicio activo
Se identifico:
```text
co.sitic.pp/.ServicioRegistro
```
En `dumpsys activity services` aparecio con informacion equivalente a:
```text
SYSTEM_ALLOW_LISTED
startRequested=true
```
El proceso permanecio activo incluso despues de intentar `force-stop`.
2.7 Receptores de arranque
Se encontro un receptor:
```text
co.sitic.pp/.CellBroadcastReceiver
```
asociado a:
```text
android.intent.action.LOCKED_BOOT_COMPLETED
android.intent.action.BOOT_COMPLETED
```
Esto ayuda a explicar por que el paquete puede volver a iniciar componentes durante el arranque.
2.8 `RECEIVE_SMS` en Xiaomi
El permiso aparecio como:
```text
android.permission.RECEIVE_SMS: granted=true
flags=[SYSTEM_FIXED|GRANTED_BY_DEFAULT|RESTRICTION_SYSTEM_EXEMPT|RESTRICTION_UPGRADE_EXEMPT]
```
Se intento:
```bat
adb shell appops set co.sitic.pp RECEIVE_SMS ignore
```
pero el firmware restauro `RECEIVE_SMS: allow`.
Tambien:
```bat
adb shell pm revoke --user 0 co.sitic.pp android.permission.RECEIVE_SMS
```
fallo porque ADB shell no disponia de:
```text
android.permission.REVOKE_RUNTIME_PERMISSIONS
```
Conclusion Xiaomi: durante estas pruebas no fue posible revocar de forma persistente `RECEIVE_SMS` sin privilegios superiores.
2.9 Segundo plano
Se aplicaron:
```bat
adb shell appops set co.sitic.pp RUN_IN_BACKGROUND ignore
adb shell appops set co.sitic.pp RUN_ANY_IN_BACKGROUND ignore
```
`RUN_IN_BACKGROUND` permanecio restringido. Durante las pruebas el firmware llego a restaurar `RUN_ANY_IN_BACKGROUND` a `allow`, por lo que debe verificarse despues de reiniciar.
2.10 NetPolicy
El UID `10168` aparecio inicialmente en una whitelist de datos en segundo plano.
Se retiro:
```bat
adb shell cmd netpolicy remove restrict-background-whitelist 10168
```
Se activo Data Saver:
```bat
adb shell cmd netpolicy set restrict-background true
```
Y se probo:
```bat
adb shell cmd netpolicy add restrict-background-blacklist 10168
```
Importante: NetPolicy restringe datos en segundo plano. No debe describirse como un firewall absoluto.
2.11 Analisis JADX del receptor SMS
Se extrajo el APK y se analizo con JADX.
Se identifico la clase:
```text
co.sitic.pp.ReceptorSMS
```
que extiende `BroadcastReceiver` y comprueba explicitamente:
```text
android.provider.Telephony.SMS_RECEIVED
```
El codigo reconstruye mensajes desde `pdus` y utiliza:
```text
SmsMessage.createFromPdu(...)
getOriginatingAddress()
getMessageBody()
```
Por lo tanto, la variante analizada contiene logica capaz de obtener el remitente y el contenido de un SMS recibido.
Tambien se identifico logica interna de validacion que, bajo condiciones especificas, puede terminar llamando una funcion de bloqueo del dispositivo. Los formatos concretos de control se omiten deliberadamente.
2.12 Prueba controlada con SMS y Logcat
En una prueba con un SMS inocuo se observo actividad SMS y, pocos segundos despues, actividad del paquete:
```text
co.sitic.pp/.jTarea
```
`dumpsys activity broadcasts` mostro un receptor del proceso `co.sitic.pp` asociado a:
```text
android.provider.Telephony.SMS_RECEIVED
```
Conclusion: no se trato solamente de un permiso declarado. Se comprobo la presencia de un receptor activo para `SMS_RECEIVED` y actividad temporal del paquete durante la prueba.
---
3. Dispositivo probado 2: Nubia Z2468N
3.1 Caracteristicas observadas
Fabricante: Nubia.
Modelo: `Z2468N`.
Device: `P780F02`.
Android: 15.
API: 35.
Verified Boot observado: `green`.
Paquete: `co.sitic.pp`.
UID: `10288`.
Ruta:
```text
/system/priv-app/Sysdll_CLA_CL/Sysdll_CLA_CL.apk
```
Tipo:
```text
SYSTEM
PRIVILEGED
```
Version observada tras la actualizacion auditada:
```text
versionCode=1110
versionName=0.06.1.30 GMS
```
3.2 Suspension
```bat
adb shell cmd package suspend --user 0 co.sitic.pp
```
Resultado:
```text
suspended=true
```
La suspension sobrevivio a reinicios normales y tambien permanecio despues de la actualizacion OTA auditada.
3.3 Servicio activo
A pesar de la suspension:
```bat
adb shell pidof co.sitic.pp
```
continuo devolviendo un PID.
Se identifico nuevamente:
```text
co.sitic.pp/.ServicioRegistro
```
con excepcion equivalente a:
```text
SYSTEM_ALLOW_LISTED
```
3.4 Force-stop
```bat
adb shell am force-stop --user 0 co.sitic.pp
```
Resultado observado:
```text
Error: Force stop of package is forbidden
```
Conclusion: el firmware Nubia protege explicitamente el paquete contra `force-stop` desde ADB.
3.5 Desinstalacion
```bat
adb shell pm uninstall --user 0 co.sitic.pp
```
Resultado observado:
```text
disallow uninstall co.sitic.pp by adb
```
La lista de usuarios mostro solamente:
```text
User 0: Propietario
```
Por lo tanto, no habia otro usuario Android real sobre el que probar una desinstalacion.
3.6 Revocacion exitosa de `RECEIVE_SMS`
Antes de la revocacion:
```text
android.permission.RECEIVE_SMS: granted=true
flags=[GRANTED_BY_DEFAULT|RESTRICTION_SYSTEM_EXEMPT|RESTRICTION_UPGRADE_EXEMPT]
```
A diferencia del Xiaomi probado, no aparecia `SYSTEM_FIXED`.
Se ejecuto:
```bat
adb shell pm revoke --user 0 co.sitic.pp android.permission.RECEIVE_SMS
adb shell appops set co.sitic.pp RECEIVE_SMS ignore
```
Resultado:
```text
android.permission.RECEIVE_SMS: granted=false
RECEIVE_SMS: ignore
```
El estado `granted=false` sobrevivio a reinicios normales.
Nota sobre AppOps duplicados
En algunas comprobaciones `appops get` mostro mas de una entrada relacionada con `RECEIVE_SMS`, por ejemplo `ignore` y `allow`.
Para determinar si el permiso runtime esta realmente concedido, la comprobacion principal utilizada fue:
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS"
```
El indicador determinante es:
```text
granted=false
```
3.7 Restricciones de segundo plano
Se aplicaron:
```bat
adb shell appops set co.sitic.pp RUN_IN_BACKGROUND ignore
adb shell appops set co.sitic.pp RUN_ANY_IN_BACKGROUND ignore
```
Se verifico:
```text
RUN_IN_BACKGROUND: ignore
RUN_ANY_IN_BACKGROUND: ignore
```
Ambos valores sobrevivieron a reinicios normales y estuvieron presentes despues de la OTA auditada.
3.8 `WAKE_LOCK`
Se aplico:
```bat
adb shell appops set co.sitic.pp WAKE_LOCK ignore
```
Se verifico:
```text
WAKE_LOCK: ignore
```
La restriccion sobrevivio al reinicio y a la posterior auditoria tras la actualizacion.
3.9 NetPolicy
Inicialmente el UID `10288` aparecia en:
```text
restrict-background-whitelist
```
Se retiro:
```bat
adb shell cmd netpolicy remove restrict-background-whitelist 10288
```
Se agrego temporalmente a blacklist:
```bat
adb shell cmd netpolicy add restrict-background-blacklist 10288
```
Y se activo:
```bat
adb shell cmd netpolicy set restrict-background true
```
Persistencia observada
Despues de reiniciar:
Data Saver siguio `enabled`.
UID `10288` siguio fuera de whitelist.
La blacklist quedo vacia.
Por lo tanto:
```text
Blacklist NetPolicy: NO persistente en el firmware probado.
```
3.10 Auditoria despues de actualizacion OTA
Despues de una actualizacion del sistema se reviso nuevamente el dispositivo.
Se conservaron:
```text
suspended=true
WAKE_LOCK=ignore
RUN_IN_BACKGROUND=ignore
RUN_ANY_IN_BACKGROUND=ignore
restrict-background=enabled
UID 10288 fuera de whitelist
```
Pero el firmware restauro:
```text
android.permission.RECEIVE_SMS: granted=true
```
Esto demuestra una diferencia critica:
> Una restriccion puede sobrevivir a reinicios normales y, aun asi, ser restablecida por una actualizacion OTA.
Se reaplico:
```bat
adb shell pm revoke --user 0 co.sitic.pp android.permission.RECEIVE_SMS
adb shell appops set co.sitic.pp RECEIVE_SMS ignore
```
Y se recupero:
```text
android.permission.RECEIVE_SMS: granted=false
```
3.11 Lista de auditoria Nubia post-OTA
```bat
adb shell getprop ro.build.fingerprint
adb shell pm path co.sitic.pp
adb shell pm list packages -U | findstr /i "co.sitic.pp"
adb shell dumpsys package co.sitic.pp | findstr /i "versionName versionCode codePath firstInstallTime lastUpdateTime"
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS suspended"
adb shell appops get co.sitic.pp | findstr /i "RECEIVE_SMS WAKE_LOCK RUN_IN_BACKGROUND RUN_ANY_IN_BACKGROUND"
adb shell cmd netpolicy get restrict-background
adb shell cmd netpolicy list restrict-background-whitelist
adb shell cmd netpolicy list restrict-background-blacklist
adb shell pidof co.sitic.pp
adb shell dumpsys activity services co.sitic.pp
```
---
4. Comparacion de resultados
4.1 Diferencia principal: SMS
Xiaomi
`RECEIVE_SMS` estaba protegido con `SYSTEM_FIXED`; ADB shell no pudo revocarlo y AppOps era restaurado por el firmware.
Nubia
`RECEIVE_SMS` no mostraba `SYSTEM_FIXED`; `pm revoke` funciono y produjo:
```text
granted=false
```
Sin embargo, una OTA volvio a conceder el permiso, por lo que debe auditarse despues de cada actualizacion.
4.2 Diferencia principal: ubicacion del APK
Xiaomi:
```text
/product/priv-app/SysDLL
```
Nubia:
```text
/system/priv-app/Sysdll_CLA_CL
```
Ambas instalaciones eran privilegiadas, pero eso no implica que tengan el mismo comportamiento ni la misma politica OEM.
4.3 Similitudes observadas
En ambos dispositivos:
`co.sitic.pp` era aplicacion del sistema privilegiada.
`cmd package suspend` fue aceptado.
El proceso/servicio podia continuar con `suspended=true`.
Se encontro `.ServicioRegistro` activo.
Los mecanismos normales de retiro/desactivacion estaban restringidos por el firmware.
Era necesario verificar cada control despues de reiniciar.
---
5. Procedimiento generico para otros dispositivos
No copies directamente UID, rutas o conclusiones de Xiaomi/Nubia.
5.1 Identificar equipo
```bat
adb devices
adb shell getprop ro.product.manufacturer
adb shell getprop ro.product.model
adb shell getprop ro.product.device
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk
adb shell getprop ro.build.fingerprint
adb shell getprop ro.boot.verifiedbootstate
```
5.2 Detectar paquete
```bat
adb shell pm list packages -s | findstr /i "sitic"
adb shell pm path co.sitic.pp
adb shell pm list packages -U | findstr /i "co.sitic.pp"
```
5.3 Capturar estado original
```bat
adb shell dumpsys package co.sitic.pp
adb shell appops get co.sitic.pp
adb shell pidof co.sitic.pp
adb shell dumpsys activity services co.sitic.pp
adb shell cmd netpolicy get restrict-background
adb shell cmd netpolicy list restrict-background-whitelist
adb shell cmd netpolicy list restrict-background-blacklist
```
Guarda las salidas antes de modificar nada.
5.4 Suspender
```bat
adb shell cmd package suspend --user 0 co.sitic.pp
adb shell dumpsys package co.sitic.pp | findstr /i "suspended"
```
5.5 Evaluar `RECEIVE_SMS`
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS"
adb shell appops get co.sitic.pp | findstr /i "RECEIVE_SMS"
```
Si el firmware permite revocarlo:
```bat
adb shell pm revoke --user 0 co.sitic.pp android.permission.RECEIVE_SMS
adb shell appops set co.sitic.pp RECEIVE_SMS ignore
```
Verifica `granted=false` antes de considerar exitosa la revocacion.
5.6 Segundo plano y WakeLock
```bat
adb shell appops set co.sitic.pp RUN_IN_BACKGROUND ignore
adb shell appops set co.sitic.pp RUN_ANY_IN_BACKGROUND ignore
adb shell appops set co.sitic.pp WAKE_LOCK ignore
```
5.7 NetPolicy
Obtiene el UID primero. Luego:
```bat
adb shell cmd netpolicy remove restrict-background-whitelist <UID>
adb shell cmd netpolicy add restrict-background-blacklist <UID>
adb shell cmd netpolicy set restrict-background true
```
No describas esta configuracion como bloqueo total de Internet. Comprueba su persistencia despues del reinicio.
5.8 Verificacion posterior
```bat
adb reboot
```
Despues:
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS suspended"
adb shell appops get co.sitic.pp | findstr /i "RECEIVE_SMS WAKE_LOCK RUN_IN_BACKGROUND RUN_ANY_IN_BACKGROUND"
adb shell cmd netpolicy list restrict-background-whitelist
adb shell cmd netpolicy list restrict-background-blacklist
adb shell pidof co.sitic.pp
```
Repite la auditoria tambien despues de cada OTA.
---
6. Analisis del APK con JADX
Primero obten la ruta real:
```bat
adb shell pm path co.sitic.pp
```
Copia el APK usando esa ruta:
```bat
adb pull /ruta/mostrada/por/pm/path.apk
```
En JADX GUI realiza busquedas globales de:
```text
SMS_RECEIVED
ReceptorSMS
registerReceiver
ServicioRegistro
jTarea
SmsMessage
getMessageBody
getOriginatingAddress
```
El analisis debe usarse para comprender componentes y riesgos, no para generar mensajes capaces de activar funciones de control.
---
7. Verificacion con Logcat
Solo sobre un dispositivo propio o autorizado y usando mensajes inocuos.
```bat
adb logcat -c
adb logcat -v time | findstr /i "SMS SMS_RECEIVED co.sitic.pp ReceptorSMS jTarea"
```
Despues de una prueba controlada tambien puede consultarse:
```bat
adb shell dumpsys activity broadcasts | findstr /i "co.sitic.pp SMS_RECEIVED"
adb shell dumpsys activity services co.sitic.pp
```
Que una aplicacion tenga `RECEIVE_SMS` concedido demuestra capacidad, pero para afirmar que proceso un SMS concreto conviene correlacionar registros y actividad durante una prueba controlada.
---
8. Comandos de reversion
Antes de modificar cualquier configuracion guarda el estado inicial.
Quitar suspension
```bat
adb shell cmd package unsuspend --user 0 co.sitic.pp
```
Restaurar AppOps modificados
```bat
adb shell appops set co.sitic.pp RUN_IN_BACKGROUND default
adb shell appops set co.sitic.pp RUN_ANY_IN_BACKGROUND default
adb shell appops set co.sitic.pp WAKE_LOCK default
adb shell appops set co.sitic.pp RECEIVE_SMS default
```
Restaurar `RECEIVE_SMS`, si corresponde
```bat
adb shell pm grant --user 0 co.sitic.pp android.permission.RECEIVE_SMS
```
Retirar blacklist NetPolicy
```bat
adb shell cmd netpolicy remove restrict-background-blacklist <UID>
```
Desactivar Data Saver global si se activo solo para pruebas
```bat
adb shell cmd netpolicy set restrict-background false
```
Este ultimo comando es global y afecta a otras aplicaciones.
---
9. Advertencias y limitaciones
`suspended=true` no significa proceso muerto
En ambos dispositivos se observo un proceso/servicio activo a pesar de la suspension.
AppOps no equivale siempre al permiso runtime
Para `RECEIVE_SMS`, comprueba tambien:
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS"
```
Data Saver no es un firewall absoluto
`netpolicy` sirve para politicas de datos en segundo plano. No garantiza aislamiento completo de red para una aplicacion privilegiada.
No modificar `/system` o `/product` a ciegas
Eliminar manualmente el APK desde una particion verificada puede provocar fallos de arranque o problemas con Android Verified Boot.
Las OTA pueden cambiar el resultado
El caso Nubia demostro que una OTA puede restaurar un permiso que habia permanecido revocado durante reinicios normales.
Las implementaciones OEM son diferentes
No generalices un resultado Xiaomi a Nubia, Samsung, Motorola, ZTE u otros fabricantes sin volver a ejecutar la fase de identificacion y auditoria.
---
10. Registro de cambios
2026-11 - Separacion por fabricantes
Separados los resultados Xiaomi y Nubia.
Anadida matriz comparativa de pruebas.
Documentadas rutas APK, UID, sistema y comportamiento observado para cada equipo.
Separados resultados comprobados de procedimientos genericos.
Anadida auditoria post-OTA especifica del Nubia.
Documentada la diferencia de `RECEIVE_SMS`: protegido con `SYSTEM_FIXED` en el Xiaomi probado y revocable en el Nubia probado.
2026-11 - Hallazgo post-OTA Nubia
Una OTA restauro `RECEIVE_SMS` de `granted=false` a `granted=true`.
La suspension, `WAKE_LOCK`, `RUN_IN_BACKGROUND` y `RUN_ANY_IN_BACKGROUND` permanecieron.
Se reaplico la revocacion de `RECEIVE_SMS` exitosamente.
---
Privacidad
Este documento ha sido anonimizado. No contiene:
numeros de serie;
IMEI;
numeros telefonicos personales;
nombres de propietarios;
PINes;
cargas SMS operativas de bloqueo/desbloqueo.
Licencia / uso
La informacion se presenta con fines de administracion, investigacion y contencion defensiva de dispositivos propios o expresamente autorizados.
