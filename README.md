# ejercicio-integracion-api
Este repositorio esta dirigido a la resolucion del ejercicio de Ingeniero de Integración de Sistemas.

Tenemos dos APIs, de donde provienen los datos que es la API A y a donde van los datos que es la API B.

## Api A

La Api A utiliza Oauth 2.0 con credenciales obtenidas del cliente. Como parametros de entrada, se facilita una ventana de tiempo donde se desea que se produzca la query.

### Obtener las credenciales

En primer lugar por tanto la Api A necesita un POST de los datos para comunicarse, este podría ser:

https://api.sistemaA.com/oauth/token

Con los parametros del usuario:

```json
{
    "grant_type": "client_credentials",
    "client_id": "<client_id>",
    "client_secret": "<client_secret>"
}
```

Y con el encabezado:

`Content-Type: application/x-www-form-urlencoded`

De esta manera, se enviaria mediante un POST en formato URI los datos del usuario y contraseña para obtener de respuesta el token autenticador:

```json
{
  "access_token": "<token>",
  "token_type": "Bearer",
  "expires_in": 3600
}

```

Con dicha clave, a continuación es cuando se podría hacer la consulta de la base de datos mediante un metodo GET con el cuerpo de formato json. 

## Api B

La Api B es el endpoint al que vamos a acceder principalmente para obtener los datos. Para su acceso, esta vez la autenticacion se produce mediante API key y se produce en api.sistemaB.com/bills


El cuerpo de la Api B es:

```json
{
    "invoices": [
        {
            "invoice_id": "123",
            "customer": "Empresa XYZ",
            "amount_due": 1500.75,
            "date_issued": "2023-05-01",
            "status": "pagada"
        },
        ...
    ]
}
```

Como buena practica, se sugiere que la Api B sea en formato de OpenApi Specification, ya que asi se tendra una mayor claridad a la hora de leer la Api al tenerlo en un formato estandarizado. 

Como ejemplo de ello:

```yml
openapi: 3.0.3
info:
  title: Sistema de Facturación B
  description: API para la consulta y gestión de facturas en el Sistema de Facturación B.
  version: 1.0.0
servers:
  - url: https://api.sistemaB.com
    description: Servidor principal de la API B
paths:
  /bills:
    get:
      summary: Consulta de Facturas
      description: Obtiene una lista de facturas dentro de un rango de fechas.
      parameters:
        - name: start_date
          in: query
          required: true
          description: Fecha de inicio para el rango de consulta (Formato: YYYY-MM-DD).
          schema:
            type: string
            format: date
        - name: end_date
          in: query
          required: true
          description: Fecha de fin para el rango de consulta (Formato: YYYY-MM-DD).
          schema:
            type: string
            format: date
      responses:
        '200':
          description: Respuesta exitosa con la lista de facturas.
          content:
            application/json:
              schema:
                type: object
                properties:
                  invoices:
                    type: array
                    items:
                      type: object
                      properties:
                        invoice_id:
                          type: string
                          description: Identificador único de la factura.
                        customer:
                          type: string
                          description: Nombre del cliente.
                        amount_due:
                          type: number
                          format: float
                          description: Monto adeudado.
                        date_issued:
                          type: string
                          format: date
                          description: Fecha de emisión de la factura.
                        status:
                          type: string
                          description: Estado de la factura.
        '400':
          description: Solicitud incorrecta (por ejemplo, formato de fecha inválido).
        '401':
          description: No autorizado (clave API incorrecta).
        '500':
          description: Error interno del servidor.
      security:
        - ApiKeyAuth: []
components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: x-api-key
  schemas:
    Invoice:
      type: object
      properties:
        invoice_id:
          type: string
          description: Identificador único de la factura.
        customer:
          type: string
          description: Nombre del cliente.
        amount_due:
          type: number
          format: float
          description: Monto adeudado.
        date_issued:
          type: string
          format: date
          description: Fecha de emisión de la factura.
        status:
          type: string
          description: Estado de la factura.
```

# Integracion de las APIs.

Una vez quedan definidas cada una de las API, lo que toca ahora es realizar un middleware que nos convierta del formato A al formato B. Esta funcion de python puede encargarse, cogiendo cada una de las posibles facturas y transformandolo al formato de apiB.

```python
def transform_facturas(facturas):
    transformed_invoices = []
    for factura in facturas:
        transformed_invoice = {
            'invoice_id': factura.id,
            'customer': factura.cliente,
            'amount_due': factura.monto,
            'date_issued': factura.fecha_emision,
            'status': 'paid' if factura.estado == 'pagada' else 'unpaid'
        }
        transformed_invoices.append(transformed_invoice)
    return transformed_invoices
```

# Prueba en directo

Como prueba en directo, se ha realizado un stack de fastAPI en python para crear un backend de api A con una base de datos mySQL y un backend de api B. El API Token ha sido hardcodeado por el tiempo disponible.

Después de esto, se ha creado un servidor en Hetzner donde ambas APIs han sido alojadas en un contenedor de docker con una especificacion de dockerfile. La intención era ir un paso más allá y depositarlo en un pod de kubernetes para asegurar la persistencia, pero no se ha podido por el tiempo.

El endpoint que puede probar el usuario es:

http://testing.juanjo.site:8001/bills/

El formato URI necesario para probarlo (entregado por Postman) es:

?start_date=2024-06-05&end_date=2024-06-30

Por tanto, en su totalidad es:

http://testing.juanjo.site:8001/bills/?start_date=2020-06-01&end_date=2020-06-30

Como mensaje se obtiene un 200 OK con el body `{"message":"Consulta de facturas"}`.





