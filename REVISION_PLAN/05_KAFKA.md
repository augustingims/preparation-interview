# 05 — Apache Kafka

## 1. Architecture Kafka

```
Producers → [Kafka Cluster] → Consumers

Kafka Cluster :
├── Broker 1 (Leader de certaines partitions)
├── Broker 2 (Follower / Leader)
└── Broker 3 (Follower / Leader)

Topic "orders" :
├── Partition 0 [msg0, msg1, msg2, ...] → Broker 1
├── Partition 1 [msg0, msg1, msg2, ...] → Broker 2
└── Partition 2 [msg0, msg1, msg2, ...] → Broker 3
```

### Concepts Clés

| Concept | Description |
|---------|-------------|
| **Topic** | Canal de messages (comme une table de logs) |
| **Partition** | Unité de parallélisme et d'ordre |
| **Offset** | Position d'un message dans une partition (immuable) |
| **Consumer Group** | Groupe de consumers partageant la charge |
| **Replication Factor** | Nombre de copies de chaque partition |
| **Leader/Follower** | Un broker leader par partition pour les I/O |

**Règle fondamentale :** L'**ordre est garanti au sein d'une partition**, pas entre partitions.

---

## 2. Producers

```java
// Configuration Producer Spring Boot
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Durabilité
        props.put(ProducerConfig.ACKS_CONFIG, "all");    // attendre tous les replicas
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // exactly-once

        // Performance
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);         // 16KB
        props.put(ProducerConfig.LINGER_MS_CONFIG, 5);              // attendre 5ms pour batcher
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, OrderEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

// Envoi de messages
@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publish(OrderCreatedEvent event) {
        // Clé = customerId → tous les events du même customer → même partition → ordre garanti
        ProducerRecord<String, OrderEvent> record = new ProducerRecord<>(
            "orders",
            event.getCustomerId().toString(), // clé de partitionnement
            event
        );

        kafkaTemplate.send(record)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to send event {}", event.getOrderId(), ex);
                    // Gérer l'échec : retry, DLQ, alerting
                } else {
                    log.debug("Event sent to partition {} offset {}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

---

## 3. Consumers

```java
// Configuration Consumer
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, OrderEvent> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "email-service-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);

        // Commit d'offset
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // commit manuel
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // Gestion des erreurs de désérialisation
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.events");
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3); // 3 threads = 3 partitions max traitées en parallèle
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // Dead Letter Queue
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate),
            new FixedBackOff(1000L, 3) // 3 retries, 1s entre chaque
        ));
        return factory;
    }
}

// Listener
@Component
public class OrderCreatedListener {

    @KafkaListener(
        topics = "orders",
        groupId = "email-service-group",
        containerFactory = "factory"
    )
    public void handleOrderCreated(
        ConsumerRecord<String, OrderEvent> record,
        Acknowledgment ack
    ) {
        try {
            log.info("Processing order {} from partition {} offset {}",
                record.value().getOrderId(),
                record.partition(),
                record.offset());

            processOrderEvent(record.value());
            ack.acknowledge(); // commit offset seulement après succès

        } catch (Exception e) {
            log.error("Error processing order event", e);
            // Ne pas ack → redelivery (selon config)
            // Ou publier sur DLQ
        }
    }
}
```

---

## 4. Garanties de Livraison

| Garantie | Description | Config |
|----------|-------------|--------|
| **At-most-once** | Perte possible, pas de doublon | auto-commit=true, acks=0 |
| **At-least-once** | Pas de perte, doublon possible | commit manuel, acks=all |
| **Exactly-once** | Pas de perte, pas de doublon | idempotent producer + transactional |

```java
// Exactly-once avec Kafka Transactions
@Transactional("kafkaTransactionManager")
public void processAndForward(OrderEvent event) {
    // Lire depuis topic A, traiter, écrire dans topic B — atomique
    kafkaTemplate.send("processed-orders", enrichEvent(event));
}

// Idempotence côté consumer (si exactly-once Kafka non disponible)
@KafkaListener(topics = "orders")
public void handle(OrderEvent event) {
    // Vérifier si déjà traité (idempotency key en BDD)
    if (processedEventRepo.existsById(event.getEventId())) {
        log.warn("Event {} already processed, skipping", event.getEventId());
        return;
    }
    processEvent(event);
    processedEventRepo.save(new ProcessedEvent(event.getEventId()));
}
```

---

## 5. Avro & Schema Registry

```java
// Avro : sérialisation binaire + contrat de schéma
// Schema Registry : registre centralisé des schémas

// Avro Schema (orders.avsc)
{
  "type": "record",
  "name": "OrderCreatedEvent",
  "namespace": "com.example.events",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "customerId", "type": "long"},
    {"name": "amount", "type": "double"},
    {"name": "status", "type": {"type": "enum", "name": "Status", "symbols": ["PENDING","CONFIRMED"]}}
  ]
}

// Configuration avec Schema Registry
props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://schema-registry:8081");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
```

---

## Questions d'Interview — Kafka

---

### Q1 : Comment Kafka garantit-il l'ordre des messages ?

**Réponse :**

L'ordre est garanti **au sein d'une partition uniquement**. Pour garantir l'ordre des événements liés à une entité (ex: tous les événements d'une commande), il faut utiliser la même **clé de partitionnement** :

```java
// Tous les events de la commande ORD-123 → même partition
kafkaTemplate.send("orders", "ORD-123", event);

// Formule de partitionnement par défaut
partition = hash(key) % numberOfPartitions
```

**Si on a besoin d'ordre global** → utiliser 1 seule partition (mais perd le parallélisme).

---

### Q2 : Qu'est-ce qu'un Consumer Group et comment fonctionne le rebalancing ?

**Réponse :**

Un Consumer Group est un ensemble de consumers qui partagent la consommation d'un topic. Chaque partition est assignée à **un seul consumer** du groupe.

```
Topic : 3 partitions
Consumer Group A : 3 consumers → 1 partition par consumer (idéal)
Consumer Group A : 2 consumers → 1 consumer traite 2 partitions
Consumer Group A : 4 consumers → 1 consumer idle (une partition max par consumer)
```

**Rebalancing :** déclenché lors d'un join/leave du groupe (nouveau consumer, crash, timeout).
- Pendant le rebalancing → **arrêt momentané du traitement**
- Stratégies : `range`, `roundrobin`, `sticky` (minimise les réassignations)

**Impact :** Ne pas oublier de commit les offsets avant le rebalancing avec `ConsumerRebalanceListener`.

---

### Q3 : Quelle est la différence entre `acks=1` et `acks=all` ?

**Réponse :**

| Config | Comportement | Risque |
|--------|-------------|--------|
| `acks=0` | Envoi fire-and-forget, pas d'attente | Perte si broker crash |
| `acks=1` | Attendre acknowledgment du leader | Perte si leader crash avant réplication |
| `acks=all` | Attendre tous les ISR (In-Sync Replicas) | Latence accrue, mais durabilité maximale |

**Recommandation production :** `acks=all` + `min.insync.replicas=2` + `replication.factor=3`

---

### Q4 : Comment gérer les messages en erreur (Dead Letter Queue) ?

**Réponse :**

```java
// Configuration DLQ automatique (Spring Kafka)
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(
            record.topic() + ".DLT", // topic DLQ = topic original + ".DLT"
            record.partition()
        )
    );

    return new DefaultErrorHandler(
        recoverer,
        new ExponentialBackOff(1000L, 2) // 1s, 2s, 4s...
    );
}

// Consumer DLQ : traitement manuel, alerting, replay
@KafkaListener(topics = "orders.DLT")
public void handleDeadLetter(
    ConsumerRecord<String, OrderEvent> record,
    @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage
) {
    log.error("Dead letter received for order {}: {}", record.key(), exceptionMessage);
    alertingService.notify(record, exceptionMessage);
    // Stocker pour investigation et replay manuel
}
```

---

### Q5 : Comment optimiser les performances Kafka ?

**Réponse :**

**Côté Producer :**
- `batch.size` + `linger.ms` : batcher les messages pour réduire les I/O
- `compression.type=snappy/lz4` : réduire la taille des messages
- `buffer.memory` : augmenter si producer bloque fréquemment

**Côté Consumer :**
- `fetch.min.bytes` + `fetch.max.wait.ms` : batcher les fetch
- `max.poll.records` : nombre de messages par poll
- Concurrency = nombre de partitions (1 thread par partition)

**Côté Kafka :**
- Nombre de partitions = facteur de parallélisme max
- Repartitionner si throughput insuffisant (irréversible → planifier)
- `log.retention.hours` : rétention selon les besoins de replay

---

## Pièges Classiques

- Consumer qui consomme trop lentement → `max.poll.interval.ms` dépassé → rebalancing en boucle
- Ne pas gérer l'idempotence côté consumer → doublons lors de retries
- Augmenter le nombre de partitions d'un topic existant → rompt le partitionnement par clé pour les messages existants
- `commitSync()` après chaque message → dégradation sévère des performances (préférer `commitAsync()` ou par batch)
- Oublier de fermer le producer proprement → messages en buffer perdus
