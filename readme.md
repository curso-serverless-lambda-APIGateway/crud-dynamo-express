# Hola mundo con express y dynamoDB

1. [Instalación de express y serverless-http](#install)
2. [Creación básica de una ruta](#newRoute)
3. [Configuración para utilizar DynamoDB](#dynamo)
4. [Instalación y configuración de dynamoDB y body-parser](#install2)
5. [Añadir un registro a DynamoDb](#post)
6. [Recuperar todos los registros](#getAll)
7. [Recuperar un registro por userId](#getOne)
8. [Trabajar con DynamoDB en local](#dynamoOffline)

<a name="install"></a>
## 1. Instalación de express y serverless-http

Utilizaremos express y serverless-http para gestionar las llamadas http a nuestra lambda

`npm install --save express serverless-http`

<a name="newRoute"></a>
## 2. Creación básica de una ruta

~~~
'use strict';

const serverless = require('serverless-http');
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hola Mundo con ExpressJS');
});

module.exports.generic = serverless(app)
~~~

<a name="dynamo"></a>
## 3. Configuración para utilizar DynamoDB

Para garantizar el acceso de la lambda a DynamoDB debemos configurar el archivo **serverless.yml**

1. Definimos el nombre de la tabla que vamos a utilizar

~~~
custom:
  tableName: 'users-table-${self:provider.stage}'
~~~

2. Habilitamos permisos para que la lambda pueda interactuar con DynamoDB

~~~
provider:
  name: aws
  runtime: nodejs12.x
  stage: dev
  iamRoleStatements:
    - Effect: Allow
      Action:
        - dynamodb:Query
        - dynamodb:Scan
        - dynamodb:GetItem
        - dynamodb:PutItem
        - dynamodb:UpdateItem
        - dynamodb:DeleteItem
      Resource:
        - { "Fn::GetAtt": ["UsersDynamoDBTable", "Arn"] }
  environment:
    USERS_TABLE: ${self:custom.tableName}
~~~

3. Definimos la tabla

~~~
resources:
  Resources:
    UsersDynamoDBTable:
      Type: 'AWS::DynamoDB::Table'
      Properties:
        AttributeDefinitions:
          -
            AttributeName: userId
            AttributeType: S
        KeySchema:
          -
            AttributeName: userId
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:custom.tableName}
~~~

<a name="install2"></a>
## 4. Instalación y configuración de dynamoDB y body-parser

Instalamos el sdk de AWS y body-parser mediante npm

`npm install --save aws-sdk body-parser`

En el archivo **handler.js** establecemos las constantes para poder utilizar estas librerías

~~~
const AWS = require('aws-sdk');
const bodyParser = require('body-parser');

const dynamoDB = new AWS.DynamoDB.DocumentClient()

app.use(bodyParser.urlencoded({extended: true}));
~~~

<a name="post"></a>
## 5. Inserción de un nuevo registro

En el archivo **handler.js** obtenemos el nombre de la tabla de la variable de entorno:

~~~
const USERS_TABLE = process.env.USERS_TABLE;
~~~

modificamos el post para insertar el nuevo registro en la tabla

~~~
app.post('/users', (req, res) => {
  const {userId, name} = req.body;

  const params = {
    TableName: USERS_TABLE,
    Item: {
      userId, name
    }
  };

  dynamoDB.put(params, (error) => {
    if (error) {
      console.log(error);
      res.status(400).json({
        error: 'No se ha podido crear el usuario'
      });
    } else {
      res.json({ userId, name })
    }
  })
});
~~~

<a name="getAll"></a>
## 6. Recuperar todos los registros

Para obtener todos los registros de dynamo, creamos una nueva ruta:

~~~
app.get('/users', (req, res) => {
  const params = {
    TableName: USERS_TABLE,
  };
  dynamoDB.scan(params, (error, result) => {
    if (error) {
      console.log(error);
      req.status(400).json({
        error: 'No se ha podido acceder a los usuarios'
      });
    } else {
      const {Items} = result
      res.json({
        succes: true,
        message: 'Usuarios cargados correctamente',
        users: Items
      });
    };
  });
});
~~~

<a name="getOne"></a>
## 7. Recuperar un registro

Para obtener un registro en concreto, creamos la nueva ruta:

~~~
app.get('/users/:userId', (req, res) => {
  const params = {
    TableName: USERS_TABLE,
    Key: {
      userId: req.params.userId
    }
  };

  dynamoDB.get(params, (error, result) => {
    if (error) {
      console.log(error);
      res.status(400).json({
        error: 'No se ha podido acceder al usuario'
      });
    }
    if (result.Item) {
      const {userId, name} = result.Item;
      res.json({userId, name});
    } else {
      res.status(404).json({
        error: 'Usuario no encontrado'
      });
    };
  })
});
~~~

<a name="dynamoOffline"></a>
## 8. Trabajar con DynamoDB en local

Para poder trabajar en local debemos instalar serverless-offline y serverles-dynamodb-local

`npm install --save-dev serverless-offline serverless-dynamodb-local`

Dentro del archivo **serverless.yml** debemos modificar la seccion *custom* y añadir la sección *plugins*:

~~~
plugins:
  - serverless-offline
  - serverless-dynamodb-local

custom:
  tableName: 'users-table-${self:provider.stage}'
  dynamodb:
    start:
      migrate: true
    stages:
      - ${self:provider.stage}
~~~

También modificaremos el archivo **handler.js** para que en el caso en el que trabajemos en modo local la relación a la base de datos sea correcta.

~~~
const USERS_TABLE = process.env.USERS_TABLE;
const IS_OFFLINE = process.env.IS_OFFLINE;

if (IS_OFFLINE === 'true') {
  dynamoDB = new AWS.DynamoDB.DocumentClient({
    region: 'localhost',
    endpoint: 'http://localhost:8000'
  });
} else {
  dynamoDB = new AWS.DynamoDB.DocumentClient();
}
~~~
