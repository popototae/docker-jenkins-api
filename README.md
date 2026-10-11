# docker-jenkins-api

Express API และ MySQL สำหรับ attractions แยก repository และ pipeline จาก monorepo เดิม

## API tests ผ่าน CLI (Bruno)

อ่าน [API-TESTING.md](API-TESTING.md) สำหรับวิธีรัน เปิด collection ใน Bruno และอ่าน tests

```powershell
npm ci --include=dev
npm run test:api
```

เริ่มจาก API บนเครื่องที่ `http://127.0.0.1:3001` ไม่ต้อง deploy หรือใช้ VPS เพื่อฝึก Bruno
ทดสอบ 3 requests (`/health`, `/attractions`, route ที่ไม่มีอยู่) รวม 12 tests ผ่าน HTTP ไปฐานข้อมูลจริง
รายงาน HTML/JSON/JUnit อยู่ใน `api-test-results/` และไม่ถูก commit

## Jenkins flow

```text
Checkout → Prepare Environment → Validate → Unit Test → Build → Deploy MySQL/API → Health Check → API Test (Bruno) → Verify Deployment
```

อ่าน [Jenkinsfile](Jenkinsfile) ตาม stage ได้โดยตรง ไม่เรียกไฟล์ .sh ใช้ NodeJS tool ชื่อ `Node22` และ polling repo นี้ประมาณทุก 2 นาที

Bruno รัน 12 tests กับ API ที่เพิ่ง deploy ผ่าน `127.0.0.1:3001` บนเครื่อง Jenkins และอ่าน MySQL จริง
เมื่อ fail build จะไม่ผ่าน รายงานอยู่ใน Test Result และ Artifacts ของ build แม้ tests ไม่ผ่าน
Jenkins ต้องมี JUnit plugin เพื่อแสดงผล `api-test-results/results.xml`

## เริ่มใช้งาน

หากย้ายจาก VPS เดิม ให้ทำตาม [MIGRATION.md](MIGRATION.md) ก่อน เพื่อใช้ฐานข้อมูลเดิมและไม่ชนพอร์ต frontend เก่า

API job ใช้ Jenkins Secret text `MYSQL_ROOT_PASSWORD` และ `MYSQL_PASSWORD` สร้าง `.env` จากนั้น build และรอ MySQL/API healthy ใช้ Compose project `docker-jenkins-pipeline` เพื่อเก็บ volume เดิม

สำหรับ Docker บนเครื่องใหม่ ให้คัดลอก `.env.example` เป็น `.env` ใส่ค่าฐานข้อมูล แล้วรัน `docker compose up -d --build --wait`

Unit tests: `npm ci --include=dev` แล้ว `npm test` (mock database ไม่ต้องเปิด MySQL) หากรัน `npm start` นอก Docker ให้ตั้ง `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` และ `DB_PORT` ใน environment หรือ `.env.local`

## GitHub Actions

มี manual workflow ใน `.github/workflows/deploy.yml` สำหรับเปรียบเทียบ ดูขั้นตอนตั้ง SSH secrets และ checkout ใน [MIGRATION.md](MIGRATION.md)
