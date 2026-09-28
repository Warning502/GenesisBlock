# CONCEPT — Multi-Machine, Multi-Provider Agent Swarm (GoVibe × GenesisBlockDB)

> **สถานะ:** Concept / discussion notes (ยังไม่ใช่ spec ที่ implement แล้ว)
> **ที่มา:** สรุปจากการคุยเรื่องใช้ GoVibe เป็น orchestrator ข้าม provider/ข้ามเครื่อง โดยมี GenesisBlockDB เป็น memory กลาง
> **ขอบเขต:** GoVibe เป็น *client* ของ GenesisBlockDB — ontology/policy ของ GoVibe อยู่ฝั่ง client ไม่ใส่ลง database core (ดู `README.md`, `docs/PRD--GENESISBLOCKDB-PLATFORM.md`)

---

## 1. GenesisBlockDB คืออะไร (ย่อ)

ฐานข้อมูล local-first เขียนด้วย Rust ที่รวมหลายอย่างไว้ใน engine เดียว:

- **Graph DB** — node / edge
- **Vector DB** — embedding ต่อ collection + HNSW (index แบบ async)
- **Relational projection** — `projection.sqlite` ภายใน (engine เป็นเจ้าของ ห้ามเขียนตรง)
- **Bitemporal** — ไม่เขียนทับ ใช้ `supersede_node` / `retract_edge` จึงย้อนเวลาได้
- **CRDT sync + ed25519 consensus** — รวมข้อมูลหลายเครื่อง/หลาย agent
- **Governance tier** — agent ภายนอกสร้าง node ระดับ `MASTER` เองไม่ได้
- **HQL** — `SEARCH`, `TRAVERSE`, `MATCH`, `MATCH … SIMILAR/HYBRID`, `CONTEXT`

เรียกใช้ได้ทาง Node addon (NAPI), REST (`/v1/*` บน `:3000`), C FFI (iOS/Android), และ MCP server (`query_hql`, `retrieve_tiered_context`, `add_knowledge`)

---

## 2. Multi-Provider: "แชร์ token" จริงๆ คือแชร์งาน

- โควต้าของ Claude / Codex / Gemini / Devin **โอนข้ามกันไม่ได้** (คนละบัญชี คนละ provider)
- สิ่งที่ทำได้คือ **route งานไปยัง provider ที่ยังเหลือโควต้า** → ในทางปฏิบัติเหมือนรวมโควต้าเป็นก้อนเดียว = *agent workforce*

การแบ่งบทบาทที่แนะนำ:

| บทบาท | ตัวอย่าง executor | เหตุผล |
|---|---|---|
| Mission control — วางแผน, สร้าง Execution DAG, ตรวจงาน | Claude | ใช้ token น้อย แต่ต้องใช้เหตุผลมาก |
| ลงมือเขียนโค้ด / งานกิน token | Codex, Devin | มีโควต้าเหลือเยอะ |
| งานที่ตรวจด้วยเครื่องได้ / งานถูก | Gemini, Antigravity, Ollama (local) | ลดต้นทุน |

### สิ่งที่ต้องแชร์จริงคือ **context**
แต่ละ session จำกันเองไม่ได้ → context ต้องอยู่นอก session ในที่กลาง = **GenesisBlockDB**

- `retrieve_tiered_context` → ส่ง context แบบ hop-scoped ให้แต่ละ agent (ไม่ต้องแปะทั้งไฟล์)
- `add_knowledge` + schema `govibe-knowledge-candidate/v1` → ทุก agent เขียนความรู้เข้าทางเดียวกัน
- Governance tier → กัน agent เขียนทับความรู้ระดับ `MASTER`
- Signed WAL + bitemporal → audit ได้ว่า agent ไหนเขียนอะไร เมื่อไร

---

## 3. Multi-Machine: Claude ตัวแม่คุมหลาย PC

### 3.1 ช่องทางสั่งงาน

| ทางเลือก | รองรับ | หมายเหตุ |
|---|---|---|
| **Claude Code Remote Control** — รัน `claude remote-control` ในโฟลเดอร์งานของแต่ละ PC (หรือ Claude Desktop) | Claude เท่านั้น | ใช้ได้ทันที session ของแต่ละเครื่องขึ้นในแอป Claude Code และตัวแม่ส่งข้อความเข้าได้ |
| **GoVibe Station daemon** บนแต่ละเครื่อง | ทุก provider | ต้องเขียน adapter เอง |

### 3.2 ความจำกลาง — สิ่งที่มีแล้วใน GenesisBlockDB

- Swarm identity (ed25519 keypair) ต่อฐานข้อมูล
- CRDT sync (Lamport clock, LWW reconcile, Merkle root) ใน core
- REST: `/v1/swarm/status`, `/v1/consensus/propose`, `/v1/consensus/vote`, `/v1/consensus/sign-vote`, `/v1/consensus/verify` (`src/router.rs`)

### 3.3 Gap ที่ยังขาด

- **ยังไม่มี REST route สำหรับแลก sync event ระหว่างเครื่อง** (CRDT sync อยู่ใน core แต่ยังไม่เปิดผ่าน REST)

ทางเลือก:
1. **ง่ายสุด (hub):** `genesis-db-server` ตัวเดียวเป็นศูนย์กลาง ทุกเครื่องอ่าน/เขียนผ่าน REST
2. **กระจายจริง (peer):** แต่ละเครื่องมี DB ของตัวเอง → ต้องเพิ่ม `/v1/sync/push`, `/v1/sync/pull` (ต้อง wire ทั้ง NAPI + REST ให้ตรงกัน)

---

## 4. หลาย executor ในเครื่องเดียว — ลำดับชั้น 3 ระดับ

```
Claude ตัวแม่ (Mission Control)  ← วาง DAG, เลือก executor, ตรวจงาน
   │
   ├─ PC-A (Station daemon)
   │    ├─ antigravity
   │    ├─ codex
   │    └─ claude
   │
   └─ PC-B (Station daemon)
        ├─ claude
        └─ devin cli
```

- ตัวแม่ **คุยกับ Station** (เครื่องละตัว) ไม่คุยกับ CLI ตรง → ตัวแม่ไม่ต้องรู้วิธีเรียกแต่ละ CLI
- Station: รายงาน executor ที่มี + สถานะ, รับงานแล้วเรียก CLI, ส่ง heartbeat + ผลงานกลับเข้า GenesisBlockDB

### 4.1 ขยาย `.agents/agent-registry.yaml` (ข้อเสนอ — ยังไม่มีในโค้ด)

ปัจจุบัน registry มี `executors` (`codex` → `codex_exec`, `ollama` → `ollama_cli`) พร้อม `allowed_modes` แต่ยังไม่มีชั้น "เครื่อง"

```yaml
stations:
  pc-a:
    endpoint: tailscale://pc-a:7400
    executors: [antigravity, codex, claude]
  pc-b:
    endpoint: tailscale://pc-b:7400
    executors: [claude, devin]

executors:
  codex:       { transport: codex_exec,  allowed_modes: [plan, atomic, audit], quota_source: openai }
  claude:      { transport: claude_cli,  allowed_modes: [plan, audit, atomic], quota_source: anthropic }
  antigravity: { transport: antigravity, allowed_modes: [atomic, doc],         quota_source: google }
  devin:       { transport: devin_cli,   allowed_modes: [atomic],              quota_source: cognition }
```

### 4.2 กติกาการเลือก executor (ตามลำดับ)

1. **Capability** — executor ทำ mode นี้ได้ไหม (`allowed_modes`)
2. **Quota** — นับตาม `quota_source` **ไม่ใช่ตามเครื่อง** (claude บน PC-A กับ PC-B บัญชีเดียวกัน = โควต้าก้อนเดียวกัน)
3. **Locality** — เลือกเครื่องที่มี repo/worktree อยู่แล้ว
4. **Availability** — ดู heartbeat; เกิน lease → reassign

### 4.3 Data model ใน GenesisBlockDB (graph)

```
(Station)-[:HOSTS]->(Executor)-[:BILLED_TO]->(QuotaSource)
(Task)-[:DEPENDS_ON]->(Task)                              ← Execution DAG
(Task)-[:ASSIGNED_TO {valid_from, valid_to}]->(Executor)
(Task)-[:PRODUCED]->(Artifact/Knowledge)
```

- การ assign เป็น edge แบบ bitemporal → ย้ายงานให้ `retract_edge` ของเดิมแล้วสร้างใหม่ ประวัติอยู่ครบ
- schema นี้เป็นของ GoVibe (client-owned) ใช้ namespace ของ client ไม่ใส่ลง core

---

## 5. ข้อควรระวัง

| เรื่อง | แนวทาง |
|---|---|
| ตัวแม่กินโควต้า Claude | เครื่องลูกส่งกลับแค่ **สรุป + หลักฐาน** (แบบ evidence fields ใน `GEMINI.md`) รายละเอียดเก็บใน DB |
| เงื่อนไขการใช้งาน provider | subscription ส่วนตัวอาจจำกัดการสั่งอัตโนมัติ — API key ปลอดภัยกว่า และห้ามแชร์บัญชีข้ามคน |
| คุณภาพงานข้าม model | ต้องผ่านเกณฑ์ที่เครื่องตรวจได้ (test/build ผ่าน) ก่อนรวมเข้า DAG |
| ความปลอดภัย | สั่งงานข้ามเครื่อง = remote code execution → ใช้ VPN/Tailscale, **ห้ามเปิด `:3000` สู่อินเทอร์เน็ต**, คำสั่งลงลายเซ็นด้วย swarm key |
| ไฟล์ชนกันในเครื่องเดียว | Station สร้าง `git worktree` แยกต่อ task |
| auth ของแต่ละ CLI ต่างกัน | Station health-check ว่ายังล็อกอิน/มีโควต้า แล้วรายงาน |
| รูปแบบผลงานไม่เหมือนกัน | บังคับ output contract เดียว เช่น `govibe-knowledge-candidate/v1` |
| เครื่องลูกหลุด | lease + heartbeat → timeout แล้ว reassign |

---

## 6. Next steps ที่เสนอ

1. ขยาย `.agents/agent-registry.yaml` ให้มี `stations` + `quota_source` และอัปเดต `.agents/validate-agent-registry.mjs`
2. เพิ่ม `/v1/sync/push` / `/v1/sync/pull` ใน GenesisBlockDB (NAPI + REST parity, มี test ใน `tests/`)
3. GoVibe: เขียน Station daemon + adapter ต่อ CLI (codex / claude / antigravity / devin)
4. GoVibe: scheduler ตามกติกาข้อ 4.2 + lease/heartbeat
