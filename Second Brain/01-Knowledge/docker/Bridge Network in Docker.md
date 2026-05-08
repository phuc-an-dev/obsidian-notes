---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Compose.md]]"
  - "[[Container Orchestration.md]]"
---

## 1. What
Bridge Network là kiểu mạng mặc định (default driver) của Docker Engine. Nó tạo ra một mạng ảo nội bộ trên máy host, cho phép các container kết nối vào đó để liên lạc với nhau, đồng thời cung cấp sự cô lập với các mạng khác và với mạng vật lý bên ngoài.

## 2. Why
Trong một hệ thống container hóa, chúng ta cần:
- **Sự cô lập**: Không muốn toàn bộ port của container bị phơi bày ra mạng vật lý.
- **Liên lạc nội bộ**: Cho phép ứng dụng web gọi tới database một cách an toàn.
- **Tự động hóa**: Không muốn phải quản lý IP tĩnh cho từng container thủ công.
Bridge Network giải quyết các vấn đề này bằng cách cấp phát IP nội bộ và hỗ trợ DNS nội bộ (trong User-defined bridge).

## 3. Mental Model
Hãy tưởng tượng Bridge Network giống như một **"Bộ chia mạng (Switch) ảo"** nằm bên trong máy tính của bạn:
- Mọi container giống như một chiếc máy tính được cắm dây cáp vào bộ switch này.
- Chúng có thể nói chuyện với nhau qua switch một cách thoải mái.
- Nếu muốn liên lạc với thế giới bên ngoài (Internet), switch sẽ chuyển yêu cầu qua một "Cổng bảo vệ" (NAT trên host).

## 4. Where it fits
Vị trí trong luồng traffic:
`Container -> eth0 (Virtual) -> docker0 (Bridge Interface) -> eth0 (Physical Host) -> Internet`

## 5. When to use
- Đây là lựa chọn tốt nhất khi bạn chạy các ứng dụng trên một máy chủ duy nhất (Standalone containers).
- Khi muốn nhiều container (vd: Web và DB) làm việc cùng nhau trong một nhóm biệt lập.
- Nên dùng **User-defined Bridge** thay vì bridge mặc định để có tính năng tự động phân giải tên miền (DNS).

## 6. When NOT to use
- Khi cần hiệu năng mạng cực cao (trường hợp này dùng `host` network để bỏ qua lớp ảo hóa).
- Khi triển khai trên cụm máy chủ (Swarm/K8s) - trường hợp này dùng `overlay` network.
- Khi cần container có địa chỉ IP thật trong mạng vật lý (dùng `macvlan`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cô lập an toàn, bảo vệ ứng dụng khỏi các truy cập trái phép. | Có độ trễ (latency) nhỏ do phải đi qua lớp NAT/Bridge của host. |
| Hỗ trợ DNS nội bộ cực kỳ tiện lợi. | Không thể giao tiếp trực tiếp giữa các container trên hai máy host khác nhau. |
| Dễ dàng cấu hình và quản lý port mapping. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Host Network | Dùng chung network với host, nhanh nhưng không có sự cô lập. |
| Overlay Network | Cho phép container trên nhiều host khác nhau liên lạc với nhau. |
| None Network | Container hoàn toàn không có kết nối mạng. |

## 9. How
Các lệnh quản lý Bridge Network:

```bash
# 1. Tạo một mạng bridge riêng (Khuyên dùng)
docker network create my-app-net

# 2. Chạy container và gắn vào mạng vừa tạo
docker run -d --name db --network my-app-net postgres
docker run -d --name web --network my-app-net -p 80:80 my-web-app

# 3. Kiểm tra thông tin mạng (xem danh sách IP container)
docker network inspect my-app-net

# 4. Liệt kê các mạng hiện có
docker network ls
```

## 10. Production concerns
### Port Mapping
Trong bridge network, container không thể truy cập từ bên ngoài trừ khi bạn dùng `-p` (Publish port). Hãy chỉ publish những port thực sự cần thiết (vd: port 80 cho web, không publish port 5432 cho DB).

### DNS Resolution
Trong User-defined bridge, Docker chạy một máy chủ DNS nội bộ. Container Web có thể gọi database bằng lệnh `ping db` (tên container) thay vì phải nhớ IP 172.17.x.x.

## 11. Common mistakes
- Mistake: Sử dụng bridge mặc định (`bridge`) và mong chờ DNS phân giải tên container hoạt động. 
  Fix: Tính năng này chỉ có trên User-defined bridge.
- Mistake: Quên rằng IP của container có thể thay đổi sau khi restart. Luôn dùng Container Name để liên lạc.

## 12. Sample project
Thiết lập một hệ thống bảo mật:
1. Tạo 2 mạng: `frontend-net` và `backend-net`.
2. Container Nginx nằm ở cả 2 mạng.
3. Container Web App chỉ nằm ở `backend-net`.
4. Kiểm tra: Nginx gọi được Web App, nhưng từ mạng ngoài không ai có thể "thấy" trực tiếp Web App.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Bridge mặc định và User-defined Bridge là gì?
   A: User-defined Bridge hỗ trợ phân giải tên miền (DNS) giữa các container, cung cấp khả năng cô lập tốt hơn và cấu hình linh hoạt hơn (MTU, MTU settings).

2. Q: Làm thế nào để container trong bridge network truy cập được internet?
   A: Docker Engine sử dụng IP Forwarding và IP Masquerade (một dạng NAT) trên máy host để chuyển tiếp yêu cầu từ container ra card mạng vật lý.

### Scenario
"Hai container của bạn nằm trên cùng một mạng bridge nhưng không ping thấy nhau. Bạn kiểm tra gì?"
-> Trả lời:
1. Kiểm tra xem cả hai đã thực sự join vào cùng một mạng chưa (`docker network inspect`).
2. Kiểm tra Firewall (iptables/ufw) trên máy host có đang chặn traffic nội bộ của Docker không.
3. Kiểm tra xem container có cài lệnh `ping` không (nhiều image alpine/slim bị lược bỏ lệnh này).

## 14. References
- Official Docs: [Bridge network driver](https://docs.docker.com/network/bridge/)
- Networking Guide: [Docker container networking](https://docs.docker.com/config/containers/container-networking/)

## 15. Real-world Code
Xem cấu hình mạng trong một file `docker-compose.yml` chuyên nghiệp:
```yaml
networks:
  app-tier:
    driver: bridge
```

## 16. Community
- Docker Networking Slack.
- Stack Overflow: Tag [docker-networking].
