# Enterprise SSL/TLS Certificate Distribution & Automation

Hệ thống tự động hóa quản lý, phân phối và cập nhật chứng chỉ SSL/TLS trên hạ tầng đa nền tảng (Kubernetes, Nginx, HAProxy) tuân thủ các nguyên tắc **DevSecOps**, **Zero Trust**, **Least Privilege** và **Defense in Depth**.

---

## 1. Kiến trúc hệ thống (Architecture Overview)

```mermaid
flowchart TD
    subgraph SecAdmin ["Quản Trị Viên An Ninh"]
        Admin[Admin / SecOps] -->|1. Upload .crt & .key trực tiếp| Vault[(HashiCorp Vault\nsecret/certs/DOMAIN)]
    end

    subgraph CI_CD ["Jenkins Controller (Bare Metal)"]
        J_Job[Jenkins Parameterized Pipeline]
        J_Job -->|2. Fetch Cert/Key bằng Token môi trường| Vault
        J_Job -->|3. Pre-flight Validation: OpenSSL Sanity Check| J_Val{Hợp lệ?}
        J_Val -->|PROD & DEPLOY| J_Appr[Manual Approval Gate]
        J_Val -->|FAIL| J_Abort[Dừng & Báo lỗi]
        J_Appr -->|Approved| Ans_Exec[Kích hoạt Ansible Engine]
    end

    subgraph Ansible_Deploy ["Ansible Automation Engine"]
        Ans_Exec -->|SSH with no_log: true| Nginx_Group
        Ans_Exec -->|SSH with no_log: true| HAProxy_Group
        Ans_Exec -->|SSH with no_log: true| K8s_Group
    end

    subgraph Targets ["Môi trường UAT / PROD (Private Network)"]
        subgraph Nginx_Group ["Nginx Web Servers (Rolling: 50%)"]
            N1[Backup cert cũ có timestamp]
            N2[Deploy crt 0644 & key 0600]
            N3[nginx -t]
            N4[systemctl reload nginx]
            N5[Rollback nếu fail]
            N1 --> N2 --> N3 --> N4
            N3 -.->|Error| N5
        end

        subgraph HAProxy_Group ["HAProxy Load Balancers (Serial: 1)"]
            H1[Backup bundle .pem cũ]
            H2[Gộp bundle .pem: crt + key]
            H3[haproxy -c -f haproxy.cfg]
            H4[systemctl reload haproxy]
            H5[Rollback nếu fail]
            H1 --> H2 --> H3 --> H4
            H3 -.->|Error| H5
        end

        subgraph K8s_Group ["Kubernetes Clusters"]
            K1[SSH tới Master/Bastion]
            K2[Tạo Secret YAML tạm trong /dev/shm]
            K3[kubectl apply -f]
            K4[Xóa/shred Secret YAML ngay lập tức]
            K1 --> K2 --> K3 --> K4
        end
    end

    subgraph Post_Verify ["Hậu kiểm tra (Post-Deployment Verification)"]
        PV[openssl s_client port 443 Handshake] --> Inspect[So khớp Serial & Expiry Date]
    end

    N4 --> PV
    H4 --> PV
```

---

## 2. Các nguyên tắc DevSecOps & Zero Trust cốt lõi

1. **Không rò rỉ Private Key vào CI/CD:**
   - Admin nạp trực tiếp cặp key vào HashiCorp Vault. Không bao giờ truyền Private Key qua tham số (Parameter) của Jenkins hoặc commit vào Git repository.
   - Thư mục tạm trên Jenkins Agent được tạo với quyền `0700` và bị hủy triệt để bằng lệnh `shred -u -z -n 3` trong khối `post { always }`.
2. **Cô lập đặc quyền theo môi trường (Isolation & Least Privilege):**
   - Tách biệt 2 Vault Token trong Jenkins Credential Manager: `vault-token-uat` và `vault-token-prod`.
   - Pipeline tự động chọn đúng Token dựa trên parameter `ENVIRONMENT`.
3. **Phòng vệ theo chiều sâu trong Ansible (Defense in Depth):**
   - Đặt `no_log: true` trên tất cả các task xử lý private key và render secret để ngăn lộ key vào Jenkins console logs.
   - Trên K8s Bastion: Manifest Secret YAML chỉ được tạo trong RAM disk (`/dev/shm`) với quyền `0600`, không ghi xuống ổ đĩa cứng vật lý và bị xóa ngay trong khối `always`.
4. **Độ sẵn sàng cao (High Availability) & Rollback an toàn:**
   - **Nginx:** `serial: 50%` đảm bảo luôn có ít nhất một nửa số node phục vụ lưu lượng; kiểm tra cú pháp `nginx -t` trước khi reload; tự động rollback phiên bản backup nếu cấu hình lỗi.
   - **HAProxy:** `serial: 1` reload từng node load balancer một; kiểm tra `haproxy -c` trước khi reload.
   - **Kubernetes:** Khai báo Secret dưới dạng `kubernetes.io/tls` chuẩn, tương thích với Ingress Controller (Nginx Ingress, Traefik, HAProxy Ingress).

---

## 3. Cấu trúc thư mục dự án

```text
.
├── Jenkinsfile                          # Pipeline chính (Parameterized, Vault, Approval, Ansible)
├── ansible.cfg                          # Cấu hình Ansible tối ưu và bảo mật
├── inventories/
│   ├── uat/
│   │   └── hosts.yml                    # Danh sách server Nginx, HAProxy, K8s Bastion UAT
│   └── prod/
│       └── hosts.yml                    # Danh sách server Nginx, HAProxy, K8s Bastion PROD
├── playbooks/
│   ├── deploy_certs.yml                 # Master playbook điều phối toàn bộ targets
│   └── roles/
│       ├── nginx_cert/                  # Role cập nhật cert cho Nginx
│       │   └── tasks/main.yml
│       ├── haproxy_cert/                # Role cập nhật PEM bundle cho HAProxy
│       │   └── tasks/main.yml
│       ├── k8s_cert/                    # Role cập nhật Secret TLS trên K8s qua Bastion
│       │   ├── tasks/
│       │   │   ├── main.yml
│       │   │   ├── apply_cluster.yml
│       │   │   └── apply_namespace.yml
│       │   └── templates/
│       │       └── tls-secret.yaml.j2
│       └── verify_cert/                 # Role kiểm tra TLS handshake thực tế qua OpenSSL
│           └── tasks/main.yml
└── README.md
```

---

## 4. Hướng dẫn thiết lập hạ tầng & Jenkinsfile để chạy Pipeline

Phần này hướng dẫn chi tiết từng bước cấu hình thực tế, từ việc phân quyền trên Vault, chuẩn bị môi trường Jenkins cho đến cách chạy pipeline lần đầu tiên.

---

### 4.1. Cấu hình HashiCorp Vault: Phân quyền & Tạo Token cho Jenkins

Trong môi trường Production, **tuyệt đối không dùng root token** cho CI/CD. Chúng ta sẽ tạo một Policy riêng chỉ cho phép Jenkins **đọc (read-only)** đúng thư mục chứa chứng chỉ `secret/certs/*`, không cho phép sửa, xóa hay xem trộm bất kỳ secret nhạy cảm nào khác (như mật khẩu Database, API keys...).

#### Bước 1: Tạo file Policy giới hạn quyền đọc (HCL)
Đăng nhập vào máy chủ Vault (hoặc máy của Admin có quyền quản trị Vault) và tạo file policy tên là `jenkins-cert-reader.hcl`:

```hcl
# Cho phép Jenkins đọc chứng chỉ (hỗ trợ Vault KV v2 mount tại secret/)
path "secret/data/certs/*" {
  capabilities = ["read"]
}

# Cho phép Jenkins xem danh sách cert để kiểm tra tồn tại
path "secret/metadata/certs/*" {
  capabilities = ["read", "list"]
}

# Hỗ trợ dự phòng nếu Vault của bạn đang dùng KV v1
path "secret/certs/*" {
  capabilities = ["read"]
}
```

Nạp policy này vào Vault bằng lệnh:
```bash
vault policy write jenkins-cert-policy jenkins-cert-reader.hcl
```
*(Bạn có thể kiểm tra lại bằng lệnh: `vault policy read jenkins-cert-policy`)*

---

#### Bước 2: Tạo Token riêng biệt cho môi trường UAT và PROD

Để đảm bảo nguyên tắc Least Privilege, ta tạo 2 token độc lập. Khi tạo token cho hệ thống tự động, cần lưu ý:
- Tham số `-orphan=true`: Cực kỳ quan trọng. Nó giúp token này trở thành token độc lập (không phụ thuộc vào session của Admin tạo ra nó). Nếu không có cờ này, khi token của Admin hết hạn thì token của Jenkins cũng sẽ tự động bị thu hồi theo, làm gãy pipeline!
- Tham số `-period=720h` (hoặc `-ttl=8760h` - 1 năm): Giúp token có hạn dài và có thể tự gia hạn (renewable).

**1. Tạo Token cho môi trường UAT:**
```bash
vault token create \
    -policy="jenkins-cert-policy" \
    -orphan=true \
    -renewable=true \
    -period="720h" \
    -display-name="jenkins-cert-uat"
```

**2. Tạo Token cho môi trường PROD:**
```bash
vault token create \
    -policy="jenkins-cert-policy" \
    -orphan=true \
    -renewable=true \
    -period="720h" \
    -display-name="jenkins-cert-prod"
```

**Kết quả trả về trên màn hình sẽ có dạng:**
```text
Key                  Value
---                  -----
token                hvs.CAESILf8... (Chuỗi token này bắt đầu bằng hvs. hoặc s.)
token_accessor       5d83a12...
token_duration       720h
token_renewable      true
token_policies       ["default" "jenkins-cert-policy"]
```

> [!TIP]
> Bạn hãy copy chuỗi giá trị ở dòng `token` (ví dụ `hvs.CAESILf8...`) để lưu vào Jenkins Credentials ở mục 4.3 bên dưới.

---

#### Bước 3: Cách nạp cặp Cert & Key mới lên Vault

Mỗi khi nhận được chứng chỉ mới từ nhà cung cấp (CA), quản trị viên cần nạp lên Vault trước khi chạy Jenkins.

**Cách 1: Nạp nhanh qua dòng lệnh (CLI):**
Chuẩn bị file chứng chỉ (Full Chain) và file Private Key:
```bash
# Đảm bảo đường dẫn cert/key chính xác:
vault kv put secret/certs/example.com \
    tls.crt=@example.com.crt \
    tls.key=@example.com.key
```
*(Nếu dùng KV v1: `vault write secret/certs/example.com tls.crt=@example.com.crt tls.key=@example.com.key`)*

**Cách 2: Nạp qua giao diện Web UI của Vault:**
1. Đăng nhập vào Vault Web UI (`https://vault.internal:8200`).
2. Vào Secret Engine `secret` -> tìm hoặc tạo thư mục `certs` -> tạo Secret mang tên domain (ví dụ: `example.com`).
3. Thêm 2 field dữ liệu dạng text (hoặc JSON):
   - Key: `tls.crt` | Value: Dán toàn bộ nội dung file `.crt` (bao gồm cả `-----BEGIN CERTIFICATE-----` đến `-----END CERTIFICATE-----`).
   - Key: `tls.key` | Value: Dán toàn bộ nội dung file `.key` (bao gồm cả `-----BEGIN RSA PRIVATE KEY-----` đến `-----END RSA PRIVATE KEY-----`).
4. Nhấn **Save**.

> [!IMPORTANT]
> **Kinh nghiệm thực chiến về Full Chain:** Trong file `.crt` nạp lên Vault, bạn nên ghép chứng chỉ domain và chứng chỉ trung gian (Intermediate CA) lại với nhau:
> ```bash
> cat domain.crt intermediate-ca.crt > fullchain.crt
> ```
> Điều này đảm bảo Nginx và HAProxy phục vụ đủ chuỗi tin cậy, tránh lỗi trình duyệt di động hay lệnh `curl` báo lỗi *"certificate signed by unknown authority"*.

---

### 4.2. Yêu cầu môi trường trên máy chủ Jenkins Controller (Bare Metal)

Vì Jenkins Controller chạy trực tiếp trên Bare Metal và là nơi gọi Ansible để kết nối tới các máy chủ trong Private Network, ta cần đảm bảo máy Jenkins có sẵn các công cụ cần thiết.

1. **Cài đặt các gói phần mềm trên máy Jenkins:**
   ```bash
   # Nếu máy Jenkins dùng RHEL / CentOS / Rocky Linux:
   sudo dnf install -y python3 python3-pip curl openssl openssh-clients
   sudo pip3 install ansible

   # Nếu máy Jenkins dùng Ubuntu / Debian:
   sudo apt-get update
   sudo apt-get install -y python3 python3-pip curl openssl openssh-client ansible
   ```
   *Giải thích lý do cần các công cụ này:*
   - `curl`: Dùng trong stage đầu của Pipeline để kết nối HTTP API kéo cert từ Vault.
   - `python3`: Dùng để parse JSON từ Vault an toàn, không sợ lỗi ký tự đặc biệt.
   - `openssl`: Dùng để thực hiện Pre-flight check (kiểm tra tính toàn vẹn của cert và key trước khi deploy).
   - `ansible`: Công cụ đẩy cert xuống các máy chủ và thực hiện zero-downtime reload.

2. **Cài đặt các Plugin bắt buộc trong Jenkins:**
   Đăng nhập Jenkins -> vào **Manage Jenkins** -> **Plugins** -> tab **Available plugins**, tìm và cài đặt:
   - **Pipeline** (`workflow-aggregator`) - *Bắt buộc* để chạy Jenkinsfile.
   - **Credentials Binding Plugin** (`credentials-binding`) - *Bắt buộc* để inject Vault token an toàn qua biến môi trường.
   - **SSH Agent Plugin** (`ssh-agent`) - *Bắt buộc* để Ansible kết nối SSH tới các server đích qua SSH Key.
   - **AnsiColor Plugin** (`ansicolor`) - *Tùy chọn*, cài nếu bạn muốn log terminal hiển thị màu mè cho dễ nhìn.

---

### 4.3. Cấu hình Credentials trong Jenkins

Vào **Manage Jenkins** -> **Credentials** -> **System** -> **Global credentials (unrestricted)** -> nhấn **Add Credentials** để tạo lần lượt 3 credential sau:

#### 1. Vault Token cho UAT (`vault-token-uat`):
- **Kind:** `Secret text`
- **Scope:** `Global`
- **Secret:** Dán chuỗi Token UAT đã tạo ở Bước 4.1 (ví dụ: `hvs.CAESILf8...`)
- **ID:** `vault-token-uat` *(Phải đặt chính xác tên này vì Jenkinsfile dùng ID này)*
- **Description:** `Vault Token read-only for UAT SSL Certs`

#### 2. Vault Token cho PROD (`vault-token-prod`):
- **Kind:** `Secret text`
- **Scope:** `Global`
- **Secret:** Dán chuỗi Token PROD đã tạo ở Bước 4.1
- **ID:** `vault-token-prod` *(Phải đặt chính xác tên này)*
- **Description:** `Vault Token read-only for PROD SSL Certs`

#### 3. SSH Key triển khai Ansible (`ansible-ssh-deploy-key`):
- **Kind:** `SSH Username with private key`
- **Scope:** `Global`
- **ID:** `ansible-ssh-deploy-key` *(Khớp với cấu hình trong Jenkinsfile)*
- **Description:** `SSH Deploy Key for Ansible Automation`
- **Username:** `devops`
- **Private Key:** Chọn `Enter directly` -> dán nội dung file SSH Private Key (file `id_rsa` hoặc `id_ed25519` của user `devops` có quyền SSH và sudo trên tất cả máy chủ Nginx, HAProxy, K8s Bastion).

> [!NOTE]
> Mặc định trong [Jenkinsfile](file:///d:/Antigravity/Cert_automation/Jenkinsfile) biến `VAULT_ADDR` được đặt là `https://vault.internal:8200`. Nếu địa chỉ Vault trong mạng của công ty bạn khác, bạn chỉ cần mở file [Jenkinsfile](file:///d:/Antigravity/Cert_automation/Jenkinsfile) và sửa lại dòng `VAULT_ADDR` ở mục `environment`.

---

### 4.4. Các bước tạo và kích hoạt Pipeline Job trên Jenkins UI

1. Tại trang chủ Jenkins, nhấn vào **New Item** ở menu bên trái.
2. Đặt tên cho Job (ví dụ: `SSL-TLS-Cert-Automation`) -> chọn loại **Pipeline** -> nhấn **OK**.
3. Cuộn xuống phần **Pipeline** ở cuối trang:
   - Mục **Definition**: Chọn `Pipeline script from SCM`.
   - Mục **SCM**: Chọn `Git`.
   - Mục **Repository URL**: Nhập địa chỉ Git repo của dự án này (ví dụ: `git@gitlab.internal:devops/cert-automation.git`).
   - Mục **Credentials**: Chọn SSH Key hoặc User/Token để Jenkins có quyền kéo code từ Git repo.
   - Mục **Branch Specifier**: Điền branch tương ứng (ví dụ: `*/main` hoặc `*/master`).
   - Mục **Script Path**: Điền đúng tên file là `Jenkinsfile`.
4. Nhấn **Save**.

#### 💡 Mẹo quan trọng cho lần chạy đầu tiên:
- Khi vừa tạo Job xong, bạn sẽ thấy menu bên trái hiển thị nút **Build Now** (chứ chưa có *Build with Parameters*).
- **Lý do:** Jenkins cần chạy qua `Jenkinsfile` một lần thì mới biết được pipeline này có những tham số nào để lưu vào database của Jenkins.
- **Cách làm:** Bạn cứ tự tin bấm **Build Now** một lần. Lần chạy này có thể bị fail hoặc bạn nhấn abort hủy cũng được, không sao cả.
- Sau khi build lần 1 xong, bạn **F5 (tải lại trang)** của Job: nút **Build with Parameters** sẽ xuất hiện với đầy đủ menu chọn:
  - `ENVIRONMENT`: `UAT` hoặc `PROD`
  - `TARGET_PLATFORM`: `ALL`, `KUBERNETES`, `NGINX`, `HAPROXY`
  - `CERT_DOMAIN`: Domain cần cập nhật (ví dụ: `example.com`)
  - `ACTION`: `DEPLOY` hoặc `VALIDATE_ONLY` (chỉ kiểm tra hợp lệ, không áp dụng)

---

## 5. Hướng dẫn chi tiết: Thêm một server HAProxy mới cần cài/thay Cert

Khi mở rộng hạ tầng hoặc thay thế một máy chủ HAProxy mới, quy trình nạp cert diễn ra hoàn toàn tự động qua các bước sau:

### Bước 1: Chuẩn bị trên máy chủ HAProxy mới (Target Host)

1. **Cài đặt HAProxy:** Đảm bảo HAProxy đã được cài đặt và service đã được bật:
   ```bash
   sudo systemctl enable haproxy
   ```

2. **Cấu hình User triển khai (`devops`):**
   Tạo user `devops` và cấp quyền `sudo` không cần mật khẩu để Ansible có thể ghi cert và reload service:
   ```bash
   sudo useradd -m -s /bin/bash devops
   echo "devops ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/devops
   sudo chmod 0440 /etc/sudoers.d/devops
   ```

3. **Cấu hình SSH Public Key:**
   Thêm SSH Public Key của Jenkins (tương ứng với `ansible-ssh-deploy-key`) vào file authorized_keys:
   ```bash
   sudo mkdir -p /home/devops/.ssh
   sudo nano /home/devops/.ssh/authorized_keys   # Dán SSH Public Key vào đây
   sudo chown -R devops:devops /home/devops/.ssh
   sudo chmod 700 /home/devops/.ssh
   sudo chmod 600 /home/devops/.ssh/authorized_keys
   ```

4. **Cấu hình HAProxy lắng nghe file SSL:**
   Trong file `/etc/haproxy/haproxy.cfg`, đảm bảo frontend được cấu hình trỏ tới đường dẫn chứng chỉ `.pem`:
   ```haproxy
   frontend https_front
       bind *:443 ssl crt /etc/haproxy/certs/example.com.pem
       mode http
       default_backend web_servers
   ```
   *(Thư mục `/etc/haproxy/certs` với quyền bảo mật `0750` sẽ được Ansible role tự động tạo nếu chưa tồn tại).*

---

### Bước 2: Cập nhật file Inventory tương ứng

Tùy vào môi trường của server mới, bạn mở file [inventories/uat/hosts.yml](file:///d:/Antigravity/Cert_automation/inventories/uat/hosts.yml) hoặc [inventories/prod/hosts.yml](file:///d:/Antigravity/Cert_automation/inventories/prod/hosts.yml).

Thêm thông tin máy chủ HAProxy mới vào nhóm `haproxy_servers`:

```yaml
    haproxy_servers:
      hosts:
        haproxy-prod-01.internal:
          ansible_host: 10.100.10.31
          haproxy_cert_dir: /etc/haproxy/certs
          haproxy_service_name: haproxy
        haproxy-prod-02.internal:
          ansible_host: 10.100.10.32
          haproxy_cert_dir: /etc/haproxy/certs
          haproxy_service_name: haproxy
        # Server HAProxy mới thêm vào:
        haproxy-prod-03.internal:
          ansible_host: 10.100.10.33
          haproxy_cert_dir: /etc/haproxy/certs
          haproxy_service_name: haproxy
```

Commit và push file inventory đã cập nhật lên Git repository.

---

### Bước 3: Kích hoạt Jenkins Pipeline để phân phối Cert

1. Vào Jenkins Job đã cấu hình, chọn **Build with Parameters**.
2. Chọn các tham số:
   - `ENVIRONMENT`: Chọn `UAT` hoặc `PROD`.
   - `TARGET_PLATFORM`: Chọn `HAPROXY` *(Chỉ triển khai riêng nhóm HAProxy, không can thiệp Nginx hay Kubernetes)*.
   - `CERT_DOMAIN`: Tên domain khớp với path trên Vault (ví dụ: `example.com`).
   - `ACTION`: Chọn `DEPLOY`.
3. Bấm **Build**.
4. **Nếu môi trường là PROD:** Pipeline dừng tại stage **Production Approval Gate**. Quản trị viên kiểm tra log hiển thị domain và target platform, sau đó bấm **Approve & Deploy to PROD**.
5. **Cơ chế triển khai tự động:**
   - Ansible duyệt qua danh sách HAProxy theo cơ chế `serial: 1` (lần lượt từng server).
   - Tự động gộp `.crt` và `.key` thành file `.pem` an toàn với quyền `0600`.
   - Thực thi `haproxy -c -f /etc/haproxy/haproxy.cfg` kiểm tra cú pháp trước khi reload.
   - Gọi `systemctl reload haproxy` zero-downtime.
   - Chạy role `verify_cert` bắt tay TLS qua OpenSSL trên cổng 443 để xác nhận chứng chỉ thực tế đang phục vụ.

---

## 6. Quy trình vận hành định kỳ khi gia hạn chứng chỉ (Runbook)

Khi nhận được cặp file `.crt` và `.key` mới từ Nhà cung cấp chứng thực số (CA):

1. **Bước 1: Nạp vào Vault**
   - Quản trị viên mở Vault UI hoặc chạy lệnh CLI cập nhật chứng chỉ mới vào path `secret/certs/<DOMAIN>`.
2. **Bước 2: Chạy kiểm thử trên UAT**
   - Vào Jenkins Pipeline, chọn:
     - `ENVIRONMENT`: `UAT`
     - `TARGET_PLATFORM`: `ALL`
     - `CERT_DOMAIN`: `<DOMAIN>`
     - `ACTION`: `DEPLOY`
   - Bấm **Build with Parameters**. Pipeline tự động fetch cert, deploy lên UAT Nginx, HAProxy, K8s và kiểm tra live handshake.
3. **Bước 3: Triển khai lên Production**
   - Vào Jenkins Pipeline, chọn:
     - `ENVIRONMENT`: `PROD`
     - `TARGET_PLATFORM`: `ALL` (hoặc từng thành phần nếu muốn chia phase)
     - `CERT_DOMAIN`: `<DOMAIN>`
     - `ACTION`: `DEPLOY`
   - Bấm **Build with Parameters**.
   - Pipeline thực hiện Pre-flight check, sau đó dừng tại bước **Production Approval Gate**.
   - Trưởng nhóm / Quản trị viên kiểm tra lại thông tin và bấm **Approve**.
   - Pipeline triển khai theo cơ chế rolling update, kiểm tra zero-downtime và trả kết quả xác thực.
