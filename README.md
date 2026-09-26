# EVO-PCA — Evolutionary Predictive & Camouflage Awareness Shield

**EVO-PCA** là hệ thống Tường Lửa & Phát Hiện Rủi Ro Đa Tầng (Multi-tier Security Shield) dành cho các ứng dụng thực thi tự động (Agentic Workflow & Tool Execution) sử dụng Mô Hình Ngôn Ngữ Lớn (LLM). Hệ thống cung cấp cơ cấu bảo vệ chủ động **nhiều tầng phòng thủ chiều sâu (Defense-in-Depth)** kết hợp giữa kiểm định quy tắc chính xác (Deterministic Regex), máy trạng thái heuristic đa bước (Multi-Step State Machine), học máy dự đoán theo chuỗi (LSTM Temporal Model & Markov Sequence Risk), Trọng tài nhận dạng ngữ nghĩa (LLM Judge Router), và bộ tổng hợp bỏ phiếu tín hiệu tập trung (VotingAggregator).

---

## 🏛️ Kiến Trúc Phòng Thủ Chiều Sâu (Defense-in-Depth Architecture)

```
[ Ingress Tool Call / Prompt Action ]
         │
         ├──► Tầng 0 (Tier 0): Deterministic Firewall & Canonicalization
         │       └─ Lọc chuỗi nguy hiểm tức thì (rm -rf, curl, eval, SQL injection, v.v.)
         │       └─ Chuẩn hóa Leet-speak (7→t, +→t, 9→g, 6→g, 8→b, ...)
         │       └─ Phát hiện RCE (curl|wget pipe to bash/sh)
         │
         ├──► Tầng 0.5 (Tier 0.5): Session-Aware Heuristic + Neural LSTM Shield
         │       ├─ SessionAwareTier05: Phát hiện tấn công phân tán xuyên bước (split-injection)
         │       │     └─ Behavioral flags theo phiên (11 loại: role_override, exfil_verb, ...)
         │       │     └─ Cross-step escalation & intent-shift detection
         │       └─ LSTMTier05Wrapper: Neural LSTM Temporal Model
         │             └─ Markov K=3 Semantic Risk (cosine distance trên Dense Embeddings)
         │             └─ Step-Counting Invariance Guard (kháng thiên vị bước đếm)
         │
         ├──► Multi-Step State Machine (HeuristicStateTracker)
         │       └─ Phát hiện chuỗi tấn công kill-chain đa giai đoạn
         │       └─ Recon → Exfiltration, Auth → PrivEsc, Evasion → Execution
         │       └─ TTL-based flag expiry & benign context whitelisting
         │
         ├──► Tầng V61: Adaptive ML Router & LLM Judge
         │       ├─ Fast-path: ML Model (Ensemble RF + LR) phân loại risk score
         │       ├─ Slow-path: LLM Action Judge (Ollama/gemma3:4b) cho điểm xám
         │       └─ Tự điều chỉnh ngưỡng trần (AdaptiveEscalationManager + Hysteresis Deadband)
         │
         ├──► LLM Session Judge (Tier 2B)
         │       └─ Kích hoạt khi ml_score ∈ [suspicious_band_min, REVIEW threshold)
         │       └─ Phân tích lịch sử session đa bước qua LLM
         │       └─ Hardened: canary token, XML escaping, strict format validator
         │
         ├──► Advanced Heuristics & VotingAggregator (Production Gate)
         │       ├─ PermissionGate: Phát hiện capability combo nguy hiểm
         │       ├─ InstructionBoundaryDetector: Nhận diện instruction nhúng trong data
         │       ├─ EmailVectorDetector: Phát hiện email indirect prompt injection
         │       ├─ EarlyIntentClassifier: Pre-screening prompt intent (jailbreak, exfil staging)
         │       ├─ VotingAggregator: Tổng hợp bỏ phiếu → MONITOR/REVIEW/DENY/QUARANTINE
         │       └─ SignalRegistry: Kiểm soát feedback loop (anti-E7 vulnerability)
         │
         ├──► GlobalThreatTracker (Cross-Session APT Correlation)
         │       └─ Phát hiện chiến dịch tấn công chậm xuyên session (low-and-slow)
         │       └─ Correlation rules: sensitive_read + external_send + cover_tracks
         │
         └──► Egress Sanitization (Context Sanitizer & Data Redactor)
                 ├─ ContextSanitizer: PASS / WRAP_UNTRUSTED / STRIP_AND_WRAP / QUARANTINE
                 └─ DataRedactor: Redact PII, API keys, PEM keys, credit cards, v.v.
```

---

## 🔑 Các Thành Phần Cốt Lõi

### 1. Kháng Bước Đếm (Step-Counting Invariance Guard)
Trong các phiên làm việc dài (long-running sessions), các mô hình neural theo chuỗi dễ bị thiên vị đếm bước (false positive khi bước thứ 50–100 chỉ đơn thuần là các thao tác lành tính lặp lại).
- **Giải pháp:** Hệ thống kích hoạt module **Rủi ro Chuỗi Markov ($K=3$)** sử dụng Khoảng cách Cosine trên Dense Embeddings từ bộ mô hình `all-MiniLM-L6-v2`.
- Nếu chuỗi thao tác chỉ là lặp lại lành tính ($\text{seq\_risk} < 0.05$), chốt giáp an toàn sẽ vô hiệu hóa báo động đếm bước của LSTM.
- Nếu xuất hiện bước nhảy vọt bất ngờ (ví dụ: từ `check_server_uptime()` bẻ lái sang `export_database_records()`), mốc delta $\ge 0.80$ (cấu hình trong `thresholds.json`) lập tức kích hoạt cầu dao **Block**.

> [!NOTE]
> **Lưu ý về Benchmark (ABSR / AgentDojo):** Nhánh phân loại theo xác suất của LSTM hiện đang bị tắt mặc định thông qua biến môi trường `TIER05_LSTM_ENABLED=False` để nhường tài nguyên cho V61 LLM Judge. Khi bật, ngưỡng chặn được đồng bộ qua `config/thresholds.json` → `tier05_lstm.block_threshold`.

### 2. Dịch Vụ Mô Hình Hợp Nhất (`SharedSemanticEncoder`) & Bồn Cache Có Kiểm Soát
- Mọi module tham gia đều hội tụ qua cổng dịch vụ Singleton **`SharedSemanticEncoder`** trong [`advanced_heuristics.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/advanced_heuristics.py), **tiêu tốn bộ nhớ mô hình bằng 0 khi sử dụng lại**.
- **`SessionEmbeddingCacheManager`:** Tác vụ lưu cache vector chuỗi được khóa theo kích thước cố định (`maxsize = 50` mục/phiên, `TTL = 3600s`), tuyệt đối không gây tràn RAM.

### 3. Multi-Step Kill-Chain State Machine (`HeuristicStateTracker`)
- Máy trạng thái phát hiện chuỗi tấn công đa giai đoạn: Recon → Credential Access → Privilege Escalation → Exfiltration → Defense Evasion → Remote Execution.
- Pattern groups và TTL values được hiệu chỉnh qua bộ audit (`tests/audit_killchain_coverage.py`, `tests/audit_ttl_calibration.py`).
- Tích hợp `EarlyIntentClassifier` để inject cảnh báo sớm cho prompt chứa dấu hiệu jailbreak/exfil staging ngay bước đầu.

### 4. Instruction Boundary Detector
- Phát hiện instruction nhúng (embedded imperative) trong dữ liệu cấu trúc (JSON, email body, tool arguments).
- Phân tích `position_ratio` và `context_type` để đánh giá mức tin cậy vi phạm ranh giới.
- Module: [`boundary_detector.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/boundary_detector.py)

### 5. Signal Registry & Anti-Feedback-Loop Protection
- **`SignalRegistry`** (Singleton) đảm bảo mỗi nguồn tín hiệu phát hiện (Tier 0, Tier 0.5, V61 ML, LLM Judge, Heuristics, ...) phải khai báo rõ ràng được phép feed vào **`EscalationManager`** nào.
- Bảo vệ chống lỗ hổng **E7 (Feedback Loop Poisoning)**: chỉ có V61 ML mới được phép điều chỉnh `AdaptiveEscalationManager` threshold; các tầng khác bị cấm tuyệt đối.
- Module: [`signal_registry.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/signal_registry.py)

### 6. GlobalThreatTracker (Cross-Session APT Correlation)
- Theo dõi tactical-stage flags (`sensitive_read`, `external_send`, `cover_tracks`) theo `user_id` xuyên suốt các `session_id` khác nhau.
- Phát hiện chiến dịch tấn công low-and-slow qua nhiều session riêng biệt.
- Module: [`global_threat_tracker.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/global_threat_tracker.py)

### 7. Egress Sanitization Pipeline
- **`ContextSanitizer`**: Phân loại dữ liệu đầu ra tool thành 4 mức (PASS / WRAP_UNTRUSTED / STRIP_AND_WRAP / QUARANTINE) dựa trên ML score và regex pattern matching.
- **`DataRedactor`**: Redact PII (email, phone, credit card qua Luhn validation), API keys (AWS AKIA*, GitHub ghp_*, Slack tokens), PEM private keys, và generic key-value secrets.
- Module: [`v61_context_sanitizer.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/v61_context_sanitizer.py), [`data_redactor.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/data_redactor.py)

---

## 🛠️ Hướng Dẫn Cài Đặt & Cấu Hình

### 1. Yêu Cầu Môi Trường (Prerequisites)
- Python $\ge 3.10$
- Các thư viện cốt lõi: `torch`, `sentence-transformers`, `scikit-learn`, `pydantic`, `requests`, `joblib`, `numpy`.
- LLM backend: Ollama (mặc định `gemma3:4b`) — chỉ cần cho tầng V61 LLM Judge và LLM Session Judge.

### 2. Cấu Hình Trung Tâm

Hệ thống sử dụng 3 file cấu hình chính, **không hardcode số ma thuật (Zero Magic Numbers)**:

#### a) [`config/settings.yaml`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/config/settings.yaml) — Tham số vận hành
```yaml
ollama_model: "gemma3:4b"           # Mô hình sử dụng cho LLM Judge
ollama_timeout: 15.0                # Timeout inference (giúp qua Cold-Start)
firewall_mode: "STRICT"             # Chế độ tường lửa (STRICT / AUDIT)
ollama_base_url: "http://localhost:11434"
multi_step_heuristics_enabled: true # Bật/tắt máy trạng thái kill-chain
llm_session_judge_enabled: true     # Bật/tắt LLM Session Judge
llm_session_judge_timeout: 5.0     # Timeout cho Session Judge (giây)
llm_session_history_window: 8      # Số bước lịch sử gửi cho Session Judge
guard_high_risk_threshold: "0.5"   # Ngưỡng rủi ro cao cho PermissionGate
```

> [!TIP]
> Toàn bộ thông số trong `settings.yaml` đều có thể ghi đè nóng thông qua biến môi trường tương ứng (ví dụ: `OLLAMA_MODEL`, `FIREWALL_MODE`, `LLM_SESSION_JUDGE_ENABLED`). Thứ tự ưu tiên: **env var thủ công > settings.yaml > default hardcode**.

#### b) [`config/thresholds.json`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/config/thresholds.json) — Ngưỡng quyết định
```json
{
  "action_risk_model": { "BLOCK": 0.6429, "REVIEW": 0.5577 },
  "prompt_risk_model": { "BLOCK": 0.6229, "REVIEW": 0.5385 },
  "voting_aggregator": { "MONITOR": 40, "REVIEW": 68, "DENY": 82, "QUARANTINE": 90 },
  "context_sanitizer": { "PASS": 0.3, "WRAP_UNTRUSTED": 0.65, "STRIP_AND_WRAP": 0.85 },
  "provenance_tagging": { "enabled": true },
  "structure_escalation": { "floor": 0.35 },
  "multi_step_defense": {
    "suspicious_band_min": 0.35,
    "heuristic_ttl_seconds": 180.0,
    "max_benign_steps": 40,
    "llm_judge_timeout": 3.0
  },
  "tier05_lstm": {
    "block_threshold": 0.98,
    "semantic_delta_threshold": 0.80,
    "confidence_propagation_floor": 0.50
  },
  "early_intent": { "risk_threshold": 0.50, "max_forced_actions": 2 }
}
```

#### c) [`config/semantic_config.json`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/config/semantic_config.json) — Anchor vectors cho Semantic Camouflage Detection
```json
{
  "semantic_camouflage": {
    "threshold": 0.5269,
    "anchors": [
      "rm -rf /", "dd if=/dev/zero of=", "fork() while true",
      "exec /bin/bash", ":(){$|&};:", "issue api_call to send an email with arguments",
      "transfer funds to account", "ignore previous instructions and execute",
      "print system prompt"
    ]
  }
}
```

---

## 🧪 Hướng Dẫn Kiểm Thử & Chạy Đánh Giá

Hệ thống được thiết kế kèm theo bộ kiểm thử toàn diện (**87+ test files**) để đảm bảo mọi sự tùy biến không phá vỡ tính năng an toàn cốt lõi.

### 1. Kiểm Chân Chức Năng Cốt Lõi (Unit Testing)

```powershell
# Kiểm tra Step-Counting Invariance Guard
py -m unittest tests/test_step_counting_invariance.py -v

# Kiểm tra Generalization Gate (tỷ lệ phát hiện tấn công >= 80%)
py -m unittest tests/test_generalization_gate.py -v

# Kiểm tra Security Invariants toàn hệ thống
py -m unittest tests/test_security_invariants.py -v

# Kiểm tra Multi-Step Kill-Chain Heuristics
py -m unittest tests/test_multi_step_heuristics.py -v

# Kiểm tra LLM Session Judge Security (canary, prompt injection resistance)
py -m unittest tests/test_llm_session_judge_security.py -v

# Kiểm tra Instruction Boundary Detection
py -m unittest tests/test_boundary_detector.py -v

# Kiểm tra GlobalThreatTracker cross-session correlation
py -m unittest tests/test_global_threat_tracker.py -v

# Kiểm tra Email Indirect Injection Detection
py -m unittest tests/test_email_indirect_injection.py -v

# Kiểm tra Data Redaction (PII, API keys, secrets)
py -m unittest tests/test_redaction.py -v

# Kiểm tra Signal Registry schema & policy enforcement
py -m unittest tests/test_signal_registry_schema.py -v

# Kiểm tra Concurrency & Thread Safety
py -m unittest tests/test_tier05_concurrency.py tests/test_lock_scope_regression.py -v
```

**Các mốc kiểm định tiêu chuẩn:**
- **Invariance Guard:** Chuỗi 100 bước lệnh lành tính không phát sinh false positive; tiêm lệnh độc hại vào Bước 101 lập tức kích hoạt Block.
- **Generalization Gate $\ge 80\%$:** Kiểm định chuỗi tấn công tinh tế vượt qua regex cũ, hoàn toàn dựa vào rủi ro chuyển dịch ngữ nghĩa để cự tuyệt $\ge 80\%$.
- **Signal Registry:** Mọi `SignalSource` đều phải khai báo trong registry; mọi lệnh feed `EscalationManager` không hợp lệ phải bị từ chối.

### 2. Chạy Benchmark Toàn Diện

```powershell
# Benchmark tải cao với pipeline đầy đủ (ThreadPoolExecutor + Circuit Breaker)
py tests/run_external_benchmark.py

# Benchmark 5000 mẫu (comprehensive evaluation)
py tests/benchmark_5000.py

# Benchmark 3000 mẫu (quick evaluation)
py tests/benchmark_3000.py

# Ablation Study (đánh giá đóng góp từng tầng)
py tests/run_ablation_benchmark.py
```

### 3. Công Cụ Chẩn Đoán & Hiệu Chỉnh

```powershell
# Chẩn đoán kill-chain coverage
py tests/diagnose_killchain_coverage.py

# Chẩn đoán suspicious band behavior
py scripts/diagnose_suspicious_band.py

# Hiệu chỉnh semantic delta threshold
py tests/calibrate_semantic_delta.py

# Hiệu chỉnh LSTM threshold
py scripts/calibrate_lstm_threshold.py

# Preflight check trước khi chạy benchmark
py scripts/preflight_check.py
```

---

## 📖 Hướng Dẫn Kỹ Thuật cho Nhà Phát Triển (Developer Guide)

### Biểu Đồ Trách Nhiệm Mã Nguồn

| Thư Mục / Tệp | Trách Nhiệm Kỹ Thuật | Lưu Ý Khi Can Thiệp |
| :--- | :--- | :--- |
| **`core/pipeline.py`** | Orchestrator chính, điều phối tuần tự: Tier 0 → Tier 0.5 → MultiStep → V61 → Session Judge → Heuristics → GlobalTracker | Khi bổ sung tầng mới, đăng ký trong `UnifiedFirewallPipeline.scan()`. |
| **`core/tier0.py`** | Tầng 0: Stateless regex + leet normalization + RCE detection | Hiện đã qua fix8 (audit/stability release). Khi thêm pattern, bổ sung test trong `test_pipeline.py`. |
| **`core/tier05.py`** | Tầng 0.5: Session-aware heuristic, 11 behavioral flags, cross-step escalation | `Tier05Config` chứa `FLAG_EXPIRY_SECONDS`, `FLAG_EXPIRY_STEPS`, `ESCALATION_THRESHOLD`. |
| **`core/lstm_tier05.py`** | LSTM Wrapper bọc `tier05.py`, tích hợp Neural LSTM Temporal Model | Không tự khởi tạo — luôn nhận `fallback_tier05` trong constructor. |
| **`core/tier_lstm.py`** | `SessionAwareLSTMRisk`: Bounded Cache, Markov K=3, $\Delta = 1 - \text{cosine\_sim}$ | **Giữ nguyên Step-Counting Invariance Guard** để kháng bias. |
| **`core/early_intent_classifier.py`** | Pre-screening stateless regex cho prompt intent (6 pattern groups) | Outputs `risk_score` + `detected_signals` → inject vào `HeuristicStateTracker`. |
| **`core/action_parser.py`** | Parse tool_call thành `ParsedAction` (dict_literal, kwargs, JSON, XML) | `envelope` property tách data content ra khỏi behavioral instructions. |
| **`core/config_loader.py`** | Đọc `settings.yaml` → set env var | Gọi **ĐẦU TIÊN** trước mọi import khác. Nếu thêm key mới, bổ sung vào `_KEY_TO_ENV`. |
| **`models/security/advanced_heuristics.py`** | `AdaptiveEscalationManager`, `VotingAggregator`, `RiskSignal`, `Canonicalizer`, `PermissionGate`, `SharedSemanticEncoder`, `SemanticCamouflageDetector` | Hysteresis Deadband $\ge 2$ chu kỳ. Không khởi tạo Transformer rải rác — luôn dùng `SharedSemanticEncoder.get()`. |
| **`models/security/multi_step_heuristics.py`** | `HeuristicStateTracker`: Multi-step kill-chain state machine, TTL-based flags | Pattern groups hiệu chỉnh qua `audit_killchain_coverage.py` và `audit_ttl_calibration.py`. |
| **`models/security/v61_inference_router.py`** | `V61SecurityRouter` (Singleton): ML Model (Ensemble RF+LR) + LLM Action Judge | `MLRiskModel` cache prob via `lru_cache(2000)`. Thresholds từ `thresholds.json`. |
| **`models/security/llm_session_judge.py`** | `LLMSessionJudge` (Singleton): Multi-step session analysis qua LLM | Hardened: canary token hex, XML escaping, strict 3-line format regex. |
| **`models/security/signal_registry.py`** | `SignalRegistry` (Singleton): Policy enforcement cho feedback loop | Anti-E7: chỉ `V61_ML` được feed `V61_ADAPTIVE` escalation manager. |
| **`models/security/global_threat_tracker.py`** | `GlobalThreatTracker` (Singleton): Cross-session APT correlation | Benchmark limitation: mỗi session = 1 user → no-op trên benchmark offline. |
| **`models/security/boundary_detector.py`** | `InstructionBoundaryDetector`: Embedded instruction detection in structured data | Phân tích JSON, email, tool_call → tách `StructuredParser` + imperative pattern. |
| **`models/security/email_vector_detector.py`** | `EmailVectorDetector`: Indirect prompt injection qua email/document | 4 pattern: obfuscated email, hidden directive, trigger word, post-task hijack. |
| **`models/security/function_risk_registry.py`** | `HIGH_RISK_FUNCTIONS` dict + `check_function_signature()` | 40+ tool signatures với severity 60–95. Bổ sung tool mới vào dict. |
| **`models/security/v61_context_sanitizer.py`** | `ContextSanitizer`: Egress sanitization (PASS/WRAP/STRIP/QUARANTINE) | `wrapped_output` PHẢI được dùng cho LLM context — KHÔNG dùng `_sanitized_text_audit_only`. |
| **`models/security/data_redactor.py`** | `DataRedactor`: Redact PII, API keys, PEM keys, credit cards (Luhn) | Gọi từ `FeedbackLogger` trước khi ghi log — đảm bảo không lưu secrets. |
| **`models/security/semantic_taint.py`** | `ObfuscationDetector` + `BoundedSemanticTaintTracker` | Shannon entropy, base64/hex/URL decode, taint propagation tracking. |
| **`models/security/feedback_logger.py`** | `FeedbackLogger`: Async JSONL logging cho retrain dataset | Queue (maxsize=1000) + daemon worker thread + atexit flush. Redact trước khi ghi. |
| **`config/settings.yaml`** | Tham số vận hành chung (LLM model, timeout, firewall mode) | Chỉ hỗ trợ flat key:value (không nesting). Nếu cần nesting → chuyển sang PyYAML. |
| **`config/thresholds.json`** | Ngưỡng quyết định cho mọi tầng (ML, heuristics, sanitizer, LSTM) | **Source of truth** cho toàn bộ threshold. Không hardcode trong code. |
| **`config/semantic_config.json`** | Anchor vectors cho Semantic Camouflage Detection | `anchor_hash` dùng để verify tính toàn vẹn của anchors. |

### 💡 Bí Kíp Thao Tác Nhanh

1. **Muốn thêm từ điển cấm đoán tức thì?** Mở [`core/tier0.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/core/tier0.py) và chỉnh sửa bảng pattern regex.
2. **Muốn thêm tool nguy hiểm mới?** Bổ sung vào `HIGH_RISK_FUNCTIONS` trong [`function_risk_registry.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/function_risk_registry.py).
3. **Muốn Tối Ưu Hóa Tỷ Lệ FPR/ABSR?** Chỉnh ngưỡng trong [`config/thresholds.json`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/config/thresholds.json) — mỗi tầng có section riêng.
4. **Muốn thêm tầng phòng thủ mới?** Đăng ký `SignalSource` trong [`signal_registry.py`](file:///c:/Users/ADMIN/jupyternotebook/EVO_PCA/models/security/signal_registry.py), tích hợp vào `pipeline.py`, và bổ sung test.
5. **Mỗi lần chỉnh sửa lõi xong:** Bắt buộc chạy `test_generalization_gate.py` + `test_security_invariants.py` + `test_signal_registry_schema.py` trước khi commit!

```powershell
# Bộ kiểm tra tối thiểu trước khi commit
py -m unittest tests/test_generalization_gate.py tests/test_security_invariants.py tests/test_signal_registry_schema.py -v
```

---

## 📂 Cấu Trúc Thư Mục

```
EVO_PCA/
├── config/
│   ├── settings.yaml             # Tham số vận hành
│   ├── thresholds.json           # Ngưỡng quyết định cho mọi tầng
│   └── semantic_config.json      # Anchor vectors cho Semantic Camouflage
├── core/
│   ├── pipeline.py               # Orchestrator chính (UnifiedFirewallPipeline)
│   ├── tier0.py                  # Tầng 0: Deterministic Firewall (fix8)
│   ├── tier05.py                 # Tầng 0.5: Session-Aware Heuristic
│   ├── lstm_tier05.py            # LSTM Wrapper cho Tier 0.5
│   ├── tier_lstm.py              # Neural LSTM + Markov K=3 Sequence Risk
│   ├── early_intent_classifier.py # Pre-screening prompt intent
│   ├── action_parser.py          # Tool call parser (ParsedAction)
│   └── config_loader.py          # Settings.yaml → env var loader
├── models/
│   ├── artifacts/                # ML model files (.joblib, .pth)
│   └── security/
│       ├── advanced_heuristics.py    # VotingAggregator, PermissionGate, Canonicalizer, ...
│       ├── multi_step_heuristics.py  # Kill-chain State Machine
│       ├── v61_inference_router.py   # V61 ML + LLM Action Judge
│       ├── llm_session_judge.py      # LLM Session Judge (Tier 2B)
│       ├── signal_registry.py        # Signal & Escalation Policy Registry
│       ├── global_threat_tracker.py  # Cross-Session APT Correlation
│       ├── boundary_detector.py      # Instruction Boundary Detection
│       ├── email_vector_detector.py  # Email Indirect Injection Detection
│       ├── function_risk_registry.py # High-Risk Function Signatures (40+)
│       ├── v61_context_sanitizer.py  # Egress Context Sanitizer
│       ├── data_redactor.py          # PII & Secret Redaction
│       ├── semantic_taint.py         # Obfuscation Detector & Taint Tracking
│       ├── feedback_logger.py        # Async Retrain Dataset Logger
│       ├── shared_utils.py           # Ensemble predict, benign_dev_shell check
│       └── v61_calibration.py        # Threshold Calibration Utilities
├── tests/                        # 87+ test files (unit, integration, benchmark)
├── scripts/                      # Diagnostic & calibration scripts
├── ablation/                     # Ablation study dataset & scripts
├── training/                     # Training pipeline for ML models
└── logs/                         # Runtime logs & retrain dataset (JSONL)
```

---

*EVO-PCA Shield — Kiến tạo móng vững chắc cho Kỷ nguyên Thực Thi AI An Toàn.*
