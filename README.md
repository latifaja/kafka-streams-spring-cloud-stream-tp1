# 📘 Kafka Streams & Spring Cloud Stream - TP1

## 🧩 Introduction
Ce projet démontre comment intégrer **Spring Cloud Stream** avec **Kafka** afin de produire, consommer et traiter en temps réel des événements de type `PageEvent`.  
L’objectif est de comprendre le fonctionnement d’un pipeline de données réactif avec Kafka Streams.

---

## ⚙️ 1. Création du Consumer simple

### 🎯 Objectif
Configurer un **consumer Kafka** capable de lire les messages envoyés sur un topic et d’afficher leur contenu.

### 🪜 Étapes
1. Créer un consumer sur le même topic que le producer.
2. Envoyer un message `"hello"` dans le topic.
3. Vérifier que le consumer reçoit bien le message dans la console.

📸 **Capture d’écran – Consumer recevant le message :**
> ![Consumer screenshot](images/image2.png)

---

## 🧱 2. Création de la classe `PageEvent`

### 🎯 Objectif
Créer un modèle d’événement représentant une page visitée, un utilisateur, une date et une durée.

### 💻 Code
```java
package org.example.kafkastreamsspringcloudstreamtp1.events;

import java.util.Date;

public record PageEvent(String name, String user, Date date, long duration) {}

````

Cette classe sera utilisée pour échanger des objets entre producer et consumer.

---

## 🚀 3. Création du contrôleur `PageEventController`

### 🎯 Objectif

Publier un objet `PageEvent` dans un topic Kafka à l’aide du **StreamBridge** de Spring Cloud Stream.

### 💻 Code

```java
package org.example.kafkastreamsspringcloudstreamtp1.controllers;

import org.example.kafkastreamsspringcloudstreamtp1.events.PageEvent;
import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.Date;
import java.util.Random;

@RestController
public class PageEventController {
    private final StreamBridge streamBridge;

    public PageEventController(StreamBridge streamBridge) {
        this.streamBridge = streamBridge;
    }

    @GetMapping("/publish")
    public PageEvent send(String name, String topic) {
        PageEvent event = new PageEvent(
                name,
                Math.random() > 0.5 ? "U1" : "U2",
                new Date(),
                10 + new Random().nextInt(1000)
        );
        streamBridge.send(topic, event);
        return event;
    }
}
```

Ce contrôleur permet de publier un événement via l’URL :
`http://localhost:8080/publish?name=page1&topic=T1`

📸 **Capture d’écran – Publication dans T1 :**

> ![Publication screenshot](images/image5.png)

---

## 🛰️ 4. Création du Consumer `PageEventHandler`

### 🎯 Objectif

Créer un bean `Consumer` qui lit les objets `PageEvent` depuis le topic Kafka et affiche leur contenu.

### 💻 Code

```java
package org.example.kafkastreamsspringcloudstreamtp1.hanndlers;

import org.example.kafkastreamsspringcloudstreamtp1.events.PageEvent;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;
import java.util.function.Consumer;

@Component
public class PageEventHandler {

    @Bean
    public Consumer<PageEvent> pageEventConsumer() {
        return (input) -> {
            System.out.println("************");
            System.out.println(input.toString());
            System.out.println("************");
        };
    }
}
```

Chaque message reçu sera affiché dans la console.

📸 **Capture d’écran – Affichage des événements dans la console :**

> ![Consumer output screenshot](images/image7.png)

---

## ⏱️ 5. Création d’un Supplier automatique

### 🎯 Objectif

Mettre en place un **supplier** qui envoie automatiquement un événement toutes les 200 ms dans le topic `T1`.

### 💻 Code

```java
@Bean
public Supplier<PageEvent> pageEventSupplier() {
    return () -> new PageEvent(
            Math.random() > 0.5 ? "P1" : "P2",
            Math.random() > 0.5 ? "U1" : "U2",
            new Date(),
            10 + new Random().nextInt(10000)
    );
}
```

Ce `Supplier` alimente le topic de manière continue, simulant une activité en temps réel.

📸 **Capture d’écran – Envoi automatique vers T1 :**

> ![Supplier screenshot](images/image10.png)

---

## ⚡ 6. Traitement temps réel avec Kafka Streams

### 🎯 Objectif

Appliquer un traitement de flux pour compter, toutes les 5 secondes, les événements dont la durée est supérieure à 100 ms.

### 💻 Code

```java
@Bean
public Function<KStream<String, PageEvent>, KStream<String, Long>> kStreamFunction() {
    return (stream) ->
            stream.filter((k, v) -> v.duration() > 100)
                    .map((k, v) -> new KeyValue<>(v.name(), 0L))
                    .groupByKey(Grouped.with(Serdes.String(), Serdes.Long()))
                    .windowedBy(TimeWindows.of(Duration.ofSeconds(5)))
                    .count()
                    .toStream()
                    .map((k, v) -> new KeyValue<>(k.key(), v));
}
```

Cette fonction permet de compter les pages actives sur des fenêtres temporelles glissantes.

📸 **Capture d’écran – Résultat du flux de traitement :**

> ![Stream processing screenshot](images/image12.png)

---

## 📊 7. Visualisation temps réel des statistiques

### 🎯 Objectif

Afficher en continu les résultats du traitement via un flux SSE (`Server-Sent Events`).

### 💻 Code

```java
@GetMapping(path = "/analytics", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<Map<String, Long>> analytics() {
    return Flux.interval(Duration.ofSeconds(1))
            .map(sequence -> {
                Map<String, Long> result = new HashMap<>();
                ReadOnlyWindowStore<String, Long> windowStore =
                        interactiveQueryService.getQueryableStore("count-store", QueryableStoreTypes.windowStore());
                Instant now = Instant.now();
                Instant from = now.minusMillis(5000);
                KeyValueIterator<Windowed<String>, Long> fetchAll = windowStore.fetchAll(from, now);
                while (fetchAll.hasNext()) {
                    KeyValue<Windowed<String>, Long> next = fetchAll.next();
                    result.put(next.key.key(), next.value);
                }
                return result;
            });
}
```

Accéder à `http://localhost:8080/analytics` pour visualiser les statistiques mises à jour en temps réel.

📸 **Capture d’écran – Visualisation des analytics :**

> ![Analytics screenshot](images/image14.png)

---

## ✅ Résultat final attendu

* Le **producer** publie régulièrement des événements.
* Le **consumer** les reçoit et les affiche.
* Kafka Streams calcule des statistiques en temps réel.
* L’endpoint `/analytics` diffuse les données de manière continue via SSE.

📸 **Capture d’écran – Résultat final complet :**

> ![Final result screenshot](images/image15.png)

---

## 🧠 Configuration requise

| Composant   | Version minimale                 |
| ----------- | -------------------------------- |
| Java        | 17+                              |
| Spring Boot | 3.x                              |
| Kafka       | Local + Zookeeper                |
| Binder      | Spring Cloud Stream Kafka Binder |

---

## ▶️ Lancement du projet

### Démarrage du serveur

```bash
mvn spring-boot:run
```

### Tests

* Publier un événement :
  `http://localhost:8080/publish?name=page1&topic=T1`
* Visualiser les statistiques :
  `http://localhost:8080/analytics`

---

## 📚 Ressources utiles

* [Documentation Spring Cloud Stream](https://docs.spring.io/spring-cloud-stream/docs/current/reference/html/)
* [Documentation Kafka Streams](https://kafka.apache.org/documentation/streams/)

```
```
