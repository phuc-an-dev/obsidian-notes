---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[EventListener in Spring Boot]]"
---

## 1. What
**Event Publisher** là thành phần trong Spring Framework (thông qua interface `ApplicationEventPublisher`) chịu trách nhiệm phát đi các sự kiện trong hệ thống, cho phép các `EventListener` khác phản ứng lại mà không cần sự gắn kết trực tiếp.

## 2. Why (problem it solves)
- **Decoupling**: Service phát sự kiện (Publisher) không cần biết service nào sẽ xử lý sự kiện đó.
- **Extensibility**: Có thể dễ dàng thêm các chức năng mới (logging, mail, analytics) bằng cách thêm Listener mà không làm thay đổi logic nghiệp vụ của Publisher.

## 3. Mental Model
> "Publisher như một **đài phát thanh**. Nó chỉ cần phát tin tức ra sóng, còn ai muốn nghe (Listener) thì tự bật đài lên. Đài phát thanh không cần biết bạn đang ở nhà hay trên xe."

## 4. Where it fits
`Business Service (Publisher) → [ApplicationEventPublisher] → ApplicationEvent → [Listeners]`

## 5. How (minimal example)
```java
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public void completeOrder(Order order) {
        // ... logic
        eventPublisher.publishEvent(new OrderCompletedEvent(order));
    }
}
```

---

## DeepL API (Note liên quan)

## 1. What
**DeepL API** là dịch vụ dịch thuật AI chất lượng cao, thường được tích hợp vào các hệ thống Spring Boot để xử lý đa ngôn ngữ (i18n) hoặc dịch nội dung tự động.

## 2. When to use
- Khi hệ thống cần dịch nội dung người dùng nhập vào hoặc dịch các thông báo hệ thống sang nhiều ngôn ngữ khác nhau trong thời gian thực.

## 3. How (Integration with RestClient)
```java
@Service
public class TranslationService {
    private final RestClient restClient = RestClient.create("https://api-free.deepl.com");

    public String translate(String text, String targetLang) {
        return restClient.post()
            .uri("/v2/translate")
            .header("Authorization", "DeepL-Auth-Key ...")
            .body(Map.of("text", List.of(text), "target_lang", targetLang))
            .retrieve()
            .body(DeepLResponse.class)
            .getTranslations().get(0).getText();
    }
}
```

## 4. Production concerns
- **API Key**: Không bao giờ hardcode, dùng `Environment` variables hoặc Vault.
- **Caching**: Dịch thuật tốn phí và thời gian, hãy cache kết quả các câu dịch giống nhau (Redis).
- **Error Handling**: API DeepL có giới hạn (429 Too Many Requests), cần implement retry logic.

## 13. Interview
### Scenario
> "Làm sao để kết hợp Event Publisher và DeepL API?"
**A**: Khi người dùng tạo một bài viết mới, Service sẽ publish `ArticleCreatedEvent`. Một `EventListener` sẽ lắng nghe event này, gọi DeepL API để dịch bài viết sang các ngôn ngữ khác, rồi lưu vào DB. Việc này giúp luồng chính (tạo bài viết) nhanh hơn đáng kể.
