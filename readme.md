Paso a paso: conexión y troubleshooting de PostgreSQL en OpenShift
Este README documenta, de forma simple y reutilizable, el flujo que usamos para crear, probar y solucionar la conexión a PostgreSQL en OpenShift, incluyendo cómo leer logs y corregir un error de autenticación.

Prerrequisitos
Acceso a un proyecto en OpenShift y permisos para crear recursos.

Imagen certificada de PostgreSQL disponible en el registro (ejemplo: registry.redhat.io/rhel8/postgresql-13).

Secret “postgresql” con estas claves:

database-user (ej.: userLI3)

database-password (ej.: OCJqLmeKqTjnuiQ0)

database-name (ej.: sampledb)

Tip: Mantener credenciales en Secrets y referenciarlas por variables de entorno es la práctica recomendada.

Despliegue e inicialización
Crear o validar el Secret
bash
oc create secret generic postgresql \
  --from-literal=database-user=userLI3 \
  --from-literal=database-password=OCJqLmeKqTjnuiQ0 \
  --from-literal=database-name=sampledb
Desplegar PostgreSQL (ejemplo básico con Deployment)
bash
cat << 'EOF' | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: registry.redhat.io/rhel8/postgresql-13
          env:
            - name: POSTGRESQL_USER
              valueFrom:
                secretKeyRef:
                  name: postgresql
                  key: database-user
            - name: POSTGRESQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql
                  key: database-password
            - name: POSTGRESQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: postgresql
                  key: database-name
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgresql-data
              mountPath: /var/lib/pgsql/data
      volumes:
        - name: postgresql-data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql
spec:
  selector:
    app: postgresql
  ports:
    - port: 5432
      targetPort: 5432
EOF
Tip: En producción usá PersistentVolume en lugar de emptyDir.

Probar la conexión
Opción A: Cliente efímero directo con oc run
bash
oc run psql-client --image=registry.redhat.io/rhel8/postgresql-13 --restart=Never -it -- \
  psql -h postgresql -U userLI3 sampledb
Si ves “cannot exec into a container in a completed pod”: el Pod efímero terminó (fase Succeeded). Volvé a crear otro o usá las opciones siguientes.

Opción B: Shell interactiva en un Pod efímero
bash
oc run psql-client --image=registry.redhat.io/rhel8/postgresql-13 --restart=Never -it -- bash
# Dentro del pod:
psql -h postgresql -U userLI3 sampledb
Opción C: Desde el Pod de PostgreSQL
bash
oc rsh deploy/postgresql
psql -h postgresql -U userLI3 sampledb
Leer logs cuando la autenticación falla
Si obtenés: FATAL: password authentication failed for user "userLI3".

Arranque del contenedor:

bash
oc logs deploy/postgresql
Verás que el logging se redirige a “log” (logging collector de PostgreSQL).

Logs de autenticación dentro del Pod:

bash
oc rsh deploy/postgresql
# Ruta típica (puede variar según imagen):
ls -l /var/lib/pgsql/data/log
tail -f /var/lib/pgsql/data/log/postgresql-*.log
Buscá entradas con:

FATAL: password authentication failed

DETAIL: Password does not match

Connection matched pg_hba.conf line ... md5

Conclusión: La contraseña del rol en PostgreSQL no coincide con la del Secret.

Corregir desincronización de credenciales
Alinear rol y contraseña con el Secret
bash
oc rsh deploy/postgresql
psql -U postgres
ALTER ROLE "userLI3" WITH LOGIN PASSWORD 'OCJqLmeKqTjnuiQ0';
\du
Verificar conexión:

bash
psql -h postgresql -U userLI3 sampledb
Tip: Usá comillas dobles si el rol fue creado case-sensitive. En nuestra demo, userLI3 funciona sin comillas al conectarse, y "userLI3" se usa en ALTER para asegurar coincidencia exacta.

Crear tabla simple e insertar datos
Crear tabla
sql
CREATE TABLE clientes (
  id SERIAL PRIMARY KEY,
  nombre VARCHAR(50),
  email VARCHAR(100)
);
Insertar registros
sql
INSERT INTO clientes (nombre, email) VALUES
('Ana Pérez', 'ana.perez@example.com'),
('Carlos Gómez', 'carlos.gomez@example.com'),
('Lucía Fernández', 'lucia.fernandez@example.com');
Verificar
sql
SELECT * FROM clientes;
\dt
\d clientes
Cierre consultivo para entrevista TAM
Problema: Falla de autenticación por desincronización entre Secret y rol.

Diagnóstico: Logs internos de PostgreSQL muestran Password does not match y regla pg_hba.conf md5.

Solución: ALTER ROLE para alinear la contraseña con el Secret.

Aprendizaje: OpenShift gestiona credenciales; PostgreSQL las valida. Los Pods efímeros terminan en Succeeded; para debugging, recrear cliente o usar oc rsh al Pod en ejecución.

Valor TAM: Convertir un error en una oportunidad educativa, guiar al cliente con pasos claros y buenas prácticas.

Limpieza opcional
bash
oc delete deployment postgresql
oc delete service postgresql
oc delete secret postgresql
oc delete pod psql-client --ignore-not-found
