---
author: Nguyễn Quang Vinh
pubDatetime: 2023-05-05
title: "Network in landing zone"
postSlug: network-landing-zone
featured: true
draft: false
tags:
  - network
  - gcp
ogImage: ""
description: "Guide tự viết cho bản thân về phần network trong landing zone"
---

# 1. Network Connectivity

Để dùng các dịch vụ của Google thì có nhiều cách để kết nối network của ta với Google, ở đây chỉ nói đến kết nối đến Google Cloud để truy cập vào VPC và Compute Engine từ on-premise network.

Có 3 product của Google cho ta lựa chọn là :

- **Cloud VPN**
- **Cloud Interconnect**
- **Cloud Router**

Cloud VPN là giải pháp low cost với băng thông thấp còn Cloud Interconnect có băng thông cao hơn.

Cloud Router cung cấp dynamic routing bằng cách dùng Border Gateway Protocol với kết nối của Cloud Interconnect và gateway Cloud VPN.

Có 2 loại VPN gateway là **HA VPN** và **Classic VPN**.

Có 2 lựa chọn cho Cloud Interconnect là Dedicated Interconnect và Partner Interconnect.

# 2. Network

Trên GCP thì thường dùng 2 loại kiến trúc cho network là **Shared VPC** và **Hub-and-spoke**. Tuy Shared VPC phổ biến hơn và đa phần trường hợp dùng kiến trúc này nhưng ta trong mô hình Hub-and-spoke ta có thể sử dụng Shared VPC.

![Một số yếu tố giúp quyết định network design. ](https://cloud.google.com/static/architecture/landing-zones/images/decide-network-design-flow.svg)

Một số yếu tố giúp quyết định network design.

### 2.1. Shared VPC

Trong mô hình này thì network policy và quyền control tài nguyên mạng được tập trung lại ở host project nên dễ quản lí hơn và đảm bảo được tính nhất quán giữa các project. Tài nguyên có thể bao gồm subnet, route, firewall.

Tài nguyên trong mạng Shared VPC kết nối với nhau qua internal IP address.

Khi triển khai với Landing zone ta sẽ dựng Shared VPC network cho từng môi trường (dev, test, prod). Mô hình này giúp các môi trường độc lập với nhau. Khi muốn giao tiếp giữa các môi trường, có thể sử dụng:

- **VPC Peering**
- **HA VPN**
- **Multi-NIC appliance**

![Sơ đồ bậc cao thiết kế mạng nếu dựng bằng fabric FAST](https://raw.githubusercontent.com/GoogleCloudPlatform/cloud-foundation-fabric/master/fast/stages/2-networking-d-separate-envs/diagram.svg)

Sơ đồ bậc cao thiết kế mạng nếu dựng bằng fabric FAST

Trong sơ đồ thiết kế trên, kết nối đến on-premise sẽ sử dụng HA VPN.

### 2.2. Hub-and-spoke

![Một sơ đồ ví dụ về hub-and-spoke](https://cloud.google.com/static/architecture/images/hub-spoke-departmental-segmentation.svg)

Một sơ đồ ví dụ về hub-and-spoke

Trong sơ đồ trên thì một VPC sẽ đống vai trò hub, dùng để chia sẻ tài nguyên cho các spoke VPC. Các spoke sẽ kết nối với hub bằng VPC peering hoặc VPN. Hub sẽ được kết nối đến on-premise network, ở giữa có thể có **centralized appliance** (transit VPC). Có thể sử dụng mô hình Shared VPC cho spoke VPC nhưng không được dùng cho hub VPC.

Mô hình hub-and-spoke không có tính bắc cầu, ví dụ như VPC A peer với VPC B, VPC A peer với VPC C không có nghĩa là B có thể truyền traffic đến trực tiếp chỗ C. Tức là các spoke không thể tương tác trực tiếp với nhau nếu kiến trúc hub-and-spoke được xây dựng bằng VPC peering. Còn với mô hình Shared VPC thì nếu 2 project trong cùng share VPC đó chung subnet thì vẫn có thể giao tiếp với nhau bình thường. Tuy có thể thay phương thức kết nối từ VPC peering thành Cloud VPN nhưng sẽ bị giới hạn băng thông, cụ thể là tổng băng thông của mỗi VPN tunnel cả ingress lẫn egress là 3 Gbps.

Mô hình này mang đến sự phân nhánh giữa các môi trường ở mức cao nhất, trong đó Shared VPC của mỗi môi trường sẽ kết nối đến on-premise bằng Cloud Interconnect hoặc Cloud VPN HA. Khi cần truyền dữ liệu giữa 2 môi trường thì sẽ truyền vòng qua mạng ở on-premise nên sẽ tốn kém hơn.

Nhìn chung việc triển khai mô hình chủ yếu sẽ rơi vào 3 trường hợp dựa theo cách kết nối trong nội bộ:

- **HA VPN**
  - Ưu điểm: tương thích tốt với các dịch vụ của GCP giúp tận dụng khả năng kết nối nội bộ, quản lý route dễ dàng, tránh được peering group shared quota và limit.
  - Nhược điểm: tốn thêm chi phí, có thể làm tăng độ trễ, cần dùng nhiều tunnel để tối đa băng thông.
  ![1](https://raw.githubusercontent.com/GoogleCloudPlatform/cloud-foundation-fabric/master/fast/stages/2-networking-a-peering/diagram.svg)

---

- **VPC Peering**

  - Ưu điểm: không tốn thêm chi phí, tối đa băng thông mà không cần cài đặt hay cấu hình, không thêm độ trễ, các môi trường đều được cô lập.
  - Nhược điểm: không có khả năng chuyển tiếp, “no selective exchange of routes”, một số quota và limit sẽ chia ra giữa các VPC trong cùng peering group.

![2](https://raw.githubusercontent.com/GoogleCloudPlatform/cloud-foundation-fabric/master/fast/stages/2-networking-b-vpn/diagram.svg)

---

- **Multi-NIC appliances**
  - Ưu điểm: thêm các tính năng bảo mật như Intrusion Prevention System, tích hợp tốt hơn với các hệ thống on-premise.
  - Nhược điểm: setup HA/fail over phức tạp. bị giới hạn bởi băng thông và khả năng scale của VM, thêm chi phí cho VM và license, “out of band management of a critical cloud component”.

![3](https://raw.githubusercontent.com/GoogleCloudPlatform/cloud-foundation-fabric/master/fast/stages/2-networking-c-nva/diagram.svg)
