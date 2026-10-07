# KẾ HOẠCH CHUNG — SentraOps

*DevSecOps và LLMOps cho ứng dụng Gen-AI trên Kubernetes*

---

## 1. Tổng quan

**Bài toán:** Ứng dụng Gen-AI (chatbot RAG) có thêm bề mặt tấn công mà quy trình DevOps truyền thống không bao phủ: prompt injection, rò rỉ dữ liệu qua câu trả lời, file model độc hại. SentraOps dựng một quy trình theo chuẩn doanh nghiệp, **Develop → Scan → Red-team → Deploy → Monitor**, và chứng minh bằng bằng chứng cụ thể rằng các lớp bảo vệ hoạt động.

**Luận điểm bảo vệ:**

> Nhóm đo mức giảm rủi ro ở từng lớp (code, image, model, cluster, câu trả lời của AI), kèm chi phí (chặn nhầm, độ trễ) và nêu rõ những gì vẫn lọt qua.
> 

**Ngoài phạm vi:** không fine-tune model, không chạy production thật, không cloud managed (GKE/EKS), không canary tự động, không OpenTelemetry, không Ansible, không SBOM.

---

## 2. Công cụ (đã cắt gọn)

| Nhóm | Công cụ |
| --- | --- |
| Ứng dụng | FastAPI, RAG đơn giản với Chroma (1 pod), Ollama + Qwen2.5 0.5B–1.5B |
| Đóng gói và hạ tầng | Docker, Helm, Terraform, k3s trên 1 VPS (khoảng 4 vCPU / 8–16 GB RAM) |
| CI/CD | GitHub Actions, Gitleaks, Semgrep, Trivy (dependency, image, IaC), ModelScan |
| LLM Security | promptfoo (red-team theo OWASP LLM Top 10), guardrail tự viết (regex PII + heuristic injection) |
| GitOps và an toàn K8s | ArgoCD, Kyverno (2 policy), NetworkPolicy, Kubernetes Secret |
| Giám sát | Prometheus, Grafana |

---

## 3. Phân vai và ranh giới bàn giao

| Vai | Thành viên | Sở hữu |
| --- | --- | --- |
| **A — Infra & App** | Nhân | Terraform, k3s, Helm, FastAPI + RAG + Ollama, Chroma, tích hợp tổng thể |
| **B — CI/CD Security** | Phước | Quy tắc Git, GitHub Actions, các security gate, ArgoCD, Kyverno, NetworkPolicy |
| **C — LLM Security & Monitoring** | Đồng | promptfoo, guardrail, đo số liệu, Prometheus, Grafana, cảnh báo |

**Hợp đồng giao diện (chốt ngay tuần 0 để tránh lệch lúc ráp):**

- A → B: Dockerfile, đường dẫn Helm chart, endpoint `/health`.
- A ↔ C: gateway có một điểm cắm middleware cho guardrail (C viết, A tích hợp) và endpoint `/metrics`.
- B → cả nhóm: pipeline chạy các bước của A và C như job độc lập. Ai làm hỏng gate nào thì người đó sửa.

Mỗi người **tự demo và tự trả lời** phần mình, đồng thời nắm sơ đồ tổng thể của cả hệ thống.

---

## 4. Quy trình doanh nghiệp (giảng viên nhấn mạnh)

**Git:**

- `main` được bảo vệ, làm việc trên `feature/*`, mọi thay đổi qua Pull Request.
- PR cần ≥1 reviewer và các status check xanh, có CODEOWNERS và PR template.
- Commit theo quy ước (conventional commits). Công việc theo dõi bằng GitHub Projects (issue → PR).
- **Hai repo:** `app` (code, Dockerfile, Helm) và `gitops` (manifest theo môi trường).

**Pipeline theo giai đoạn:**

1. **PR:** lint + unit test → Gitleaks → Semgrep → Trivy (dependency + IaC)
2. **Merge vào main:** build image → Trivy image → ModelScan (artifact model) → push registry với tag SHA
3. **Deploy:** tự cập nhật tag trong repo `gitops` → ArgoCD sync vào `dev`
4. **staging/prod:** phê duyệt thủ công (GitHub Environments; cần kiểm tra giới hạn của repo private)
5. **Sau deploy:** smoke test + promptfoo mini (~10 prompt, bộ đầy đủ chạy tay hoặc theo lịch)


---

## 5. Kế hoạch theo tuần

| Tuần | A | B | C | Mốc cuối tuần |
| --- | --- | --- | --- | --- |
| **0** · 6–11/10 | Chốt VPS và chi phí | Tạo 2 repo, quy ước Git, GitHub Projects | Threat model 1 trang (mối đe dọa → lớp kiểm soát → bài test → chỉ số), cùng B | Chốt scope, ngày, phần cứng, hợp đồng giao diện |
| **1** · 12–18/10 | Terraform + k3s; FastAPI gọi Ollama bằng Docker Compose | Branch protection, CODEOWNERS; CI khung (lint, test, build) | Cài promptfoo; soạn ~40 prompt tấn công + ~40 prompt hợp lệ | App chạy local, CI xanh |
| **2** · 19–25/10 | RAG + Chroma, Dockerfile, Helm chart | Gitleaks, Semgrep, Trivy, push registry | **Baseline** chưa có guardrail: đo ASR | Pipeline chặn được secret giả; có số liệu "trước" |
| **3** · 26/10–1/11 | Deploy Helm thủ công lên k3s (dev) | Cài ArgoCD, sync từ `gitops`, pipeline tự bump tag | Guardrail v1; đo ASR, chặn nhầm, độ trễ | **Lát cắt dọc chạy được từ commit đến cluster** |
| **4** · 2–8/11 | Tách dev/staging/prod, health check, resource limit | Kyverno (2 policy), NetworkPolicy, bước ModelScan, phê duyệt prod | Prometheus + Grafana | Đủ các lớp bảo vệ |
| **5** · 9–15/11 | Tích hợp, sửa lỗi | Gate promptfoo smoke trong CI, Trivy IaC | Alert rule, metric guardrail | Vòng end-to-end đầu tiên |
| **6** · 16–22/11 | Chạy kịch bản A, ghi bằng chứng | Chạy kịch bản B, ghi bằng chứng | Chạy kịch bản C, ghi bằng chứng | Đủ bằng chứng cho 6 kịch bản. **Đóng băng tính năng 22/11** |
| **7** · 23–29/11 | Tấn công chéo (mỗi người thử vượt lớp bảo vệ của người khác), vá lỗi, viết báo cáo phần mình |  |  | Báo cáo bản nháp |
| **8** · 30/11–6/12 | Diễn tập từ môi trường sạch, quay video dự phòng, luyện Q&A |  |  | Sẵn sàng |
| **9** · 7–8/12 | Bảo vệ |  |  |  |

**Quy tắc sinh tồn:** nếu cuối tuần 3 chưa có lát cắt dọc chạy được thì cắt ngay (RAG dùng ngữ cảnh cố định thay vì Chroma, bỏ ModelScan) chứ không kéo dài. Sau 22/11 không thêm tính năng.

---

## 6. Sáu kịch bản demo

**A — Infra & App**

1. **Dựng lại từ số 0:** destroy rồi apply bằng một lệnh, app chạy lại như cũ.
2. **Sự cố và rollback:** đẩy một bản lỗi, rollback bằng `git revert` qua ArgoCD, đo thời gian phục hồi. Thêm cảnh sửa tay trên cluster rồi ArgoCD tự kéo về đúng trạng thái trong Git.

**B — CI/CD Security**

1. **PR bị chặn:** PR chứa secret giả và dependency có lỗ hổng, pipeline đỏ, không merge được.
2. **Chặn chuỗi cung ứng hai lớp:** CI chặn model/image xấu (ModelScan, Trivy); cluster chặn pod chạy root hoặc registry lạ (Kyverno).

**C — LLM Security & Monitoring**

1. **Guardrail trước/sau:** bảng ASR, tỷ lệ chặn nhầm, độ trễ cộng thêm; mỗi cấu hình chạy 3 lần; nêu cả điều guardrail không chặn được.
2. **Cảnh báo thời gian thực:** spam prompt độc, Grafana bắn alert, rồi đưa prompt đó vào bộ test hồi quy để vòng thật sự khép kín.

---