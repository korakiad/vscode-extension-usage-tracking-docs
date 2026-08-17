# VS Code Extension Usage Tracking Docs

เอกสารสำหรับวางแผนและส่งต่องาน Implement usage tracking ของ VS Code extension ภายในองค์กร ซึ่งมี MCP tools, chat participants/skills, Language Model tools และ commands/scripts

Repository นี้มีเฉพาะเอกสาร Markdown ไม่มี source code หรือ history จาก extension project

## เริ่มจากตรงไหน

1. อ่าน [execution handoff](docs/handoff/company-vscode-extension-usage-tracking.md) บน company laptop
2. ตรวจ API และข้อจำกัดจาก [VS Code API research](docs/research/vscode-usage-tracking-api.md)
3. ทำ Phase 1 inventory ใน private company repository ก่อนออกแบบหรือแก้ implementation

เอกสารตั้งใจให้ใช้เป็นจุดเริ่มต้น ไม่ใช่คำยืนยันว่า public API รุ่นล่าสุดตรงกับ VS Code version ที่บริษัทใช้อยู่ ทุก API ต้องตรวจซ้ำกับ `@types/vscode`, MCP SDK และ policy ของเครื่องบริษัท
