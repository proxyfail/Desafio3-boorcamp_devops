# Desafio3-boorcamp_devops

#  Importante! 

# Para este desafio realize la registracion de una cuenta en AWS, la misma cuenta con un trial de 30 dias. No es un dato menor ya que se puede virtualizar el entorno de AWS pero preferi no hacerlo ya que eso acarreaba varias instalaciones y configuraciones. Aclarado esto, en este 'step by step', solo mencionare los pasos para realizar este desafio en la consola de AWS, ademas adjuntare fotos ilustrativas de lo que se ingresa y como sale la ejecucion de los scripts por consola.

Paso 1: Crear un bucket en S3

Primero, necesitamos un nombre único para el bucket. Usaremos un sufijo basado en la fecha para asegurar la unicidad.
Bash

BUCKET_NAME="my-s3-write-bucket-$(date +%Y%m%d%H%M%S)"
REGION="us-east-1" # Puedes cambiar la región según tu preferencia

echo "Creando el bucket: $BUCKET_NAME en la región: $REGION"
aws s3api create-bucket --bucket "$BUCKET_NAME" --region "$REGION" --create-bucket-configuration LocationConstraint="$REGION"

echo "Bucket '$BUCKET_NAME' creado exitosamente."

Salida Esperada:
JSON

{
    "Location": "/my-s3-write-bucket-20250527193038"
}

Paso 2: Crear un rol con una política que permita escribir en el bucket

Ahora crearemos un rol IAM y adjuntaremos una política que le permita escribir en el bucket S3 recién creado.

Primero, definimos la política de confianza del rol, que especifica quién puede asumir este rol. Inicialmente, no permitiremos que nadie lo asuma, y lo actualizaremos en el Paso 4.
Bash

ROLE_NAME="S3BucketWriterRole"
TRUST_POLICY_FILE="trust-policy.json"
S3_POLICY_FILE="s3-write-policy.json"

cat << EOF > "$TRUST_POLICY_FILE"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:root" # Esto se actualizará en el paso 4
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Obtener el ID de la cuenta para la política de confianza inicial (puede ser cualquier cuenta válida, luego lo cambiaremos)
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
sed -i "s/ACCOUNT_ID/$ACCOUNT_ID/" "$TRUST_POLICY_FILE"

echo "Creando el rol IAM: $ROLE_NAME"
aws iam create-role --role-name "$ROLE_NAME" --assume-role-policy-document file://"$TRUST_POLICY_FILE"

echo "Rol '$ROLE_NAME' creado exitosamente."

# Definir la política de permisos para el rol
cat << EOF > "$S3_POLICY_FILE"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::$BUCKET_NAME/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::$BUCKET_NAME"
    }
  ]
}
EOF

POLICY_NAME="S3BucketWriterPolicy"

echo "Creando la política de permisos: $POLICY_NAME"
aws iam put-role-policy --role-name "$ROLE_NAME" --policy-name "$POLICY_NAME" --policy-document file://"$S3_POLICY_FILE"

echo "Política '$POLICY_NAME' adjuntada al rol '$ROLE_NAME' exitosamente."

Salida Esperada para aws iam create-role:
JSON

{
    "Role": {
        "Path": "/",
        "RoleName": "S3BucketWriterRole",
        "RoleId": "AROA...",
        "Arn": "arn:aws:iam::ACCOUNT_ID:role/S3BucketWriterRole",
        "CreateDate": "2025-05-27T22:30:38Z",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::ACCOUNT_ID:root"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}

Salida Esperada para aws iam put-role-policy (no hay salida por defecto si es exitoso).
Paso 3: Generar un usuario IAM llamado s3-support y crear credenciales programáticas

Crearemos un nuevo usuario IAM y generaremos sus claves de acceso. Es crucial guardar estas claves en un lugar seguro ya que solo se mostrarán una vez.
Bash

USER_NAME="s3-support"

echo "Creando el usuario IAM: $USER_NAME"
aws iam create-user --user-name "$USER_NAME"

echo "Usuario '$USER_NAME' creado exitosamente."

echo "Creando credenciales de acceso para el usuario '$USER_NAME'"
CREDENTIALS=$(aws iam create-access-key --user-name "$USER_NAME")

ACCESS_KEY_ID=$(echo "$CREDENTIALS" | jq -r '.AccessKey.AccessKeyId')
SECRET_ACCESS_KEY=$(echo "$CREDENTIALS" | jq -r '.AccessKey.SecretAccessKey')

echo "---------------------------------------------------------"
echo "¡ATENCIÓN! Guarda estas credenciales de forma segura:"
echo "ACCESS_KEY_ID:     $ACCESS_KEY_ID"
echo "SECRET_ACCESS_KEY: $SECRET_ACCESS_KEY"
echo "---------------------------------------------------------"

Salida Esperada para aws iam create-user:
JSON

{
    "User": {
        "Path": "/",
        "UserName": "s3-support",
        "UserId": "AIDAU...",
        "Arn": "arn:aws:iam::ACCOUNT_ID:user/s3-support",
        "CreateDate": "2025-05-27T22:30:38Z"
    }
}

Salida Esperada para aws iam create-access-key:
JSON

{
    "AccessKey": {
        "UserName": "s3-support",
        "Status": "Active",
        "CreateDate": "2025-05-27T22:30:38Z",
        "SecretAccessKey": "YOUR_SECRET_ACCESS_KEY_HERE",
        "AccessKeyId": "YOUR_ACCESS_KEY_ID_HERE"
    }
}

Paso 4: Actualizar la política de confianza del rol para que permita al usuario s3-support asumir el rol

Ahora modificaremos la política de confianza del rol S3BucketWriterRole para que el usuario s3-support sea el único principal autorizado para asumirlo.
Bash

ROLE_ARN=$(aws iam get-role --role-name "$ROLE_NAME" --query 'Role.Arn' --output text)
USER_ARN=$(aws iam get-user --user-name "$USER_NAME" --query 'User.Arn' --output text)

TRUST_POLICY_FILE_UPDATED="trust-policy-updated.json"

cat << EOF > "$TRUST_POLICY_FILE_UPDATED"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "$USER_ARN"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

echo "Actualizando la política de confianza para el rol: $ROLE_NAME"
aws iam update-assume-role-policy --role-name "$ROLE_NAME" --policy-document file://"$TRUST_POLICY_FILE_UPDATED"

echo "Política de confianza del rol '$ROLE_NAME' actualizada exitosamente para permitir al usuario '$USER_NAME' asumir el rol."

Salida Esperada (no hay salida por defecto si es exitoso).
Paso 5: Conectar el CLI con las credenciales del usuario s3-support

Configuraremos un perfil de AWS CLI para el usuario s3-support utilizando las credenciales programáticas obtenidas en el Paso 3.
Bash

echo "Configurando el perfil de AWS CLI para el usuario '$USER_NAME'..."

aws configure set aws_access_key_id "$ACCESS_KEY_ID" --profile "$USER_NAME"
aws configure set aws_secret_access_key "$SECRET_ACCESS_KEY" --profile "$USER_NAME"
aws configure set region "$REGION" --profile "$USER_NAME"
aws configure set output json --profile "$USER_NAME"

echo "Perfil '$USER_NAME' configurado exitosamente."
echo "Para usar este perfil, añade '--profile $USER_NAME' a tus comandos AWS CLI."

Paso 6: Asumir el rol para poder escribir en el bucket

Ahora, utilizaremos el usuario s3-support para asumir el rol S3BucketWriterRole y obtener credenciales temporales. Luego, usaremos esas credenciales para escribir un archivo en el bucket de S3.
Bash

echo "Asumiendo el rol '$ROLE_NAME' con el usuario '$USER_NAME'..."

ASSUMED_ROLE_CREDENTIALS=$(aws sts assume-role --role-arn "$ROLE_ARN" --role-session-name "S3WriteSession" --profile "$USER_NAME")

ASSUMED_ACCESS_KEY_ID=$(echo "$ASSUMED_ROLE_CREDENTIALS" | jq -r '.Credentials.AccessKeyId')
ASSUMED_SECRET_ACCESS_KEY=$(echo "$ASSUMED_ROLE_CREDENTIALS" | jq -r '.Credentials.SecretAccessKey')
ASSUMED_SESSION_TOKEN=$(echo "$ASSUMED_ROLE_CREDENTIALS" | jq -r '.Credentials.SessionToken')

echo "Credenciales temporales obtenidas exitosamente."
echo "AccessKeyId: $ASSUMED_ACCESS_KEY_ID"
echo "SecretAccessKey: $ASSUMED_SECRET_ACCESS_KEY"
echo "SessionToken: $ASSUMED_SESSION_TOKEN"

# Exportar las credenciales temporales como variables de entorno
export AWS_ACCESS_KEY_ID="$ASSUMED_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="$ASSUMED_SECRET_ACCESS_KEY"
export AWS_SESSION_TOKEN="$ASSUMED_SESSION_TOKEN"

echo "Credenciales temporales exportadas a las variables de entorno."

# Crear un archivo de prueba local
echo "Este es un archivo de prueba creado por el rol asumido." > test-file.txt

echo "Copiando 'test-file.txt' al bucket '$BUCKET_NAME' usando las credenciales del rol..."
aws s3 cp test-file.txt "s3://$BUCKET_NAME/test-file.txt"

echo "Archivo 'test-file.txt' copiado a s3://$BUCKET_NAME/test-file.txt exitosamente."

# Limpiar las variables de entorno para evitar usar credenciales temporales caducadas
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN

echo "Variables de entorno de credenciales temporales limpiadas."

Captura de la salida de comandos para asumir el rol y copiar un archivo al bucket:

(La captura de pantalla incluiría la ejecución del comando aws sts assume-role y la posterior ejecución de aws s3 cp)

Salida de aws sts assume-role:
JSON

{
    "Credentials": {
        "AccessKeyId": "ASIA...",
        "SecretAccessKey": "YOUR_TEMPORARY_SECRET_ACCESS_KEY_HERE",
        "SessionToken": "YOUR_TEMPORARY_SESSION_TOKEN_HERE",
        "Expiration": "2025-05-27T23:30:38Z"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA...:S3WriteSession",
        "Arn": "arn:aws:sts::ACCOUNT_ID:assumed-role/S3BucketWriterRole/S3WriteSession"
    }
}

Salida de aws s3 cp:

upload: ./test-file.txt to s3://my-s3-write-bucket-20250527193038/test-file.txt

Diagrama de Objetos IAM para Asumir un Rol
Fragmento de código

graph TD
    A[Usuario IAM: s3-support] -->|1. Llamada a sts:AssumeRole| B(Servicio AWS STS)
    B -->|2. Valida la política de confianza del rol| C[Rol IAM: S3BucketWriterRole]
    C -->|3. AWS STS genera credenciales temporales| B
    B -->|4. Envía credenciales temporales al usuario| A
    A -->|5. Utiliza credenciales temporales| D[Bucket S3: my-s3-write-bucket-...]
    D -->|6. Escribe objeto en S3| D

Explicación del Diagrama:

    Usuario IAM (s3-support): Este es el principal que desea realizar una acción (escribir en S3) para la cual no tiene permisos directos. En su lugar, tiene permisos para llamar a la operación sts:AssumeRole.
    Llamada a sts:AssumeRole: El usuario s3-support invoca el servicio AWS Security Token Service (STS) con el comando aws sts assume-role, especificando el ARN del rol S3BucketWriterRole que desea asumir.
    Servicio AWS STS: STS es el servicio encargado de emitir credenciales de seguridad temporales. Cuando recibe la solicitud de sts:AssumeRole, realiza dos validaciones clave:
        Política de Confianza del Rol: STS verifica la política de confianza del rol S3BucketWriterRole. Esta política define quién (qué principales) están autorizados a asumir este rol. En nuestro caso, la política permite que el usuario s3-support lo asuma.
        Política de Permisos del Principal: Aunque no se muestra directamente en el flujo, STS también verifica si el usuario s3-support tiene permiso para realizar la acción sts:AssumeRole en el rol especificado.
    Generación de Credenciales Temporales: Si ambas validaciones son exitosas, AWS STS genera un conjunto de credenciales de seguridad temporales (clave de acceso, clave secreta y token de sesión). Estas credenciales tienen los permisos definidos en la política de permisos del rol S3BucketWriterRole.
    Envío de Credenciales Temporales al Usuario: STS devuelve estas credenciales temporales al usuario s3-support.
    Utilización de Credenciales Temporales: El usuario s3-support (o la aplicación/script que lo utiliza) configura estas credenciales temporales en su entorno (por ejemplo, exportándolas como variables de entorno de la CLI).
    Escritura de Objeto en S3: Con las credenciales temporales del rol S3BucketWriterRole activas, el usuario s3-support ahora puede interactuar con el bucket S3 y realizar operaciones permitidas por la política del rol, como s3:PutObject (escribir archivos).

En resumen, el usuario s3-support no tiene permisos directos para escribir en S3. En su lugar, tiene permiso para "convertirse" temporalmente en el rol S3BucketWriterRole, que sí tiene los permisos necesarios para interactuar con S3. Esto es una práctica de seguridad recomendada conocida como "Least Privilege", donde los usuarios solo tienen los permisos necesarios para realizar sus tareas, y los permisos elevados se otorgan de forma temporal a través de la asunción de roles.