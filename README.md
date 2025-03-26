### Структура проекта (Gradle)
```
src
├── main
│   ├── java
│   │   └── com
│   │       └── example
│   │           └── microservice
│   │               ├── config
│   │               ├── controller
│   │               ├── service
│   │               ├── client
│   │               │   └── ExternalServiceClient.java
│   │               └── MicroserviceApplication.java
│   └── resources
│       └── application.yml
└── test
    ├── java
    │   └── com
    │       └── example
    │           └── microservice
    │               ├── service
    │               ├── client
    │               │   └── ExternalServiceClientTest.java
│                   └── MicroserviceApplicationTests.java
    └── resources
        ├── __files
        └── mappings
```

---

### 1. Добавьте зависимости в `build.gradle`
```groovy
plugins {
    id 'org.springframework.boot' version '3.1.4'
    id 'io.spring.dependency-management' version '1.1.3'
    id 'java'
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.cloud:spring-cloud-contract-wiremock:4.0.3'
    testImplementation 'com.github.tomakehurst:wiremock-standalone:2.35.0'
}

test {
    useJUnitPlatform()
}
```

---

### 2. Пример HTTP-клиента (Feign)
```java
// src/main/java/.../client/ExternalServiceClient.java
@FeignClient(name = "external-service", url = "${external.service.url}")
public interface ExternalServiceClient {
    @GetMapping("/api/data")
    ResponseEntity<String> getData(@RequestHeader("Authorization") String token);
}
```

---

### 3. Полный пример теста с WireMock
```java
// src/test/java/.../client/ExternalServiceClientTest.java
@SpringBootTest
@AutoConfigureWireMock(port = 8081) // (1) Запуск WireMock на порту 8081
@EnableFeignClients // (2) Активация Feign-клиентов
@ActiveProfiles("test") // (3) Использование тестового профиля
public class ExternalServiceClientTest {

    @Autowired
    private ExternalServiceClient client;

    @Test
    void testGetMockedData() {
        // Arrange: Настройка стаба (stub)
        stubFor(get(urlPathEqualTo("/api/data")) // (4) Метод HTTP GET
                .withHeader("Authorization", equalTo("Bearer test123")) // (5) Проверка заголовка
                .willReturn(aResponse() // (6) Определение ответа
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("{\"result\": \"success\"}")));

        // Act: Вызов тестируемого метода
        ResponseEntity<String> response = client.getData("Bearer test123");

        // Assert: Проверка результата
        assertEquals(200, response.getStatusCode().value());
        assertTrue(response.getBody().contains("\"result\": \"success\""));

        // Verify: Проверка вызова (7)
        verify(exactly(1), // (8) Количество вызовов
                getRequestedFor(urlPathEqualTo("/api/data"))
                        .withHeader("Authorization", equalTo("Bearer test123")));
    }
}
```

---

### Список методов WireMock для тестов
| **Метод** | **Описание** | **Перевод** |
|----------|-------------|------------|
| `stubFor()` | Создает стаб (заглушку) для HTTP-запроса | Создать заглушку |
| `verify()` | Проверяет, что запрос был выполнен | Проверить вызов |
| `get()` | Определяет стаб для GET-запроса | Метод GET |
| `post()` | Определяет стаб для POST-запроса | Метод POST |
| `put()` | Определяет стаб для PUT-запроса | Метод PUT |
| `delete()` | Определяет стаб для DELETE-запроса | Метод DELETE |
| `urlEqualTo()` | Точное совпадение URL | URL равно |
| `urlPathEqualTo()` | Совпадение пути URL (без параметров) | Путь URL равно |
| `withHeader()` | Проверка заголовка запроса | С заголовком |
| `withRequestBody()` | Проверка тела запроса | С телом запроса |
| `willReturn()` | Определяет ответ на запрос | Вернет |
| `aResponse()` | Создает объект ответа | Ответ |
| `withStatus()` | Устанавливает HTTP-статус ответа | Со статусом |
| `withBody()` | Устанавливает тело ответа | С телом |
| `exactly()` | Точное количество вызовов | Ровно |
| `atLeast()` | Минимальное количество вызовов | Как минимум |
| `atMost()` | Максимальное количество вызовов | Не более чем |
| `getRequestedFor()` | Проверка GET-запроса в verify() | Проверка GET-запроса |
| `postRequestedFor()` | Проверка POST-запроса в verify() | Проверка POST-запроса |

---

### 4. Настройка через JSON-файлы (альтернатива коду)
Создайте файл `src/test/resources/mappings/external-service.json`:
```json
{
  "request": {
    "method": "GET",
    "urlPath": "/api/data",
    "headers": {
      "Authorization": {
        "equalTo": "Bearer test123"
      }
    }
  },
  "response": {
    "status": 200,
    "body": "{\"result\": \"success\"}",
    "headers": {
      "Content-Type": "application/json"
    }
  }
}
```

---

### 5. Настройка `application.yml` для тестов
```yaml
# src/test/resources/application.yml
external:
  service:
    url: http://localhost:${wiremock.server.port}
```

---

### Ключевые особенности:
1. **Программные стабы** через `stubFor()` позволяют динамически настраивать ответы
2. **Декларативные стабы** через JSON-файлы в `mappings/` подходят для сложных сценариев
3. **Проверки verify()** гарантируют, что запросы были отправлены
4. **Интеграция с Feign** позволяет тестировать реальные HTTP-клиенты
5. **Профиль `test`** изолирует настройки для тестовой среды

Для микросервисов рекомендуется:
- Использовать **рандомизированные порты** через `@AutoConfigureWireMock(port = 0)`
- Комбинировать с **Spring Cloud Contract** для проверки контрактов API
- Тестировать **негативные сценарии** (например, таймауты, ошибки 5xx)
