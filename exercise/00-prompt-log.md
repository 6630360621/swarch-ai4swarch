**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:**
**agent ที่ใช้:** เช่น GitHub Copilot (Agent mode), Gemini Code Assist, Gemini CLI, Claude Code, Codex, Cursor, Cline: …
**โมเดล:** …
**แพ็กเกจที่ใช้:** เช่น Copilot Student, Copilot Free, Gemini รุ่นฟรี: …

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | vscode copilot model auto | `prompts/1-…` run A | 1-recover-architecture.md and demo.proto | true | 5 | |
| 2 | 1-B | vscode copilot model auto | `prompts/1-…` run B | AGENTS.md, 1-recover-architecture.md, demo.proto, README.md, adservice.yaml, cartservice.yaml, checkoutservice.yaml, currencyservice.yaml, emailservice.yaml, frontend.yaml, loadgenerator.yaml, paymentservice.yaml, productcatalogservice.yaml, recommendationservice.yaml, shippingservice.yaml, and kustomization.yaml. | true | 5 | |
| 3 | 2-a | vscode copilot model auto | `prompts/2-…` run a | AGENTS.md, 2-write-adr.md และ template.md. | true | 5 | |
| 4 | 2-b | vscode copilot model auto | `prompts/2-…` run b | AGENTS.md, 2-write-adr.md, scenario.md, 0001-use-redis-for-cart-storage.md, template.md, main.go และ 2b-adr.md. | true | 5 | |
| 5 | 3 | vscode copilot model auto | `prompts/3-…` | AGENTS.md, 3-review-adr.md, 0002-2b-adr.md และ scenario.md | true | 5 | |
| 6 | | | | | | | |
| 7 | | | | | | | |
