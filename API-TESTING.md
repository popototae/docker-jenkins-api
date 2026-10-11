# ทดสอบ API ด้วย Bruno

## เป้าหมายการฝึก

ลองส่ง request ใน Bruno อ่าน response เขียน tests แล้วใช้ CLI รัน collection เดียวกัน
เริ่มบนเครื่องตัวเองได้ ไม่ต้อง deploy ขึ้น VPS และ Jenkins ใช้ collection เดียวกันหลัง deploy

## เริ่มรันกับ API บนเครื่อง

เปิด API และ MySQL บนเครื่องก่อน จากนั้นเปิด terminal ในโฟลเดอร์ `01_api`:

```powershell
npm ci --include=dev
npm run test:api
```

CLI ถูกติดตั้งเป็น dev dependency ของโปรเจกต์ ไม่ต้องติดตั้งแบบ global
และไม่ต้องเปิดโปรแกรม Bruno Desktop เพื่อใช้คำสั่งนี้

flow:

```text
npm run test:api
  → สร้างโฟลเดอร์ api-test-results หากยังไม่มี
  → เข้าโฟลเดอร์ bruno
  → bru run --env Local
  → อ่านไฟล์ requests .bru และ environment
  → ส่ง HTTP ไป API ที่ http://127.0.0.1:3001
  → API อ่าน MySQL จริง
  → รัน tests และสร้างรายงาน
```

CLI ไม่เปิด API หรือ MySQL ให้ ต้องเปิดระบบก่อนรัน ทั้งชุดมีเฉพาะ GET จึงไม่แก้ไขข้อมูล
แต่ละ request ตั้ง timeout 5 วินาทีไว้ในส่วน `settings` ของไฟล์ `.bru`

## ไฟล์ไหนทำอะไร

| ไฟล์ | หน้าที่ |
|---|---|
| `bruno/bruno.json` | ประกาศว่าโฟลเดอร์นี้เป็น collection ชื่อ Attractions API |
| `bruno/01-health.bru` | request `/health` และ tests ตรวจ API/ฐานข้อมูล |
| `bruno/02-attractions.bru` | request `/attractions` และ tests ตรวจข้อมูล |
| `bruno/03-not-found.bru` | request route ที่ไม่มีอยู่ และ tests ตรวจ 404 |
| `bruno/environments/Local.bru` | `baseUrl` เป็น `http://127.0.0.1:3001` |
| `bruno/environments/VPS.bru` | `baseUrl` เป็น `http://138.2.70.183:3001` |
| `package.json` | คำสั่ง npm ที่เรียก Bruno CLI โดยตรง |

แต่ละ request แยกไฟล์ จึงเปิดอ่านหรือแก้ได้ง่าย โดยใช้ไฟล์เดียวกันกับ GUI และ CLI

## ตรวจอะไรบ้าง

มี 3 requests รวม 12 tests:

ในสรุปของ CLI ช่อง `Tests` จะแสดง 12/12 ส่วน `Assertions` เป็น 0/0 เพราะเราเขียนในบล็อก `tests {}`
ซึ่งใช้ `test()` และ `expect()` ไม่ได้ใช้บล็อก `assert {}` ของ Bruno

| Request | Tests |
|---|---|
| `GET /health` | HTTP 200, JSON, `status` เป็น `ok`, `db` เป็น `true` |
| `GET /attractions` | HTTP 200, JSON, array, ฟิลด์/ชนิดข้อมูลครบ, IDs ไม่ซ้ำ, พิกัดเป็นตัวเลขและอยู่ในช่วงที่ถูกต้อง |
| `GET /does-not-exist` | HTTP 404 และข้อความแจ้ง route ที่ไม่มีอยู่ตามพฤติกรรม Express ปัจจุบัน |

แต่ละสถานที่ต้องมี `id`, `name`, `detail`, `coverimage`, `latitude`, `longitude`
latitude อยู่ระหว่าง -90 ถึง 90 และ longitude ระหว่าง -180 ถึง 180
MySQL DECIMAL อาจส่งพิกัดเป็น string จึงยอมรับทั้ง number และ string ตัวเลข

ไม่กำหนดจำนวนแถวหรือลำดับตายตัว array ว่างถือว่าถูกต้อง แต่เมื่อว่างจะไม่มีแถวให้ตรวจฟิลด์หรือพิกัด
ชุดนี้ไม่ได้ตรวจว่า URL รูปภาพภายนอกดาวน์โหลดได้

## อ่าน test อย่างไร

เปิด `bruno/01-health.bru` จะเห็นส่วน `meta`, `get`, `headers`, `tests` และ `docs`
`get` บอก URL ที่จะยิง ส่วน `tests` คือ JavaScript สำหรับตรวจ response เช่น:

```js
test("API status is ok", function () {
  expect(res.getBody().status).to.equal("ok");
});
```

- `test()` ตั้งชื่อเคสที่แสดงในผลลัพธ์
- `res.getStatus()` อ่าน HTTP status code
- `res.getHeader()` อ่าน response header
- `res.getBody()` อ่าน response body ที่ parse แล้ว
- `expect()` ตรวจว่าผลตรงกับสัญญาของ API

ถ้าเปลี่ยน API เป็น `status: "okkkkk"` เคสนี้จะ fail เช่นเดียวกับ unit test เดิม
เราเปลี่ยนค่าที่คาดหวังเมื่อเปลี่ยนสัญญาของ API จริง ๆ

## เปิด collection ใน Bruno Desktop

1. เปิด Bruno แล้วเลือก Open Collection
2. เลือกโฟลเดอร์ `01_api/bruno` ที่มี `bruno.json`
3. เลือก environment `Local`
4. เปิด request แล้วกด Send เพื่อทดสอบทีละรายการ
5. ดูแท็บ Tests เพื่ออ่าน/แก้ assertions หรือใช้ Collection Runner เพื่อรันทั้งชุด

โปรแกรมบันทึกการแก้ไขกลับเป็นไฟล์ใน collection โดยตรง ไม่ต้อง export JSON เพื่อใช้ CLI
ต้องการติดตั้ง GUI ดู [Bruno Downloads](https://www.usebruno.com/downloads)

## รันกับ API บนเครื่อง

เปิด API ที่พอร์ต 3001 และ MySQL ก่อน:

- Docker: เตรียม `.env` ตาม `.env.example` แล้ว `docker compose up -d --build --wait`
- Node โดยตรง: ตั้งค่าฐานข้อมูลใน `.env.local` แล้วเปิดอีก terminal รัน `npm start`

จากนั้น:

```powershell
npm run test:api
```

หากไม่มี API จะ fail เพราะชุดนี้ส่ง HTTP จริง ต่างจาก unit tests ที่ mock ฐานข้อมูล

## เปลี่ยน URL หรือเลือก request เดียว

ใน PowerShell ใช้ `npm.cmd` เมื่อส่ง options เพิ่ม เพื่อส่ง flags ต่อให้ CLI ครบ:

```powershell
npm.cmd run test:api -- --env-var "baseUrl=http://127.0.0.1:3001"
```

ใช้ URL ของ API โดยไม่เติม `/` ท้าย URL บน Linux/macOS ใช้ `npm run ... -- ...` ตามปกติ

รันเฉพาะ health โดยใช้ CLI โดยตรงจากโฟลเดอร์ collection:

```powershell
cd bruno
..\node_modules\.bin\bru.cmd run 01-health.bru --env Local
cd ..
```

## รายงานและ exit code

คำสั่ง npm สร้างรายงาน 3 รูปแบบใน `api-test-results/`:

- `results.html` เปิดใน browser เพื่ออ่านผล
- `results.json` เก็บรายละเอียด requests/responses และผล tests
- `results.xml` เป็น JUnit สำหรับระบบ CI

โฟลเดอร์นี้ไม่ถูก commit หรือส่งเข้า Docker build และรอบถัดไปเขียนรายงานเดิมทับ
เมื่อผ่านควรเห็น requests 3/3 และ tests 12/12 ผ่านทั้งหมด
ตรวจ exit code หลังรันได้ด้วย `$LASTEXITCODE`: ผ่านเป็น 0 และ fail เป็นค่าที่ไม่ใช่ 0

## ลองให้ test fail โดยไม่แก้ API

ชี้ไป path ผิด:

```powershell
npm.cmd run test:api -- --env-var "baseUrl=http://127.0.0.1:3001/does-not-exist"
$LASTEXITCODE
```

คำสั่งนี้ตั้งใจให้ fail เพราะ `/health` และ `/attractions` จะกลายเป็น URL ที่ไม่มีอยู่
ไม่ได้หยุด service หรือแก้ข้อมูล
รายงานจะเป็นผล fail ของรอบนี้ รัน `npm run test:api` อีกครั้งเพื่อคืนรายงานเป็นผลของ URL ปกติ

## ต่างจาก tests อื่นอย่างไร

| คำสั่ง | สิ่งที่ตรวจ |
|---|---|
| API `npm test` | Express ผ่าน Supertest และ mock `pool.query`; ไม่ต้องเปิด MySQL |
| API `npm run test:api` | Bruno CLI → HTTP → API บนเครื่อง → MySQL จริง; ไม่มีเบราว์เซอร์ |
| Frontend `npm run test:browser` | UI ใน Chromium จริง โดยจำลอง API |
| Frontend `npm run test:browser:live` | Chromium → Next.js → API → MySQL จริง |

เคสฐานข้อมูลเสียยังตรวจด้วย unit tests เดิม ไม่ต้องหยุด MySQL ที่ใช้งานอยู่

## ถ้าต้องการตรวจระบบที่ deploy แล้วในภายหลัง

มี environment `VPS` และคำสั่ง `npm run test:api:vps` ไว้เป็นตัวเลือกสำหรับระบบที่เปิดอยู่เดิม
การฝึก Bruno และ CLI ตามขั้นตอนข้างบนใช้แค่ `Local` ได้ทั้งหมด

## Pipeline

Jenkinsfile เพิ่ม stage `API Test (Bruno)` หลัง Deploy และ Health Check:

```text
Unit Test → Build → Deploy → Health Check → API Test (Bruno) → Verify Deployment
```

stage สร้างโฟลเดอร์รายงาน เข้า `bruno` แล้วเรียก `bru run --env Local` พร้อม reporter 3 รูปแบบโดยตรง
`Local` ใน Jenkins หมายถึง `127.0.0.1:3001` ของเครื่อง Jenkins ที่ deploy API อยู่ ไม่ใช่คอมพิวเตอร์ของผู้พัฒนา
Jenkins รับ JUnit report `api-test-results/results.xml` เพื่อแสดง Test Result และเก็บรายงานทั้งหมดเป็น Artifacts
ส่วน `post { always { ... } }` ทำให้ยังเก็บผลได้แม้ test fail; คำสั่ง CLI ที่ exit ไม่ใช่ 0 ทำให้ build fail
ต้องมี JUnit plugin และ credentials ของ MySQL เดิม ไม่ต้องเพิ่ม SSH credentials สำหรับ tests
CLI อยู่ใน dev dependencies ที่ Jenkins ติดตั้งอยู่แล้วใน Unit Test stage
Dockerfile ใช้ `npm ci --omit=dev` จึงไม่ติดตั้งเครื่องมือทดสอบนี้ใน production container
GitHub Actions ยังเป็น flow เดิม การเพิ่ม tests รอบนี้ทำเฉพาะ Jenkins

เอกสารทางการ:
[Bruno CLI](https://docs.usebruno.com/bru-cli/overview),
[CLI options](https://docs.usebruno.com/bru-cli/run/options),
[Reports](https://docs.usebruno.com/bru-cli/run/reports),
[JavaScript API](https://docs.usebruno.com/testing/script/javascript-reference)
