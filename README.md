# Projets Spring Boot avec RabbitMQ

Ce dépôt contient deux mini-projets Spring Boot qui démontrent l'utilisation de RabbitMQ (AMQP) pour échanger des messages entre microservices.

## Vue d'ensemble

### Mini-projet 1 : Messagerie JSON
Démontre une communication de base entre un Producer et un Consumer via RabbitMQ avec des messages JSON.

**Architecture :**
- REST Producer (port 8123) → RabbitMQ (TopicExchange) → Consumer (port 8223) → logs console

### Mini-projet 2 : Messagerie + MySQL
Étend le flux en persistant les messages consommés dans une base de données MySQL.

**Architecture :**
- REST Producer (port 8081) → RabbitMQ (DirectExchange) → Consumer → MySQL (table user)

## Prérequis

- JDK 17 ou supérieur
- Maven 3.6+
- RabbitMQ en cours d'exécution (Docker recommandé)
  - Port AMQP : 5672
  - Port UI web : 15672
- MySQL (pour le mini-projet 2)
  - Base de données : testdb
  - phpMyAdmin (optionnel)

## Structure des projets

```
TP31/
├── spring-rabbitmq-producer/          # Mini-projet 1 - Producer
│   ├── src/main/java/com/oussama/rabbitmicro/
│   │   ├── SpringRabbitmqProducerApplication.java
│   │   ├── MQConfig.java
│   │   ├── CustomMessage.java
│   │   └── MessagePublisher.java
│   └── src/main/resources/application.properties
│
├── spring-rabbitmq-consumer/          # Mini-projet 1 - Consumer
│   ├── src/main/java/com/oussama/rabbitmicro/
│   │   ├── SpringRabbitmqConsumerApplication.java
│   │   ├── MQConfig.java
│   │   ├── CustomMessage.java
│   │   └── MessageListener.java
│   └── src/main/resources/application.properties
│
├── microservices-messaging-producer/  # Mini-projet 2 - Producer
│   ├── src/main/java/oussama/microservices/messaging/
│   │   ├── MessagingApplication.java
│   │   ├── config/RabbitMQConfig.java
│   │   ├── domain/User.java
│   │   ├── service/ProducerService.java
│   │   └── controller/ProducerController.java
│   └── src/main/resources/application.yml
│
└── microservices-messaging-consumer/  # Mini-projet 2 - Consumer
    ├── src/main/java/oussama/microservices/messagingconsumer/
    │   ├── MessagingConsumerApplication.java
    │   ├── config/RabbitMQConfig.java
    │   ├── domain/User.java
    │   ├── repository/UserRepository.java
    │   └── service/ConsumerService.java
    └── src/main/resources/application.yml
```

## Installation et configuration

### 1. Démarrer RabbitMQ

**Avec Docker :**
```bash
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

L'interface web sera disponible sur : http://localhost:15672
- Utilisateur : guest
- Mot de passe : guest

### 2. Configuration MySQL (Mini-projet 2 uniquement)

Créer la base de données :
```sql
CREATE DATABASE testdb;
```

Modifier les paramètres de connexion dans `microservices-messaging-consumer/src/main/resources/application.yml` si nécessaire :
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb?useSSL=false&serverTimezone=UTC
    username: root
    password: votre_mot_de_passe
```

## Mini-projet 1 : Messagerie JSON

### Configuration

**Producer (spring-rabbitmq-producer) :**
- Port : 8123
- Exchange : `2ite_micro_message_exchange` (TopicExchange)
- Queue : `2ite_micro_message_queue`
- Routing Key : `message_routingKey`

**Consumer (spring-rabbitmq-consumer) :**
- Port : 8223
- Écoute la même queue et affiche les messages dans la console

### Compilation et lancement

**1. Compiler le Producer :**
```bash
cd spring-rabbitmq-producer
mvn clean install
mvn spring-boot:run
```

**2. Compiler le Consumer :**
```bash
cd spring-rabbitmq-consumer
mvn clean install
mvn spring-boot:run
```

### Test

**Envoyer un message via Postman ou curl :**

**POST** http://localhost:8123/publish

**Body (JSON) :**
```json
{
  "message": "je suis oussama de 2ite"
}
```

**Réponse attendue :**
```
Message Published
```

**Vérifications :**
1. Vérifier dans l'interface RabbitMQ (http://localhost:15672) :
   - Exchange `2ite_micro_message_exchange` visible
   - Queue `2ite_micro_message_queue` visible
   - Binding avec `message_routingKey` visible

2. Vérifier les logs du Consumer :
   - Les objets `CustomMessage` reçus doivent s'afficher dans la console

## Mini-projet 2 : Messagerie + MySQL

### Configuration

**Producer (microservices-messaging-producer) :**
- Port : 8081
- Exchange : `user.exchange` (DirectExchange)
- Routing Key : `user.routingkey`

**Consumer (microservices-messaging-consumer) :**
- Port : 8080
- Queue : `user.queue`
- Base de données : MySQL `testdb`
- Table : `user` (créée automatiquement via JPA)

### Compilation et lancement

**1. S'assurer que MySQL est démarré et que la base `testdb` existe**

**2. Compiler le Producer :**
```bash
cd microservices-messaging-producer
mvn clean install
mvn spring-boot:run
```

**3. Compiler le Consumer :**
```bash
cd microservices-messaging-consumer
mvn clean install
mvn spring-boot:run
```

### Test

**Envoyer un message via Postman ou curl :**

**POST** http://localhost:8081/api/produce

**Body (JSON) :**
```json
{
  "userId": "1",
  "userName": "oussama"
}
```

**Réponse attendue :**
```json
"user sent: User(userId=1, userName=oussama)"
```

**Vérifications :**
1. Vérifier dans l'interface RabbitMQ (http://localhost:15672) :
   - Exchange `user.exchange` visible
   - Queue `user.queue` visible
   - Binding avec `user.routingkey` visible

2. Vérifier les logs du Consumer :
   - Messages "persisted User(...)"
   - Messages "User recieved: User(...)"

3. Vérifier dans MySQL :
   ```sql
   SELECT * FROM user;
   ```
   Les lignes envoyées doivent apparaître dans la table.

## Concepts techniques

### Exchange Types

**Mini-projet 1 : TopicExchange**
- Permet un routage flexible avec des patterns de routing key
- Exemple : `message_routingKey`
- Évolutif pour plusieurs patterns de messages

**Mini-projet 2 : DirectExchange**
- Routage exact par matching de routing key
- Plus simple et performant pour des cas d'usage directs

### Sérialisation JSON

Les deux projets utilisent `Jackson2JsonMessageConverter` pour :
- Sérialiser les objets Java en JSON côté Producer
- Désérialiser le JSON en objets Java côté Consumer

### Configuration RabbitMQ

**Mini-projet 1 :**
- Configuration basique avec `spring.rabbitmq.addresses`
- Déclaration dynamique de l'exchange, queue et binding via `@Bean`

**Mini-projet 2 :**
- Configuration explicite avec `CachingConnectionFactory`
- Authentification (username/password)
- Durabilité activée pour l'exchange et la queue

### Persistence MySQL

**Mini-projet 2 - Consumer :**
- Utilise Spring Data JPA
- Table `user` créée automatiquement via `ddl-auto: update`
- Entité `User` avec `@Entity` et `@GeneratedValue` pour l'ID

## Dépannage

### Erreur de connexion RabbitMQ
- Vérifier que RabbitMQ est démarré : `docker ps`
- Vérifier le port 5672 : `netstat -an | findstr 5672` (Windows)

### Erreur de connexion MySQL
- Vérifier que MySQL est démarré
- Vérifier les credentials dans `application.yml`
- Vérifier que la base `testdb` existe

### Messages non reçus
- Vérifier que le Consumer est démarré
- Vérifier que les noms d'exchange, queue et routing key sont identiques
- Vérifier dans l'interface RabbitMQ que le binding est correct

## Auteur

Projets créés dans le cadre d'un TP sur les microservices avec RabbitMQ.

