โอเค งั้นผมจะส่งเนื้อหาไฟล์ n8n-install-guide.md ให้เลย
คุณก็อปทั้งหมดไปวางในไฟล์ใหม่บนเครื่อง หรือกด “Create new file” ใน GitHub แล้วตั้งชื่อเป็น n8n-install-guide.md ได้เลย

markdown
Copy
Edit
# n8n Installation & Management Guide

## 1. การติดตั้ง n8n ด้วย Docker Compose
รองรับทั้ง `docker compose` (ใหม่) และ `docker-compose` (เก่า)  
ถ้าระบบใช้เวอร์ชันใหม่ แนะนำให้ใช้ `docker compose` แทน

### 1.1 สร้างโฟลเดอร์
```bash
mkdir ~/n8n-docker
cd ~/n8n-docker
1.2 สร้างไฟล์ docker-compose.yml
yaml
Copy
Edit
version: '3.9'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=false
      - N8N_TRUST_PROXY=true
      - N8N_RUNNERS_ENABLED=true
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - n8n_network

  runner:
    image: n8nio/n8n:latest
    container_name: n8n_runner
    restart: always
    environment:
      - N8N_RUNNERS_ENABLED=true
      - N8N_RUNNER_MODE=own
      - N8N_RUNNER_CONNECTION_MODE=server
      - N8N_RUNNER_SERVER_URL=http://n8n:5678
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
    depends_on:
      - n8n
    networks:
      - n8n_network

volumes:
  n8n_data:

networks:
  n8n_network:
    driver: bridge
1.3 เริ่มใช้งาน
bash
Copy
Edit
docker compose up -d
2. สคริปต์จัดการ manage-n8n.sh
สคริปต์นี้ทำได้หลายอย่าง เช่น

Full Reset + Backup

Restart + Backup

Restore ล่าสุด

Update + Backup

Full Maintenance (Backup + Update + Restart + Renew SSL)

2.1 รองรับทั้ง docker compose และ docker-compose
ในสคริปต์จะมีโค้ด:

bash
Copy
Edit
if command -v docker compose &> /dev/null; then
    DC="docker compose"
elif command -v docker-compose &> /dev/null; then
    DC="docker-compose"
else
    echo "❌ ไม่พบ docker compose หรือ docker-compose"
    exit 1
fi
หมายความว่า:

ถ้าระบบมี docker compose จะใช้ตัวนี้

ถ้าไม่มี จะ fallback ไปใช้ docker-compose

ถ้าไม่มีทั้งคู่ จะหยุดการทำงาน

3. ปัญหาที่เจอบ่อยและวิธีแก้
3.1 Error: unknown shorthand flag: 'f' in -f
สาเหตุ: ใช้คำสั่ง docker -f แทนที่จะใช้ docker compose -f หรือ docker-compose -f
วิธีแก้: ให้เปลี่ยนในสคริปต์ทั้งหมดมาใช้ $DC ตามโค้ด fallback ด้านบน

3.2 Runner เชื่อมต่อไม่ได้ (403)
สาเหตุ: ค่า N8N_RUNNERS_ENABLED หรือ URL ไม่ถูกต้อง
วิธีแก้: ตรวจสอบให้ตรงกันทั้งใน service n8n และ runner

4. คำสั่งที่ใช้บ่อย
bash
Copy
Edit
$DC up -d        # รันคอนเทนเนอร์
$DC down         # ปิดคอนเทนเนอร์
$DC restart      # รีสตาร์ทคอนเทนเนอร์
$DC logs -f      # ดู log แบบ real-time
5. การ Backup และ Restore แบบ Manual
5.1 Backup
bash
Copy
Edit
tar -czvf n8n-backup-$(date +%F-%H%M%S).tar.gz ~/.n8n
5.2 Restore
bash
Copy
Edit
tar -xzvf n8n-backup-ไฟล์.tar.gz -C ~/
Tip:
ถ้าใช้งานบนระบบใหม่ แนะนำใช้ docker compose (เวอร์ชันใหม่) เพื่อรองรับอนาคต
แต่ยังคง fallback ไปใช้ docker-compose ได้ในสคริปต์ เพื่อความเข้ากันได้กับระบบเก่า

yaml
Copy
Edit

---

ต้องการให้ผมสร้าง **ลิงก์ `wget` สำหรับโหลดไฟล์นี้ตรงจาก SSH** ให้เลยมั้ย  
แบบคุณแค่รัน 1 คำสั่งก็ได้ไฟล์นี้ลง VM ทันที
