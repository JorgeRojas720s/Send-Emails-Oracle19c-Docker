# 📧 Enviar Correos desde Oracle 19c en Docker (Gmail / SSL / TLS)

Este repositorio contiene una guía paso a paso para enviar correos electrónicos utilizando Oracle 19c ejecutado en un contenedor Docker, configurando Wallet, certificados, ACLs y paquetes UTL_MAIL/UTL_SMTP para permitir conexión segura (TLS/SSL) hacia servidores SMTP como Gmail.


## I PARTE: Intalacion de certificados
### 📂 1️⃣ Crear carpeta local para montar el Wallet
```
mkdir C:\oracle\wallets\Example
```

### 🐳 2️⃣ Crear el contenedor Oracle 19c con volumen
```
docker run -d --name oracle19c \
-p 1521:1521 -p 5500:5500 \
-v C:\oracle\wallets\Example:/opt/oracle/smtp_wallet \
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


### 🔐 4️⃣ Crear la wallet dentro del contenedor
```
orapki wallet create -wallet /home/oracle/smtp_wallet -pwd Example123 -auto_login
mkdir -p /tmp/certs
cd /tmp/certs
````

### 📥 5️⃣  Descargar la cadena de certificados de Gmail (smtp.gmail.com) usando OpenSSL
```
echo -n | openssl s_client -connect smtp.gmail.com:587 -starttls smtp -showcerts 2>/dev/null \
| sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > gmail_chain.pem
```

### 4️⃣ Separar la cadena en certificados individuales
```
csplit -f cert- gmail_chain.pem '/-----BEGIN CERTIFICATE-----/' '{*}'
```

### 7️⃣ Obtener el certificado principal del servidor smtp.gmail.com
```
echo -n | openssl s_client -connect smtp.gmail.com:587 -starttls smtp 2>/dev/null \
| openssl x509 > /tmp/gmail_cert.pem
```


### 8️⃣ Agregar los certificados al wallet
```
orapki wallet add -wallet /home/oracle/smtp_wallet -trusted_cert \
-cert /tmp/gmail_cert.pem -pwd Example123

orapki wallet add -wallet /home/oracle/smtp_wallet -trusted_cert \
-cert /tmp/certs/cert-02 -pwd Example123

orapki wallet add -wallet /home/oracle/smtp_wallet -trusted_cert \
-cert /tmp/certs/cert-03 -pwd Example123

```

### 9️⃣ Verificar que el wallet contiene los tres certificados correctos
```
orapki wallet display -wallet /home/oracle/smtp_wallet -pwd Example123
```
Deberías ver algo como::
```
CN=smtp.gmail.com
CN=GTS Root R4,O=Google Trust Services LLC,C=US
CN=WE2,O=Google Trust Services,C=US
```

---

## II PARTE: Configuracion del ACL (Access Control List)


## 1. Crear el ACL con SYS
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


## 2. Verificar que se creó correctamente
```
SELECT acl, principal, privilege, is_grant 
FROM dba_network_acl_privileges 
WHERE acl LIKE '%smtp_gmail%';
```


## 3. Configurar parámetros SMTP
```
ALTER SYSTEM SET smtp_out_server = 'smtp.gmail.com:587' SCOPE=SPFILE;

-- Si VALUE es null, probar este comando alternativo:

ALTER SYSTEM SET smtp_out_server = 'smtp.gmail.com:587' SCOPE=BOTH;
```

## 4. Reiniciar la base de datos (opcional)
```
SHUTDOWN IMMEDIATE;
STARTUP;
```

## 5. Verificar que todo está configurado
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

## 6. Probar conexión básica
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

---
---
---



# Extra: Si al reiniciar el contenedor no esta el listener, puedes hacer esto:

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
