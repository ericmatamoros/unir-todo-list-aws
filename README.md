# todo-list-aws

Este proyecto contiene un ejemplo de solución **SAM + Jenkins**. Contiene una aplicación API RESTful de libreta de tareas pendientes (ToDo) y los pipelines que permiten definir el CI/CD para productivizarla.

## Estructura

A continuación se describe la estructura del proyecto:
- **src** - en este directorio se almacena el código fuente de las funciones lambda con las que se va a trabajar
- **test** - Tests unitarios y de integración. 
- **samconfig.toml** - Configuración de los stacks de Staging y Producción
- **template.yaml** - Template que define los recursos AWS de la aplicación
- **localEnvironment.json** - Permite el despliegue en local de la aplicación sobreescribiendo el endpoint de dynamodb para que apunte contra el docker de dynamo

## Consulta de Outputs

aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text

