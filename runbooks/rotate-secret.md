# Runbook: Rotate DB Secret với ESO

## Mục đích
Hướng dẫn xoay vòng (rotate) mật khẩu database mà không gây downtime.

## Điều kiện tiên quyết
- ESO operator đang chạy trong namespace `external-secrets`
- `SecretStore` `fake-store` đã được tạo trong namespace `demo`
- Pod mount `db-secret` dưới dạng volume (không phải env)

## Quy trình Rotate (≤ 60s, zero downtime)

### Bước 1: Cập nhật giá trị trên nguồn bên ngoài

**Nếu dùng AWS Secrets Manager:**
```bash
aws secretsmanager update-secret \
  --secret-id demo/db/password \
  --secret-string "new-password-$(date +%s)" \
  --region ap-southeast-1
```

**Nếu dùng Fake provider (lab):**
```bash
# Chỉnh sửa eso/secret-store.yaml, thay đổi value của demo/db/password
# Rồi commit & push → ArgoCD sync → ESO sẽ detect thay đổi
```

### Bước 2: Theo dõi ESO sync

```bash
# Kiểm tra ExternalSecret status
kubectl get externalsecret db-creds -n demo -w

# Xem chi tiết sync status
kubectl describe externalsecret db-creds -n demo

# Kiểm tra Secret đã cập nhật chưa (chờ tối đa refreshInterval = 1m)
kubectl get secret db-secret -n demo -o jsonpath='{.data.password}' | base64 -d; echo
```

### Bước 3: Xác minh Pod KHÔNG restart

```bash
# AGE của pod phải giữ nguyên sau khi rotate
kubectl get pod -n demo -l app=api

# Xác nhận file trong container đã được cập nhật
kubectl exec -n demo \
  $(kubectl get pod -n demo -l app=api -o name | head -1) \
  -- cat /etc/db-secret/password
```

## Kỳ vọng
| Kiểm tra | Kỳ vọng |
|----------|---------|
| `kubectl get secret db-secret -o jsonpath='{.data.password}' \| base64 -d` | Giá trị mới trong < 60s |
| `kubectl get pod -n demo` | AGE không đổi (pod không restart) |
| `grep -ri password .` | Không có plaintext password trong repo |

## Xử lý sự cố

### ESO không sync
```bash
kubectl logs -n external-secrets \
  $(kubectl get pod -n external-secrets -l app.kubernetes.io/name=external-secrets -o name | head -1) \
  | grep -i error
```

### Secret không cập nhật trong pod
- Volume mount tự reload sau khi kubelet detect Secret thay đổi (~1-2 phút)
- Nếu app đọc file mỗi request → không cần làm gì thêm
- Nếu app cache giá trị khi start → cần thêm logic reload hoặc restart app (không phải pod)
