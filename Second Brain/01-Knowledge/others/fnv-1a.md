---
created: 2026-04-17
tags:
  - type/concept
  - status/draft
  - lang/java
related: []
---

# Thuật toán băm FNV-1a (FNV-1a Hash Algorithm)

## 1. What

FNV-1a (Fowler-Noll-Vo variant 1a) là một thuật toán băm phi mã hóa (non-cryptographic hash function) được thiết kế tối ưu cho tốc độ và phân tán đều trên các chuỗi ngắn. Thuật toán hoạt động bằng cách xử lý từng byte của input theo trình tự: XOR byte đó với hash hiện tại, sau đó nhân kết quả với một số nguyên tố đặc biệt (FNV prime). Đây là biến thể cải tiến của FNV-1 bằng cách đảo ngược thứ tự hai phép toán, giúp tăng hiệu ứng avalanche (một bit thay đổi đầu vào làm thay đổi nhiều bit đầu ra).

---

## 2. Why

Trước khi FNV phổ biến, các lựa chọn hash cho hash table và checksum gặp nhiều vấn đề:

- Các hàm băm mã hóa như MD5 hay SHA-256 quá chậm cho các tác vụ thời gian thực như tra cứu hash table, vì chúng được thiết kế cho bảo mật chứ không phải tốc độ.
- Các hàm băm đơn giản như cộng dồn mã ASCII (`sum of char codes`) tạo ra rất nhiều xung đột khi dữ liệu chỉ khác nhau một chút (ví dụ: "abc" và "abd" có thể ra cùng một hash).
- Java built-in `String.hashCode()` dùng polynomial rolling hash (31 làm base) ổn định nhưng có thể bị khai thác để tạo hash collision có chủ đích (HashDoS attack).

FNV-1a ra đời để đáp ứng sự cân bằng: tốc độ cực nhanh (chỉ là một vòng lặp với XOR và nhân), phân tán dữ liệu tốt trên chuỗi ngắn, và đơn giản đủ để triển khai thủ công mà không cần thư viện ngoài.

---

## 3. Mental Model

Hãy tưởng tượng FNV-1a như một **máy trộn màu siêu tốc**.

Mỗi byte dữ liệu đưa vào là một giọt màu. Máy thực hiện hai bước cho mỗi giọt:

Bước 1 — **XOR**: "Đánh tan" giọt màu mới vào hỗn hợp hiện tại. Phép XOR đảm bảo rằng mỗi byte đều trực tiếp làm thay đổi trạng thái hash, ngay cả khi byte đó là 0. Đây là điểm cải tiến của FNV-1a so với FNV-1 — trong FNV-1, nếu XOR sau khi nhân, thì các byte giá trị nhỏ ở cuối chuỗi có ảnh hưởng rất hạn chế.

Bước 2 — **Nhân với FNV Prime**: "Xoay tròn" hỗn hợp thật mạnh. Phép nhân với một số nguyên tố đặc biệt (được chọn kỹ để có tính chất lan truyền bit tốt trên cả 32 hoặc 64 bit) làm cho kết quả XOR ở bước 1 lan tỏa ra toàn bộ chiều rộng của giá trị hash. Tràn số (overflow) xảy ra một cách có chủ đích — đây là một phần của thuật toán, không phải lỗi.

Kết quả cuối cùng là một màu sắc duy nhất đại diện cho toàn bộ các giọt màu. Chỉ thay đổi một byte trong input, kết quả hash sẽ hoàn toàn khác.

---

## 4. Where it fits

```
Raw Data (String / byte[])
    |
    v
FNV-1a loop: for each byte => hash = (hash XOR byte) * FNV_PRIME
    |
    v
Hash Value (32-bit int hoặc 64-bit long)
    |
    v
Hash Table Index     Cache Key     Shard Key     Checksum
```

FNV-1a nằm ở tầng thấp nhất của stack, là primitive được các cấu trúc dữ liệu và hệ thống phân tán gọi đến.

---

## 5. When to use

- Hash table / hash map nội bộ: khi cần hash function nhanh để tính bucket index, đặc biệt với key là string ngắn đến trung bình.
- Cache key generation: tạo khóa ngắn, duy nhất cho các đối tượng cache (session data, query result), nhanh hơn MD5 nhiều lần.
- Checksum nhanh: kiểm tra xem nội dung file/config có thay đổi giữa hai lần đọc hay không, không cần toàn vẹn mã hóa.
- Sharding key: phân tán dữ liệu vào các bucket/shard dựa trên hash của user_id hoặc tenant_id — yêu cầu phân tán đều, không yêu cầu bảo mật.
- Fingerprint trong memory: nhận diện nhanh các object duplicate trong tập dữ liệu lớn dựa trên hash của nội dung, trước khi đi vào so sánh byte-by-byte.
- Embedded / resource-constrained environments: thiết bị nhúng ít RAM/CPU chỉ có thể chạy phép XOR và nhân — FNV-1a chỉ cần dùng bấy nhiêu.

---

## 6. When NOT to use

- Bảo mật (Security): tuyệt đối không dùng cho hash mật khẩu, chữ ký số, MAC, hoặc bất kỳ mục đích xác thực nào. FNV-1a không có tính chất one-way (có thể bị tấn công brute-force để đảo ngược), không có salt, và dễ bị tấn công collision có chủ đích.
- Lưu hash vào database để so sánh lâu dài: nếu logic nghiệp vụ yêu cầu "hash này không bao giờ xung đột", hãy dùng SHA-256. FNV-1a là 32/64-bit — với 2^32 giá trị, xác suất xung đột khá cao khi tập dữ liệu lớn (birthday paradox).
- Hash flooding protection: FNV-1a không có tham số secret key, kẻ tấn công có thể tính trước input gây xung đột để làm map/dict chậm xuống O(n). Dùng SipHash nếu dữ liệu đến từ user không tin tưởng.
- Dữ liệu lớn và cần hiệu suất tối đa: với chuỗi dài hơn 256 byte, MurmurHash3 hoặc xxHash có throughput cao hơn do khai thác SIMD và xử lý nhiều byte một lúc.

---

## 7. Trade-offs

| Ưu điểm (Pros) | Nhược điểm (Cons) |
|------|------|
| Triển khai cực đơn giản — chỉ vài dòng code, không cần thư viện | Không có tính bảo mật, dễ bị tạo collision có chủ đích |
| Tốc độ cao trên mọi loại CPU, kể cả CPU cũ không có SIMD | 32-bit variant có không gian hash nhỏ (4 tỷ giá trị), xung đột cao khi tập dữ liệu lớn |
| Phân tán rất tốt với chuỗi ngắn (< 64 bytes) | Không có tham số seed/key để ngẫu nhiên hóa kết quả |
| Kết quả xác định (deterministic) — cùng input luôn ra cùng output | Overflow khi tính toán là dự kiến — cần hiểu để không bị bối rối |
| Không phụ thuộc thư viện ngoài | Không phù hợp với dữ liệu dài lớn (MurmurHash3, xxHash tốt hơn) |
| Hoạt động tốt trên embedded / resource-constrained systems | Java không có built-in — phải tự implement hoặc dùng Guava |

---

## 8. Alternatives

| Alternative | Tốc độ | Avalanche | Bảo mật | Use case chính |
|---|---|---|---|---|
| FNV-1a (32/64-bit) | Rất nhanh | Tốt với chuỗi ngắn | Không | Hash table, checksum nhanh |
| MurmurHash3 (128-bit) | Nhanh | Rất tốt | Không | Hash table quy mô lớn, Bloom filter |
| xxHash (64-bit) | Nhanh nhất | Rất tốt | Không | Checksum file, deduplication |
| SipHash (128-bit) | Trung bình | Tốt | Có (secret key) | Hash map khi key đến từ user (chống HashDoS) |
| MD5 (128-bit) | Chậm | Rất tốt | Không (broken) | Checksum file (legacy), không dùng cho security |
| SHA-256 (256-bit) | Rất chậm | Xuất sắc | Có (one-way) | Password hash (với bcrypt), chữ ký số |
| Java String.hashCode() | Rất nhanh | Trung bình | Không | Built-in Java collections |
| Guava Hashing.murmur3_32() | Nhanh | Tốt | Không | Java project cần hash function tin cậy |

---

## 9. How

Đây là implement đầy đủ FNV-1a trong Java, hỗ trợ cả 32-bit và 64-bit variant, cùng với utility method tiện ích:

```java
/**
 * FNV-1a (Fowler-Noll-Vo) non-cryptographic hash function.
 *
 * Thuật toán:
 *   hash = OFFSET_BASIS
 *   for each byte b in input:
 *       hash = hash XOR b
 *       hash = hash * FNV_PRIME
 *
 * Constants nguồn: http://www.isthe.com/chongo/tech/comp/fnv/
 */
public final class Fnv1a {

    // ---------------------------------------------------------------
    // 32-bit constants
    // ---------------------------------------------------------------
    private static final int FNV32_OFFSET_BASIS = 0x811c9dc5;       // 2166136261
    private static final int FNV32_PRIME        = 0x01000193;       // 16777619

    // ---------------------------------------------------------------
    // 64-bit constants
    // ---------------------------------------------------------------
    private static final long FNV64_OFFSET_BASIS = 0xcbf29ce484222325L; // 14695981039346656037
    private static final long FNV64_PRIME        = 0x00000100000001B3L; // 1099511628211

    // Utility class -- không instantiate
    private Fnv1a() {}

    // ---------------------------------------------------------------
    // 32-bit API
    // ---------------------------------------------------------------

    /**
     * Tính FNV-1a 32-bit hash của một mảng byte.
     * Trả về int (có thể âm trong Java vì Java không có unsigned int).
     */
    public static int hash32(byte[] data) {
        int hash = FNV32_OFFSET_BASIS;
        for (byte b : data) {
            hash ^= (b & 0xFF);       // XOR với byte không dấu (0..255)
            hash *= FNV32_PRIME;      // overflow là có chủ đích -- Java wrap-around tự động
        }
        return hash;
    }

    /**
     * Tiện ích: hash một String sử dụng UTF-8 encoding.
     */
    public static int hash32(String text) {
        return hash32(text.getBytes(java.nio.charset.StandardCharsets.UTF_8));
    }

    /**
     * Trả về kết quả unsigned như long để dễ so sánh (tránh số âm gây nhầm).
     */
    public static long hash32Unsigned(byte[] data) {
        return Integer.toUnsignedLong(hash32(data));
    }

    // ---------------------------------------------------------------
    // 64-bit API
    // ---------------------------------------------------------------

    /**
     * Tính FNV-1a 64-bit hash của một mảng byte.
     * Long trong Java là signed, nhưng phép tính wrap-around là đúng.
     */
    public static long hash64(byte[] data) {
        long hash = FNV64_OFFSET_BASIS;
        for (byte b : data) {
            hash ^= (b & 0xFFL);      // XOR với byte không dấu cast sang long
            hash *= FNV64_PRIME;      // overflow wrap-around -- là một phần của thuật toán
        }
        return hash;
    }

    /**
     * Tiện ích: hash một String sử dụng UTF-8 encoding (64-bit).
     */
    public static long hash64(String text) {
        return hash64(text.getBytes(java.nio.charset.StandardCharsets.UTF_8));
    }

    /**
     * Tiện ích: kết hợp hash của nhiều trường thành một hash 64-bit.
     * Hữu ích khi tạo composite key từ nhiều field.
     */
    public static long hash64(String... parts) {
        long hash = FNV64_OFFSET_BASIS;
        for (String part : parts) {
            if (part == null) {
                // XOR với một sentinel value cho null
                hash ^= 0xFFFFFFFFFFFFFFFFL;
                hash *= FNV64_PRIME;
            } else {
                byte[] bytes = part.getBytes(java.nio.charset.StandardCharsets.UTF_8);
                for (byte b : bytes) {
                    hash ^= (b & 0xFFL);
                    hash *= FNV64_PRIME;
                }
            }
        }
        return hash;
    }

    /**
     * Tiện ích: tính bucket index để phân tán dữ liệu vào N shard.
     * Sử dụng Long.remainderUnsigned để tránh số âm gây ra bucket âm.
     */
    public static int bucket(String key, int numBuckets) {
        long h = hash64(key);
        // Long.remainderUnsigned xử lý đúng cả khi h < 0 (bit 63 = 1)
        return (int) Long.remainderUnsigned(h, numBuckets);
    }
}
```

Kiểm tra kết quả với các giá trị chuẩn (test vector):

```java
// Test vectors chuẩn được công bố tại http://www.isthe.com/chongo/tech/comp/fnv/
public class Fnv1aTest {

    @Test
    void test32BitKnownVectors() {
        // Giá trị này được xác nhận bởi tác giả thuật toán
        // FNV-1a 32-bit của chuỗi rỗng = offset basis
        assertEquals(0x811c9dc5, Integer.toUnsignedInt(Fnv1a.hash32("")));

        // "a" -> 0xe40c292c
        assertEquals(0xe40c292cL, Fnv1a.hash32Unsigned("a".getBytes(UTF_8)));

        // "foobar" -> 0xbf9cf968
        assertEquals(0xbf9cf968L, Fnv1a.hash32Unsigned("foobar".getBytes(UTF_8)));
    }

    @Test
    void test64BitKnownVectors() {
        // FNV-1a 64-bit của "foobar" = 0x85944171f73967e8
        assertEquals(0x85944171f73967e8L, Fnv1a.hash64("foobar"));
    }

    @Test
    void testAvalancheEffect() {
        // Một ký tự khác nhau phải tạo ra hash hoàn toàn khác
        long h1 = Fnv1a.hash64("hello");
        long h2 = Fnv1a.hash64("hellp"); // chỉ thay đổi ký tự cuối
        assertNotEquals(h1, h2);
        // So sánh bit: nhiều bit phải khác nhau
        long diff = h1 ^ h2;
        int differentBits = Long.bitCount(diff);
        assertTrue(differentBits >= 20, "Expected at least 20 bit differences, got " + differentBits);
    }

    @Test
    void testBucketDistribution() {
        // Kiểm tra phân tán đều trên 10 bucket
        int[] counts = new int[10];
        for (int i = 0; i < 10_000; i++) {
            int bucket = Fnv1a.bucket("user:" + i, 10);
            counts[bucket]++;
        }
        // Mỗi bucket nên có khoảng 1000 phần tử, cho phép lệch 10%
        for (int count : counts) {
            assertTrue(count > 900 && count < 1100,
                "Bucket count out of range: " + count);
        }
    }
}
```

Sử dụng với Guava nếu không muốn tự implement:

```java
// Guava cung cấp MurmurHash3 và các hàm băm khác -- FNV không có built-in
// Nhưng Guava.Hashing.goodFastHash() sử dụng MurmurHash3, hợp lý cho production
import com.google.common.hash.Hashing;
import java.nio.charset.StandardCharsets;

long murmurHash = Hashing.murmur3_128()
    .hashString("my-cache-key", StandardCharsets.UTF_8)
    .asLong();

// Nếu cần FNV-1a đúng chuẩn, dùng implementation ở trên hoặc thư viện:
// https://github.com/benzguo/fnv-java (lightweight)
```

---

## 10. Production Concerns

### Scaling

- 32-bit FNV-1a chỉ có 4 tỷ giá trị có thể. Với tập dữ liệu lớn (hàng chục triệu item), xác suất xung đột tăng nhanh theo birthday paradox. Luôn dùng 64-bit variant trong production với tập dữ liệu > 100k item.
- Khi dùng làm shard key, đảm bảo `numBuckets` là số nguyên tố hoặc power-of-two để phân tán đều hơn. Dùng `Long.remainderUnsigned()` để tránh bucket index âm khi hash value là long có bit 63 bằng 1.
- Để benchmark chính xác trong Java, dùng JMH (Java Microbenchmark Harness). JIT warm-up làm cho vòng lặp đầu tiên có vẻ chậm hơn thực tế.

### Failure

- Hash collision: FNV-1a không đảm bảo collision-free. Trong hash map hoặc cache, xử lý collision bằng separate chaining hoặc open addressing. Không assume "hash khác nhau" có nghĩa là "giá trị khác nhau".
- HashDoS attack: nếu key trong hash map đến từ nguồn không tin cậy (HTTP headers, query params), FNV-1a có thể bị khai thác để tạo các key có cùng hash, làm degradation hash map xuống O(n). Biện pháp: dùng SipHash-1-3 với secret key ngẫu nhiên khi khởi động.
- Overflow là tính năng, không phải lỗi: phép nhân `hash *= FNV32_PRIME` sẽ tràn số trong Java (vì Java không có unsigned int). Đây là có chủ đích — kết quả wrap-around theo module 2^32 là đúng. Không thêm bất kỳ kiểm tra overflow nào.

### Monitoring

- Nếu dùng FNV-1a làm bucket key trong hệ thống phân tán, theo dõi phân phối: mô hình hot-spot xảy ra nếu tập key có pattern đặc biệt (ví dụ: tất cả user ID bắt đầu bằng "1000"). Expose histogram số phần tử per bucket qua Micrometer.
- Track collision rate trong hash map nội bộ nếu performance-sensitive: nếu trung bình chain length > 3, xem xét đổi sang hash function khác hoặc tăng số bucket.

---

## 11. Common Mistakes

- Mistake: Dùng FNV-1 thay vì FNV-1a — nhân trước, XOR sau — vì sao chép nhầm thứ tự phép toán.
  Fix: Nhớ quy tắc FNV-1a: **XOR trước, nhân sau**. "1a" trong tên có nghĩa là "XOR first". Kiểm tra bằng test vector: `hash("a")` với FNV-1a 32-bit phải ra `0xe40c292c`.

- Mistake: Quên cast byte sang unsigned trước khi XOR, để byte âm (-128 đến -1 trong Java) làm sai kết quả.
  Fix: Luôn dùng `(b & 0xFF)` cho 32-bit hoặc `(b & 0xFFL)` cho 64-bit trước khi XOR. Java byte là signed (-128..127), nhưng FNV cần giá trị unsigned (0..255).

- Mistake: Dùng FNV-1a để hash mật khẩu hoặc tạo token xác thực, vì nghe "hash" tưởng là bảo mật.
  Fix: FNV-1a là non-cryptographic. Để hash mật khẩu dùng Argon2 (khuyến nghị nhất), bcrypt, hoặc scrypt. Để tạo token dùng HMAC-SHA256 hoặc JWT.

- Mistake: So sánh kết quả hash32() bằng `==` với literal int dương mà không biến đổi về unsigned, gây ra so sánh sai vì Java int là signed.
  Fix: Dùng `Integer.toUnsignedLong(Fnv1a.hash32(data)) == 0xe40c292cL` hoặc dùng phương thức `hash32Unsigned()` trả về long.

---

## 12. Sample Project

**Dự án: Simple In-memory Shard Router**

Constraint cứng:
- Phải phân tán các key vào đúng 8 shard (0-7).
- Không được dùng `HashMap` hoặc bất kỳ built-in map nào của Java.
- Phải tự implement hash table với FNV-1a làm hash function.
- Phân phối phải đều: sau khi insert 80.000 key ngẫu nhiên, mỗi shard chỉ chứa chênh lệch không quá 5% so với giá trị lý tưởng (10.000 key/shard).
- Xử lý collision bằng separate chaining (mỗi bucket là một LinkedList).
- Viết JUnit test kiểm tra distribution và collision rate.

Cách implement:
1. Viết `Fnv1aHasher` với hàm `hash64(String key)`.
2. Tạo `HashTable<K,V>` với array `buckets[]` là array of `LinkedList<Entry<K,V>>`.
3. `put(K, V)`: tính `bucketIndex = (int) Long.remainderUnsigned(hash64(key.toString()), buckets.length)`.
4. `get(K)`: tính bucket, duyệt chain để tìm entry có key khớp.
5. Viết test kiểm tra distribution uniformity với 80.000 key.

---

## 13. Interview

### Core Q&A

**Q: FNV-1a khác FNV-1 ở điểm nào, và tại sao FNV-1a tốt hơn?**
A: FNV-1 thực hiện: `hash = (hash * prime) XOR byte`. FNV-1a thực hiện: `hash = (hash XOR byte) * prime`. Sự khác biệt: trong FNV-1a, phép XOR xảy ra trước phép nhân, nên mỗi byte tác động trực tiếp vào hash trước khi được "khuếch đại" bởi phép nhân. Điều này cải thiện avalanche effect đặc biệt với các byte nhỏ ở cuối chuỗi — trong FNV-1, các byte cuối chỉ được XOR vào sau khi nhân, nên chúng có ít tác động hơn lên các bit cao.

**Q: Tại sao FNV prime được chọn là số nguyên tố, và tại sao là các giá trị cụ thể đó?**
A: Số nguyên tố đảm bảo rằng phép nhân "lan truyền" các bit input ra toàn bộ chiều rộng của hash value tốt hơn số hợp số. Các giá trị cụ thể (32-bit prime = 16777619, 64-bit prime = 1099511628211) được chọn qua thử nghiệm thực nghiệm để cho avalanche effect tốt nhất và tỉ lệ xung đột thấp nhất trên các tập dữ liệu thực tế như chuỗi, số, binary data.

**Q: Overflow có phải là lỗi trong FNV-1a không?**
A: Không. Overflow là có chủ đích và là một phần của thuật toán. Phép nhân `hash * FNV_PRIME` được tính theo modulo 2^32 (hoặc 2^64), tạo hiệu ứng wrap-around làm tăng tính ngẫu nhiên. Trong Java, phép nhân int và long đã wrap-around tự động theo two's complement, nên không cần xử lý thêm.

**Q: FNV-1a có thể dùng cho distributed sharding không?**
A: Có thể nhưng cần lưu ý: (1) FNV-1a là deterministic — cùng input luôn ra cùng hash, phù hợp để route request đến đúng shard. (2) Với 32-bit, không gian hash có thể gây "hot shard" nếu key distribution không đều — 64-bit tốt hơn. (3) Nếu thay đổi số shard (resharding), mỗi key sẽ map sang shard mới, cần có chiến lược migration dữ liệu. Consistent hashing (Rendezvous / Ketama) giải quyết vấn đề này tốt hơn.

**Q: Tại sao FNV-1a không dùng cho bảo mật dù có thuộc tính "không thể đảo ngược"?**
A: FNV-1a không có thuộc tính "không thể đảo ngược" (one-way) theo nghĩa mã học. Vì nó là hàm đơn giản với không gian hash nhỏ (32/64 bit), brute-force để tìm input có cùng hash là khả thi. Ngoài ra nó không có salt/key, nên hai người dùng cùng password sẽ có cùng hash. Cuối cùng, nó rất nhanh — điều này tốt cho hash table nhưng là lỗi cho bảo mật (tấn công brute-force chạy nhanh hơn).

**Q: FNV-1a so sánh với MurmurHash3 như thế nào trong thực tế?**
A: FNV-1a đơn giản hơn (ít code hơn, không cần thư viện), phân tán tốt hơn với chuỗi ngắn (< 32 bytes), và dễ kiểm chứng bằng test vector chuẩn. MurmurHash3 có throughput cao hơn trên chuỗi dài (xử lý 4-8 byte một lúc nhờ SIMD), và được dùng rộng rãi hơn trong các hệ thống lớn (Redis, Cassandra, Elasticsearch). Trong thực tế, với chuỗi key dưới 64 bytes (user_id, cache key...), hai hàm cho kết quả tương đương; chọn MurmurHash3 nếu project đã có Guava.

**Q: Làm thế nào tạo composite hash từ nhiều field trong Java?**
A: Tiến hành hash tất cả các field liên tiếp trong cùng một vòng lặp, không tạo chuỗi trung gian:
```java
public static long hash64(String... parts) {
    long hash = FNV64_OFFSET_BASIS;
    for (String part : parts) {
        byte[] bytes = part.getBytes(StandardCharsets.UTF_8);
        for (byte b : bytes) {
            hash ^= (b & 0xFFL);
            hash *= FNV64_PRIME;
        }
    }
    return hash;
}
```
Tránh ghép các field thành một chuỗi rồi hash vì: (1) tốn bộ nhớ allocation, (2) "user:1" + "0" và "user:10" + "" sẽ cho cùng kết quả nếu ghép đơn giản.

**Q: Làm sao biết nên chọn 32-bit hay 64-bit FNV-1a?**
A: Chọn 64-bit nếu: tập dữ liệu có thể vượt 10.000 item, cần làm shard key, hoặc hash được lưu vào storage. Chọn 32-bit chỉ khi: phần cứng 32-bit, cần tiết kiệm bộ nhớ, hoặc cần tương thích với hệ thống cũ. Quy tắc đơn giản: trong JVM hiện đại 64-bit, luôn dùng 64-bit FNV-1a — long operation không chậm hơn int trên CPU 64-bit.

### Scenario

**Scenario 1**: Bạn cần tạo cache key từ userId (long) và resourceType (String). Làm thế nào dùng FNV-1a?

Trả lời: Không nên chuyển long sang String rồi concat vì dễ gây key collision (`userId=1, resource="23"` và `userId=12, resource="3"` có thể cho cùng chuỗi). Thay vào đó, hash từng phần riêng hoặc serialize bằng cách đưa từng byte của long vào vòng lặp FNV:
```java
public static long hashUserResource(long userId, String resourceType) {
    long hash = FNV64_OFFSET_BASIS;
    // Hash 8 bytes của userId (big-endian)
    for (int i = 7; i >= 0; i--) {
        hash ^= ((userId >> (i * 8)) & 0xFFL);
        hash *= FNV64_PRIME;
    }
    // Hash bytes của resourceType
    for (byte b : resourceType.getBytes(StandardCharsets.UTF_8)) {
        hash ^= (b & 0xFFL);
        hash *= FNV64_PRIME;
    }
    return hash;
}
```

**Scenario 2**: Hệ thống của bạn bị tấn công DDoS bằng cách gửi hàng triệu request có key gây xung đột hash map. FNV-1a có chịu được không?

Trả lời: Không. FNV-1a là deterministic và public — kẻ tấn công có thể tính trước các key gây xung đột (tất cả map đến cùng bucket). Giải pháp: chuyển sang SipHash-1-3 với một secret key ngẫu nhiên được sinh khi khởi động JVM. SipHash có secret key 128-bit, kẻ tấn công không biết key nên không thể tạo collision có chủ đích. Trong Java, Guava chứa `Hashing.sipHash24()` sẵn.

**Scenario 3**: Bạn cần chia dữ liệu người dùng vào 16 database shard bằng userId. Sau 6 tháng, cần tăng lên 32 shard. FNV-1a xử lý việc này thế nào?

Trả lời: FNV-1a đơn thuần sẽ hash lại tất cả userId vào không gian mới, nên 50% dữ liệu cần migrate. Đây là vấn đề của mọi hash-based sharding, không riêng FNV. Giải pháp tốt hơn cho usecase này là consistent hashing (Ketama algorithm): thêm shard mới chỉ ảnh hưởng đến dữ liệu của shard kế bên, phần còn lại giữ nguyên. FNV-1a vẫn được dùng bên trong consistent hashing làm hàm hash cơ sở.

**Scenario 4**: Bạn muốn dùng FNV-1a để generate idempotent ID cho message trong queue (giống checksum). Cần lưu ý gì?

Trả lời: FNV-1a 64-bit phù hợp nếu: (1) không gian ID là private (không expose ra ngoài), (2) chấp nhận xác suất xung đột thấp (khoảng 1 trong 18 tỷ cho 64-bit). Nếu message ID là public hoặc có bảo mật, dùng SHA-256. Nếu yêu cầu collision-resistant đảm bảo, dùng UUID v4 (random). Thêm nữa, nếu cần idempotency key cho distributed system, nên dùng content-addressable hash như SHA-256 có giá trị chuẩn hơn.

**Scenario 5**: Khi implement FNV-1a trong Java, dòng nhân `hash *= FNV32_PRIME` trên `int` có thể bị compiler cảnh báo overflow hoặc sẽ cho kết quả sai không?

Trả lời: Không sai và không có vấn đề. Java int multiplication là modulo 2^32 theo two's complement — đây chính xác là những gì FNV-1a cần. Compiler không cảnh báo vì overflow trong Java là defined behavior (khác C/C++ là undefined behavior). Khi cần in kết quả dạng unsigned (để so sánh với test vector), chuyển bằng `Integer.toUnsignedString()` hoặc `Integer.toUnsignedLong()`.

---

## 14. References

- FNV hash algorithm official page (tác giả gốc, có đầy đủ test vectors): http://www.isthe.com/chongo/tech/comp/fnv/
- FNV Wikipedia: https://en.wikipedia.org/wiki/Fowler%–Noll%–Vo_hash_function
- Guava Hashing (MurmurHash3, SipHash trong Java): https://guava.dev/releases/snapshot-jre/api/docs/com/google/common/hash/Hashing.html
- SipHash paper (Juan Garay, Jean-Philippe Aumasson): https://131002.net/siphash/
- MurmurHash3 reference implementation: https://github.com/aappleby/smhasher
- xxHash specification: https://github.com/Cyan4973/xxHash/blob/dev/doc/xxhash_spec.md
- Baeldung — Hashing in Java: https://www.baeldung.com/java-hashcode
- Java `Integer.toUnsignedLong()` Javadoc: https://docs.oracle.com/en/java/docs/api/java.base/java/lang/Integer.html#toUnsignedLong(int)

---

## 15. Real-world Code

- Cassandra sử dụng Murmur3Partitioner (bình chọn FNV cho version cũ): https://github.com/apache/cassandra/blob/trunk/src/java/org/apache/cassandra/dht/Murmur3Partitioner.java
- Redis dùng SipHash cho hash table key để chống HashDoS: https://github.com/redis/redis/blob/unstable/src/siphash.c
- Go runtime dùng AES-based hash trên CPU hỗ trợ, FNV làm fallback: https://github.com/golang/go/blob/master/src/runtime/hash32.go
- Guava Hashing source — góc tham khảo cho Java hash function: https://github.com/google/guava/blob/master/guava/src/com/google/common/hash/Hashing.java
- FNV Java implementation nhẹ (không phụ thuộc): https://github.com/jakedouglas/fnv-java

---

## 16. Community

- Stack Overflow — "Why does Java's String.hashCode() use 31 as the multiplier?": https://stackoverflow.com/questions/299304/why-does-javas-string-hashcode-use-31-as-the-multiplier
- Stack Overflow — "When to use FNV vs MurmurHash vs CityHash": https://stackoverflow.com/questions/11899616/murmurhash-what-is-it
- Stack Overflow — "What is hash flooding / HashDoS?": https://stackoverflow.com/questions/8669946/application-vulnerability-due-to-non-random-hash-functions
- Reddit r/programming — "Hash functions: an empirical comparison": https://www.reddit.com/r/programming/comments/hash_function_comparison
- smhasher benchmark — thống kê so sánh các hash function phi mã hóa: https://github.com/rurban/smhasher
- Thorben Janssen blog — "How to choose the right hash function for your use case": https://thorben-janssen.com/hash-functions-java/
