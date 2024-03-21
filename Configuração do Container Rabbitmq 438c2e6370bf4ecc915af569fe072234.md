# Configuração do Container Rabbitmq

## 1 - Instalar Docker

- Windows
    - Instalar [WSL](https://learn.microsoft.com/pt-br/windows/wsl/install)
    - Instalar [Docker](https://docs.docker.com/desktop/install/windows-install/)

## 2 - Configurar Container no Docker Desktop

1. Fazer pull da imagem do rabbitmq
2. Criar um volume
    1. Docker volumes servem para persistência dos dados.
    2. Caso o container seja reiniciado ou desligado os dados não são perdidos.
    3. É possível associar mais de um container ao mesmo volume.
    4. É possível refazer um container usando um volume, mantendo as configurações e dados.
3. Criar o container do RabbitMQ
4. Acessar o container no modo bash e ativar o plugin mqtt

```jsx
docker volume create rabbitmq_data
docker volume create rabbitmq_config

docker run -e RABBITMQ_DEFAULT_USER=rabbitmq -e RABBITMQ_DEFAULT_PASS=rabbitmq \
 -v rabbitmq_data:/var/lib/rabbitmq -v rabbitmq_config:/etc/rabbitmq  \
 -d --hostname rabbitmq --name rabbitmq -p 1883:1883 -p 5672:5672 -p 15672:15672 \
 rabbitmq:3-management
 
docker exec -it rabbitmq bash
rabbitmq-plugins enable rabbitmq_mqtt
exit
 
 
```

## 3 - Configurando Fila no RabbitMQ

1. Acessa o servidor web do RabbitMQ `localhost:15672`
2. Crie uma fila
    
    ![Untitled](Configurac%CC%A7a%CC%83o%20do%20Container%20Rabbitmq%20438c2e6370bf4ecc915af569fe072234/Untitled.png)
    
3. Ligue essa a fila as Exchanges usando routing keys
    
    ![Untitled](Configurac%CC%A7a%CC%83o%20do%20Container%20Rabbitmq%20438c2e6370bf4ecc915af569fe072234/Untitled%201.png)
    

## 4 - Código

### Configuração do ambiente

1. Instale o Python
2. Instale a biblioteca paho-mqtt
    1. pip install paho-mqtt

### Publisher

```jsx
import paho.mqtt.client as mqtt
import random

# Credenciais de acesso ao broker MQTT
user = "rabbitmq"
password = "rabbitmq"

# Callback executado quando o cliente MQTT se conecta ao broker
def on_connect(client, userdata, flags, reason_code, properties):
    print(f"Connected with result code {reason_code}")

# Criando um cliente MQTT
mqttc = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2) 
mqttc.username = user
mqttc.password = password
mqttc.on_connect = on_connect  # Definindo a função de callback para conexão

# Conectando ao broker MQTT
mqttc.connect("localhost", 1883, 60)

# Iniciando o loop de eventos do cliente MQTT em uma thread separada
mqttc.loop_start()

for i in range(3):
    celsius = random.randint(16, 35)
    fahrenheit = celsius * 9/5 + 32

    # Publicando temperatura em Celsius e Fahrenheit para cada usuário
    mqttc.publish(f"temperatura.c.{i}", f"Temperatura do usuario {i}: {celsius} graus Celsius")
    mqttc.publish(f"temperatura.f.{i}", f"Temperatura do usuario {i}: {fahrenheit} graus Fahrenheit")
   
#Fechando a conexão
mqttc.disconnect()
```

### Subscriber

```jsx
import paho.mqtt.client as mqtt

# Credenciais de acesso ao broker MQTT
user = "rabbitmq"
password = "rabbitmq"

# Função de callback chamada quando a inscrição é feita
def on_subscribe(client, userdata, mid, reason_code_list, properties):
    # Verificando se a inscrição foi bem sucedida ou não
    if reason_code_list[0].is_failure:
        print(f"Broker rejected you subscription: {reason_code_list[0]}")
    else:
        print(f"Broker granted the following QoS: {reason_code_list[0].value}")

# Função de callback chamada quando o cliente se conecta ao broker MQTT
def on_connect(client, userdata, flags, reason_code, properties):
    print(f"Connected with result code {reason_code}")
    
    # Inscrevendo-se nos tópicos relevantes para o usuário
    mqttc.subscribe(f"temperatura.c.{usuario}", 0)
    mqttc.subscribe(f"temperatura.f.{usuario}", 0)

# Função de callback chamada quando uma mensagem é recebida
def on_message(client, userdata, message):
    print(message.payload)
    
# Obtendo o número do usuário da entrada do usuário
usuario = input("Insira o número do usuário: ")

# Criando um cliente MQTT
mqttc = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
mqttc.username = user
mqttc.password = password

# Definindo as funções de callback
mqttc.on_connect = on_connect
mqttc.on_subscribe = on_subscribe
mqttc.on_message = on_message

# Conectando ao broker MQTT
mqttc.connect("localhost", 1883, 60)

# Iniciando o loop de eventos do cliente MQTT
mqttc.loop_forever()

```

### Publisher AMQP

```python
import pika

credentials = pika.PlainCredentials('rabbitmq', 'rabbitmq')
parameters = pika.ConnectionParameters('localhost', 5672, '/', credentials)
connection = pika.BlockingConnection(parameters)
channel = connection.channel()

# Declare a exchange do tipo 'fanout'
channel.exchange_declare(exchange='promocao', exchange_type='fanout')

# Publica uma mensagem para a exchange
channel.basic_publish(exchange='promocao', routing_key='', body='Promoção!')

connection.close()
```

### Subscriber AMQP

```python
import pika

credentials = pika.PlainCredentials('rabbitmq', 'rabbitmq')
parameters = pika.ConnectionParameters('localhost', 5672, '/', credentials)
connection = pika.BlockingConnection(parameters)
channel = connection.channel()

def callback(ch, method, properties, body):
    print(f" Mensagem Recebida: {body.decode()}")

channel.queue_bind(exchange='promocao', queue='urubu_da_promo')
channel.basic_consume(queue='urubu_da_promo', on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```