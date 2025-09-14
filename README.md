# microservices-rabbitmq
Repositorio de práctica para aprender mensajería con RabbitMQ usando una arquitectura de microservicios. Contiene dos microservicios: uno que envía (producer) y otro que recibe/activa acciones cuando llegan mensajes desde RabbitMQ (consumer). Ideal para practicar patrones de integración, colas, exchanges y pruebas locales con Docker.

```mermaid
sequenceDiagram
    participant C as Cliente
    participant OS as Order Service
    participant DB as Base de Datos
    participant RMQ as RabbitMQ
    participant NS as Notification Service

    Note over C,NS: Flujo Principal - Creación de Orden
    C->>OS: POST /orders {order data}
    OS->>DB: Guardar pedido
    DB-->>OS: Confirmación (ID generado)
    OS->>RMQ: Publicar mensaje (order.exchange, order.created)
    RMQ-->>OS: Confirmación de publicación
    OS-->>C: 201 Created (respuesta HTTP)

    Note over RMQ,NS: Procesamiento Asíncrono
    RMQ->>NS: Entregar mensaje (order.created.queue)
    NS->>NS: Procesar notificación
    alt Notificación exitosa
        NS-->>RMQ: ACK (confirmación)
        NS->>NS: Registrar éxito
    else Error en notificación (primer intento)
        NS-->>RMQ: NACK (rechazo con reintento)
        RMQ->>NS: Reintentar entrega (después de backoff)
        NS->>NS: Reintentar procesamiento
        alt Segundo intento exitoso
            NS-->>RMQ: ACK
            NS->>NS: Registrar éxito
        else Error en segundo intento
            NS-->>RMQ: NACK (rechazo con reintento)
            RMQ->>NS: Reintentar entrega (después de backoff)
            NS->>NS: Reintentar procesamiento
            alt Tercer intento exitoso
                NS-->>RMQ: ACK
                NS->>NS: Registrar éxito
            else Error en tercer intento
                NS-->>RMQ: Rechazar y enviar a DLQ
                RMQ->>RMQ: Mover mensaje a order.dlq
                NS->>NS: Registrar fallo permanente
            end
        end
    end

    Note over RMQ,NS: Procesamiento de Mensajes Fallidos (DLQ)
    RMQ->>NS: Entregar mensaje de order.dlq
    NS->>NS: Manejar mensaje fallido
    NS-->>RMQ: ACK (confirmar procesamiento DLQ)
    NS->>NS: Registrar error y generar alerta
```
