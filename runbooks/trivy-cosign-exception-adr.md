# ADR: Exception CVE chưa có Patch

## Status: TEMPLATE (điền vào khi cần xin exception)

---

## Ngữ cảnh

Trong pipeline CI, Trivy scan với `ignore-unfixed: true` đã được bật — CVE chưa có patch tự động bỏ qua. Tuy nhiên, có trường hợp CVE HIGH/CRITICAL đã có patch nhưng chúng ta cần thời gian để nâng version.

## Quyết định

Khi CI fail do CVE HIGH/CRITICAL, nhóm phải:

1. **Ưu tiên 1**: Fix CVE bằng cách nâng base image hoặc dependency version.
2. **Nếu không thể fix ngay**: Viết ADR này với thời hạn rõ ràng (tối đa 14 ngày).
3. **Workaround trong CI**: Thêm CVE vào `.trivyignore` với comment và expiry.

## Template Exception

```
CVE-XXXX-XXXXX
# Lý do: vendor chưa ra patch cho alpine:3.xx
# Ảnh hưởng: <mô tả ngắn>
# Ngày tạo exception: YYYY-MM-DD
# Ngày hết hạn: YYYY-MM-DD (tối đa 14 ngày)
# Người approve: <tên>
# Ticket: <link jira/linear>
```

## Ví dụ file `.trivyignore`

```
# Tạo file .trivyignore trong src/api/ để bỏ qua CVE cụ thể
CVE-2023-12345
# alpine musl libc - vendor chưa có patch
# Expire: 2026-07-01
# Ticket: INFRA-123
```

## Hệ quả

- Exception phải được review mỗi 2 tuần
- Không được extend quá 2 lần
- Nếu vendor ra patch trong thời gian exception → phải fix ngay

---

# Runbook: Trivy + Cosign Supply Chain

## Thiết lập ban đầu (1 lần)

### 1. Tạo Cosign Keypair

```bash
# Chạy trên máy local, KHÔNG chạy trong CI
cosign generate-key-pair

# → Tạo ra:
#   cosign.key  (PRIVATE - đưa vào GitHub Secret, XÓA khỏi máy sau khi upload)
#   cosign.pub  (PUBLIC  - commit vào signing/cosign.pub trong repo)
```

### 2. Upload Private Key lên GitHub Secrets

```
GitHub repo → Settings → Secrets and variables → Actions → New repository secret

Tên: COSIGN_PRIVATE_KEY
Giá trị: nội dung cosign.key (toàn bộ, bao gồm -----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----)

Tên: COSIGN_PASSWORD
Giá trị: passphrase bạn nhập khi tạo key pair
```

### 3. Cập nhật Public Key trong repo

```bash
# Copy nội dung cosign.pub
cat cosign.pub

# Paste vào signing/cosign.pub (thay thế PLACEHOLDER)
# Commit lên repo
git add signing/cosign.pub
git commit -m "feat: add cosign public key for image signing"
git push origin lab2
```

### 4. Cập nhật ClusterImagePolicy

```bash
# Lấy public key dạng base64 để paste vào policies/cluster-image-policy.yaml
cat signing/cosign.pub | base64 -w0
# Paste giá trị này vào spec.authorities[0].key.data
```

## Kiểm chứng (Nghiệm thu Lab 2.2)

### Test A: Push image có CVE HIGH → CI phải đỏ

```bash
# Thêm package có lỗ hổng vào Dockerfile (test only)
# Kiểm tra GitHub Actions → job phải fail tại step "Scan image với Trivy"
```

### Test B: Deploy image chưa ký → Admission reject

```bash
# Gắn label enforce cho namespace (sau khi đã có image ký thành công)
kubectl label namespace demo policy.sigstore.dev/include=true

# Thử deploy image không có signature
kubectl run test-unsigned \
  --image=nginx:alpine \
  --namespace=demo

# Kỳ vọng: admission webhook reject với lỗi về missing signature
```

### Test C: Deploy image đã ký → Pass

```bash
# Image được build từ CI (đã qua Trivy + Cosign) phải chạy được
kubectl get rollout api -n demo
# Kỳ vọng: Healthy
```

## Verify Signature thủ công

```bash
# Verify 1 image cụ thể
cosign verify \
  --key signing/cosign.pub \
  ghcr.io/kaphudong/w10-api:<version>

# Kết quả thành công: in ra JSON với payload chứa thông tin image
```
