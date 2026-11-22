Contencion de `co.sitic.pp` / SysDLL en Android con ADB sin root
Guia defensiva, reproducible y anonimizada para propietarios, administradores e investigadores autorizados. Resume pruebas realizadas en dispositivos Android donde `co.sitic.pp` viene preinstalado como aplicacion `SYSTEM/PRIVILEGED`.
> Esta guia busca reducir capacidades sensibles sin modificar `/system` o `/product`, sin root y sin desbloquear el bootloader. No incluye cargas SMS, PINes, IMEI ni instrucciones para activar bloqueos.
Advertencias
Los resultados dependen del fabricante y firmware.
Registra el estado original antes de cada cambio.
No borres manualmente APK de `/system/priv-app` o `/product/priv-app`. Android Verified Boot puede impedir el arranque.
No copies numeros de transaccion de `service call package` entre versiones de Android.
`suspended=true` no garantiza que un servicio privilegiado se detenga.
Data Saver y `netpolicy` no son un firewall absoluto.
1. Requisitos
Windows y cable USB de datos.
Android SDK Platform-Tools.
Depuracion USB activada y autorizada.
Opcional: JADX GUI para analizar el APK.
```bat
adb devices
```
El equipo debe aparecer como `device`, no `unauthorized`.
2. Identificar dispositivo, compilacion y AVB
```bat
adb shell getprop ro.product.manufacturer
adb shell getprop ro.product.model
adb shell getprop ro.product.device
adb shell getprop ro.build.version.release
adb shell getprop ro.build.version.sdk
adb shell getprop ro.build.fingerprint
adb shell getprop ro.boot.verifiedbootstate
```
Si Verified Boot devuelve `green`, evita modificar particiones verificadas.
3. Detectar `co.sitic.pp`
```bat
adb shell pm list packages -s | findstr /i "sitic"
adb shell pm path co.sitic.pp
adb shell pm list packages -U | findstr /i "co.sitic.pp"
```
Rutas observadas:
```text
/product/priv-app/SysDLL/SysDLL.apk
/system/priv-app/Sysdll_CLA_CL/Sysdll_CLA_CL.apk
```
No asumas que todas las variantes son iguales. Guarda el UID devuelto, porque cambia entre dispositivos.
4. Estado inicial
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "codePath versionName versionCode system privileged installed suspended"
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS"
adb shell appops get co.sitic.pp
adb shell pidof co.sitic.pp
adb shell dumpsys activity services co.sitic.pp
```
En las pruebas se observo el servicio:
```text
co.sitic.pp/.ServicioRegistro
```
Tambien se observo `SYSTEM_ALLOW_LISTED`, por lo que el sistema podia mantenerlo activo incluso con restricciones normales.
5. Suspender el paquete
```bat
adb shell cmd package suspend --user 0 co.sitic.pp
adb shell dumpsys package co.sitic.pp | findstr /i "suspended"
```
Resultado deseado:
```text
suspended=true
```
Comprueba si el proceso sigue vivo:
```bat
adb shell pidof co.sitic.pp
adb shell dumpsys activity services co.sitic.pp
```
Para revertir:
```bat
adb shell cmd package unsuspend --user 0 co.sitic.pp
```
6. Probar detencion y desinstalacion
```bat
adb shell am force-stop --user 0 co.sitic.pp
adb shell pidof co.sitic.pp
```
Algunos firmwares prohiben expresamente el `force-stop`.
Para comprobar si se permite retirar el paquete del usuario principal:
```bat
adb shell pm uninstall --user 0 co.sitic.pp
```
Algunos firmwares responden que la desinstalacion por ADB esta prohibida. No inventes otros IDs. Lista usuarios reales:
```bat
adb shell pm list users
```
Si solo existe `User 0`, probar `--user 10`, `999`, etc. no aporta una via valida.
7. Restringir segundo plano
```bat
adb shell appops set co.sitic.pp RUN_IN_BACKGROUND ignore
adb shell appops set co.sitic.pp RUN_ANY_IN_BACKGROUND ignore
adb shell appops get co.sitic.pp | findstr /i "RUN_IN_BACKGROUND RUN_ANY_IN_BACKGROUND"
```
Resultado deseado:
```text
RUN_IN_BACKGROUND: ignore
RUN_ANY_IN_BACKGROUND: ignore
```
Para revertir:
```bat
adb shell appops set co.sitic.pp RUN_IN_BACKGROUND default
adb shell appops set co.sitic.pp RUN_ANY_IN_BACKGROUND default
```
8. Restringir `WAKE_LOCK`
```bat
adb shell appops set co.sitic.pp WAKE_LOCK ignore
adb shell appops get co.sitic.pp | findstr /i "WAKE_LOCK"
```
Para revertir:
```bat
adb shell appops set co.sitic.pp WAKE_LOCK default
```
9. Restringir `RECEIVE_SMS`
Comprueba ambos niveles:
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS"
adb shell appops get co.sitic.pp | findstr /i "RECEIVE_SMS"
```
9.1 AppOp
```bat
adb shell appops set co.sitic.pp RECEIVE_SMS ignore
```
Algunos firmwares lo restauran inmediatamente a `allow`.
9.2 Permiso runtime
Si el firmware lo permite:
```bat
adb shell pm revoke --user 0 co.sitic.pp android.permission.RECEIVE_SMS
```
Verifica:
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS"
```
Resultado determinante:
```text
android.permission.RECEIVE_SMS: granted=false
```
`appops get` puede mostrar entradas duplicadas, como `ignore` y `allow`. Para saber si el permiso runtime esta concedido, usa como fuente principal `dumpsys package` y su linea `granted=true/false`.
En un firmware con `SYSTEM_FIXED`, ADB shell no pudo revocarlo. En otro firmware sin esa bandera, la revocacion funciono y sobrevivio a reinicios normales.
Para restaurar voluntariamente el permiso:
```bat
adb shell pm grant --user 0 co.sitic.pp android.permission.RECEIVE_SMS
adb shell appops set co.sitic.pp RECEIVE_SMS default
```
10. Restringir datos en segundo plano
Obtiene el UID real:
```bat
adb shell pm list packages -U | findstr /i "co.sitic.pp"
```
Sustituye `<UID>`:
```bat
adb shell cmd netpolicy list restrict-background-whitelist
adb shell cmd netpolicy list restrict-background-blacklist
adb shell cmd netpolicy remove restrict-background-whitelist <UID>
adb shell cmd netpolicy add restrict-background-blacklist <UID>
adb shell cmd netpolicy set restrict-background true
```
Verifica:
```bat
adb shell cmd netpolicy get restrict-background
adb shell cmd netpolicy list restrict-background-whitelist
adb shell cmd netpolicy list restrict-background-blacklist
```
Una prueba mostro que la blacklist desaparecia tras reiniciar, aunque el UID permanecia fuera de whitelist y Data Saver seguia activo.
Para revertir:
```bat
adb shell cmd netpolicy remove restrict-background-blacklist <UID>
```
Solo reincorpora el UID a whitelist si quieres restaurar expresamente esa excepcion.
11. Verificacion despues de reiniciar
```bat
adb reboot
```
Despues del arranque:
```bat
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS suspended"
adb shell appops get co.sitic.pp | findstr /i "RECEIVE_SMS WAKE_LOCK RUN_IN_BACKGROUND RUN_ANY_IN_BACKGROUND"
adb shell cmd netpolicy get restrict-background
adb shell cmd netpolicy list restrict-background-whitelist
adb shell cmd netpolicy list restrict-background-blacklist
adb shell pidof co.sitic.pp
adb shell dumpsys activity services co.sitic.pp
```
No consideres persistente un cambio hasta verificarlo despues de reiniciar.
12. Auditoria obligatoria despues de una actualizacion OTA
Una actualizacion del sistema puede restaurar permisos aunque hayan sobrevivido a reinicios normales.
Hallazgo reproducible
Antes de una actualizacion OTA se mantenia:
```text
android.permission.RECEIVE_SMS: granted=false
```
Despues de actualizar, la ruta del APK, UID, suspension y AppOps principales continuaban, pero el firmware restauro:
```text
android.permission.RECEIVE_SMS: granted=true
```
Permanecieron:
```text
suspended=true
WAKE_LOCK=ignore
RUN_IN_BACKGROUND=ignore
RUN_ANY_IN_BACKGROUND=ignore
restrict-background=enabled
UID fuera de restrict-background-whitelist
```
Lista de comprobacion post-actualizacion
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
Si `RECEIVE_SMS` vuelve a `granted=true` y el firmware permite revocarlo:
```bat
adb shell pm revoke --user 0 co.sitic.pp android.permission.RECEIVE_SMS
adb shell appops set co.sitic.pp RECEIVE_SMS ignore
adb shell dumpsys package co.sitic.pp | findstr /i "RECEIVE_SMS"
adb shell appops get co.sitic.pp | findstr /i "RECEIVE_SMS"
```
La comprobacion principal debe volver a ser:
```text
android.permission.RECEIVE_SMS: granted=false
```
Conclusion: que una restriccion sobreviva a reinicios no significa que sobrevivira a una actualizacion OTA.
13. Copiar el APK para analisis
Usa la ruta exacta de `pm path`:
```bat
adb pull /ruta/real/al/APK.apk
```
Ejemplos observados:
```bat
adb pull /product/priv-app/SysDLL/SysDLL.apk
adb pull /system/priv-app/Sysdll_CLA_CL/Sysdll_CLA_CL.apk
```
Esto solo copia el APK, no modifica el telefono.
14. Analisis defensivo con JADX
Abre el APK en JADX GUI y busca:
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
En una variante se identifico `co.sitic.pp.ReceptorSMS`, que:
extendia `BroadcastReceiver`;
comprobaba `android.provider.Telephony.SMS_RECEIVED`;
reconstruia mensajes desde `pdus`;
obtenia remitente y cuerpo del SMS;
aplicaba logica interna que podia terminar invocando una funcion de bloqueo.
Esto explica por que `READ_SMS: ignore` no basta: recibir un SMS por broadcast es distinto de consultar la base de mensajes.
Esta guia excluye deliberadamente los formatos de control y cualquier dato capaz de activar bloqueo o desbloqueo.
15. Verificacion controlada con Logcat
Solo en un equipo propio o autorizado y con un SMS inocuo:
```bat
adb logcat -c
adb logcat -v time | findstr /i "SMS SMS_RECEIVED co.sitic.pp ReceptorSMS jTarea"
```
Envia un mensaje de prueba normal, espera unos segundos y termina con `Ctrl+C`. No uses formatos obtenidos mediante ingenieria inversa.
Tambien puedes revisar:
```bat
adb shell dumpsys activity broadcasts | findstr /i "co.sitic.pp SMS_RECEIVED"
adb shell dumpsys activity services co.sitic.pp
```
16. Perfil de contencion recomendado
Cuando el firmware acepte cada operacion:
```text
suspended=true
RECEIVE_SMS permission=granted=false
RECEIVE_SMS AppOp=ignore
RUN_IN_BACKGROUND=ignore
RUN_ANY_IN_BACKGROUND=ignore
WAKE_LOCK=ignore
UID fuera de restrict-background-whitelist
UID en restrict-background-blacklist, si persiste
restrict-background=enabled
```
Esto es contencion, no eliminacion fisica. El proceso puede seguir activo si el fabricante lo mantiene mediante servicios privilegiados.
17. Comandos de recuperacion
```bat
adb shell cmd package unsuspend --user 0 co.sitic.pp
adb shell appops set co.sitic.pp RUN_IN_BACKGROUND default
adb shell appops set co.sitic.pp RUN_ANY_IN_BACKGROUND default
adb shell appops set co.sitic.pp WAKE_LOCK default
adb shell appops set co.sitic.pp RECEIVE_SMS default
adb shell pm grant --user 0 co.sitic.pp android.permission.RECEIVE_SMS
adb shell cmd netpolicy remove restrict-background-blacklist <UID>
```
Si activaste Data Saver solo para la prueba:
```bat
adb shell cmd netpolicy set restrict-background false
```
Este ultimo comando es global y afecta a todas las aplicaciones.
18. Metodo recomendado
Identificar dispositivo, compilacion, ruta y UID.
Guardar el estado original.
Aplicar un cambio.
Verificarlo inmediatamente.
Reiniciar.
Verificar nuevamente.
Repetir toda la auditoria despues de cada OTA.
Registro de cambios
2026-11: auditoria post-OTA
Documentado que una actualizacion del sistema restauro `RECEIVE_SMS` de `granted=false` a `granted=true`.
Confirmado que la revocacion pudo reaplicarse en el firmware actualizado.
Anadida lista completa para comparar fingerprint, version, ruta, UID, permiso, AppOps, NetPolicy, proceso y servicios.
Aclarado que `dumpsys package` es la comprobacion principal si `appops get` muestra resultados duplicados.
Privacidad
La guia esta anonimizada. No incluye numeros de serie, IMEI, telefonos personales, nombres de propietarios, PINes ni cargas SMS operativas.
