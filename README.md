# 📧 Enviar Correos desde Oracle 19c en Docker (Gmail / SSL / TLS)

Este repositorio contiene una guía paso a paso para enviar correos electrónicos utilizando Oracle 19c ejecutado en un contenedor Docker, configurando Wallet, certificados, ACLs y paquetes UTL_MAIL/UTL_SMTP para permitir conexión segura (TLS/SSL) hacia servidores SMTP como Gmail.




## 🐳 Parte I: Creación del Contenedor y Base de Datos
 -  Si no quieres un volumen para los certificados, solo no lo colocas y omites el paso 1
 -  Si ya tienes un contenedor pasar la parte II

### 📂 1️⃣ Crear carpeta local para montar el Wallet
```
mkdir C:\oracle\wallets\Example
```

### 🐳 2️⃣ Crear el contenedor Oracle 19c con volumen
```
docker run -d --name oracle19c \
-p 1521:1521 -p 5500:5500 \
-v C:\oracle\wallets\Example:/home/oracle/smtp_wallet \
oracle/database:19.3.0-ee
```
✅ Así, /opt/oracle/smtp_wallet dentro del contenedor quedará sincronizada con C:\oracle\wallets\Example en tu PC.



### 🗄️ 3️⃣ Crear la base de datos desde bash del contenedor
```
 dbca -silent -createDatabase \
  -templateName General_Purpose.dbc \
  -gdbname Example -sid Example \
  -responseFile NO_VALUE \
  -characterSet AL32UTF8 \
  -memoryPercentage 30 \
  -emConfiguration NONE \
  -sysPassword Oracle123 \
  -systemPassword Oracle123 \
  -createAsContainerDatabase false
```




## 📜 Parte II: Instalación de Certificados

### 1️⃣ Crear el directorio dentro del contenedor
```
mkdir -p /home/oracle/smtp_wallet
cd /home/oracle/smtp_wallet
````

### 2️⃣ Descargar los certificados
```
curl -o gtsr4.crt https://pki.goog/repo/certs/gtsr4.pem
```

```
curl -o we2.crt https://pki.goog/repo/certs/gts1c3.pem
```

```
echo | openssl s_client -connect smtp.gmail.com:465 -showcerts 2>/dev/null | \
sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > smtp_gmail.crt
```

### 3️⃣ Crear el Wallet con auto-login (Coloca la contraseña que quieres)
```
orapki wallet create -wallet /home/oracle/smtp_wallet -pwd TuPassword123 -auto_login
```

### 4️⃣ Agregar los certificados al Wallet
```
orapki wallet add -wallet /home/oracle/smtp_wallet -trusted_cert -cert gtsr4.crt -pwd TuPassword123
```

```
orapki wallet add -wallet /home/oracle/smtp_wallet -trusted_cert -cert we2.crt -pwd TuPassword123
```

```
orapki wallet add -wallet /home/oracle/smtp_wallet -trusted_cert -cert smtp_gmail.crt -pwd TuPassword123
```

### Verificar que el wallet contiene los tres certificados correctos
```
orapki wallet display -wallet /home/oracle/smtp_wallet
```

Deberías ver algo como:
```
CN=smtp.gmail.com
CN=GTS CA 1C3,O=Google Trust Services LLC,C=US
CN=GTS Root R4,O=Google Trust Services LLC,C=US
```







## 🔐 Parte III: Configuración ACL y Parámetros SMTP

### 1️⃣ Crear el ACL con SYS
```
BEGIN
    DBMS_NETWORK_ACL_ADMIN.CREATE_ACL(
        acl => 'smtp_gmail_acl.xml',
        description => 'ACL para SMTP Gmail',
        principal => 'SYS',  --- Aca lo puedes cambiar por el usuario que quieras, mientras tenga los permisos
        is_grant => TRUE,
        privilege => 'connect'
    );
    
    DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE(
        acl => 'smtp_gmail_acl.xml',
        principal => 'SYS', --igual...
        is_grant => TRUE,
        privilege => 'resolve'
    );
    
    DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL(
        acl => 'smtp_gmail_acl.xml',
        host => 'smtp.gmail.com'
    );
    
    COMMIT;
END;
```


### 2️⃣ Verificar ACL creada
```
SELECT acl, principal, privilege, is_grant 
FROM dba_network_acl_privileges 
WHERE acl LIKE '%smtp_gmail%';
```


### 3️⃣ Configurar parámetros SMTP
```
ALTER SYSTEM SET smtp_out_server = 'smtp.gmail.com:587' SCOPE=SPFILE;
-- Alternativa si VALUE es NULL
ALTER SYSTEM SET smtp_out_server = 'smtp.gmail.com:587' SCOPE=BOTH;
```

### 4️⃣ Verificar que todo está configurado
  - Ver ACL
```
SELECT * FROM dba_network_acls;
```
  - Ver privilegios
```
SELECT * FROM dba_network_acl_privileges WHERE principal = 'SYS';
```

  - Ver parámetro SMTP
```
SELECT name, value FROM v$parameter WHERE name LIKE 'smtp%';
```

  - Probar resolución de DNS
```
SELECT UTL_INADDR.get_host_address('smtp.gmail.com') FROM DUAL;
```

### 5️⃣ Probar conexión básica
```
DECLARE
    v_conn UTL_SMTP.connection;
BEGIN
    v_conn := UTL_SMTP.OPEN_CONNECTION('smtp.gmail.com', 587);
    DBMS_OUTPUT.PUT_LINE('Conexión exitosa');
    UTL_SMTP.QUIT(v_conn);
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error conexión: ' || SQLERRM);
END;
```




## 🔑 Parte IV: Configuración de Gmail App Password

### 1️⃣ Ir a Configuración de Seguridad de Google
```
https://myaccount.google.com/security
```

### 2️⃣ Ingresas aqui
<img width="525" height="98" alt="image" src="https://github.com/user-attachments/assets/b6ef2961-9bb5-42be-b4f3-c91ed8c9401f" />

### 3️⃣ Verificar que tengas la verificación en 2 pasos activada

### 4️⃣ En la parte inferior de la pagina das clic aqui
<img width="524" height="118" alt="image" src="https://github.com/user-attachments/assets/cca251c0-0d5d-408a-951f-1fbc9783bea0" />

### 5️⃣ Le colocas un nombre y le das a crear
<img width="406" height="244" alt="image" src="https://github.com/user-attachments/assets/f352116b-ea00-49db-8c34-f3314c365c10" />

### 6️⃣ Por ultimo, copias la contraseña y esa sera tu app password
 - La contraseña tendrá formato: xxxx xxxx xxxx xxxx
 - Guárdala en un lugar seguro, no podrás verla nuevamente






## 💻 Parte V: Implementación del Procedimiento

### 📧 Crear Procedimiento para Envío de Emails
```
CREATE OR REPLACE PROCEDURE send_email(
    p_recipient IN VARCHAR2,
    p_subject IN VARCHAR2,
    p_message IN VARCHAR2
) AS
    v_conn UTL_SMTP.connection;
    v_usuario VARCHAR2(50) := 'test123@gmail.com'; --!Cambiar por su correo
    v_password VARCHAR2(50) := 'xxxx xxxx xxxx xxxx'; --!Cambiar por su app password
    v_username_encoded VARCHAR2(1000);
    v_password_encoded VARCHAR2(1000);
BEGIN
    v_conn := UTL_SMTP.OPEN_CONNECTION(
        host => 'smtp.gmail.com',
        port => 587,
        wallet_path => 'file:/home/oracle/smtp_wallet',
        wallet_password => 'TuPassword123' --!Cambiar por su password del wallet
    );
    
    DBMS_OUTPUT.PUT_LINE('Conexión establecida...');
    
    UTL_SMTP.EHLO(v_conn, 'oracle-server');
    DBMS_OUTPUT.PUT_LINE('EHLO enviado...');
    
    UTL_SMTP.STARTTLS(v_conn);
    DBMS_OUTPUT.PUT_LINE('TLS iniciado...');
    
    UTL_SMTP.EHLO(v_conn, 'oracle-server');
    DBMS_OUTPUT.PUT_LINE('EHLO después de TLS...');
    
    -- Autenticación con AUTH LOGIN
    DBMS_OUTPUT.PUT_LINE('Iniciando autenticación LOGIN...');
    
    v_username_encoded := UTL_RAW.CAST_TO_VARCHAR2(
        UTL_ENCODE.BASE64_ENCODE(UTL_RAW.CAST_TO_RAW(v_usuario))
    );
    
    v_password_encoded := UTL_RAW.CAST_TO_VARCHAR2(
        UTL_ENCODE.BASE64_ENCODE(UTL_RAW.CAST_TO_RAW(v_password))
    );
    
    DBMS_OUTPUT.PUT_LINE('Enviando AUTH LOGIN...');
    UTL_SMTP.COMMAND(v_conn, 'AUTH LOGIN');
    
    DBMS_OUTPUT.PUT_LINE('Enviando username...');
    UTL_SMTP.COMMAND(v_conn, v_username_encoded);
    
    DBMS_OUTPUT.PUT_LINE('Enviando password...');
    UTL_SMTP.COMMAND(v_conn, v_password_encoded);
    
    DBMS_OUTPUT.PUT_LINE('Autenticación completada...');
    
    UTL_SMTP.MAIL(v_conn, v_usuario);
    UTL_SMTP.RCPT(v_conn, p_recipient);
    
    UTL_SMTP.OPEN_DATA(v_conn);
    UTL_SMTP.WRITE_DATA(v_conn, 'From: ' || v_usuario || UTL_TCP.CRLF);
    UTL_SMTP.WRITE_DATA(v_conn, 'To: ' || p_recipient || UTL_TCP.CRLF);
    UTL_SMTP.WRITE_DATA(v_conn, 'Subject: ' || p_subject || UTL_TCP.CRLF);
    UTL_SMTP.WRITE_DATA(v_conn, 'Date: ' || TO_CHAR(SYSDATE, 'DD-MON-YYYY HH24:MI:SS') || UTL_TCP.CRLF);
    UTL_SMTP.WRITE_DATA(v_conn, UTL_TCP.CRLF);
    UTL_SMTP.WRITE_DATA(v_conn, p_message || UTL_TCP.CRLF);
    UTL_SMTP.CLOSE_DATA(v_conn);
    
    DBMS_OUTPUT.PUT_LINE('Mensaje enviado...');
    
    UTL_SMTP.QUIT(v_conn);
    
    DBMS_OUTPUT.PUT_LINE('✅ Email enviado exitosamente a: ' || p_recipient);
    
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('❌ Error al enviar email: ' || SQLERRM);
        BEGIN
            UTL_SMTP.QUIT(v_conn);
        EXCEPTION
            WHEN OTHERS THEN
                NULL;
        END;
        RAISE;
END send_email;
```

### 🧪 Probar el Envío de Email
```
BEGIN
    send_email(
        p_recipient => 'correoDestino@gmail.com', 
        p_subject => 'Hola',
        p_message => 'Este cooreo fue enviado a traves de oracle con sql'
    );
END;

```


# Fin 😜

---
---
---

# Datos Extras: Si al reiniciar el contenedor no esta el listener, puedes hacer esto:

## Crear o sobrescribir listener.ora 
```
cat <<EOF > /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = Example)
      (SID_NAME = Example)
      (ORACLE_HOME = /opt/oracle/product/19c/dbhome_1)
    )
  )
EOF
```

## Reiniciar el listener

```
lsnrctl stop
lsnrctl start
```

## Verificar que el listener lo reconoce
```
lsnrctl status
```
