# n8n Installation & Management Guide
>(พร้อมระบบ Backup, Restore, Update, Maintenance และ Runner)


## 1. การติดตั้ง n8n ด้วย Docker Compose
ปัญหา docker compose vs docker-compose
- Docker รุ่นใหม่ (20.10 ขึ้นไป) ใช้ docker compose (มีช่องว่าง)
- Docker รุ่นเก่า ใช้ docker-compose (มีขีดกลาง) ซึ่งเป็น binary แยก
- สคริปต์นี้ตรวจอัตโนมัติ → ถ้ามี docker compose จะใช้ก่อน ถ้าไม่มีจะ fallback ไป docker-compose
รองรับทั้ง `docker compose` (ใหม่) และ `docker-compose` (เก่า)  
>แนะนำให้ติดตั้ง ระบบใช้เวอร์ชันใหม่ แนะนำให้ใช้ `docker compose` แทน
ต้องติดตั้ง:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-plugin curl nano tar
```
>หมายเหตุ: docker compose (แบบมีช่องว่าง) เป็นเวอร์ชันใหม่ที่มากับ Docker plugin แล้ว ถ้าใช้ระบบเก่าอาจต้องใช้ docker-compose (มีขีดกลาง) แทน

### 1.1 เพิ่มสิทธิ์ให้ user ปัจจุบันใช้ Docker ได้ (ไม่ต้องพิมพ์ sudo ทุกครั้ง)
```bash
sudo usermod -aG docker $USER
```
>แล้ว ออกจาก SSH และเข้าใหม่ เพื่อให้สิทธิ์มีผล

### 1.2 สร้างโฟลเดอร์หลัก คำสั่ง:
```bash
mkdir -p ~/n8n-docker
mkdir -p ~/n8n-backups
cd ~/n8n-docker
```
>หมายเหตุ: บางระบบ Docker เวอร์ชันใหม่ใช้คำสั่ง docker compose แทน docker-compose


## 2. โครงสร้างโฟลเดอร์ ไฟล์

### 2.1 โครงสร้าง
```bash
~/n8n-docker/
  ├── manage-n8n.sh      # สคริปต์จัดการ n8n
  ├── docker-compose.yml # จะถูกสร้างใหม่อัตโนมัติเมื่อ Full Reset
  └── n8n-backups/       # เก็บไฟล์สำรอง
```


## 3. ไฟล์ `docker-compose.yml` (รองรับ Runner)
```yaml
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
```


## 4. สคริปต์ `manage-n8n.sh` (เวอร์ชันเสถียร)
รองรับ:
- Full Reset + Backup
- Restart + Backup
- Restore
- Update
- Full Maintenance
- เลือกใช้ docker compose หรือ fallback เป็น docker-compose อัตโนมัติ
- มี timestamp log
- หยุด script ถ้า error (set -euo pipefail)

### 4.1 สคริปต์ `manage-n8n.sh` (เวอร์ชันเสถียร)
```bash
#!/bin/bash
set -euo pipefail

timestamp() { date +"[%Y-%m-%d %H:%M:%S]"; }

# ตรวจหา docker compose
if command -v docker compose &> /dev/null; then
  DC="docker compose"
elif command -v docker-compose &> /dev/null; then
  DC="docker-compose"
else
  echo "$(timestamp) ❌ ไม่พบ docker compose หรือ docker-compose"
  exit 1
fi

BACKUP_DIR="$HOME/n8n-backups"
COMPOSE_FILE="$HOME/n8n-docker/docker-compose.yml"
mkdir -p "$BACKUP_DIR"

echo "========= Manage n8n ========="
echo "1. Full Reset + Backup"
echo "2. Restart + Backup"
echo "3. Restore ล่าสุด"
echo "4. Update + Backup"
echo "5. Full Maintenance"
read -rp "เลือกหมายเลข: " choice

backup_n8n() {
  echo "$(timestamp) 📦 กำลัง Backup..."
  tar -czf "$BACKUP_DIR/n8n-backup-$(date +%Y%m%d-%H%M%S).tar.gz" -C "$HOME" n8n-docker
  echo "$(timestamp) ✅ Backup เสร็จ: $(ls -t $BACKUP_DIR | head -n 1)"
}

case $choice in
  1)
    read -rp "พิมพ์ YES เพื่อยืนยันการล้างระบบ: " confirm
    [[ "$confirm" != "YES" ]] && { echo "$(timestamp) ❌ ยกเลิก"; exit 1; }
    backup_n8n
    echo "$(timestamp) 🛑 Stopping & Removing..."
    $DC down -v || true
    docker system prune -af --volumes || true
    echo "$(timestamp) ⬇️ Pull latest image..."
    docker pull n8nio/n8n:latest
    echo "$(timestamp) 📝 Creating new docker-compose.yml..."
    cat << 'YAML' > "$COMPOSE_FILE"
(ใส่ docker-compose.yml จากข้อ 3)
YAML
    echo "$(timestamp) ⬆️ Starting n8n..."
    $DC up -d
    ;;
  2)
    backup_n8n
    $DC restart
    ;;
  3)
    latest_backup=$(ls -t "$BACKUP_DIR"/*.tar.gz | head -n 1)
    [ -z "$latest_backup" ] && { echo "$(timestamp) ❌ ไม่มี backup"; exit 1; }
    echo "$(timestamp) 🔄 Restoring from $latest_backup..."
    tar -xzf "$latest_backup" -C "$HOME"
    $DC up -d
    ;;
  4)
    backup_n8n
    docker pull n8nio/n8n:latest
    $DC up -d
    ;;
  5)
    backup_n8n
    docker pull n8nio/n8n:latest
    $DC up -d
    docker exec n8n certbot renew || true
    docker restart nginx || true
    $DC logs --tail=50
    ;;
  *)
    echo "$(timestamp) ❌ Invalid choice"
    ;;
esac
```

### 4.2 ตั้ง Cron ให้ Full Maintenance อัตโนมัติ
```bash
crontab -e
```
เลือก nano (ตัวเลือก 1)
เพิ่มบรรทัดนี้:
```cron
0 3 * * 0 /bin/bash /home/username/n8n-docker/manage-n8n.sh 5 >/dev/null 2>&1
```
อธิบาย 
- `0 3 * * 0` = รันทุกวันอาทิตย์เวลา 03:00 น.
- /bin/bash /home/thongtrachu_t/n8n-docker/manage-n8n.sh <<< '5' = รันเมนูตัวเลือก 5 (Full Maintenance) อัตโนมัติ
- /dev/null 2>&1 = ซ่อน output ไม่ให้ส่งเมลแจ้งเตือน
หลังบันทึก กด `CTRL + O` → `Enter` → `CTRL + X` เพื่อออกจาก nano


## 5. หมายเหตุการใช้งาน
- ถ้าเลือก ตัวเลือก 1 (Full Reset) ข้อมูล Workflow จะถูกลบ
- ถ้าใช้ Runner แล้วให้เปิด `N8N_RUNNERS_ENABLED=true` ใน `docker-compose.yml`
- ระบบใช้ `$DC` เพื่อตรวจสอบว่ามี `docker compose` หรือ `docker-compose` ก่อนใช้

## 6. ปัญหาที่พบและวิธีแก้
  1. unknown shorthand flag: '-f'
     - เกิดจากใช้ 'docker -f' แทน 'docker compose -f' หรือ 'docker-compose -f'
     - แก้โดยเช็ก '$DC' และใช้ '$DC -f' แทนการเรียก 'docker' ตรง ๆ

  2. Runner เชื่อมต่อไม่ได้
      - ตรวจสอบให้ container `runner` และ `n8n` อยู่ใน network เดียวกัน
      - เช็กว่า `N8N_RUNNER_SERVER_URL` ชี้ไปที่ service `n8n`



## 7. คำสั่งพื้นฐาน
```bash
$DC ps        # ดู container

$DC logs -f   # ดู log

$DC down      # หยุด service

$DC up -d     # เริ่ม service
```

## 8. เทคนิคอื่น ๆ
- Backup ก่อนทำทุกอย่าง ป้องกัน workflow หาย
- ถ้าจะย้ายเครื่อง → ย้ายทั้งโฟลเดอร์ ~/n8n-docker และ ~/n8n-backups
- n8n-manager.sh เพิ่ม Auto Update Version n8n + Backup
- n8n-manager.sh เพิ่ม Restore แบบเลือกไฟล์ได้
- n8n-manager.sh ควรเพิ่ม certbot renew, restart nginx, $DC logs --tail=50 ทุกครั้ง
- n8n-manager.sh เมื่อเลือก Full Reset, Update, Full Maintenance ควรเรียก docker-compose.yml

```bash
create_compose() {
    log "📝 Creating new docker-compose.yml..."
    cat << 'YAML' > "$COMPOSE_FILE"
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
YAML
}
```