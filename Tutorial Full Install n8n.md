# Tutorial Full Install n8n

## Step 0 – ภาพรวมการติดตั้งและสถาปัตยกรรมระบบ

### **📌 สิ่งที่เราจะได้จากการติดตั้ง**

- n8n (เครื่องมือ Automation แบบ Open Source) ทำงานบน **Google Cloud e2-micro (Always Free tier)**
- ใช้ **Docker** เพื่อรัน container ของ n8n
- ใช้ **Nginx Reverse Proxy** \+ SSL (Let’s Encrypt / Cloudflare)  
  เก็บข้อมูลใน **Local Persistent Volume** เพื่อป้องกันข้อมูลหาย
- สามารถรันใน subdomain เช่น `n8n.thho.me` หรือเป็นโดเมนใหม่ก็ได้
- ใช้งานได้ฟรี (ตามเงื่อนไข Free tier ของ GCP) แต่พร้อมสำหรับ Production

---

### **🗂 ส่วนประกอบหลัก (Key Components)**

| ส่วนประกอบ                                       | หน้าที่                                                   | เหตุผลที่เลือก                                 |
| ------------------------------------------------ | --------------------------------------------------------- | ---------------------------------------------- |
| **Compute Engine VM**                            | รันระบบปฏิบัติการ Debian 12                               | e2-micro ฟรีตลอดตามเงื่อนไข Free tier          |
| **Docker Engine \+ Container n8n**               | รัน n8n ในสภาพแวดล้อมแยกออกจาก OS หลัก                    | ง่ายต่อการอัปเดต/ย้ายเซิร์ฟเวอร์               |
| **Nginx Reverse Proxy**                          | จัดการ SSL, ป้องกันการโจมตี, forward request ไป container | เพิ่มความปลอดภัยและยืดหยุ่น                    |
| **SSL Certificate (Let’s Encrypt / Cloudflare)** | เข้ารหัสการสื่อสาร HTTPS                                  | ป้องกันการดักฟังข้อมูล                         |
| **SQLite Database**                              | เก็บ workflows และข้อมูลระบบ                              | ติดตั้งง่าย ไม่ต้องดูแลเพิ่ม                   |
| **Local Persistent Storage**                     | เก็บข้อมูล container และฐานข้อมูล                         | ป้องกันข้อมูลสูญหายเมื่อ container ถูก restart |

---

### **🔍 การไหลของข้อมูล (ตามภาพ Architecture)**

![][image1]

1. **ผู้ใช้ (Internet/User)** → พิมพ์ `https://n8n.thho.me`
2. **DNS (Cloudflare)** → แปลงโดเมนเป็น IP ของ Google Cloud VM
3. **VM Instance (Debian 12\)** รับ request ผ่าน **Nginx**
4. **Nginx** ทำหน้าที่:
   - จัดการ SSL (Let’s Encrypt / Cloudflare)
   - Forward request ไป `localhost:5678` (port ของ n8n ใน container)
5. **Docker Engine** → ส่ง request เข้า container `n8nio/n8n`
6. **n8n Application** → ประมวลผล workflow, เข้าถึง database (SQLite) และส่งผลกลับ
7. **ผลลัพธ์** → กลับไปยังผู้ใช้ผ่าน Nginx และ HTTPS

---
### **💡 จุดเด่นของสถาปัตยกรรมนี้**

- **ฟรี:** อยู่ใน Free tier ของ GCP ถ้าคุณใช้ e2-micro \+ traffic ไม่เกินกำหนด
- **ปลอดภัย:** SSL \+ Reverse Proxy \+ Cloudflare ปกป้องจากการโจมตีโดยตรง
- **ง่ายต่อการดูแล:** Docker \+ docker-compose อัปเดตและย้ายระบบได้ง่าย
- **พร้อมใช้ใน Production:** มีการตั้งค่า timeout, websocket, และ header ให้เหมาะกับงาน Automation

---

### **⚠ ข้อควรระวัง**

- e2-micro มีทรัพยากรจำกัด (RAM 1 GB) → ถ้า workflow ใหญ่มาก อาจต้องอัปเกรด
- SQLite ดีสำหรับเริ่มต้น แต่ถ้า workflow เยอะ ควรย้ายไป PostgreSQL
- ต้องตั้ง backup (snapshot หรือ copy volume) เผื่อ VM หรือ container พัง

---

### **Deployment Options & Prerequisites**

### **🎯 เป้าหมายของขั้นตอนนี้**

- เลือกวิธีการติดตั้ง n8n บน Google Cloud ที่เหมาะกับความต้องการ
- เตรียมสิ่งที่ต้องมี (Prerequisites) ก่อนเริ่มติดตั้งจริง

---

## **1 เลือกวิธีติดตั้ง (Deployment Method)**

### **ตัวเลือกที่ 1: Compute Engine (VM)**

- **เหมาะสำหรับ**:
  - ต้องการควบคุมระบบปฏิบัติการเต็มรูปแบบ
  - ต้องการการตั้งค่า security เฉพาะ
  - ระบบทำงานแบบคาดการณ์ได้ (predictable workloads)
- **ข้อดี**:
  - Full control OS \+ software
  - ง่ายต่อการ backup/restore (snapshot VM)
  - ทำงานร่วมกับโครงสร้าง VM เดิมได้
  - รองรับฟีเจอร์ทั้งหมดของ n8n
- **ข้อเสีย**:
  - ต้องดูแลระบบเองทั้งหมด (update, patch, security)

---

### **ตัวเลือกที่ 2: Compute Engine \+ Docker**

- **เหมาะสำหรับ**:
  1. ต้องการ setup ง่าย, update สะดวก, ย้ายเซิร์ฟเวอร์ได้ง่าย
  2. ต้องการติดตั้งแบบแยกเป็น stack (n8n \+ Nginx \+ SSL \+ volumes)
- **โครงสร้าง**:

  1. **Docker Compose** – จัดการ container ทั้งหมดใน stack เดียว
  2. **Nginx Reverse Proxy** – รับ request และ forward ไปยัง n8n container
  3. **Certbot** – จัดการ SSL ฟรี (Let’s Encrypt)
  4. **Custom Domain** – ใช้โดเมนของคุณให้ดูเป็นทางการ

- **ข้อดี**:

  - ง่ายต่อการเริ่ม/หยุด/อัปเดต
  - ย้ายไปเครื่องใหม่ได้ง่าย (แค่ย้าย docker-compose.yml \+ volumes)
  - ใช้ Docker volumes แยกข้อมูลออกจากตัว app

- **ข้อเสีย**:

  - ต้องติดตั้ง Docker Engine และ Docker Compose เพิ่ม
  - ถ้าไม่เข้าใจ Docker อาจต้องเรียนรู้เพิ่มเล็กน้อย

---

## **2 สิ่งที่ต้องเตรียม (Prerequisites)**

| สิ่งที่ต้องมี                               | เหตุผล                                                 |
| ------------------------------------------- | ------------------------------------------------------ |
| **Google Cloud Account \+ Billing Enabled** | Free tier ต้องเปิด billing ถึงจะใช้ได้                 |
| **Domain Name \+ DNS Management**           | ต้องใช้ชี้ subdomain (เช่น `n8n.example.com`) มาที่ VM |
| **Basic Terminal Knowledge**                | รู้คำสั่งพื้นฐาน Linux (`ssh`, `apt`, `nano`, etc.)    |
| **เวลาประมาณ 30 นาที**                      | ติดตั้ง \+ ตั้งค่า \+ ทดสอบระบบ                        |

---

### **💡 การวิเคราะห์**

- ในคู่มือนี้จะใช้ **Compute Engine \+ Docker Compose** เพราะ:
  - ทำงานได้ใน Always Free tier
  - ง่ายต่อการดูแลและอัปเดต
  - แยกส่วนการทำงานชัดเจน (Nginx, n8n, SSL)
- การเตรียม prerequisites ให้ครบตั้งแต่ต้นจะลดปัญหาในขั้นตอนหลัง (เช่น DNS ยังไม่ตั้ง จะทำ SSL ไม่ผ่าน)

---

# Step 1 – Google Cloud Infrastructure Setup

### **🎯 เป้าหมายของขั้นตอนนี้**

- สร้างโครงสร้างพื้นฐานบน Google Cloud เพื่อรองรับการติดตั้ง n8n
- ใช้ Free Tier (e2-micro) เพื่อประหยัดค่าใช้จ่าย
- เตรียม VM พร้อม Static IP เพื่อให้บริการผ่านอินเทอร์เน็ตได้อย่างถาวร

---

## **1.1 สร้าง Google Cloud Project**

**ทำไมต้องสร้าง Project ก่อน?**

- Project คือ "กล่อง" เก็บทุก resource ของระบบนี้ (VM, IP, Network, etc.)
- ช่วยแยก **billing** (ค่าใช้จ่าย) ออกจาก Project อื่น
- จัดการสิทธิ์ (permissions) ได้เป็นระบบ

**ขั้นตอน:**

1. เข้า **Google Cloud Console**
2. คลิก **Project Dropdown** (มุมซ้ายบน)
3. เลือก **"New Project"**
4. ตั้งชื่อ เช่น `n8n-automation`
5. Organization: ใช้ค่า default เว้นแต่มีความต้องการเฉพาะ
6. คลิก **"Create"** และรอให้ระบบสร้าง Project

**ผลลัพธ์:**  
 ได้ Project ใหม่ที่ใช้สำหรับ deployment n8n โดยเฉพาะ และมีขอบเขต billing แยกชัดเจน

---

##

## **1.2 สร้าง Virtual Machine (VM)**

**เหตุผล:**

- n8n ต้องการ Server ที่รันได้ตลอดเวลา → VM ตอบโจทย์
- ใช้ e2-micro เพราะ:
  - ฟรีใน Always Free tier
  - 1 vCPU \+ 1 GB RAM เพียงพอสำหรับ workflow ส่วนใหญ่
  - ถ้าจำเป็น สามารถอัปเกรดเป็น e2-medium ได้ในอนาคต

**ขั้นตอน:**

1. ไปที่ **Compute Engine → VM Instances → Create Instance**
2. ตั้งค่า:

   - **Name:** `n8n-server`
   - **Region:** เลือก _us-central1_, _us-west1_ หรือ _us-east1_ (เข้า Free Tier)
   - **Zone:** เลือกใดก็ได้ใน Region นั้น
   - **Series:** E2
   - **Machine type:** e2-micro (Free Tier)  
      _(หมายเหตุ: ถ้าต้องการ RAM/CPU มากขึ้น เลือก e2-medium แต่จะมีค่าใช้จ่าย)_

3. **Boot disk:**

   - OS: **Ubuntu 22.04 LTS** หรือ **Debian 12**
   - Disk type: Standard persistent disk
   - Size: 30 GB (สูงสุดสำหรับ Free Tier)

4. **Firewall:**
   - ✅ Allow HTTP traffic
   - ✅ Allow HTTPS traffic

**การวิเคราะห์:**

- Firewall ที่เลือกไว้ จะเปิดพอร์ต 80 และ 443 ให้เข้าถึงจากภายนอกได้
- การใช้ Debian หรือ Ubuntu ขึ้นกับความถนัด (ในคู่มือนี้ใช้ Debian 12\)

---

## **1.3 จอง Static IP Address**

**เหตุผล:**

- ถ้า VM restart แล้วได้ IP ใหม่ → Domain จะชี้ผิด → ระบบใช้ไม่ได้
- Static IP แก้ปัญหานี้ได้ โดยผูก IP ตายตัวกับ VM

**ขั้นตอน:**

1. ไปที่ **VPC network → IP addresses**
2. คลิก **"Reserve External Static Address"**
3. ตั้งชื่อ: `n8n-static-ip`
4. Type: **Regional**
5. Region: เลือกให้ตรงกับ VM
6. Attach to: `n8n-server`
7. คลิก **"Reserve"**

**ผลลัพธ์:**  
 คุณจะได้ IP คงที่ (Static IP) จดเก็บไว้ → ใช้ตั้งค่า DNS ใน Step ถัดไป

---

### **📌 สรุปภาพรวม Step 1**

หลังจบขั้นนี้ คุณจะมี:

- Project GCP แยกสำหรับ n8n
- VM พร้อมระบบปฏิบัติการ \+ Firewall เปิด HTTP/HTTPS
- Static IP สำหรับชี้โดเมนในภายหลัง

---

# Step 2 – Domain DNS Configuration

### **🎯 เป้าหมายของขั้นตอนนี้**

- ตั้งค่า DNS เพื่อให้โดเมนหรือซับโดเมนของคุณ (`n8n.thho.me`) ชี้ไปยัง Static IP ของ VM ที่เราสร้างใน **Step 1.3**
- ทำตั้งแต่ตอนนี้เพราะ DNS อาจใช้เวลานานในการ propagate

---

## **2.0 ลงทะเบียนโดเมนและตั้งค่า DNS**

### **A-1 ลงทะเบียนโดเมนกับ Cloudflare Registrar (ถ้ายังไม่มีโดเมน)**

Cloudflare Registrar ให้บริการ **ลงทะเบียนโดเมนในราคาตามต้นทุนจริง โดยไม่มี markup หรือค่าต่ออายุแพง**  
 **ขั้นตอน:**

1. เข้าสู่ระบบ Cloudflare Dashboard
2. ไปที่ **Domain Registration → Register Domains**
3. ค้นหาชื่อโดเมนที่ต้องการ (เช่น thho.me) และตรวจสอบว่าใช้ได้
4. เลือกระยะเวลาการลงทะเบียน และกรอกข้อมูลผู้ติดต่อ (Registrant, Admin, Technical, Billing) ครบถ้วน
5. ชำระเงิน และรอระบบสร้างโดเมน (ใช้เวลาแค่ไม่ถึงนาที)

## ลิงก์สำคัญ: Cloudflare ใช้ระบบ Full Setup—ถ้าโดเมนลงทะเบียนผ่าน Cloudflare Registrar คุณจะจัดการ DNS ได้ผ่าน Cloudflare โดยตรง

### **A-2 ตั้งค่า DNS ใน Cloudflare**

**หลังจากโดเมนลงทะเบียนเรียบร้อย (หรือมีโดเมนแล้ว):**

1. เข้าสู่ **Cloudflare Dashboard** เลือกโดเมนที่ต้องการจากรายการ
2. ไปที่แท็บ **DNS** ด้านซ้าย

3. คลิก “Add record”
   - **Type:** A
   - **Name/Host:** `n8n` (สำหรับ subdomain `n8n.thho.me`)
   - **Content/Value:** Static IP จาก **Step 1.3**
   - **TTL:** 300 (หรือค่า default ก็ได้)
   - **Proxy status:** เลือก DNS Only หรือสามารถเปิด Proxy (สัญลักษณ์เมฆ) ตามต้องการ แต่สำหรับ n8n อาจเลือก DNS Only เพื่อหลีกเลี่ยงปัญหาการทำงานของ WebSocket และบริการ real‑time
4. กด Save — ง่ายนิดเดียว\!

---

### **B-1.0 ลงทะเบียนโดเมนกับ Domain กับ Namecheap และตั้งค่า DNS ใน Cloudflare**

**เหตุผลที่ใช้ Namecheap:**

- ราคาลงทะเบียนและต่ออายุถูก
- ฟรี WHOIS Privacy Protection
- จัดการโดเมนง่าย
- สามารถใช้ Nameserver ภายนอกได้ (Cloudflare)

**ขั้นตอนการจดโดเมน:**

1. เข้าเว็บ Namecheap.com
2. ค้นหาชื่อโดเมนที่ต้องการ เช่น `thho.me`
3. ถ้าว่าง ให้กด **Add to Cart**
4. เลือกระยะเวลาการลงทะเบียน (1 ปีขึ้นไป)
5. เข้าสู่ระบบ (หรือสมัครบัญชีใหม่)
6. ยืนยันว่าฟีเจอร์ **WHOIS Privacy Protection** เปิดอยู่ (ฟรี)
7. ชำระเงิน → รอระบบยืนยัน (ปกติไม่เกิน 1 นาที)
8. เมื่อเสร็จแล้ว โดเมนจะอยู่ใน **Dashboard → Domain List**

---

### **B-1.1 เปลี่ยน Nameserver ของ Namecheap ให้ใช้ Cloudflare**

เพื่อให้เราจัดการ DNS ผ่าน Cloudflare ได้

1. สมัครบัญชี Cloudflare และเข้าสู่ระบบ
2. คลิก **Add a Site** → ใส่ชื่อโดเมนที่จดกับ Namecheap เช่น `thho.me`
3. เลือกแผน **Free Plan** → ดำเนินการต่อ
4. Cloudflare จะสแกน DNS เดิมของคุณและแสดงรายการ Record ปัจจุบัน
5. คลิก **Continue** → Cloudflare จะแสดง Nameserver ใหม่ 2 ตัว (เช่น `xxxx.ns.cloudflare.com` และ `yyyy.ns.cloudflare.com`)
6. กลับไปที่ Namecheap:
   - ไปที่ **Domain List → Manage** ของโดเมน
   - ในช่อง **Nameservers** เลือก **Custom DNS**
   - ใส่ Nameserver ของ Cloudflare ทั้งสองตัว
   - กด **Save**
7. รอการเปลี่ยน Nameserver ให้สำเร็จ (ปกติ 15 นาที – 24 ชั่วโมง)

---

###

###

### **B-2 เพิ่ม A Record ใน Cloudflare**

เมื่อ Nameserver ชี้มาที่ Cloudflare แล้ว:

1. เข้า **Cloudflare Dashboard** → เลือกโดเมน
2. ไปที่แท็บ **DNS** → คลิก **Add Record**
   - **Type:** A
   - **Name:** `n8n` (สำหรับ `n8n.thho.me`)
   - **IPv4 address:** Static IP จาก Step 1.3
   - **TTL:** 300 (หรือ Auto)
   - **Proxy status:** เลือก **DNS only** (สีเทา) เพื่อหลีกเลี่ยงปัญหา WebSocket ของ n8n
3. กด **Save**

---

## **2.1 ตรวจสอบการ Propagation**

**ทำไมต้องตรวจสอบ:**

- DNS ไม่อัปเดตพร้อมกันทั่วโลกทันที อาจใช้เวลาตั้งแต่ **5 นาที** ถึง **48 ชั่วโมง**
- ต้องมั่นใจว่าชื่อโดเมนของคุณชี้ไปยัง IP ที่ถูกต้องก่อนติดตั้ง SSL และ Reverse Proxy

**วิธีตรวจสอบ:**

- **ออนไลน์:**
  - whatsmydns.net → กรอก `n8n.thho.me` เลือก A Record
- **คำสั่งใน Terminal:**

  bash:  
  `dig n8n.thho.me +short`  
  `nslookup n8n.thho.me`

**ผลลัพธ์ที่ถูกต้อง:**

- ต้องได้ IP ตรงกับ Static IP ที่คุณตั้งไว้ใน Step 1.3

---

### **📌 สรุป Step 2**

หลังจบขั้นนี้:

- `n8n.thho.me` จะชี้ไปยัง Static IP ของ VM
- DNS เริ่ม propagate ทั่วโลก
- พร้อมสำหรับขั้นตอนต่อไป: **การเข้า VM และติดตั้ง Docker \+ n8n**

##

# Step 3 – Server Preparation

### **🎯 เป้าหมายของขั้นตอนนี้**

- เชื่อมต่อไปยัง VM ที่สร้างไว้ใน Step 1
- อัปเดตแพ็กเกจและระบบให้เป็นเวอร์ชันล่าสุด เพื่อความปลอดภัยและความเข้ากันได้ก่อนติดตั้ง n8n

---

## **3.1 เชื่อมต่อไปยังเซิร์ฟเวอร์ (Connect to Server)**

**เหตุผล:**

- Google Cloud มีระบบ **Browser-based SSH** ทำให้เชื่อมต่อได้ง่ายโดยไม่ต้องติดตั้งโปรแกรมเพิ่มเติม (เช่น PuTTY, Terminal) หรือจัดการ SSH key เอง
- ใช้การเข้ารหัสและการตรวจสอบสิทธิ์ผ่านบัญชี Google Cloud โดยตรง

**ขั้นตอน:**

1. เข้า **Google Cloud Console** → **Compute Engine → VM instances**
2. หาชื่อ VM ของคุณ (เช่น `n8n-server`)
3. คลิกปุ่ม **SSH** ในคอลัมน์ "Connect"
   - ระบบจะเปิดหน้าต่าง Terminal ใหม่ในเบราว์เซอร์
4. รอจนกว่า terminal พร้อมให้พิมพ์คำสั่ง

**การวิเคราะห์:**

- วิธีนี้ปลอดภัยและเร็วที่สุดสำหรับการเข้าถึง VM ใหม่

ถ้าต้องการเชื่อมผ่าน local terminal สามารถใช้คำสั่ง:

```bash
gcloud compute ssh n8n-server --zone=<ZONE>
```

- _(ต้องติดตั้ง Google Cloud SDK)_

---

## **3.2 อัปเดตระบบ (Update System Packages)**

**เหตุผล:**

- ปิดช่องโหว่ด้านความปลอดภัย
- ปรับให้ซอฟต์แวร์ในระบบเข้ากันได้กับแพ็กเกจใหม่ที่จะติดตั้ง
- ลดปัญหา dependency conflict

**คำสั่ง:**

```bash
# อัปเดตรายการแพ็กเกจจาก repository
sudo apt update
# อัปเกรดแพ็กเกจทั้งหมดเป็นเวอร์ชันล่าสุด
sudo apt upgrade -y
```

**การทำงานทีละบรรทัด:**

1. `sudo apt update`

   - ดึงรายการแพ็กเกจล่าสุดจาก repo ของ Debian/Ubuntu
   - ไม่ติดตั้งอะไร แค่ sync ข้อมูลว่าแพ็กเกจไหนมีเวอร์ชันใหม่

2. `sudo apt upgrade -y`
   - ดาวน์โหลดและติดตั้งแพ็กเกจเวอร์ชันใหม่
   - ใช้ `-y` เพื่อยืนยันอัตโนมัติ ไม่ต้องกด `y` ระหว่างทาง

**หมายเหตุ:**

- ควรทำการ update & upgrade ก่อนติดตั้งซอฟต์แวร์ทุกครั้ง
- ถ้าระบบถามให้ restart service บางตัว ให้เลือกค่า default

---

✅ **ผลลัพธ์หลังจบ Step 3**

- เชื่อมต่อ VM ได้สำเร็จ
- ระบบอัปเดตล่าสุด พร้อมติดตั้ง Docker และ n8n ในขั้นตอนต่อไป

---

# Step 4 – Docker and Docker Compose Installation

### **🎯 เป้าหมายของขั้นตอนนี้**

- ติดตั้ง Docker และ Docker Compose เพื่อใช้ในการรัน n8n แบบ container
- ตั้งค่าให้ Docker ทำงานอัตโนมัติเมื่อเครื่องรีบูต เพื่อให้ระบบ n8n พร้อมใช้งานตลอดเวลา

---

## **4.1 ติดตั้ง Docker**

**เหตุผล:**

- Docker ทำให้สามารถแพ็กแอป (รวม dependencies) ลงใน container ที่รันได้เหมือนกันทุกเครื่อง
- ลดปัญหา "ทำไมเครื่องผมรันได้ แต่เครื่องคุณรันไม่ได้"
- อัปเดตง่าย เพียง pull image ใหม่แล้ว restart

**คำสั่งติดตั้ง:**  
bash  
`sudo apt install docker.io -y`

**การทำงานของคำสั่งนี้:**

1. ติดตั้ง `docker.io` จาก repository ของ Debian/Ubuntu
2. การใช้แพ็กเกจจาก distro จะทำให้ Docker integrate กับ `systemd` ได้ดี และได้รับอัปเดตความปลอดภัยผ่าน `apt`
3. `-y` ยืนยันการติดตั้งอัตโนมัติ

**ตรวจสอบเวอร์ชันหลังติดตั้ง:**  
bash  
`docker --version`

---

## **4.2 ติดตั้ง Docker Compose**

**เหตุผล:**

- Docker Compose ช่วยให้เราจัดการหลาย container พร้อมกันด้วยไฟล์ `.yml` เดียว
- สำหรับ n8n เราจะใช้ Docker Compose จัดการ:
  - container n8n
  - volume สำหรับเก็บข้อมูล
  - network ระหว่าง container และ reverse proxy

**คำสั่งติดตั้ง:**  
bash  
`sudo apt install docker-compose -y`  
**ตรวจสอบเวอร์ชัน:**  
bash  
`docker-compose --version`  
**วิเคราะห์:**

- การใช้ `apt install docker-compose` ติดตั้งจาก repo ทำให้การอัปเดตง่าย และ integrate กับระบบได้ดี
- ในบางกรณีถ้าต้องการเวอร์ชันใหม่ล่าสุด อาจติดตั้งจาก GitHub release ของ Docker Compose

---

## **4.3 ตั้งค่า Docker Service**

**เหตุผล:**

- เพื่อให้ Docker ทำงานตลอดเวลา และเริ่มอัตโนมัติเมื่อเซิร์ฟเวอร์รีบูต
- ช่วยให้ n8n container กลับมาทำงานทันทีแม้มีการ maintenance หรือ restart เครื่อง

**คำสั่ง:**  
bash  
`# เริ่มบริการ Docker ทันที`  
`sudo systemctl start docker`

`# ตั้งให้ Docker เริ่มอัตโนมัติเมื่อบูตเครื่อง`  
`sudo systemctl enable docker`

`# ตรวจสอบสถานะการทำงานของ Docker`  
`sudo systemctl status docker`

**การวิเคราะห์:**

- `start` → สั่งให้ Docker daemon ทำงานทันที
- `enable` → ตั้งให้ Docker daemon เริ่มทำงานทุกครั้งที่เครื่องเปิด
- `status` → ตรวจสอบว่าบริการกำลังทำงานอยู่ (`active (running)` คือปกติ)

---

✅ **ผลลัพธ์หลังจบ Step 4**

- ระบบมี Docker และ Docker Compose พร้อมใช้งาน
- Docker จะเริ่มอัตโนมัติเมื่อเซิร์ฟเวอร์รีบูต
- พร้อมสำหรับขั้นตอนต่อไป: **Step 5 – n8n Installation and Configuration with Docker Compose**

---

# Step 5 – Create Docker Compose Configuration (Optional)

### **🎯 เป้าหมายของขั้นตอนนี้**

- สร้างไฟล์ **Docker Compose** เพื่อให้การจัดการ n8n ง่ายขึ้น (เริ่ม/หยุด/อัปเดตในคำสั่งเดียว)
- จัดเก็บการตั้งค่าแบบ version-controlled เพื่อให้ทำซ้ำได้ง่าย
- จัดการ **persistent data** ให้ข้อมูลไม่หายเมื่อ container รีสตาร์ท

---

## **5.1 สร้างโฟลเดอร์โปรเจกต์**

เก็บไฟล์ทุกอย่างของ n8n ไว้ในที่เดียวเพื่อความเป็นระเบียบ  
bash  
`mkdir ~/n8n-docker && cd ~/n8n-docker`  
**การวิเคราะห์:**

- `mkdir ~/n8n-docker` → สร้างโฟลเดอร์ใหม่ใน home directory ของผู้ใช้
- `cd ~/n8n-docker` → เข้าไปในโฟลเดอร์ที่สร้างไว้
- โฟลเดอร์นี้จะเก็บ:
  - `docker-compose.yml` (ไฟล์ตั้งค่า Docker Compose)
  - ไฟล์อื่น ๆ เช่น `.env` หรือ script ที่เกี่ยวข้อง

---

## **5.2 สร้างไฟล์ Docker Compose**

bash  
`nano docker-compose.yml`

จากนั้นใส่เนื้อหานี้ (แก้ `n8n.example.com` เป็นโดเมนจริง เช่น `n8n.thho.me`):  
yaml  
`services:`  
 `n8n:`  
 `image: n8nio/n8n:latest`  
 `container_name: n8n`  
 `restart: unless-stopped`  
 `ports:`  
 `- "5678:5678"`  
 `environment:`  
 `- N8N_HOST=n8n.thho.me`  
 `- WEBHOOK_TUNNEL_URL=https://n8n.thho.me/`  
 `- WEBHOOK_URL=https://n8n.thho.me/`  
 `- GENERIC_TIMEZONE=UTC`  
 `volumes:`  
 `- n8n_data:/home/node/.n8n`  
 `networks:`  
 `- n8n_network`

`volumes:`  
 `n8n_data:`  
 `driver: local`

`networks:`  
 `n8n_network:`  
 `driver: bridge`

**การวิเคราะห์แต่ละบรรทัด:**

- `services` → กลุ่ม container ที่จะรันใน stack นี้
- `image` → ใช้ image จาก Docker Hub (`n8nio/n8n:latest`)
- `container_name` → ตั้งชื่อ container ให้เรียกง่าย
- `restart: unless-stopped` → ให้ restart อัตโนมัติถ้าหยุดทำงาน ยกเว้นหยุดด้วยมือ
- `ports` → map port 5678 ของ container → port 5678 ของเครื่อง host (ให้ Nginx เชื่อมต่อได้)
- `environment` → ตัวแปรสภาพแวดล้อมของ n8n:
  - `N8N_HOST` → โดเมนที่ใช้รัน n8n
  - `WEBHOOK_TUNNEL_URL` และ `WEBHOOK_URL` → ใช้สร้างลิงก์ webhook ที่ถูกต้อง
  - `GENERIC_TIMEZONE` → ตั้ง timezone ให้ตรงกับที่ต้องการ (UTC หรือ Asia/Bangkok)
- `volumes` → เก็บข้อมูลใน local volume `n8n_data` เพื่อไม่ให้หายเมื่อลบ container
- `networks` → ใช้ bridge network แยกเฉพาะ stack นี้ เพื่อความปลอดภัย

---

## **5.3 รัน n8n ด้วย Docker Compose**

bash  
`sudo docker-compose up -d`

**การวิเคราะห์:**

- `up` → สร้างและรัน container ตามไฟล์ `docker-compose.yml`
- `-d` → รันแบบ detached (เบื้องหลัง) ให้เรากลับมาใช้ terminal ได้ต่อ

---

## **5.4 ตรวจสอบการทำงานของ n8n**

bash  
`# ดู container ที่กำลังรัน`  
`sudo docker-compose ps`

`# ดู log ของ container n8n`  
`sudo docker-compose logs n8n`

`# ทดสอบการเชื่อมต่อภายในเครื่อง`  
`curl -I http://localhost:5678`

**สิ่งที่ต้องเห็น:**

- `docker-compose ps` → container `n8n` สถานะ "Up"
- `docker-compose logs n8n` → มีข้อความ `n8n ready on ::, port 5678`
- `curl` → ได้ HTTP/1.1 200 OK

---

✅ **ผลลัพธ์หลังจบ Step 5**

- n8n รันใน Docker container พร้อม configuration
- ข้อมูล workflow และ credentials ถูกเก็บใน persistent volume  
  พร้อมเชื่อมต่อกับ Nginx Reverse Proxy (Step 6\)

---

# Step 6 – Nginx Installation and Configuration

### **🎯 เป้าหมายของขั้นตอนนี้**

- ติดตั้ง **Nginx** เพื่อทำหน้าที่ Reverse Proxy
- ช่วยให้การเข้าถึง n8n ปลอดภัยและรองรับ HTTPS
- เตรียมการเชื่อมต่อกับ Certbot (Step ถัดไป) สำหรับ SSL

---

## **6.1 ติดตั้ง Nginx**

**คำสั่ง:**  
bash  
`sudo apt install nginx -y`

**การทำงานของคำสั่งนี้:**

- ติดตั้งแพ็กเกจ Nginx จาก repository ของ Debian/Ubuntu
- `-y` คือ auto-confirm ไม่ต้องกด `y` ระหว่างติดตั้ง

**เหตุผล:**

- Nginx ทำหน้าที่เป็น "ประตูหน้า" ของระบบ ช่วยทำ:
  - **SSL termination** (เข้ารหัส HTTPS ที่ Nginx แล้วส่ง plain HTTP ภายใน)
  - **Security filtering** (เพิ่ม HTTP security headers)
  - **Reverse Proxy** ให้กับ n8n container
  - **รองรับหลายเว็บในเซิร์ฟเวอร์เดียว** (ผ่าน Virtual Host)

---

## **6.2 สร้างไฟล์ config ของ Nginx สำหรับ n8n**

**คำสั่ง:**  
bash  
`sudo nano /etc/nginx/sites-available/n8n.conf`

**เนื้อหาไฟล์ (ปรับ `n8n.example.com` เป็นโดเมนจริง เช่น `n8n.thho.me`):**  
nginx  
`server {`  
 `listen 80;`  
 `server_name n8n.thho.me;`

    `# Redirect all HTTP traffic to HTTPS (will be added by Certbot)`
    `location / {`
        `proxy_pass http://localhost:5678;`
        `proxy_http_version 1.1;`

        `# Disable buffering for real-time functionality`
        `chunked_transfer_encoding off;`
        `proxy_buffering off;`
        `proxy_cache off;`

        `# WebSocket support for n8n's real-time features`
        `proxy_set_header Upgrade $http_upgrade;`
        `proxy_set_header Connection "upgrade";`

        `# Essential proxy headers for proper functionality`
        `proxy_set_header Host $host;`
        `proxy_set_header X-Real-IP $remote_addr;`
        `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`
        `proxy_set_header X-Forwarded-Proto $scheme;`

        `# Extended timeout for long-running workflows`
        `proxy_read_timeout 86400;`
        `proxy_connect_timeout 60;`
        `proxy_send_timeout 60;`
    `}`

`}`

**การวิเคราะห์:**

- **`listen 80`** → รอรับการเชื่อมต่อ HTTP ปกติ (Certbot จะเพิ่ม HTTPS ในภายหลัง)
- **`proxy_pass http://localhost:5678;`** → forward request ไปยัง n8n container ที่รันบนพอร์ต 5678
- **`chunked_transfer_encoding off;`** และ **`proxy_buffering off;`** → ปิดการ buffer เพื่อให้ข้อมูล real-time ส่งถึง browser ทันที
- **WebSocket Support** → ใช้ `Upgrade` และ `Connection "upgrade"` เพื่อให้ฟีเจอร์ real-time ของ n8n ทำงานได้
- **Proxy Headers** → ส่งข้อมูลต้นฉบับ เช่น IP ผู้ใช้จริง (`X-Real-IP`)
- **Timeout** → 86400 วินาที (24 ชั่วโมง) เพื่อรองรับ workflow ที่รันนาน

---

## **6.3 เปิดใช้งาน config ของ Nginx**

**คำสั่ง:**  
bash  
`sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/`  
`sudo nginx -t`  
`sudo systemctl restart nginx`  
`sudo systemctl status nginx`

**การทำงานทีละบรรทัด:**

1. `ln -s` → สร้าง symbolic link เพื่อเปิดใช้งาน site config (`sites-available` → `sites-enabled`)
2. `nginx -t` → ตรวจสอบ syntax ของ config ป้องกัน error ก่อน restart
3. `systemctl restart nginx` → รีสตาร์ท Nginx ให้ใช้ config ใหม่
4. `systemctl status nginx` → ตรวจสอบว่ายังรันอยู่ (`active (running)` คือปกติ)

---

## **คำแนะนำเพิ่มเติม**

1. **ปิดการเข้าถึงตรงไปที่ port 5678**
   - หลังจากใช้ Nginx proxy แล้ว ควรปิด firewall ของ port 5678 จาก public internet และให้เข้าผ่าน Nginx เท่านั้น

bash

`sudo ufw allow 80,443/tcp`  
`sudo ufw deny 5678/tcp`

2. **เตรียม SSL ต่อใน Step 7**

   - ในขั้นตอนนี้ยังเป็น HTTP เท่านั้น
   - ควรต่อด้วย Certbot เพื่อให้เป็น HTTPS พร้อม auto-renew

3. **ตรวจสอบการทำงาน**

ทดสอบจากเครื่อง client:

bash  
`curl -I http://n8n.thho.me`

- ต้องได้ HTTP 200 OK จาก Nginx และเป็นเนื้อหาจาก n8n

---

✅ **ผลลัพธ์หลังจบ Step 6**

- Nginx ทำงานเป็น Reverse Proxy สำหรับ n8n แล้ว
- การเชื่อมต่อผ่าน `http://n8n.thho.me` จะถูกส่งไปยัง n8n container port 5678 ภายในเครื่อง
- พร้อมต่อ Step 7 เพื่อติดตั้ง SSL และ redirect ไป HTTPS

---

##

# Step 7 – SSL Certificate Setup

### **🎯 เป้าหมายของขั้นตอนนี้**

- ติดตั้งและตั้งค่า **SSL Certificates** สำหรับ `n8n.thho.me`
- ใช้ **Certbot** \+ **Nginx plugin** เพื่อขอใบรับรองจาก Let’s Encrypt และตั้งค่า redirect HTTP → HTTPS อัตโนมัติ

---

## **7.1 ติดตั้ง Certbot**

**คำสั่ง:**  
bash  
`sudo apt install certbot python3-certbot-nginx -y`  
**การทำงาน:**

- ติดตั้ง `certbot` → เครื่องมือขอใบรับรองจาก Let’s Encrypt
- ติดตั้ง `python3-certbot-nginx` → plugin ที่ทำให้ Certbot สามารถแก้ไข config ของ Nginx อัตโนมัติ
- `-y` → ยืนยันการติดตั้งอัตโนมัติ

**เหตุผล:**

- Let’s Encrypt เป็น **Certificate Authority (CA)** ฟรีและเชื่อถือได้โดยทุกเบราว์เซอร์
- ใช้ plugin Nginx ช่วยลดความซับซ้อนในการแก้ไฟล์ config ด้วยมือ

---

## **7.2 ขอและติดตั้ง SSL Certificate**

**คำสั่ง:**  
bash  
`sudo certbot --nginx -d n8n.thho.me`  
_(แก้ `n8n.thho.me` เป็นโดเมนจริงของคุณ)_  
**สิ่งที่จะเกิดขึ้นระหว่างรันคำสั่ง:**

1. **กรอก Email Address** → ใช้สำหรับแจ้งเตือนหมดอายุและประกาศด้านความปลอดภัย
2. **ยอมรับ Terms of Service** ของ Let’s Encrypt
3. **เลือกว่าจะรับ Email จาก EFF หรือไม่** (optional)
4. **เลือก redirect HTTP → HTTPS** (แนะนำให้เลือก) เพื่อบังคับการเชื่อมต่อที่ปลอดภัย
5. Certbot จะ:
   - ตรวจสอบว่าชื่อโดเมนชี้มาที่ IP ของเครื่องคุณ (DNS ต้องตั้งถูกต้องก่อน)
   - ขอใบรับรองจาก Let’s Encrypt
   - แก้ไข config ของ Nginx ให้ใช้ HTTPS และ redirect HTTP

**ตัวอย่าง config หลัง Certbot แก้ไข (เพิ่ม SSL block) เช็คว่าได้เพิ่มเข้ามาแล้ว:**  
bash  
`sudo nano /etc/nginx/sites-available/n8n.conf`

nginx  
`server {`  
 `listen 443 ssl;`  
 `server_name n8n.thho.me;`

    `ssl_certificate /etc/letsencrypt/live/n8n.thho.me/fullchain.pem;`
    `ssl_certificate_key /etc/letsencrypt/live/n8n.thho.me/privkey.pem;`
    `include /etc/letsencrypt/options-ssl-nginx.conf;`
    `ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;`

    `location / {`
        `proxy_pass http://localhost:5678;`
        `proxy_http_version 1.1;`
        `...`
    `}`

`}`

`server {`  
 `listen 80;`  
 `server_name n8n.thho.me;`  
 `return 301 https://$host$request_uri;`  
`}`

---

##

## **7.3 ตรวจสอบการตั้งค่า SSL**

**ตรวจด้วย Browser:**

- เข้า `https://n8n.thho.me` → ต้องมี 🔒 Lock icon ใน address bar
- ตรวจสอบ certificate details → ต้องออกโดย Let’s Encrypt

**ตรวจด้วย command line:**  
bash  
`curl -I https://n8n.thho.me`  
ต้องได้ HTTP 200 OK และ protocol `HTTP/2` หรือ `HTTP/1.1` ผ่าน `https`

---

##

## **7.4 ปัญหาที่เกิดขึ้นระหว่างทำ**

รอบแรก **Certbot ไม่สามารถยืนยันโดเมนได้** ขึ้น error:  
java  
`Timeout during connect (likely firewall problem)`

**สาเหตุจริง**

- DNS ของ `n8n.thho.me` ชี้ไป **IP เก่า** (34.66.xx.xx)
- แต่ VM ที่รัน Nginx อยู่มี IP ใหม่ (35.206.xx.xx)
- ทำให้ Let's Encrypt ต่อเข้า IP เก่า → ล้มเหลว

---

###

## **7.5 วิธีแก้ปัญหา**

1. **ตรวจสอบ IP ปัจจุบันของ VM**  
    bash  
   `curl ifconfig.me`  

2. **เช็ก DNS ปัจจุบัน**  
    bash  
   `dig n8n.thho.me +short`

3. ถ้าไม่ตรง → แก้ A record ใน **Cloudflare** ให้ชี้ไป IP ใหม่
4. ปิด Proxy (ให้เป็น "DNS only") ตอนขอ SSL
5. รอ DNS propagate (ปกติ ≤ 5 นาทีถ้า TTL ต่ำ)
6. ทดสอบพอร์ต 80 ว่าเข้าถึงได้  
    bash  
   `curl -I http://n8n.thho.me`  
    ต้องได้ `HTTP/1.1 200 OK`

---

## **7.6 ขอกันใหม่**

หลังแก้ DNS ให้ถูกต้อง รันคำสั่งเดิม:  
bash  
`sudo certbot --nginx -d n8n.thho.me`

**ผลลัพธ์สำเร็จ:**  
swift  
`Successfully received certificate.`  
`Certificate is saved at: /etc/letsencrypt/live/n8n.thho.me/fullchain.pem`  
`Key is saved at:         /etc/letsencrypt/live/n8n.thho.me/privkey.pem`  
`Congratulations! You have successfully enabled HTTPS`

---

## **7.7 ตรวจสอบ SSL**

bash  
`curl -I https://n8n.thho.me`

## ต้องได้ status code 200 หรือ 301 (redirect ไป https)

## **7.8 ทดสอบต่ออายุอัตโนมัติ**

bash  
`sudo certbot renew --dry-run`

## ถ้าผ่าน แปลว่าทุก 90 วัน Certbot จะต่ออายุให้เองโดยไม่ต้องทำอะไร

## **7.9 ตรวจสอบ Cron Job ของ Certbot**

bash  
`systemctl list-timers | grep certbot`

- ปกติจะมี task auto-renew ทุกวัน

---

## **7.10 วิธีป้องกันไม่ให้พลาดรอบหน้า**

- **จอง Static IP** ใน Google Cloud และใช้ IP นั้นใน DNS  
   (ไปที่ Google Cloud → VPC network → External IP addresses → Reserve)
- ทุกครั้งก่อนขอ SSL ให้ **ตรวจสอบ IP ปัจจุบันกับ DNS ว่าตรงกัน**
- ใช้ `DNS only` ใน Cloudflare ระหว่างขอ SSL  
   (หลังขอเสร็จจะเปิด Proxy ก็ได้)

---

##

##

## **คำแนะนำเพิ่มเติม (เพิ่มจากเนื้อหาเดิม)**

### **ความปลอดภัยเพิ่มเติม**

### **⚠️ ข้อควรระวัง**

- เปิดใช้ HSTS แล้ว ถ้าเว็บมีปัญหา SSL เบราว์เซอร์จะไม่ยอมให้เข้าผ่าน HTTP เลย

- ถ้าพึ่งติดตั้ง SSL ใหม่ ควรทดสอบให้แน่ใจก่อนว่าเว็บเข้าผ่าน HTTPS ได้สมบูรณ์ 100% ก่อนเปิด HSTS

- เปิด **HTTP Strict Transport Security (HSTS)** ใน config Nginx เพื่อบังคับใช้ HTTPS:

nginx  
`add_header Strict-Transport-Security "max-age=31536000" always;`

###

**3.1 เปิดไฟล์ config ของเว็บ**

bash  
`sudo nano /etc/nginx/sites-available/n8n.conf`  
**3.2 เพิ่มบรรทัด HSTS ใน block ที่เป็น HTTPS เท่านั้น**  
 เช่น ใน `server { listen 443 ssl; ... }`

nginx  
`server {`  
 `listen 443 ssl;`  
 `server_name n8n.thho.me;`

    `ssl_certificate /etc/letsencrypt/live/n8n.thho.me/fullchain.pem;`
    `ssl_certificate_key /etc/letsencrypt/live/n8n.thho.me/privkey.pem;`

    `add_header Strict-Transport-Security "max-age=31536000" always;`

    `location / {`
        `proxy_pass http://localhost:5678;`
        `proxy_http_version 1.1;`
        `proxy_set_header Upgrade $http_upgrade;`
        `proxy_set_header Connection "upgrade";`
        `proxy_set_header Host $host;`
        `proxy_set_header X-Real-IP $remote_addr;`
        `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`
        `proxy_set_header X-Forwarded-Proto $scheme;`
        `proxy_read_timeout 86400;`
        `proxy_connect_timeout 60;`
        `proxy_send_timeout 60;`
    `}`

`}`  
**หมายเหตุ**:

- `max-age=31536000` คือ 1 ปี (วินาที)
- `always` บังคับให้ส่ง header นี้ในทุกการตอบสนองจากเซิร์ฟเวอร์

**3.3 บันทึกไฟล์แล้วตรวจสอบ syntax**  
 bash  
`sudo nginx -t`

3.4 **โหลดค่าใหม่**  
 bash  
`sudo systemctl reload nginx`

**3.5 ทดสอบว่า header ถูกส่งออกมา**  
 bash  
`curl -I https://n8n.thho.me`

pgsql  
`Strict-Transport-Security: max-age=31536000`

---

ถ้าต้องการบังคับกับทุก subdomain ให้เพิ่ม `; includeSubDomains`  
 nginx  
`add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;`

---

✅ **ผลลัพธ์หลังจบ Step 7**

- `https://n8n.thho.me` ใช้งานได้ด้วย HTTPS
- Nginx redirect HTTP → HTTPS อัตโนมัติ
- Certbot ตั้งค่า auto-renew ใบรับรองทุก 90 วัน

---

## **อธิบายสรุปภาพ Flowchart**

### **ขั้นตอนการขอ SSL Certificate**

1. **ตรวจสอบ IP ของ VM**

   - ใช้ `curl ifconfig.me` เพื่อดึง IP ปัจจุบันของเครื่อง

2. **เช็กว่า DNS ชี้มาที่ IP เดียวกันหรือไม่**

   - `dig yourdomain.com +short`
   - ถ้าไม่ตรง จะต้องอัปเดต DNS (แก้ A record)

3. **เปิดพอร์ต HTTP (80) และ HTTPS (443) จริง**

   - ต้องแน่ใจว่า Google Cloud Firewall \+ VM Firewall อนุญาตรับเข้า port เหล่านี้

4. **ทดสอบเข้าถึงผ่าน HTTP ได้ไหม**

   - `curl -I http://yourdomain.com`
   - ถ้าได้ HTTP 200 OK → ต่อไปขอ SSL ได้

5. **รัน Certbot**
   - `sudo certbot --nginx -d yourdomain.com`
   - ระบบจะขอใบ SSL แล้วกำหนดให้ Nginx ใช้งาน HTTPS พร้อม redirect  

6. **ตรวจสอบผลลัพธ์**

   - `curl -I https://yourdomain.com`
   - ต้องเจอ status เช่น `HTTP/1.1 200 OK` หรือ redirect

7. **ตั้ง auto renew อัตโนมัติ**
   - ทดสอบ: `sudo certbot renew --dry-run`
   - ถ้าใช้งานได้ ระบบจะต่ออายุเองอัตโนมัติเมื่อใกล้หมดอายุ

---

## **จุดที่มักพลาด**

- **DNS ไม่ตรงกับ IP VM**

  - เพราะ VM เปลี่ยน IP (ถ้ายังไม่ใช้ Static IP) หรือไม่ได้อัปเดต A record

- **Cloudflare Proxy เปิด (เมฆส้ม)**

  - ทำให้ Certbot ไม่ระบุ IP จริงของ VM เมื่อมี HTTP challenge
  - แก้ง่ายๆ: ปิด Proxy ชั่วคราว (DNS only)

- **พอร์ต 80/443 ถูกบล็อก**
  - อาจลืมตั้ง Firewall ของ Google Cloud หรือ firewall ใน VM เอง

---

## **SSL Request & Verification Flow**

(สำหรับ `sudo certbot --nginx -d yourdomain.com`)  
\[เริ่มต้น\]  
 ↓  
ตรวจสอบ IP ปัจจุบันของ VM  
 ↓  
curl ifconfig.me → \[Your VM IP\]  
 ↓  
ตรวจสอบ DNS ชี้ไป IP เดียวกันหรือไม่?  
 ↓  
dig yourdomain.com \+short  
 ↓  
 ┌─────────────────────────────┐  
 │ DNS IP \== VM IP ? │  
 └───────────────┬─────────────┘  
 │ ใช่  
 │  
 ▼  
 เปิดพอร์ต 80, 443  
 ↓  
 curl \-I http://yourdomain.com  
 ↓  
 ┌─────────────────────────────┐  
 │ ได้ HTTP 200 / 301 ? │  
 └───────────────┬─────────────┘  
 │ ใช่  
 │  
 ▼  
 รัน certbot \--nginx \-d yourdomain.com  
 ↓  
 Certbot ติดต่อ Let's Encrypt  
 ↓  
 Let's Encrypt เข้า http://yourdomain.com/.well-known/acme-challenge  
 ↓  
 ยืนยันโดเมนสำเร็จ  
 ↓  
 ออกใบ Cert \+ แก้ Nginx ให้ใช้ HTTPS  
 ↓  
 ตรวจสอบ SSL → curl \-I https://yourdomain.com  
 ↓  
 ตั้ง cron ต่ออายุอัตโนมัติ  
 ↓  
 \[เสร็จสิ้น\]

## **Step 8: Access and Initial Configuration**

### **8.1 เข้าถึง n8n ครั้งแรก**

1. **ตรวจสอบว่า DNS พร้อมใช้งานแล้ว** ใช้คำสั่ง:

bash  
`nslookup n8n.thho.me`  
 หรือ

bash  
`dig n8n.thho.me +short`

- IP ที่แสดงควรตรงกับ Static IP ของ VM ที่ตั้งไว้ใน **Step 1.3**

2. **เปิดเว็บเบราว์เซอร์** ไปที่:

arduino  
`https://n8n.thho.me`

- ถ้าทำถูกขั้นตอนใน **Step 7** จะเห็นสัญลักษณ์กุญแจ 🔒 (SSL certificate valid) ที่ address bar

3. **ทำตาม Setup Wizard**
   - สร้าง **User Account แรก** → username, email, password
   - กรอกข้อมูลพื้นฐาน เช่น Timezone (สามารถแก้ทีหลังได้ใน Settings)

---

### **8.2 การตั้งค่าความปลอดภัยเบื้องต้น**

**1\. ใช้รหัสผ่านที่แข็งแรง**

- ผสม **ตัวพิมพ์ใหญ่, ตัวพิมพ์เล็ก, ตัวเลข, สัญลักษณ์พิเศษ**
- อย่างน้อย **12 ตัวอักษรขึ้นไป**
- หลีกเลี่ยงคำเดาง่าย เช่น `password123`, `n8n2025`

**2\. กำหนดสิทธิ์ผู้ใช้ (User Roles)**

- ถ้ามีทีมงาน ให้สร้าง User แยกแต่ละคน
- ใช้ Role:
  - **Owner / Admin** → จัดการระบบทั้งหมด
  - **Editor** → สร้าง/แก้ workflow
  - **Viewer** → ดูอย่างเดียว

**3\. ทดสอบการทำงานของ Webhook**  
สร้าง Workflow ง่าย ๆ เช่น:  
 sql  
`Trigger → Webhook`  
`Respond → HTTP Response`

- กด “Execute Workflow” แล้วเรียกผ่าน:  
   bash  
  `curl -X GET "https://n8n.thho.me/webhook-test"`  

- ถ้าได้ Response → แสดงว่าเซิร์ฟเวอร์เข้าถึงได้จากภายนอก

**4\. ตรวจสอบการเก็บข้อมูล (Data Persistence)**

- สร้าง Workflow → Save

Restart container:  
 bash  
`sudo docker-compose restart`

- กลับไปเช็กว่า Workflow ยังอยู่ → ถ้าหาย แสดงว่า Volume mapping ใน Step 5 อาจผิดพลาด

---

### **ข้อควรระวัง**

- ถ้าเข้า `https://n8n.thho.me` แล้วขึ้น **"Not Secure"** → แปลว่า SSL อาจไม่ถูกติดตั้ง หรือ DNS ยังชี้ไปผิด IP
- หลังจากเปิดใช้งานแล้ว ควรสำรองไฟล์ `~/.n8n` หรือ Volume `n8n_data` อย่างน้อยสัปดาห์ละครั้ง
- เปิดการบังคับใช้ HTTPS เสมอ (HSTS) → ถ้ายังไม่ได้ทำ ให้ย้อนกลับไปเพิ่มใน Step 7 หรือเพิ่ม **Step 9: Security Hardening**

---

## **Step 9: Maintenance and Updates – n8n.thho.me**

### **9.1 Updating n8n**

อัปเดต n8n ด้วย Docker Compose เพื่อความปลอดภัยและได้ฟีเจอร์ใหม่ โดยไม่กระทบข้อมูลเดิม  
bash  
CopyEdit  
`cd ~/n8n-docker`  
`sudo docker-compose pull`  
`sudo docker-compose up -d`  
`sudo docker-compose logs n8n`

- `pull` → ดึง image เวอร์ชันล่าสุดจาก Docker Hub
- `up -d` → รีสร้าง container โดยใช้ image ใหม่
- Logs ควรไม่มี error และแสดงข้อความว่า **n8n ready on port 5678**

---

### **9.2 Managing Your Deployment**

คำสั่งพื้นฐานที่ใช้ดูแลระบบ  
bash  
CopyEdit  
`sudo docker-compose down             # หยุดทุก service`  
`sudo docker-compose up -d            # เริ่มใหม่`  
`sudo docker-compose logs             # ดู log ทั้งหมด`  
`sudo docker-compose logs n8n         # ดู log เฉพาะ n8n`  
`sudo docker-compose restart n8n      # รีสตาร์ทเฉพาะ n8n`

---

### **9.3 Backup Your Data**

ข้อมูล n8n เก็บใน Docker Volume `n8n-docker_n8n_data`  
 สำรองข้อมูลแบบ `.tar.gz` เพื่อกู้คืนภายหลัง  
bash  
CopyEdit  
`mkdir -p ~/n8n-backups`  
`sudo docker run --rm \`  
 `-v n8n-docker_n8n_data:/data \`  
 `-v ~/n8n-backups:/backup \`  
 `ubuntu:latest \`  
 `tar czf /backup/n8n-backup-$(date +%Y%m%d_%H%M%S).tar.gz -C /data .`

`cd ~/n8n-backups && ls -t n8n-backup-*.tar.gz | tail -n +11 | xargs rm -f`

- เก็บแค่ 10 backup ล่าสุด เพื่อลดการใช้พื้นที่

---

### **9.4 Monitor System Health**

เช็กสถานะและประสิทธิภาพเครื่อง  
bash  
CopyEdit  
`sudo docker-compose ps`  
`sudo docker-compose logs --tail=50 n8n`  
`free -h   # ใช้ RAM เท่าไร`  
`df -h     # เหลือพื้นที่ดิสก์เท่าไร`  
`uptime    # โหลดเครื่องและเวลาทำงาน`

---

### **9.5 SSL Certificate Renewal**

Certbot จะต่ออายุ SSL อัตโนมัติทุก 90 วัน  
 แต่ควรทดสอบว่า auto-renew ทำงานจริง  
bash  
CopyEdit  
`sudo certbot renew --dry-run`  
`sudo systemctl status certbot.timer`  
`sudo certbot renew   # ถ้าต้องต่ออายุเอง`

---

## **Troubleshooting Common Issues**

### **1\. Container ไม่ยอม Start**

bash  
CopyEdit  
`sudo docker-compose ps`  
`sudo docker-compose logs n8n`  
`sudo netstat -tlnp | grep 5678`  
`sudo docker-compose restart`

### **2\. SSL Certificate มีปัญหา**

bash  
CopyEdit  
`sudo certbot certificates`  
`nslookup n8n.thho.me`  
`sudo certbot --nginx -d n8n.thho.me --force-renewal`

### **3\. Nginx Config Error**

bash  
CopyEdit  
`sudo nginx -t`  
`sudo tail -f /var/log/nginx/error.log`  
`sudo systemctl restart nginx`

### **4\. Docker Compose Error**

bash  
CopyEdit  
`sudo docker-compose config`  
`sudo docker-compose down && sudo docker-compose up -d --force-recreate`  
`sudo docker-compose down -v && sudo docker-compose up -d   # ลบ volume (ข้อมูลหาย)`

---

✅ ตอนนี้ **n8n.thho.me** ของคุณพร้อมใช้งานในระดับ Production แล้ว

- ได้ SSL ปลอดภัย
- สำรองข้อมูลได้
- อัปเดตง่าย
- มีวิธีแก้ปัญหาพื้นฐาน

---

**9.6 ตั้งค่า Task Runner ให้ทำงานผ่าน Proxy ได้จริง**  
จุดที่ทำให้คุณเจอปัญหา คือ:

1. `X-Forwarded-For` ถูกตั้ง แต่ `trust proxy` ใน Express ไม่เปิดจริง
2. Task Runner พยายามต่อกับ `n8n main` ผ่าน URL ที่ไม่ถูกต้อง
3. Proxy (nginx) ไม่ส่ง header ครบ ทำให้ authentication fail

---

### 1\. โครงสร้างไฟล์ที่ต้องรู้

หลังจากที่เราใช้ `docker-compose` \+ mount config แล้ว  
 ไฟล์หลักที่เราจะใช้มี 3 จุด

| ไฟล์ / โฟลเดอร์              | อยู่บน Host (เครื่องคุณ)              | ใช้ทำอะไร                      |
| ---------------------------- | ------------------------------------- | ------------------------------ |
| `docker-compose.yml`         | `~/n8n-docker/docker-compose.yml`     | กำหนด service / env / volumes  |
| n8n settings file (`config`) | `~/.n8n/config`                       | บังคับค่า trustProxy และอื่น ๆ |
| nginx config                 | `/etc/nginx/sites-available/n8n.conf` | ตั้งค่า reverse proxy และ SSL  |

---

### **1\. เพิ่ม trust proxy ใน container**

สร้างไฟล์ config ให้ n8n อ่านตอน start: n8n settings file (`config`)  
เปิดไฟล์  
bash  
`mkdir -p ~/.n8n`  
`nano ~/.n8n/config`

วางโค้ดนี้:  
js  
`module.exports = {`  
 `generic: {`  
 `express: {`  
 `trustProxy: true`  
 `}`  
 `}`  
`}`  
Save File เวลาแก้ไฟล์ด้วย `nano` ให้กดตามนี้:

1. พิมพ์/วางโค้ด
2. **กด `CTRL + O`** (ตัว O ไม่ใช่เลขศูนย์) เพื่อบันทึก
3. กด `Enter` เพื่อยืนยันชื่อไฟล์
4. **กด `CTRL + X`** เพื่อออกจาก nano

---

### **2\. แก้ `docker-compose.yml`**

เพิ่ม 2 service: main กับ runner  
 และ mount config เข้า main \+ runner  
bash  
`cd ~/n8n-docker`  
`nano docker-compose.yml`

yaml  
`version: '3.3'`

`services:`  
 `n8n:`  
 `image: n8nio/n8n:latest`  
 `restart: always`  
 `environment:`  
 `- N8N_HOST=n8n.thho.me`  
 `- N8N_PROTOCOL=https`  
 `- WEBHOOK_TUNNEL_URL=https://n8n.thho.me/`  
 `- N8N_RUNNERS_ENABLED=true`  
 `- N8N_RUNNERS_LICENSE_CERT_AUTO_RENEW=true`  
 `ports:`  
 `- "5678:5678"`  
 `volumes:`  
 `- n8n_data:/home/node/.n8n`  
 `- ~/.n8n/config:/home/node/.n8n/config`  
 `networks:`  
 `- n8n_network`

`runner:`  
 `image: n8nio/n8n:latest`  
 `restart: always`  
 `environment:`  
 `- N8N_RUNNERS_ENABLED=true`  
 `- N8N_RUNNERS_HOST=n8n`  
 `- N8N_RUNNERS_PROTOCOL=https`  
 `- N8N_RUNNERS_AUTH_BASIC=true`  
 `volumes:`  
 `- ~/.n8n/config:/home/node/.n8n/config`  
 `depends_on:`  
 `- n8n`  
 `networks:`  
 `- n8n_network`

`networks:`  
 `n8n_network:`

`volumes:`  
 `n8n_data:`

Save File เวลาแก้ไฟล์ด้วย `nano` ให้กดตามนี้:

1. พิมพ์/วางโค้ด
2. **กด `CTRL + O`** (ตัว O ไม่ใช่เลขศูนย์) เพื่อบันทึก
3. กด `Enter` เพื่อยืนยันชื่อไฟล์
4. **กด `CTRL + X`** เพื่อออกจาก nano

---

### **3\. อัปเดต nginx ให้ส่ง header ครบ**

แก้ `/etc/nginx/sites-available/n8n.conf`:  
bash  
`sudo nano /etc/nginx/sites-available/n8n.conf`

nginx  
`location / {`  
 `proxy_pass http://127.0.0.1:5678;`  
 `proxy_http_version 1.1;`  
 `proxy_buffering off;`  
 `proxy_cache off;`

    `proxy_set_header Upgrade $http_upgrade;`
    `proxy_set_header Connection "upgrade";`

    `proxy_set_header Host $host;`
    `proxy_set_header X-Real-IP $remote_addr;`
    `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`
    `proxy_set_header X-Forwarded-Proto $scheme;`

    `proxy_read_timeout 86400;`
    `proxy_connect_timeout 60;`
    `proxy_send_timeout 60;`

`}`  
Save File เวลาแก้ไฟล์ด้วย `nano` ให้กดตามนี้:

1. พิมพ์/วางโค้ด
2. **กด `CTRL + O`** (ตัว O ไม่ใช่เลขศูนย์) เพื่อบันทึก
3. กด `Enter` เพื่อยืนยันชื่อไฟล์
4. **กด `CTRL + X`** เพื่อออกจาก nano

---

### **4\. Restart ทั้งระบบ**

bash  
CopyEdit  
`sudo docker-compose down -v`  
`sudo docker-compose up -d --build`  
`sudo systemctl restart nginx`

---

## \*\*\*ชุดคำสั่ง **พร้อมรัน** ได้เลย\*\*\*

## **สคริปต์ติดตั้ง & เซ็ตค่า n8n \+ nginx ครบวงจร**

รันทีเดียว ทำครบทุกอย่าง

### **วิธีใช้**

1\. บันทึกโค้ดนี้เป็นไฟล์ เช่น `setup-n8n.sh`  
 bash  
`nano setup-n8n.sh`

---

bash  
`#!/bin/bash`

`# 1. สร้างโฟลเดอร์และเข้าไป`  
`mkdir -p ~/n8n-docker`  
`cd ~/n8n-docker`

`# 2. สร้างไฟล์ docker-compose.yml`  
`cat > docker-compose.yml << 'EOF'`  
`version: '3.3'`

`services:`  
 `n8n:`  
 `image: n8nio/n8n:latest`  
 `restart: always`  
 `environment:`  
 `- N8N_HOST=n8n.thho.me`  
 `- N8N_PROTOCOL=https`  
 `- WEBHOOK_TUNNEL_URL=https://n8n.thho.me/`  
 `- N8N_RUNNERS_ENABLED=true`  
 `- N8N_RUNNERS_LICENSE_CERT_AUTO_RENEW=true`  
 `ports:`  
 `- "5678:5678"`  
 `volumes:`  
 `- n8n_data:/home/node/.n8n`  
 `- ~/.n8n/config:/home/node/.n8n/config`  
 `networks:`  
 `- n8n_network`

`runner:`  
 `image: n8nio/n8n:latest`  
 `restart: always`  
 `environment:`  
 `- N8N_RUNNERS_ENABLED=true`  
 `- N8N_RUNNERS_HOST=n8n`  
 `- N8N_RUNNERS_PROTOCOL=https`  
 `- N8N_RUNNERS_AUTH_BASIC=true`  
 `volumes:`  
 `- ~/.n8n/config:/home/node/.n8n/config`  
 `depends_on:`  
 `- n8n`  
 `networks:`  
 `- n8n_network`

`networks:`  
 `n8n_network:`

`volumes:`  
 `n8n_data:`  
`EOF`

`# 3. สร้าง config file สำหรับ n8n`  
`mkdir -p ~/.n8n`  
`cat > ~/.n8n/config << 'EOF'`  
`module.exports = {`  
 `generic: {`  
 `express: {`  
 `trustProxy: true`  
 `}`  
 `}`  
`}`  
`EOF`

`# 4. แก้ nginx config`  
`sudo tee /etc/nginx/sites-available/n8n.conf > /dev/null << 'EOF'`  
`server {`  
 `server_name n8n.thho.me;`

    `listen 443 ssl http2;`
    `ssl_certificate /etc/letsencrypt/live/n8n.thho.me/fullchain.pem;`
    `ssl_certificate_key /etc/letsencrypt/live/n8n.thho.me/privkey.pem;`
    `include /etc/letsencrypt/options-ssl-nginx.conf;`
    `ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;`

    `add_header Strict-Transport-Security "max-age=31536000" always;`

    `location / {`
        `proxy_pass http://127.0.0.1:5678;`
        `proxy_http_version 1.1;`
        `proxy_buffering off;`
        `proxy_cache off;`

        `proxy_set_header Upgrade $http_upgrade;`
        `proxy_set_header Connection "upgrade";`

        `proxy_set_header Host $host;`
        `proxy_set_header X-Real-IP $remote_addr;`
        `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`
        `proxy_set_header X-Forwarded-Proto $scheme;`

        `proxy_read_timeout 86400;`
        `proxy_connect_timeout 60;`
        `proxy_send_timeout 60;`
    `}`

`}`

`server {`  
 `listen 80;`  
 `server_name n8n.thho.me;`  
 `return 301 https://$host$request_uri;`  
`}`  
`EOF`

`# 5. รีสตาร์ท docker-compose และ nginx`  
`sudo docker-compose down -v`  
`sudo docker-compose up -d --build`  
`sudo systemctl restart nginx`

`echo "ติดตั้งและเซ็ต n8n + nginx เสร็จแล้ว! เข้าใช้งานได้ที่ https://n8n.thho.me"`

---

2\. วางโค้ด → กด `CTRL+O` → Enter → `CTRL+X`

3\. ให้สิทธิ์รัน  
 bash  
`chmod +x setup-n8n.sh`

4\. รัน  
 bash  
`./setup-n8n.sh`

#### ข้อดีของสคริปต์นี้คือ **แม้เครื่องพัง / ลบคอนฟิก ก็รันใหม่แล้วได้ระบบเหมือนเดิม 100%**

---

✅ หลังทำ Step 9 นี้:

- ปัญหา `trust proxy` จะหาย เพราะเราบังคับผ่าน config file
- Task Runner จะต่อกับ main ได้จริง
- 403 จะไม่โผล่อีก (ถ้า URL และ Auth ถูก)
- เข้าหน้าเว็บและใช้งาน workflow พร้อม runner ได้

---

## **คู่มือ Cloudflare Tunnel สำหรับ n8n**

### **1\. สิ่งที่ต้องเตรียมก่อนเริ่ม**

- **บัญชี Cloudflare** (สมัครฟรีที่ https://dash.cloudflare.com/sign-up)
- โดเมนที่คุณเป็นเจ้าของ (เช่น `thho.me`) และชี้ Nameserver ไปยัง Cloudflare แล้ว
- เครื่องเซิร์ฟเวอร์ที่ติดตั้ง **n8n** ไว้ (Docker หรือแบบปกติก็ได้)
- สิทธิ์ SSH เข้าไปที่เครื่องเซิร์ฟเวอร์ได้

---

### **2\. เพิ่ม DNS Records ใน Cloudflare**

1. เข้า **Cloudflare Dashboard** → เลือกโดเมน `thho.me`
2. ไปที่เมนู **DNS**
3. เพิ่ม 2 Record นี้:

   - **Root domain** → ให้ผู้ใช้เข้า `thho.me`

     - Type: `A`
     - Name: `@`
     - IPv4 address: `127.0.0.1` _(หรือเว้นไว้ถ้าใช้ Tunnel แบบเต็ม)_
     - Proxy status: ☁️ **Proxied** (ส้ม)

   - **Subdomain สำหรับ n8n**
     - Type: `CNAME`
     - Name: `n8n`
     - Target: `your-tunnel-id.cfargotunnel.com` (เราจะได้ค่านี้ตอนสร้าง Tunnel)
     - Proxy status: ☁️ **Proxied** (ส้ม)

## 💡 **หมายเหตุ:** ถ้ายังไม่มีค่า `your-tunnel-id.cfargotunnel.com` ให้เว้นว่างไว้ก่อน แล้วไปทำขั้นตอน **สร้าง Tunnel**

### **3\. สร้าง Cloudflare Tunnel**

บนเซิร์ฟเวอร์ (ผ่าน SSH):  
bash  
`# ติดตั้ง cloudflared`  
`sudo mkdir -p /etc/cloudflared`  
`sudo apt update && sudo apt install -y cloudflared`

จากนั้นรัน:  
bash  
`sudo cloudflared tunnel login`

- จะเปิด Browser ให้คุณล็อกอิน Cloudflare และเลือกโดเมน `thho.me`

สร้าง Tunnel:  
bash  
`sudo cloudflared tunnel create n8n-tunnel`

- ระบบจะสร้าง Tunnel ID และไฟล์ `.json` เก็บไว้ที่ `/etc/cloudflared/`

---

### **4\. ตั้งค่า Tunnel ให้ชี้ไปที่ n8n**

สร้างไฟล์ `/etc/cloudflared/config.yml`:  
bash  
`sudo nano /etc/cloudflared/config.yml`

ใส่:  
yaml  
`tunnel: YOUR-TUNNEL-ID`  
`credentials-file: /etc/cloudflared/YOUR-TUNNEL-ID.json`

`ingress:`  
 `- hostname: n8n.thho.me`  
 `service: http://localhost:5678`  
 `- service: http_status:404`

## แทนที่ `YOUR-TUNNEL-ID` ด้วยค่า ID จากตอนสร้าง Tunnel

### **5\. ผูก Tunnel กับ DNS อัตโนมัติ**

รัน:  
bash  
`sudo cloudflared tunnel route dns n8n-tunnel n8n.thho.me`

## Cloudflare จะสร้าง DNS record ประเภท CNAME ให้ชี้ไปที่ Tunnel ID ของคุณ

### **6\. เริ่มรัน Tunnel**

bash  
`sudo cloudflared tunnel run n8n-tunnel`

ถ้าต้องการให้รันอัตโนมัติทุกครั้งที่เครื่องเปิด:  
bash  
`sudo cloudflared service install`

---

### **7\. ทดสอบ**

เปิด Browser เข้า:  
arduino  
`https://n8n.thho.me`

## ถ้า n8n รันอยู่ในพอร์ต `5678` ก็จะเข้าได้ทันที โดยไม่ต้องเปิดพอร์ตใน Firewall และไม่ต้องใช้ Public IP

📌 **สรุปขั้นตอนสำคัญ**

1. สมัคร Cloudflare และเพิ่มโดเมน
2. สร้าง Tunnel → ได้ Tunnel ID
3. ตั้งค่า `config.yml` ให้ชี้ไป `http://localhost:5678`
4. ผูก DNS record ไปยัง Tunnel ID
5. รัน `cloudflared tunnel run`

---

## Launching the Tunnel (Finally\!)

![][image2]  
![][image3]  
![][image4]  
![][image5]  
![][image6]  
![][image7]  
![][image8]

---

##

### Run cloudflared as a system service for reliability.

Install the Service: Registers with systemd.  
sudo cloudflared service install

Enable and Start:  
sudo systemctl enable \--now cloudflared

Check Status:  
systemctl status cloudflared

See if it’s running.  
View Logs:  
journalctl \-u cloudflared \-f

# **ส่วนเพิ่มเติม ระบบดูแลอัตโนมัติ**

---

## **1️⃣ ตั้ง timezone ของ VM ให้เป็นเวลาไทย**

bash  
`sudo timedatectl set-timezone Asia/Bangkok`

ตรวจสอบว่าเปลี่ยนแล้ว:  
bash

`timedatectl`  
ถ้าขึ้น `Time zone: Asia/Bangkok` แสดงว่าพร้อมแล้ว

---

#

# **โหมดที่ 6: Full Maintenance (Backup \+ Update \+ Renew SSL \+ Restart Nginx \+ Log Check)**

ในสคริปต์นี้ผมได้เพิ่มให้:

- ✅ แก้ Permission Warning อัตโนมัติ (`N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`)
- ✅ เปิด Task Runner (`N8N_RUNNERS_ENABLED=true`)
- ✅ Backup ก่อนทำงานทุกครั้ง และลบ Backup เก่าเกิน 7 วัน
- ✅ Restart Nginx อัตโนมัติหลังทำงาน
- ✅ เลือกได้ว่าจะ Full Reset, Restart+Backup หรือ Restore ล่าสุด

---

### **วิธีใช้งาน**

bash  
CopyEdit  
`nano ~/n8n-docker/manage-n8n.sh`  
`# วางโค้ดด้านบน`  
`chmod +x ~/n8n-docker/manage-n8n.sh`  
`~/n8n-docker/manage-n8n.sh`

---

**สิ่งที่ผมจะปรับในสคริปต์เดิม**

1. เพิ่ม `docker compose` แทน `docker-compose` (เพราะในบางระบบ version ใหม่ใช้ `docker compose`)  
   แต่จะเช็กก่อน ถ้าไม่มี จะ fallback กลับไปใช้ `docker-compose`

2. เพิ่ม `set -euo pipefail` เพื่อให้สคริปต์หยุดทันทีถ้ามี error และใช้ตัวแปรที่ไม่ถูกกำหนดไม่ได้

3. เพิ่ม log timestamp ให้ทุกการทำงาน เพื่อเวลา restore หรือ debug จะได้ง่าย

4. ปรับ `restore_backup` และ `restore_latest` ให้ restart n8n \+ maintenance_tasks อัตโนมัติหลัง restore

---

นี่คือสคริปต์ที่ **ปรับปรุงแล้ว** \+ **ระบบรัน Full Maintenance ทุกสัปดาห์เวลา 03:00 น. ตามเวลาไทย**  
bash  
CopyEdit  
`#!/bin/bash`  
`set -euo pipefail`

`# ===== CONFIG =====`  
`BACKUP_DIR="$HOME/n8n-backups"`  
`VOLUME_NAME="n8n-docker_n8n_data"`  
`IMAGE_NAME="n8nio/n8n:latest"`  
`DOCKER_COMPOSE_FILE="$HOME/n8n-docker/docker-compose.yml"`  
`LOG_TAG="[n8n-maintenance $(date '+%Y-%m-%d %H:%M:%S')]"`

`# ===== Detect docker compose command =====`  
`if command -v docker compose &>/dev/null; then`  
 `DC_CMD="docker compose"`  
`else`  
 `DC_CMD="docker-compose"`  
`fi`

`log() {`  
 `echo "$LOG_TAG $1"`  
`}`

`backup_data() {`  
 `mkdir -p "$BACKUP_DIR"`  
 `BACKUP_FILE="$BACKUP_DIR/n8n-backup-$(date +%Y%m%d-%H%M%S).tar.gz"`  
 `log "📦 กำลัง Backup..."`  
 `docker run --rm -v "$VOLUME_NAME":/data -v "$BACKUP_DIR":/backup alpine \`  
 `tar czf "/backup/$(basename "$BACKUP_FILE")" -C /data .`  
 `log "✅ Backup เสร็จ: $BACKUP_FILE"`

    `find "$BACKUP_DIR" -type f -name "*.tar.gz" -mtime +7 -exec rm {} \; || true`

`}`

`restore_backup() {`  
 `echo "📂 เลือกไฟล์ Backup ที่ต้องการกู้:"`  
 `select FILE in "$BACKUP_DIR"/*.tar.gz; do`  
 `if [ -n "$FILE" ]; then`  
 `log "♻️ กำลังกู้ข้อมูลจาก $FILE"`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" down`  
 `docker run --rm -v "$VOLUME_NAME":/data -v "$FILE":/backup.tar.gz alpine \`  
 `sh -c "rm -rf /data/* && tar xzf /backup.tar.gz -C /data"`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" up -d`  
 `maintenance_tasks`  
 `break`  
 `fi`  
 `done`  
`}`

`restore_latest() {`  
 `FILE=$(ls -t "$BACKUP_DIR"/*.tar.gz 2>/dev/null | head -n 1 || true)`  
 `if [ -n "$FILE" ]; then`  
 `log "♻️ กำลังกู้ข้อมูลจาก $FILE"`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" down`  
 `docker run --rm -v "$VOLUME_NAME":/data -v "$FILE":/backup.tar.gz alpine \`  
 `sh -c "rm -rf /data/* && tar xzf /backup.tar.gz -C /data"`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" up -d`  
 `maintenance_tasks`  
 `else`  
 `log "❌ ไม่พบไฟล์ Backup"`  
 `fi`  
`}`

`generate_compose() {`  
`cat << YAML > "$DOCKER_COMPOSE_FILE"`  
`version: '3.9'`  
`services:`  
 `n8n:`  
 `container_name: n8n`  
 `image: $IMAGE_NAME`  
 `restart: always`  
 `ports:`  
 `- "5678:5678"`  
 `environment:`  
 `- N8N_TRUST_PROXY=true`  
 `- N8N_BASIC_AUTH_ACTIVE=false`  
 `- N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true`  
 `- N8N_RUNNERS_ENABLED=true`  
 `volumes:`  
 `- n8n_data:/home/node/.n8n`

`volumes:`  
 `n8n_data:`  
`YAML`  
`}`

`maintenance_tasks() {`  
 `log "🔄 Restart nginx..."`  
 `sudo systemctl restart nginx`  
 `log "🔑 Renew SSL..."`  
 `sudo certbot renew --quiet`  
 `log "📜 ตรวจสอบ Log n8n..."`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" logs --tail=30 n8n || true`  
`}`

`# ===== MENU =====`  
`if [[ "${1:-}" == "--auto" ]]; then`  
 `log "⚙️ รันโหมด Full Maintenance อัตโนมัติ"`  
 `backup_data`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" down`  
 `docker pull "$IMAGE_NAME"`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" up -d`  
 `maintenance_tasks`  
 `exit 0`  
`fi`

`echo "========= Manage n8n ========="`  
`echo "1. Full Reset + Backup"`  
`echo "2. Restart + Backup"`  
`echo "3. Restore เลือกไฟล์"`  
`echo "4. Restore ล่าสุด"`  
`echo "5. Update + Backup"`  
`echo "6. Full Maintenance (Backup + Update + Renew SSL + Restart nginx + Log)"`  
`read -p "เลือกหมายเลข: " choice`

`case $choice in`  
 `1)`  
 `backup_data`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" down -v`  
 `docker pull "$IMAGE_NAME"`  
 `generate_compose`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" up -d`  
 `maintenance_tasks`  
 `;;`  
 `2)`  
 `backup_data`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" restart`  
 `maintenance_tasks`  
 `;;`  
 `3)`  
 `restore_backup`  
 `;;`  
 `4)`  
 `restore_latest`  
 `;;`  
 `5)`  
 `backup_data`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" down`  
 `docker pull "$IMAGE_NAME"`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" up -d`  
 `maintenance_tasks`  
 `;;`  
 `6)`  
 `backup_data`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" down`  
 `docker pull "$IMAGE_NAME"`  
 `$DC_CMD -f "$DOCKER_COMPOSE_FILE" up -d`  
 `maintenance_tasks`  
 `;;`  
 `*)`  
 `log "❌ ตัวเลือกไม่ถูกต้อง"`  
 `;;`  
`esac`

---

### **ติดตั้งระบบ Auto Run ทุกสัปดาห์เวลา 03:00 น. (เวลาไทย)**

1\. เปิด cron:  
bash  
`crontab -e`

2\. กดเลข 1  
3\. เพิ่มบรรทัดนี้:  
bash  
`0 3 * * 1 /bin/bash $HOME/n8n-docker/manage-n8n.sh --auto >> $HOME/n8n-docker/auto-maintenance.log 2>&1`

`4.` กด Ctrl+O \> กดปุ่ม Enter \> กด Ctrl+X

- `0 3 * * 1` \= ทุกวันจันทร์เวลา 03:00 (เวลาไทย)

- `--auto` \= ให้สคริปต์รัน Full Maintenance แบบไม่ถาม

- log จะถูกเก็บไว้ที่ `auto-maintenance.log`

---

ตอนนี้คุณจะมี **สคริปต์ดูแลระบบครบ** และระบบจะรันเองทุกสัปดาห์ ผมแนะนำให้คุณรัน: `~/n8n-docker/manage-n8n.sh`  
สักรอบทดสอบก่อน ว่าทำงานได้ครบ

---

### **ตรวจสอบว่าเพิ่มสำเร็จ**

bash  
`crontab -l`

คุณจะเห็นบรรทัด:  
bash  
`0 3 * * 0 /bin/bash /home/thongtrachu_t/n8n-docker/manage-n8n.sh <<< '6' >/dev/null 2>&1`

---

##

## **ลบ container / image / volume / network ที่ไม่ใช้งาน** ออกทั้งหมดได้ด้วยคำสั่งนี้:

bash  
`# หยุด container ที่รันอยู่ทั้งหมด`  
`sudo docker stop $(sudo docker ps -aq) 2>/dev/null || true`

`# ลบ container ทั้งหมดที่หยุดแล้ว`  
`sudo docker rm $(sudo docker ps -aq) 2>/dev/null || true`

`# ลบ image ที่ไม่ถูกใช้งาน`  
`sudo docker image prune -a -f`

`# ลบ volume ที่ไม่ถูกใช้งาน`  
`sudo docker volume prune -f`

`# ลบ network ที่ไม่ถูกใช้งาน`  
`sudo docker network prune -f`

`# ตรวจสอบพื้นที่หลังทำความสะอาด`  
`sudo docker system df`

ถ้าจะให้จบในคำสั่งเดียวแบบรวมกัน (หยุด-ลบ-เคลียร์-ตรวจสอบ) ก็ทำได้แบบนี้:  
bash  
`sudo docker stop $(sudo docker ps -aq) 2>/dev/null || true && \`  
`sudo docker rm $(sudo docker ps -aq) 2>/dev/null || true && \`  
`sudo docker image prune -a -f && \`  
`sudo docker volume prune -f && \`  
`sudo docker network prune -f && \`  
`sudo docker system df`

---

# **เพิ่มสิทธิ์ให้ user ปัจจุบันใช้ Docker ได้** (ไม่ต้องพิมพ์ sudo ทุกครั้ง)

bash  
`sudo usermod -aG docker $USER`

## \*\*แล้ว **ออกจาก SSH และเข้าใหม่** เพื่อให้สิทธิ์มีผล\*\*

### **คำสั่งรีเซ็ตให้กลับมาใช้ image และคำสั่งเริ่มต้นที่ถูกต้อง**

bash  
`cd ~/n8n-docker`  
`sudo docker-compose down -v`  
`sudo docker pull n8nio/n8n:latest`  
`sudo docker-compose up -d`  
`sudo docker-compose logs -f n8n`

---

\*\* ตรวจ Log \*\*

## `sudo docker-compose logs -f n8n`

**สิ่งที่ผมจะปรับในสคริปต์เดิม**

1. เพิ่ม `docker compose` แทน `docker-compose` (เพราะในบางระบบ version ใหม่ใช้ `docker compose`)  
   แต่จะเช็กก่อน ถ้าไม่มี จะ fallback กลับไปใช้ `docker-compose`

2. เพิ่ม `set -euo pipefail` เพื่อให้สคริปต์หยุดทันทีถ้ามี error และใช้ตัวแปรที่ไม่ถูกกำหนดไม่ได้

3. เพิ่ม log timestamp ให้ทุกการทำงาน เพื่อเวลา restore หรือ debug จะได้ง่าย

4. ปรับ `restore_backup` และ `restore_latest` ให้ restart n8n \+ maintenance_tasks อัตโนมัติหลัง restore

---

[image1]: data:image/png;base64,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
[image2]: data:image/png;base64,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
[image3]: data:image/png;base64,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
[image4]: data:image/png;base64,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
[image5]: data:image/png;base64,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
[image6]: data:image/png;base64,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
[image7]: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAloAAAFnCAYAAAB+V+cEAABMQ0lEQVR4Xu3dB5QU5brvf8/Za+9z193nnPs/9xx3vBh2VlFJRoIgSlJJkkWiBJUkIBkJSs4ZJKigBIkSBMk5S8455yEMYQZmhufP8/Z+i+rq7pmGmZoA389az6qqt6qrq6trun5TPVPvI88//7zky5ePoiiKoiiKSuN6xNuQWerNtypLg55bpcXIKxRFURRFUakuzRWaL7yZw8/KlEFLd0b+/AVC2imKoiiKolJTmi80Z3jb/apMF7TS88VTFEVRFPVwVnrljUwVtNLrRVMURVEUlTWrQIECsmbNGlPeeeGqcuXIXxUmlzteeukladSokSQmJsqUKVNC5kdbYYNWu3btQtoiVc2aNUPawtXUqVOlX79+MnHixJB5tmq1nuGML1q0SI4fPx6yzL3W4MGDQ9q0tm7dGtKmdezYsZA277oOHDggkyZNCpl/rzV06NCg6eXLl5uhfUNXrVolZcqUMePuA2rmzJlOW+HCheX777+XOXPmOPObN28uq1evlvnz5zvLvfvuu2EPTF2uatWqYee5S+flz59fSpUq5SxnH1O7dm356aefnLYJEyYEPXblypWyYsUKZ36zZs2kd+/eznp0G5YsWSIFCxY0bTq0267lfqwO9YfMu31a/fv3D2nz1rx580La7rXq168f0uYu737U99GODxw4UJo0aWLG9Rj3PpaiKOphrs6dOzvj1atXD5mvVbp0aef8Y0vPg97lbOk5xttmy5073PXoo4/K66+/Lt27d5fbt29LUlJSyDLecp+33BUStDRk6QnV255c6YnW2xapvCchdxUqXNwZb9WqlaxduzZkGXd9/fXX8tZbb5lAplWnTh358ssvnfk7d+50wtp3333ntPXp08e0b9++3Vn28OHD5vk0aOk2Hjp0yJk3ZswY5zlat24tAwYMMEFLp9evXy/Tp0834UIf6w6HI0eOlFGjRpmgoO0tW7Y0gUJP9nv27DHPM23aNBk0aJA0btxYPv/8c+eg0bBhA5WWfQP15GzbNZjptn366afmMRpodFqDlj7eHlxFixaVIUOGmHFdToOK7ju7bvueVKpUyQz1tem6tH3ZsmVBy3nDzmuvvWaeRwOYTtsgPXv2bBMqdHzhwoXmdetj9LcD3W5dv12vlvsY0kCu76v3ObX0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lNdff92Ev+XLl0ujRo1M0BowYIC5I/uaNWtM0NLtKVasmPO4+33ehIQE85rLlCnjnZXs+tI8aAEAgAebBg530FLVqlUzQUvpFa9JkyaZcQ1akUQKJ9Fo2LChzJgxQxYvXmym3UFLw9c777zjXNHybuv9PK99zeFeuwbKSEKCVo4cOeSRRx6RbNmyybRp0yQuLs6066U6bddSdqj++Mc/mml7uc69nJu2nTlzxhnX0st/9vEAACBzu3DhgjOuV3ncNGjdunXLmb59+7Zrrj8SExPNUAOWm4atzCAk3bgDz/Tp052gZdsLFy4cspwGJXebDn/961878y1Nmu5lXnjhBRk9ejRBCwAAPJBC0k2VKlVM6NHLYnpFy1552rVrlzOu3OM2KF27ds29KsO93JIlS+Txxx932hs0aGDWS9ACAAAPItINAACATwhaAAAAPiFoAQAA+ISgBQAA4BOCFgAAgE8IWgAAAD4haAEAAPiEoAUAAOATghYAAIBPCFoAAAA+IWgBAAD4hKAFAADgk4hBKynptrcprH/7t38zHUKXLVv2njuGnjt3rrcpyPLly71NQezj8+TJIyNGjJDf/va3niUAAAAyTsRktOLnC96miGzA0uFjjz3mjP/mN78x4zly5DDTN27ckF//+tcmGBUoUECyZ88u06ZNk0cffVTy588v//7v/+48VkuDls576qmnpEqVKmbek08+aYa6Ln18XFycmX766afN0D7+hRdekIsXL5rHurclWrt375Zy5cp5mwEAAKIWMWjlez/5q0lu7qBllS9f3hkfPny43Lx5U1q0aGGmixYtaoZ6RUqDltLHrlu3zoxXqFDBDO0VLZ333nvvmXEbtJQ+ftWqVXL79m3ZsGGDadPgFR8fL2vWrDFBS9mQpctGS4PWlStXvM0AAABRixi01O5DV71NYYULWg0aNJD/+I//MOM2aKm2bdtKmzZtzPivfvUrJ2i1bt1apk+fbsbr168vxYsXDwpadqhBS9dtH6/c4UuNGjXKPF+4oPWLX/xCChcubKb1yhoAAIBfkg1afnjiiSekadOm3uY0pQGqRIkS3mYAAIB0le5B62Fy7Uait8kYP+e4GRapu9oM2w3aZf75QId2XrF6q6VS8w3SoMtWmbXktPPYV9+7+5WuLrNwzTlpO2CXmbZf9w6ZcEgSE29L6UbrJP5mknQbtc95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
[image8]: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAloAAAHNCAYAAADG0NWMAAB2AElEQVR4XuydB5gUx5n+bdn+353t851Plu8sny377uSgYBsBAomcRBZZ5CgyIieRRM455yxyBpFzjkvOaQlLXmDZhV02UX/eGn29NTWzO7PLDswu7+95vqeqv6ru6e7p8E51z/f94J0//EHlzp2bRqPRaDQajZbO9gPbEUxWpGRV1WbsTdVh8iMajUaj0Wi0dLP2kx6qSg0GeWiP9LagFVrYCXny5PXw02g0Go1Go6WXfTX4uKrdfr6HP70sKIVW4z57PHw0Go1Go9FogbAChYp7+NLLgk5oYSTL9tFoNBqNRqMF0jCyZfvEPvjgA1WgQAEVGxurEhISPNpTsmSF1t69ez18yVmXLl08fN4My1yxYoUaNmyYR5tYvU7LnfrmzZtVjx49PPr4Yw0aNPDw+WPdu3dXISEhasaMGR5tabHhw4c79d27dzv1du3aqc6dO7vtZ6k3btxYVa9eXbVv316VKlVKjRo1SvuxP1AuXrzYrX+nTp10vW7durq0vzuZ9tYmVrRoUadtz549auvWrfpz4evYsaP69ttv1ZAhQ5zloL/9GUuWLHGWU6RIEbVx40a3z/7666/1/jA/x5y/fPnyTtvatWv1d2F+xrp169ym/bGaNWvqUvadbeY6+Gv47mwfjUaj0V6N9erVy6nXrl3bo922gkVKe/jEvL2zVahQIfXXv/5V1alTRy1YsEAlJiZqs/uZhnt3yZIldd2r0EruZpyS+RJbuAmjTOlmDzOH73BjLlu2rEcfMQii2bNn6zpuzNeuXVNXrlzRJYTW8uVJou369evq8uXLut6sWTM1adIkNW3aNL0zFi1apHfIypUr9c0dfSG08ubNq/thHizzs88+c7vZyzKLFy+uli5d6vjQ98yZM06/TZs2OduNviNGjNA364ULF6qdO3fqPuZ+ad26tRZaqJcuXVrt2LFDC87evXu79YO6LlGihK43bdrU+bxBgwZpAQFhV65cOWf5KHft2qUKFy7siF18vrnMuXPn6vLLL7/UZf/+/XUJgYT9hPqWLVucZco+xjZhesCAAap+/fr6wO/Xr59u69Onj9s65M+fX5c4YFGagkwM6y/7BuuEafRZv3699mEbZRvgx7LkWEBfWJ48efQ0hCmEFvrNnDlT+yFO8V2vXr1a+1HHMps0aeJ8FvYBfLJMfHc4ZvDdYT/Onx+4Z/o0Go1G889MoYVrM67rdh9YlSYjVZO++/STs5qtZ6oSFRp79MEf8Gwf7gvvvvuu+uKLL1Tz5s1VfHy8evbsmUe/5Myr0MLNVW5S/lqtWrU8fKaZQsy+qZpWrEwdp96wYUN9A4cgkZuyaRA19jRGUo4ePeqMaEFUoJwwYYLeSahDuKAMDQ3VozX4YsaOHat9ptAaM2aMs2xsn4wsickyR44cqaerVq2qJk+erM6fP++2PyAsRCSJ+MDNWkZTIJggitCOEiZtEFoQIrIsiAasL8QYpiF0RPjAZN9ChGAbRNSsWbPG6SPCBwaBaY5m9ezZUwtMWc8aNWroEiNcWCcRYNgXIoQqVqyoS+xDWQ7KNm3a6M+XUSX4cJDKZ8t3is9DG0oRmDj+zFEotNsjWtgP0mb6u3Xrpj/XHK0SoYU6fp2gXLZsmcf8FSpU0NuF70uEJaxKlSpu3x2OK/l8Go1Go706M4WWt2nT5B+HuZP5s91Xg455+HBPgbbAwMXo0aP1YMq5c+c8+iVnXoUWzLwZ+zJfIksMogajO7bftLbjbnn4YCJmbJNHWHjsZLeJ4QZu+2SkJ1++fI4Pgie5fmIygmSbCDrThg4d6uGDff75507dXn5qzHx8J0OUYuY2e/t+7Ed/MIgJlCLOkrPKlSt7+Mx5ZDmmiWDzZraoF7GVkkGAojS/P9tk9EuElWneRkrN70VMPkfM176h0Wg02qsx+0e3N4PIajXikipbo5NHGyIdIKyU7S9Tpox67733VPbs2fWPcTzhkR/qyRleDcJTOdSTFVqvyrypSRqNRqPRaLRAWkp/xsMP9h//+Me63rZtW4/2lCzohBYspY2l0Wg0Go1GS09L6R+HL2pBKbRgGNnCY8RAxrag0Wg0Go32ehoeFSLSQaAHd4JWaNFoNBqNRqNldKPQotFoNBqNRguQUWjRaDQajUajBcgotGg0Go1Go9ECZD6FFgJ/eQv+5S02U4Fy1VX5kevUZ192UOWGr1b5ipdx2hDfAtHb7XlMSyk1j7+2fft2D58/liNHDvWXv/xF/eQnP/Fo82YSMf1FLLxxNhV56qB6uGOdulboXbc2f+KBeDMEd0UU+Vu3bulyzpw5ThsirCO2FCK3m/Mg4r29HNskfpgdkyy5uGKptdCW2dS5ZlnUucZ/f15+pC63yOa0TZ8+3aN/MBm+q1WrVuk6gpkiXRFig3n7DuFDfBXb/7KsfH7PeG8wb+tq292cuZ5b3ueWR9ftdlmGGRwX5ivNEVJNYd6BAwc6vlatWnn0exEbPHaumjB7rTbTj89Fdgi7v93H9sH8uQa0zoV95t9+Q6BcnE/btm3T0wj4i9IMnAszg/7aJsGDbTPnSW57bPNnvZFhAeuN7xzfo9lmxupD1g1JHSaZF+zlpWSp7S+W9/l5mNfy9e3bV5eyPv6YHI/IFCHrIiUCYJvXUPglmHNqDDEVcV8FyKdnt6eUfcUMyCyGQNfJnXt2X29+M7tGMJmvWIbetAkMOgYBxW2/GLYd55/c06ZOnap93vaBr2PHTh+XrNAyDyZvX4rXjXm+QtW/u6nyFi6m62YbQtgjcjgijGN5CPaFYJD16tXTQUwRBAwrJ2IMG4IDeMOGDW7LwXxYDuaBaEB0biwP8w0ePFgLLakjXQouBB7r6cV+97vfqbfeekv96Ec/Un/84x8dv2y/LBOfj+joiMYOv4Tkx/a1bNlSz4PI7ohqj5NT+tufB7tZ72/6pIrYs1Fd7+F+Y5H1xvZjm7CdOPBxoOCzkzuB7LQA5rTcUBDcVYKO4uKIE9hcnnnBFMPBZ95YsE5IOwQ/9gO+L3yH+H7lO8I6iwDxZVponTqhnjzfHzeP7VaXm/zNaWvRooUuEa1d9gsEZKNGjXRdLoLjx4/XJcQOtgl9EeleItDjBmYe13IhwnpineFLSdR9Wbu6ujEwpzbTb54fOJ4htJAlILkbsewvHN/2emAa3xPSRKGvHbwVgXGRCgiiWW7Ivix/7jyqR/EKam7ZumpBuXoe64E61hllSjk+7+T4VEWMn6vu5PyHrosfaY1QyvGP9TPnw+fIMdW1a1fHb16MsD1yjmEa3ymOUewHlDiHcL1AgFoz3ZQ/BnHVoKlLtJgXTeQtQynnF85bZDzAeuF4MNcVN2cc2zimcDxi3+M8NPt7O8+v5/zU2G8fe91vcpxgHbAMmZZYPfiOsA+xfhMnTtTHCm7s2DcQp1gfc1loR8ozmQfT5vntLTCvN0vu+xbD943PlX1ntiE4NbJzTJkyRR9bSFeF7ZB1keuqXNfkHJVzHH4zT6xkl/DHvi5aVo0uXU1Vzl9EtSlaRnUu5goKjeMIgZOxrrg+mJ8vWR5Q4tqCY1G+W/jlGgPBaO5LSdGGY0WuA7hm41hBXVKz+WM4pnBMAjsLCc4f7EuIV5wrOB7hM89X7COsg9wzJTg4AkhLhhVZf/TDukldsnCY2wYfoqHb6xkMhn1hXndN86pNvJg9P75rU2hBPyBwO/Y5vl+keMP+QYljGeci7m9yHEmqPdSxLHwPOObwvaar0CpYqa6qsea+KlLL8xcXhAhKJHzESWVGVcWBgwiqWDncSMwRF28R13FyI/0NRAc2Chsk64iLptTN1DW+DELrl7/8pXrjjTfUL37xC8ePZclFC9Oy3vhM7ED5FT5r1ixnHnwZcuNKKXrs0LI51TP8erlzQ4UV+8Djl1eHDh309st3gH0IcVmtWjWPZYmlJLTwawwJq+0o+rhomyeYN7NHrrA+uJCK0IIP+0JSGeE7si++Kdnlhh88F1i71LW5fdX5M6fcRrQgWkUMiQ+Reu1lmIaTROq4eCPqP9ZXcm7iFzgC0OECBDGGZeNXqLfjDSYiK3xJKxWxc5yKOuhKxwPD9pu/srCclH59mlkXsB64QMp64JgyR7yQKNyeX8zbeenNRGjNKVNX34Ds+b0Ja2+W3I3XzPmIG6wttLwdW7iZmccHrgHm9ojQkmkzkbe9fF82bOJC1WfoVF3/vGzSiI+ILrk4IsWS3CDNY0364dqFmxPSceA7Nm+oyZ3nKQkte79J2iiJJi3bD0GH6wluLjg+cB7LDw5zFBCGH31YhjkPsmpIzk6Mjo0bN07/ILDX1bbkvm/T5NjHOeatbd68efqmJEIL62IKLTH5ESj7UYQPDNcrrLOdgis562gILRzzdjtECq4PWB8IGjn35HNxbcE5KN8tzP4xJ4ZjwbzWwHBvQonvT1LB+WP4bvFjAtcobwmLcU2RYwNmCkQYriPfffedsy+9ZWHBsSz5bTGNH3tyb5ZlmuehPTodLJbeQgvbjWPMFlooJcUd9jWOFZQwCFjcL7APca5JTl4RWrJ87O9khZY8LsQF0Xz8JOZtY/J9VkLV2R6v6u2J92hLb/PnJo6blLeLfHKGEPt4fGj6kvsy08PyPLebtT5SNypl9WhLq0FQ4ORHHXhLKxSMdq1VdnWuWVbHMG33eVFLTkTR/LMen+ZSI7+3dp96PkoKdqtYuab6pt9oDz/MvGGlt8l+G+bnfguWmxvWVb7vHp96F1qmmSNQqTHcpAJ5nU0PS+9H2ckZhBZGSo4fD1zwTNvS8pgzmM2bNoHZ4t7fp13pYckKLRqNRqPRaDTaixmFFo1Go9FoNFqAjEKLRqPRaDQaLUBGoUWj0Wg0Go0WIPuBfmOaEEIIIYSkOxRahBBCCCEBgkKLEEIIISRAUGgRQgghhASIlyq0bo8tqW4OyuFmhBBCCAlu7t27p4OpBtIQ/T8z8tKE1sPVPT1EFsUWIYQQEvzExcXZLuInARdatqjyZoQQQgghmZFXLrTujP9cPVxRUVvEd9Xs2b3yfpnNqveEs6pet8O6viMk3O5CCCGEEJIuINF31apVbbdfBExo2YLq1rA86vboz9SdieXU3WlV1b3ZdVXU3um6b3z4KRWx7kstthKiwtwX5IWx8y5r+6zBbjV4+gVVsslet/ZLly6pZs2aqZkzZ6q3337brS2tnD9/3nY5/OAHrt2IrO8xMTHqwIEDVg/v4IurVs0/cQn2799vuzTy+aklrfMRQsjrzr59+3Q5cOBAN3/evHnVxYsXVaNGjdz8IEuWLLYrzVSvXl3lyJG2J0IHDx506rNnz3az1LB8+XLbpZo3b267PFi0aJGb+cvGjRvViRMnbLcma9as6smTJ7bbITQ01G07MZ0aILJgq1atcvP7s/8Cdqe1hZY3uzWioO4rI1qwyC1trCW506L/cadet+tho8WdhQsX6tIUWitXrtQngQiMPXv2qJs3b6q33nrL6WOKjx/96EfqjTfe0F8uhFZCQoKaN2+e+sUvfuH0AZhnwoQJqnPnzmrXrl36BPjLX/6i2/AC4Ztvvqmzvz969Ej7oqOjVceOHfV8ptC6ffu2+uabb9TPf/5zPd+AAQO0/9e//rWHKEJfgM9ZsmSJ48uZM6ezzhB9EHP4bHP+hw8f6ml7mYQQQlIHRJVJbGysUy9Xrpy+z8j7TSK0Tp8+rUaPHu30Swv16tXTZViYa3Biy5YtqnDhwo4AbNiwoapbt66+F545c0bt3LlTff755yoiIkItXbrUWc7atWtViRIltPnL4sWLtfXu3VuXAkTns2fPdB2C6OnTp7oO39atW51+QIQL1ju15M+fXxUsWFDf33bs2KFatGihhdbIkSN1JPa9e12DL9gXxYoVc+aT7UzNtgp37971OqLlz/4L2J029lqIT4u7e0HF3zvpZolP7tqLcuN2+FN14MQDbRVbH1Cb9t1V1TokqXPBm9CCKDEFBtQnRIkc/JiWNogy1HFg/uMf/3BGtEaNGqX9BQoU0NM4kDAdFRWlp6dOnaoPagiud999VwumPHnyqMGDB+t2MGzYMNW0aVNHaEF0AXxRM2bM0H6MjonQ+vTTT91E0aFDhxxVLb8eihQpouc31xm0bNlSbdu2Tftx0gmY/tnPfuaIP0IIIf7Trl07fZ1v0KCBm18EVK5cuVSpUqV0HWILmCNaEAYvSkhIiP7hDCBYMJAg4N6EH9sQJKBWrVrq5MmTui5CRIBYSAsLFixwm5bthPCBABTRBY4dO+bUXxTc9zCaBvEDTKGFz4QAE+xRxLRsK0YARRh6E1u+CJjQCiR4L6vX+KR3tDbtTVmcpcRvfvMb9ZOf/MR2pwkc1BiRAhgOhfghhBCSuWnSpInbNH5sC+YIlyA/cE0h8iLcv39flxAg3n48YxQrM4KnTAACy8QUWumBKbJgtWvXtrukiFehJaMnffv2VcuWLdMCwvSb5TvvvKPrv/3tb3X505/+VJcYJsSQng3mgbiROqhUqZKev3v37mrz5s1md0IIIYSQDEuyQmv16tX6kRaE1vr16/V7RXj0NmjQIDehJXUIpTp16ujhSRuILnk+a85jz2/2I4QQQkhwwDhaacer0CKEEEIIERgZPu1QaBFCCCGEBAgKLUIIIYSQAEGhRQghhBASICi0CCGEEEICBIUWIYQQQkiAoNAihBBCCAkQFFqEEEIIIQGCQosQQgghJEBQaBFCCCGEBIigF1qP46PVhjt71JpbO9NsFx9fsxfrgISfly5dUufOnfPLrl+/bi/CDST3tOfx186fP+81ASkhhBBCMiZBLbQgkGzRlFaDWLORjOdpwdu8knz7RQkNDbVdhBBCCMmA+BRaUU/i1d5jD1R8wjPHV7tziNFDqaFDh7pNv/32227TacUWS/7YorNrPXxihx6esj/C4fDhw27TiYmJbtO+8Ca8fPHo0SPb5XDr1i3bRQghhJAMhk+hdeDEAw+zhdYPfpC0GNQhtNavX6+nV65cqX74wx867WDYsGHqjTfecObzJlKSe1w4bucmp563UD7Vqn8HVbjkZ6p0lbJqyq55auTqyar3zMG6vUSFUqrdkM5u8wt4XChgHePj43UpdOrUSZd4nHf69GkVFhampyMjI50+vhg9erTq06ePKl++vBoxYoQuwZgxY9SAAQPUsWPH9LKrVKmi6tevr44cOeLMi/UhhBBCSMbGp9Dyhj9CKyoqSk8fOHBA+54+fer02bBhg/rlL3/pzBcdHe20CbbQqj94ueqzzH20qsOIbqpg8cJq0KLR6rsb29WsQ0u00EJb895tValKn7v1T05oJcf8+fPVtWvX1KRJk+wmv3ny5IkuixUr5vggtOLi4rTQwrIHDx6shZYJhRYhhBCS8UkXoeUPIjhswsPDbZcGL8HbIikttvrmDqe+7vZuZ/nmS+dLlizR5e7dSe2BFDoQWr64ePGi7SKEEEJIBsOn0Hq/zGYPm7z4it0tIPBleEIIIYRkZHwKrVcNwzsQQgghJKMS9EKLEEIIISSjQqFFCCGEEBIgKLQIIYQQQgIEhRYhhBBCSICg0CKEEEIICRAUWoQQQgghAYJCixBCCCEkQFBoEUIIIYQECAotQgghhJAAQaFFCCGEEBIgKLQIIYQQQgKET6HVvN8xNXbeZccIIYQQQoh/+BRaJss237RdbnxYoL26fTfCdhNCCCGEvJb4FFqnLkaq98ts1paS0Hrzvfq6/HbJThUbG6+u37yvjpwMddpDjnsfDStXb6iq1nS0ylKko8pRsqvdTAghhBCSYUk3odW+97e6/MPHzZ1y14GzTvuT6Ke6LFSpj+MDz549U806TVMPIh6rEZPXuLURQgghhGRkfAotk5SEFiGEEEIIcSdVQouQ1FC6dGlVokQJXZeye/fuzrT4CCGEkMwKhRYJGCtWrNDlyJEjHWElQqt27dpJHQkhhJBMCoUWCRgitBYsWOAxspWQkKBOnDjh9CWEEEIyIxRaJGBAVLVo0cKpg2bNmumyZs2aqmrVqk5fQgghJDNCoUUIIYQQEiAotAghhBBCAgSFFiGEEEJIgKDQIoQQQggJEH4JrT1H7+uyWOM9qkH3I1Zr2kAkeG9EHVnol5HgJ0uWLG6WmJhod0l38DmEEEJIsJCi0Crfcr/tSpHsxTvrEvkLhTMXbjj17kMW6RyIRSr3VafPh6n7D6PUX/O2ddpjrux3hNSIlmXUjO61PASWGPqS4AaiJyoqSterVav2UkTQy/gMQgghxF98Cq2IqDhd/6rvsRRT8CBn4c3bD3VdhNblq3ccH0BOw4KVeus6hBZYt/Wo0x51dJEjpBYPaqI6fJFTLRrYWF3bMkn7Fg5olCS2nvclwY0ptGQaYGRLRrnGjBnj1g7Lnj2748ubN6/jN/v17t3b8Ul7aGio4zt9+rTjv337tjMvIYQQ8jLxKbRMUhJaf8rdWpdFq/ZzhFatFmN12WvYEvWbfzRWDdtPVsdOXVV/ytXKEVoyCibYI1cRh+bp8lHIfBV5eIHjJ8EPRM7XX3+tunTp4iaWbNF09OhRpy9A6p7cuXOrO3fuqBw5crj1NUupS+DTjRs3eu2zfPlyp04IIYS8TFIUWq+CxCcPPMSWbdEXttqzkSAEYqdXr16qevXqHuLItM6dO+sSo6ImAwcOVNOnT3emvYkos25OT5gwwVn+3Llz3foQQgghL4ugE1ok8wCRI48OURfRZIsj8c2bN0/X8ViwWLFi6vDhw6pw4cJufcxS6tHR0boeHh7usezHjx97+AghhJCXBYUWCRim0IqMjHQEz/Hjx53RJvN9LPHZQkpMBJUtnKR99uzZTluHDh0c/1dffeXWnxBCCHlZUGgRQgghhAQICi1CCCGEkABBoUUIIYQQEiAotAghhBBCAgSFFiGEEEJIgEhRaG3Yc1e9X2azNkIIIYQQkjpSFFrjF4Tqcu+xB+pv5be4N1qUrTvEdnnwq/cb6FyH5PWgZMmS6ty5c7bbgytXrtiuFEFsrE2bNqkSJUro6WnTpukS0xUqVFA3btxQLVu2NGcJKFiXYCR75W22yyuIE5vcjynEJkst169ft11uLFmyxHYRQkimJUWhNeF7oQUOnHiQ1OAFEVrIbXj4RKiufzN4kcpSpKPRy0WVJqPUhwXaq/fzt1OzFu1w/Ks3HVYlawxQfy/UQU9v3H5cl8Wr9Xf6vEzefK++tuiYWLuJpJLGjRurVatW6XqbNm1UgwYN1Pbt27VBHHXq1EkLFqTfOXDggDNf8+bNdYmb89OnT3UdQstEhBqElogv4dSpU2rcuHH6M6ZOnap948eP1yIQbNmyRV27dk3P9+jRIx3JvlGjRurhw4c63teMGTN0v/bt26v169frOuJydezYUUeynz9/vmrWrJnrw5RrfbEdW7dudQTHwoWulFExMTF6GaNGjdLbM3nyZGc+ALEzYMp5XS/acI8uJy0KVTmqblcflN2sth9KEj25a7rOm4qtD6i1u+7o+tqdt/WPonXfT2epuFW662Vv2X9PjZh90c0HTl2MVENnJPmRAPybb77RdXw/JthPgwcPdtqxTa1atdL1nj176hLbjf22f/9+tXbtWmcZrVu3Vk2bNtXzIxhtpUqV9HEAxo4dqxYvXqy//x07duh9I98R9hm+e7TZ3y8hhAQ7KQotjGgt2XhDdR11Wg2beVGt2XFbhd2JsbtpRGgNGrfS8T2KjFaVGgx3poWcJbuqXQfOqs+q9HV8cfEJukQexA3bjqnYuHin7Um06wb7Kgh/EGm7iJ9ISp1bt26poUNd+S+9AREEA6VKlVK7d+92a0cKHSxLlidCy0zZM2fOHH0Txo0ffP7557qE0AIQPnaKH4gk8MUXX+h54+Ndx1xCgutYDAsL0+mBANrFjxyMQFL72Cl+0FfWt0aNGjqQKoCYFD9G3CAspA2I8Dl48qHqMvK0mrgwVE9/+9315+dDoqrc7qCeTrS2Y/nmmyrqiWvdkfxd6uaIliy794SzHj5/EUFbu3Ztx4dAtKbQBCIwu3bt6gijy5cv6xKCW0a06tat65rhOQcPHlT169fXYkpAu/3d4/gghJCMRIpCCyNaDx7FqdqdQxyzL/LeiIh8osuEhEQ9qpUcaDMFVfzz/ib2NMlYYARJbrRnzpxxxI+NiKxFixbpG62MenjrAyC0MNphCzJ8FkbDUOLxIRChhZEVGSGpWbOmmjJliq6LT9YTbSEhz4/zxETti4iI0H7c8EVgiNACWF8s2wSiAPPGxsbqZZmg/5EjR1RcXJyeRgR7AcLn4youcQShBTC9aust1XrgCX3u4fQzBdL63XfUh2Vdj/UxynXtVrQe/QL4gQTkPcuHkXF6dEwoUn+313cwMTKF78BG9hFG/qQdQlJGtARTaMm+ALVq1XJEGYSmKbTQx5vQkjbQr18/fUxdvJg0+kYIIcFOikIL4JGhGCGBAu8C4ZHV60yFVvttl1dkxOplc/XqVdtFCCHEBz6FFiGEEEIISRsUWoQQQgghAYJCiwScOn1jVe6mTzOsFW/3VK3Z63oRnhBCCEkNFFokYNToxbAYhBBCXm8otMhL5fD5RFWyQ6wup36X9FJ3vq+eqss3k/7R+jDKVa/YLVadver+T9dLN3z/8zUlLoa5z79qt/+jVfX6UzwSQgjxHwot8tIRgWUKLXDumksAVe8Z6wit+5FJoqjhoDi19XCiFlrLdiSo0h1j1b2IZ2rJtiSh9DhaqRq9Y1W5zrFqwLeu5e856QoTUvmbWNViRJwatyxeRTxfftfJcfqRYNE2rjhtiCZy9EKiGr88XrUaFadFno0t0gghhJCUoNAiL53khFaDQUnCplrPpHrhVi4hVKytq5QRra8nxKkZa91HoyC09pxIVLPWJSgE9DfCtGk/2hdvdc3zOEap7lPjVIn2ruVCZAGMrmHencc847hRaBFCCEkNfgmtew9iVeeRp9XTWM8bDyGp4avhcfrRIUo8PvSF/Zgw0YvOER9GqbzhLe4tAn+aIkzwlW2pVh8fHQghhBCDFIWWP1HgTbIXd6UrGT5ptdq084SKiYlTf8rdWvskmfSRk6Hqjzmaq9t3I3TU6D9/3/77bElpPD4t/Y36nxwtnOly9VzpWx5EPNZt4H9ztlBZi3bSn9G5/zynb3py/PRVbSRtbDrkReFkYFbs9P9dLkIIIQSkKLRmr7zm1H2l4IFoQkJp0GXAfJWvnCvBbKd+SSIoMipal90GLlBht+7rhM0CfMLkOZvVyg0hav8RV6oNEVoFK/XW5enzYbrE/AtW7FH37gcmH2H4gyi3dSRpo/7AjB3eoUynWPXdHoosQgghqSdFoWXzt/JbbJfDO9m/0uXUuVt1DsN85Xuqv+Ztq85ccOWcA29naaLLAhV7aaHVstsMPUolCWOv3QjX5c79Z9TYGRscAVWsWn916codnYz6yvW72tfqm5laBOWv0EvVajFW+wghhBBCggm/hNbyzTdtV5q4cv2e7UozPYcutl2EEEIIIUGFX0KLEEIIIYSkHgotQgghhJAAQaFFAsa+ffvUmjVraDQajZYJDdd44hsKLUIIIYSQAEGhRQghhBASICi0CCGEEEICBIUWIYQQQkiAoNAihBBCCAkQfgmtLBW3qqNnI2y3G2XrDtER36UOELk9NjbeLW8hIS+LmTNn2q6g4vbt27aLEEJeOdsPubK0gHsPYo0Wkhb8Elr+IOLqV+830HUke4bQWrhyr9UzY3H01BXbRV6AuXPnquPHj6uxYwOfNilnzpy2K1myZMliu5IlNX1T4vDhw7YroNSsWdN2EUKIinoSb7scDp9OeZCF+CZVQqt8y/22y0GE1u+zNdP1mQu3uyVk3rHvjFPPKOw6cFZbQmKi3UTSCIRWSEiILidOnKjmzJmj/U+fPlXZsmVT27Zt09NZs2bVgubSpUtq0aJFKn/+/OqLL77QbfA3bNhQ1xs3bqzn27JliwoNDVX58uVTpUqV0m0QWlWrVtX1HTt26PkSje8Sy3zw4IGuo23oUFfyctRFTEkd+Tjj4uIcH7hw4YIqV66croNVq1apLl26OJ9pLke259GjR2rAgAG6LkIrPDxclSxZ0lmOuc6CbBOWL8tZvXq123oPGzZMl+3atXPW96OPPlJt2rTRfm9CS5aF/rKuhJDXCwitJr2OagPthpx0chuHnH6oCtTdpfYdc10rSepJUWi9X2azqtUpRBWuv1sdOOHayWF3YqxevpHk0IRAYC1YsEAlJCRoYfL48WPtx40eogXiAZgiJSoqSgsrU2hBpIHu3bvreSC0pK1w4cK6DqFVunRpXccomi0kMF90dLReF7T17NlT+xcvXuz0RSnrdOTIEccHINJE8IGDBw+qKVOmOJ9pbkOVKlW06IGomTdvnpvQio2NVQcOHHCWY64zMB8xTpgwwRFHwFzvzZs363Ljxo26xPpmz55dC1BQokQJXZrYyyKEvH7YI1oQWrj/Awit0s32Pb9Gu3UhqSBFoRV644k6ePKhOn7edSEWsUVIegGBBcyRpjt37jj1tHL//n2nLqNW3jA/F0B4AYgOCDABI26p4eHDh7ZLI6ImOeLjXRe8lNbZG7Legj/re+vWLV36WidCCCFpJ0WhRQghhBBC0g6FFiGEEEJIgKDQIoQQQggJEKkWWg8eP1PbTyWoba+5EUIIIYT4wm+hdSfimRZZNJcRQgghhPjCb6FlC43X3QghhBBCfJFuQmvphjOq27D1KnupoR5tKVmnQWs9fBnBCCGEEEJ8ka5C6+1s3dT+E7d1CVuz46LT/t8fd9fl+l2XdVm89iRdQmih767DYR7LDGYjhBBCCPFFugotlJ+WG6mylhyishQfrKdL1Z2s1u28pMVUoarjHKGF/kVrTnCEFsxeZjAbCSx9+/Z1m/7DH/7g1H/84x8bLckjkdL9xez/gx/4fWrolDl//OMfdf3Pf/6zLmV+b8v5+uuvVUREhG7z1p4WkJrnZbFixQodod4byQVq9ZepU6fqUlIxpTcIRDtkyJBU73dJgUQIIanF76uNLTRedyPph7eb8/vvv69T9IBf/OIXWmghWvp///d/uwmtf/3Xf9XlkiVL1H/+5386fvCTn/zEbRpAkMhyIyMjdQ5Cwezftm1bpy5AIO3evVvX69dPyuMphIWF6dyLQG7kZvqc06dP6xLCDELLZuTIkTqiO7YRIL1P9erVdf1Xv/qVjhpft25dnYqoWbNmqk+fPs68P/rRj3QJIYF1/Mtf/qJ++tOfOj4AEWOmBZI22LJly3SUfLPt3/7t33SJ7Zb0R+CHP/yhFlqFChXSqY9Onjypfv/73+u2d955x+kHZHlIuyTTSDd0/vx5nQEA08jJeOzYMZ2OCftt7dq1atOmTXobwI0bN1SOHDl0/cMPP9QpkgTM/9vf/lZ98803qmXLlnr/vfnmm0773/72N6deq1YtnffRFFrIu1mkSBFdR+5LgHRJd+/e1fVf/vKXuoTQKlq0qK6DAgUKOHVCCEkJv4UWuHg70UNwvK5G0ocOHTrYLn3Dr127ttuIz+9+9zv1ySef6LqICqTZkfYZM2a4Zv4e+LEMG3OZKP/0pz85demPPIqo4+ZvA0FnIommha+++koLNvkM2Q7QqFEjtX//fp3ax5vQArJ+yNMoQMABiAzJ42gDISbiD+tojtj85je/Uf/4xz+0b9CgQVrotWrVSgsK5D/E9qIN/SBqBXMZJj//+c+dES0IrM6dO6s33nhDXb9+XR06dMjpN3nyZC1YvvvuOz0N0bhu3TrdF0AgdevWTScFF+QzMaJljmpCDEm6JlNIoX+1atWcumlg+fLler/HxLhytGKbTaG1a9cut3mxfhBRyDNpLgNCC7kjIea//fZbt88ghJCUSPWVAiKDcbQYRys9kKTKNuaNFAmYT506pW/uEAtHjx51hBZEx7lz53TdFlpdu3bVJUYhcKMFkkdQQLJqsHfvXrf+//Iv/6LrcuMVMMrm7bHl22+/rZo3b67rEFoA85YpU0bXzZEniBkRWhhZk9E1AWLywoULertLlizpPLLCPsD2itDCupgJrSG0MJqDBNhYR1MEXL16VfsgagQRCiJeOnbsqP7f//t/TiJr7M/w8HBdx2e1a9fObV5TaGHZ06ZN00KrYsWKbv38LZGQW+obNmzwKrTA//3f/+k+2DfS3xRaf//73/V+kUe4AKNxOH5AckILI3oYCYTIb9KkiR7Nw6imHF/yPaxcuVKXs2bN0sfVBx98oKcJISQ5Ui20CMlofPTRR7bLb8x3wwghhJDUQqFFCCGEEBIg0lVovfme5wvC3pgyd4vtIoQQQgjJdKSr0Hr85KkuIbiyFOmoHj56oo6fvuq0432QqXO3OoLs1p2HKuT4ZbV510l14fItNWLyGqcvIYQQQkhGJ92E1rUb4SohIVF1G7hAC6k1m4+owePdX/T9c+7WuhShNWS8699IB45c1GXJGgNcHQkhhBBCMgHpJrSSIzom1qnvOnDWaHGBUS/wKDLaaiGEEEIIydgEXGgRQgghhLyuUGiRoIQpeFLHy0rBIyly0kpqt3fPnj069pUvrly54hbzzKynF4i/ZQZkJYQQf0jdVY+QAMAUPC4ClYIHkdvBqFGjnHZsO/j888/1vh47dqwz/3/913+pa9euqbfeekv7ELwUscjMFDkAATsR6BP06NFDFSxYUNfbt2+vU9f89a9/1ZHhnzx54qTCkZQ/H3/8sQ7MKkhAUexXCTAbGxurfvazn2mhhe8J+xfbjKj5lStXdlL0YHkQuhBXpUuX1kFFzTqQbQFI/XPixAn13nvvOSl2sN/RX8C+7t+/vzMNTKE1fvx4vZ8gPMuXL699mMb2Tpo0yQmwamMG4yWEvB5QaJFXClPwuCPrl54peIYPH+4matAunw8BZApXc98AiAYIJrsdSBR8gGjqAqLDYz6A/hAlKCE67f0P8H2byzVBRHYZ0UKfXLlyOZH7TbBNMoqFfmYdEe+lDrA/kZ8REf0BBK2s1/Tp07UPoql3795O6h5gCi0Rueb2IAckIutDtAGk7gFIOYQfAdI3uW0lhGRO0vWMvxrmStnhjZkLt9su8prDFDwvJwUPkGlpB/gciDdvQgspaJA0WnySNBp1pMiROsSF1CVPoi20YPj+khNaderUUY8ePdJ1jA4h/6KA78IUWhh1kn3Sq1cvpx+2ITmhhWMG24I0QhjB8ia0kLoH+wKjaEDSFclIG/AmtLDNGNULDQ3Vjy+x3zAP8jwKly5d0iNrrVu3VhcvXtTHo/ndEUIyN+kmtGYt2qHLiMgnatnag+rm7YfqT7laqUWr9qmsRTupMnWG6NAPFesP1/3ylu2hOvSeo7buPqWKVHZ/H4eQ9OR1T8GD95xskUkIIeTlkG5CS+g/armq1nS0Gjh2pZr0bdLLxR991kk16pD0Kw/UaTnu+a/wOPXbj5IeOxBCCCGEZBbSVWj954eu4fCBY1boEa74hEQdCR5CyhZaY6ev10KrU795euSLEEIIISSzka5CKzWMmZb+f78mhBBCCAkmXpnQIoQQQgjJ7FBoEUIIIYQECAotQgghhJAAQaFFgpIXTcFjxpFq06aN2rZtm4f/RUDcqpeBRF4XwsOTj1XnjV//+te2K0UQpf9F9hGitL/I/MlhxrMSvvzyS1WpUiVdR7R3xBEzPzu91wPxvoT0XnaguXLjiVN/v4x7aqqNe+6qdbvuuPlszoVGqQ/LbdF1zF+04R6nbi8P0416HnXqoH73I6pCq/1mNzV81kW36ZT4W3nXZ2/ed08vc8gMVwDeml+HOH2iYxJ0OWyma7nol7NaUvxGTO8+ct+ZDlZWrFhhu1IEx+KECRNst09u3rypszhIXEISODLW1YJkSgKRgidPnjy6RNoZBKqE2JLgpgDRypHWBQIsf/782oeglrNmzdL1//3f/9Ul0rwguCUChaIUEO0bSGqcTp066dQ4ABcwgKji0t6vXz/tQ7BPCQQKIEzy5s3rpLkBCHparlw5narnn//5n52+SHEj21CjRg2djsdbOpmtW7eqzz77TNexbHyGuW9CQkLUu+++q+tY5syZM3V96NCh6p133tEXbklZgwCuNWvW1G2S8gZAwEH0SdR4BJlF0FHsZ1OE4LMPHjzopM4BCEpq3hgkor2kHoK47Nmzp67LvoLQQjBQQdILQVwhmCq2BwFH8dm5c+dW58+fd6vv3LlT/c///I8zPwK3nj17Vl2+fNkJtNq0aVO9rUK9evWc/fgf//EfjtBCHcvGukkanu3bXTd0WUfsV6R7MlMq4XibM2eOrgNJh4RjXYL0PngUp/LX2anrdbseVgdPPlSlmu1VU5de0b5Ji1zlnueC4ctuh1X9b464FvacEbMvqraDTqiR315SN++6Itpnr7xNhZx+qHLXdMU5zFVjh14eKFzflU4KAuTC1cfqUZQr1hrabzyfv2yLJGGUo6pr+x5HJzjiafqyq86yvHHrXozuO3rOJT0tQuvS9ce6bD8k6XwCExeGqmodDup6ywHH1YJ1rowIBevtchNaJh2GupaRt/ZOR2hVanNAl7Ke1TsecpZ7+HRSRoZvV13XJfYJiItPVA17uPYn9ldCwjO1YG2Yyvf8+5i8+IrqPNKVQgvL/267K7UWzgkYzrls2bJpH8CxIcfVokWLdIorHAcIiCvnEY6NChUq6DqOAWS/ABIEGOAYwvmB61WOHDl0GixJFYVzHmB55vkk2S5wTcKxCpA+TK5Pcs7jmlCsWDF9jqCvHLtYd0kXRtIHCi3ySpCRGYnO7g0IKuS6AxA2Mjpj/gKT9DQjRoxwfEDEE4QChBYwBYBE9sYFCNG6kdtPkAsZhA7mefbsmZ6Ojo52+vz7v/+7cyHFOspFDCAFz+LFi5350N6qlSuEydKlS91Ej6yTmVYGOfYkRQ7EA7h165batWuX2rJli25DNHv88sX2IyCpjLBJxHEZAcJ6ymdIxHVcyJH2BoILHDjgujEBRD6X/rgIgwEDBjjtJuY+v3fvnp4PUdxlfkRSN0GEdgEXdxOkCcL3IGIGwtZM4GyPYiL9Er5DiByIOwgWMyUTSm91OZ6+/fZbt9Q/c+fO1euO9ZDvDWAdZL8har+5TPkeEYW+QYMGOtOB7Mtq1ao5/WS7RDwCCFcwaNAgt23DTf9pbKKuX7sVrbqPPeO0gQFTzuty09676sOyLvEhqwuRBTBfiSZ71eyV19T1265jdvryq65OyiVAmvZOSi+F6VMXXbkvIeCOn3+khQmWYyKfh/4lm7gEFgRQoS9dgs1m8PQLuu8/KmzV0yK0RLQlJibtZ+FsaNL1AKNV05a61ltG02yhBSEKcXr07CMttLbsT2o3R9vsUTfhzOUovZ0QaiJGZX9mey5SIf6AiEMwbv5lp+74xo3TJdJ3AfkhhmMJIgngOMDxAJGP4830mceAOXqLdphcr3A+T5s2Tdch1CDCML8ptCSNlKQsMzHP+TFjxug6fiyULVtW15HFACB7BUk/KLTIKyOQKXggtOSXIYRWvnz5dF1y8smFCxdE3KRx00XaFIwgYZQDogYiRm6sSMli5vzDBQ9CR1LjmI+VcHGVVDzSvmPHDv2rFyNv5mNQ88ZtprmRC6+0Sx0XRYBtw7S3dDLw/9M//ZOeNoUWRKfMi/UC2G/mqBnS50h/jOYsWLDAWbYpyIBcqLEvkX4G85lCC3kgMSqJaTN1DjDT7ADcNCT1EEYTsd4A3wkSVtuPDjE6hZEqiF8krcaNAaOb5v606xgxRP5B1CG0pA3gxmfuX4DRBIxC4RhDjkrsB6TPwUgY+sr3CMGNNuxL+QGBGy5GOMyUSps2bXJGGMx0SeZN9oOym9XdB7F6dGXl1luqQN1d2g8fOHTqoTpw4oF+3AchIKNFoMv3Iy7lWu7XAgjTUU9co1Rlm+9z+kF0YLTpyJkI3Y7HayK0Oo1wHRcQHeevJokLrNe+Yw9URFScm2iREa2e4846vm0H7zmjP9J3woJQt0eHGLWD0MLImdCszzE3Adik11EtOiG+TKGF5Zds6vpcCK0W/Y/rOoSWPEqEmJPPXrrpph6dunozWm+HDYQW1r9Ol8Nq6AzXY8eL1x7r0UIRWhgZhHAFe5/vh0+ru0QZwDmN6wjSLEkGBgh68xwD+/btc35Q4ccBgPjGdcVbGiyAXKRINi/XK/ygwvIwMobzHo/P0V+EFq49uEbieMWxjBJgxEx+cMo5bwotuX7hnMIPRYyYk/Qj6RtNBhystTuHOJbcL4OydYeoX73fwHY7nDznGqYlJLNjPh4igaN79+6OgH4d+ayB95GkV4E8rkstIg4zG8GQusv88ZcaOnTo4Iykk/QhRaGFX054Pi2/WvALCs/7vQGhJUyes1nVbTVeC6/ew5eqQpX6qLBbwf8Sos2b79XXFh3j+jVJCCGBRB530YLfCPGXFIUWDiYIrQbdXS8IXr7+RIsvb5hCS4DAejtLE52OJyMKLRD+wDWkTgghhBCSWlIUWgD/chk777KbEfIy2XYqIV3s1kPPl28JIYSQQOJTaBHyKrHFkj82Z0OYh0+MYosQQsjLhEKLBDUikP6Yo5VT7zBko1Mfv+SsU/+4VC9dNuiyWH1UvLsat+iM+nOe9mrj0aduYosQQgh5WVBokaDGFFoipMT6TNyrcpbp60x3HbVNDZp+SAutIbOOaF/Lvqvd5qHQIoQQ8jKh0CJBjS2SXtT2nqfQIoQQ8vKg0CJBjy2W0mp8P4sQQsjLhkKLEEIIISRAUGgRQgghhAQICi1CCCGEkADhl9BqPfAEUw6QVHPnzh3bRQghJJPAa7x/+CW0wP7j3lPvCJUbj3RyAiLX4Seluqm9h8477b/+sKFTN8n1+Tc64zghhBBCSGbDp9C6divaqfsa1araZJRTf+uDBm5CK/xBlFO3adfrW9sVFGCdkVSaEEIIISQtpJvQKlK5r7oaFq7rfUYsVR8WaK+F1qUr3ocW4b9x+4HKXryzioh8YjcTQgghhGR40k1opUTfkctsFyGEEEJIpsen0CKEEEIIIWmDQosQQgghJEBQaBFCCCGEBAgKLUIIIYSQAEGhRQghhBASICi0CCGEEEICBIUWIYQQQkiAoNAir5zw8HC3NEwJCQlGq4sjR47YrqCjQ4cOtssrbdq0cZtev3692zQhhJDMQ6qEVpH6u22Xw+27Eer32ZrZbq/MWrTDdpHXmFmzZumyXbt2uhwyZIjZrLJkyaLLbdu2Ob45c+aoGTNm6PrAgQMd/9ixY3UJYbZgwQL18OFDPT19+nR14cIFXd+yZYu6deuW2r59u56Wz8MyN23apOcZPHiw9oGhQ4c69RMnTqgVK1boeq1atRx//vz59fxASqzDpEmTnD7Yjr59++qyf//+atq0ado/YcIEValSJZWYmKinv/rqK7VkyRJnvrCwMC1G8XmxsbF6XgBf586dnX6EEEKCj1QJrZzVXDcmb7Ts5rrpASSYXr3psHovX1s1Ze4WPQ3WbD6iug9ZpBas2KPOXbqpnj6N0yMZSN8TjCDPIUySZZPAIEILPHr0SJ0/f1599NFHju/y5ctanEBYCCK+pOzatavat2+frlerVk2NGuXKu5kjRw7XDM8RYSZgZKlUqVK6juXIskqUKOH4RNR89tlnrpmec/DgQV326tXL8ZnLkVLWQbDXWcpy5co50+KD8BLOnDmjHj9+rIoVK6ZFnj2/ua8IIYQEFz6F1uDprlEA0KTXUaPFOxBPY6atU6OnrlO9hrl+lZtCC0BoCTlLdlVzlyU/Uvaq+axKcIrAzIQIrWvXrukSQgtUrlzZ6QN69Ojh1G2x0aJFCz2qBLFVtGhRR+TIY7qZM2eqyZMnu2b+HrTVrVtXHThwQM8ny6pTp44uMf3JJ5847TbehFaVKlV0mRqhhREtmRafjIoBCC3QrFkzdfv2bbf5N2zYoI0QQkhw4lNogZVbb6kPy26x3R48fvLUdqnERNe7NzExcVaLi10Hztou8poBwQBxhMdiQISWEBERofuY4sMWKxBaeBRYvXp1PfJjCy30wyNDgH4QLGi7dOmSbsMjQW9CS8rvvvtO102wzoIIrZiYGGcUzRZaX375pfr888891t0UWiBbtmxaVAGMpCUntHbu3Knr8siREEJI8OGX0CIkMwFxImKFEEIICSQUWoQQ8pozbNgwNX78eDVu3Dj19KnnkwlCSNqh0CKEkJcERlLxpw6EAlm9erXKnTu39uPxL9rwZ5CQkBCnr1kK9nRawKP206dPu/nkX7m+2Lzvnu3ySdPex2yXVw6fjtDl+2U2q1Xbbqn1u+9YPZJn+vKrtitV1O/uPYQM1qNqe9cfYLyRr85Op56adVi++aZTn7cmTHUYelLXT1+KdPwmoWFPbJcbXUaeVpMWXbHdmuyVk/6xTV4+FFqEEPKSMMUThBb+aAEgtCC6zD9Y4J+nhQoV0u/9yXz4R63U0d67d29dz5Mnj8qXL5+ud+nSRZfC3Llz9WeZAs0UWli+TUqjWhBaExeGaqEAcfJxFddNvFSzvU4IoEmLQtWl64+1YFq84Yaq1Ma1naDX+LPqHxW26vqaHbfV2l131PRlV7VQMIWWgGXlqOr6xzs+s1mfYyrqSbyau/q6io5J0G33HsSqjsNPOfOARetv6OWIpQTaG/U8qstNe+/azRq0fVRpq2o72CWIDpx4oEbMvqiFVo+xrvcoK7Tarwp/vw8qtj6gLfJxvBZroGjDPWr3kft6m8q33O9a8HPCH7reT+0/5bz+nIFTz6sW/Y9rX5nm+9Tfy29xhNa65/tr2tKrKv/zzz11MUmULXy+vULbQSfUh+W2qD1H76s9zz8vS8Wt6u4D12dgHet0Oazfn5669Ir+bvB5O0PC1aOoeGcZJP2g0CKEkJeELbSkDqGFeGlAAt9eueIanZB3CvFHCPPPEPgDBsKagJEjR6oxY8boenIkJ7SEw4cPawsNDXXz24jQArlr7nAbmYHw+GaMS3SYAscc0YIwMoVPgbq7nOmQ0664d2a7PcJTrcMhVbyx65/rEA2fNXAJG2+jSf6ILIA+ZZ8LGpQQft5AW42vD2mxJdMRUXFuI1oQLib9Jp1Tic+e6b63w59qMSks3Zi030Ro9Rx3Vvd9GBmnqnc85LRDnInQkm2atfKax7btP/5Ai097u80RrRt3XcJaxOHSTTf1531Q1vd+ImmDQou8cnADsIOUvipSeizTuHFjvZ5m4FT0l8c/W7dudfxg7dq1bnG80kJy64OAqmhDUFYTrJu5fslRtWpVfaPNlSuX3eRBhQoV3KavXvW8odkg+GyBAgVst8OxY/49SnrdiIx0FxVmRoQbN5JGLGwWL15su1JNmTJltPkiOaE1efEVte2g67GijHLJzb5xz6TQQPCNmnPJmYbQio1LVF+0PeA2ogXD8mOeJuiRHgAxcO1WtK7LqJgIrZQe7/kDRtW8gfXACBHIU2unHkkT/4NHnkJLBBJKiESsP/reMYTW38pvcRuBQnu1Dq71R/1caJQqUG+Xnt4REq4fCYrQwj4QoWkCMYWRL1C62T7nUSj2K0a2th90xSEUoQUgThFVAJ9nizaSflBokaBAbrwSHR4v5Zrg1zqiuTdt2tTxSaBPII9eAIKMTpkyRdchji5evOj0A2iTqO7Dhw9XAwYM0HXE4TKFDcJExMXF6XATeEyDZSFdDgKEStBSM9gohJY9agAk/MLo0aP1aIVsCyhZsqQusTyEpgDyOMlcHwgjEzNyPcJWNGzYUNcxKiIBTPE4SdIZde/eXbprcYZo97bQwudhHYE8hgKIZg+wPxCCo2DBgnoa35X5PeExFvYX9gksa9as2i/rhjZ8BspPP/3UmU+Ws2jRIh0CA+Li5s2bqnjx4k77smXLnP6EEKVy1XjxDCslm+xVq3fctt0knaHQIq8c3JDr1aun69evX9fiokaNGm594Mev7QcPHuhp3IyRVscEN2cTyZ/o7X2Tc+fOOXWk7Tl50vXehQgbxKySdZFHOY0aNdIjB6Y4EaEFEWKPaAFTeMlyZFtEiIGFCxc6dWCujxk/TIDQQgBXpAsy58VjH6wfYm/JMgCEokS5B19//bWb0Kpfv74umzRp4qyzlOZ6YjsRYFb2rZmjUvaXkDdvXnX27FntL1y4sCPQgCzTXI6IY8QRM/NGYv41a9Y404QQkpHwS2jhJbrhs9xHBQhJT2TUY+LEiap8+fK6/vHHHzvtX3zxhb4BA4gDGZGS90wQwf3UKdfLsBUrVnReEsZIiSxPaNmypZt4kNQ8ECkiFDAyhCCmOXPm1CNlGHESoYV5kRIHSFocBFUVoSWjcnhsiMeHgowqybbgfZvjx4/rZUFsSu5FQdYH80lk+lWrVukSQmvHjh36cRL6IEejzIP1W7lypV4n5E0UIMhEWNlCC0IH69W+fXv9V3+8HySBVM19hQCqsp+xb81HTRBH2F8ChBZAsNc+ffqoBg0aOP9sM/vJckRo4buEMDt69KjeNzgmEMGfEEIyIj6F1oWrrhsKmLAgNKnBC38rmPQrlJBgAsKDJA8EEFL5QOQQQghJP1IUWvJCovmSHF7sS46T51wvCd5/GKWGTVzt5Ams0sSVimRviOvXbNTjGJ0HMfxBlGtGQgghhJBMSIpCCyBeCdi45676dpVLSHnjneyuRxRT525VTTtNVS27zVC/z9ZMHTp2Wc1evFPFJySqwydC9SOK+cv3qE795qnmXaa7L4QQQgghJBPhU2gBvK863sdjQ0JeBvJvP0IIISQj4JfQIiRYMMMUEEIIIcEOhRZ55eBfewLCLiDfG8C//vDXfjNAZu3atZ06IYQQEuxQaJFXCoJXAoRY2L9/vzO9e7cr2nN8fLxbrCYE4SSEEEIyChRa5JXjLamtNxCckxBCCMlIUGgRQgghhAQICi1CCCGEkABBoUUIIYQQEiD8Flp7jty3XYQQQgghJAX8ElpdR7kS9/oiOibWdnnF/BcZIYQQQkhmxS+h9SQm+fyGwgcF2jl1pNq5cv2urv8pd2tdflKqm07DU6/1BN2eETh++qo2QgghhJC04FNoSVLpE+ddQSR98fRpnLp3P1KVrTtEvflefTVm2jp14uw1XQ4cu1L3iYyKtuYKXiQxNiGEEEJIavEptMCc766rbQfv2W43bt+NUG990EDXV24IUQePXtJ18SHB9P4jF1Xesj3Ums1HnPkIIYQQQjIrfgmt/HV22q4UiYv3/aiREEIIISSz45fQIoQQQgghqYdCixBCCCEkQFBoEUIIIYQECAotQr6nefPmKleuXLbbK5UqVXKb/uijj9TYsWPdfADLa93aFeLk8OHDKlu2bGrFihVWr+SpW7eu7SKEEJKBoNAi5HsKFy7s1I8cOaK2bNmiPv30U5U1a1bVpUsXdfDgQTV//nwtniC0WrRoYcydxNdff+3U161b59QhtECxYsVU0aJF1aRJk/T0iRMnVEREhCpZsqQWVmiTzy9QoIDatm2batcuKU4dIYSQjAOFFiHfc/euK8hudLQrzhsyGNSqVUsLIwgtYdeuXR4jWuDq1aTgtqZoO3bsmC5FaHXs2FG1bdvWaQdhYWHq5MmTWlhBaAk5c+bU5caNGx0fIYSQjAOFFiEGT548cStflMePH9suh/j4eF1iNMsb6bUOhBBCXh0+hdakRaFq7LzL2iKi4uxmQgghhBCSDCkKrSwVtzopeMQeRbl+hdvs2HdGl3VbjbdaCCGEEEJeT1IUWhBWJjtDwlXYnRg3n4C8hsKN2w9U9uKd1agpa40eSv3fJy3V133nqt9lbaan+49arst3P21ldgsailTu67ZdJP3pO8u7cA8kS7d7z1wwZswY20UIeQ2Ii4tTq1evdqb3nExUuZs+TXfrMc39qZDdHkx29uozt3UlaSdFodVuyElVu3OImz2NTbS7aVp9M1OXfy/UQRWt2k99WKC9GjL+O7c+SDQNoVWoUh89PXH2Jl0Gq9AC4Q8ibRdJR3BCE0LIqwZiS3jyki5LtrgJNiPpQ4pCCyNattC6cz/9dv79h1Gq78hlav0217+yyOsHT2ZCyOuKLWyCzUj6kKLQIiTQmCdzvq9SPslHLUp6zOitfci8eLVqd4J6lPwf/QghJM0ken+g40FyryfY2MIGhusgwMMUu01s5a4Ep37pxjOn/sU3sR59X8RI+kChRV4p9sncfWqcevbs+QXNy+sBk1a6hFarUXF6vp3HEtXp0KSOJTvEqnX7XUIrKlqpm+HPVOXnF55afVwXH1yQ/OXLL7/U5fTp090bCCGvJea16usJrseMy3YkqGbD4lSTIXGqaNunatrqBBXx2CV8irX1LVRsYWNfD+9FePYZuzRe7T7heods0dYEFflEqSo9YtXafQmO0Lof6VqH8cvjVZ5mT7VADL35TB05n6gmfO+7EPZMX2+HL4z3+Axv60LSDoUWeaV4O5mHL/B8Qb7f7Hh9scBFLC7eNV/Toe4vlpbplCS0BBFX3j4nJfxNxUMIef2YuzFBbTqUqIq2eaoWbklQ5TrHav/T7y9J/l5vbGHjbT67/fy1Z6puv1h9vcM0fpjO3+Qa4RKh1WZ0nCrcytW/QAuX4MIPUVle7b6ufueuJY2GeTOSPlBokVfKi57Mj11B3L2S4Ocwf3IgQjuChiJCPCHk9ebE5UQtWkD1nrH6hx/K9QeShBb+VSiP8vzBFjawtfuSLlx2m22ftfH0mWa/jgEr2NL/ZZD0gUKLvFJ4MhNCXldsYRNsRtIHCi3ySuHJTAh5XbGFTbAZSR8otMgrxQ7g9zJI7h9BZhwdQsjrhRmwOLlrRHpjC5tgMnnhn7w4FFqEEEIIIQHCL6EV8zRBFW+8x3aniouht20XIYQQQkimxqfQajvohC5LNNmryjTfZ7UmUbnRCNVz6GJ1736keif7V6p5l+mqz4il6tqNcFWlySi1YMUe1azTNLVs7UHVoN0kPc8fczRXlRoMVzMXbtfTSNED7tyLcObPXaa79kU9jlFZi3ZSDyIeOyl8/jdnC10SQgghhAQjKQqttbvuOPXTlyLVpesph9yGoHovX1s1Zto6PR0TE6cuX3UtA0mmwSeluqnO/ec58/w+WzP19PvgIxBaEFMC5v/vj5rq+qNI1//4keQZ85+5cMMjlyIhhBBCSDCRotACkxdf0eXFaymLLOHhoye6LF6tv6rdcpwjtDCi1WXAfDVwzAq3JNKod+rnEl4yonX4RKgz/9mLN/TIFYTWbz9qoi5duaN+9X4DdTf8kfrDx82d5RBCCCGEBBs+hRYYO++yunDVP6FFCCGEEEJc+CW0CCGEEEJI6qHQIgEjMjKSRqPRaJnYiG8otEjAsE9IGo1Go2UuI76h0CIBwz4haTQajZa5jPiGQosEDPuEpNFoNFrmMuIbCi0SMOwTEjZ+/HjVqVMnbXv37vVof1X27bffevgyumGbHjx44OEPZrt8+bKHL612//79TPm9vmzL7PsxM29baqxBgwYePn+M+IZCiwQM+4T0dlLa7elh9evX9/ClZI8ePVK1atXSpd32Ki1Lliy6bNKkiUebL5Ntsv3BbLly5dJl1qxZPdrEZJ94M4h42zd69Oig+17TYiJA27Zt69Hmr3nbP4MHD/bwmbZp0yZdYj9mpOOpUqVKHvUlS5Z49MOxkVmOEX8M58+UKVM8/C9ixDcUWiRg2Cekt5PSbk8PE6HVu3dvtxvzF198obZu3ap27Njh+OXX7PTp092mA2FVq1bVZvuTs1KlSun1FKF18eJFPS3r3qNHD10fNmyY2zaJvYxtCpTJtkB05ciRQ9exPeKX7e3Tp4+ezps3r5uQkG3GMWAvOyMahNbQoUP1/rhx44YKDw/XxzLaZJ/kzJlTl+hjHguLFy/Wpbl/li1bpksIrdOnTzvLwf66cOGC0w/HGMqMth/9EVr2MZIRz5PU2pkzZ9yODVxTpC7XEqlXrFhR11u0aKGnkxOjxDc+hRaClYqt3HrLbnZAVPf387dz8+Up291tmrxe2Cekt5PSbk8PE6GFi0O7du0c/8cff6wuXbqk6+ZFFReQlC4k6WE1atRwhJa/YgtCCyXWrWHDhk5dLpT9+vXTdbmRetsme5nBbOXLl9eluY0lSpRQhQoV0nX8Eje3CdsrQgujYZMnT/ZYZqC/15dl5iNVCC2UcixjG0+ePOnsmzJlyuhpc360yf6BYFu+fLmu4xH+3bt3nT4rV67U9d27d+vywIEDTluFChXclhnM5o/Qgr2Mcz+YDK8SyHeMbcaxIm3mtQR1EVrNmjXT+w7Hjb08GPGNT6H1ODpB3bgbowrX3203uSHpc549e6b2H7mocxJCaCFVzuDxq3TbhcvJC7VgBNsAi46JtZuIH9gnpH1SxsXFebSnp7Vs2dLDBzN/xYnNmjXLwxdsJjdYbxYREeHhwzZltHe0ZHQFdv36dY92MREHKRneLZo9e7aHPzOZt3fajh496jZtjlDBHj586DHPrVu3nPq9e/fc2jL7fszM25aSmdcM85jwdkylZMQ3PoUW6Dn+rO3yAEJr94Fzuo68hiK0kNswIxP+gAdSWrFPSBqNRqNlLiO+8UtovShx8Qm6RGJo8vpgn5A0Go1Gy1xGfPNShBZ5PbFPSBqNRqNlLiO+odAiAePcuXM0Go1Gy8RGfEOhRQghhBASICi0CCGEEEICBIUWIYQQQkiAoNAiLxXESRLGjBljtKQ/CGKJtCEgNtYVCw2B+MyydOnSrs7f+6KionRZoEABxx8MIDgpIqS/7iDquZSHDx92gpsCKUH+/Pmd+qRJk3Qb9h/iiuXJk8dpE8zj0kSW2ahRI6/zmZ8vIDDugAED3Hxg/vz5ukQQ3Rfl2dNIlRjzyHY7yHHcpk0bu8kNe90zC962iy9uk1cFhRbJlJw4ccJtGkFKgXkBRnDdKlWqqOHDhzttuEEFC1gfRHFGuhVE5Y6JidF+rDMSwGY2smXL5ogQRNIvXry4ri9atMgRxPjOoqOjVfXq1Z35gC0qbKGM4LiCHAvZs2fX0egB9qktZJFXcMOGDbreoUMHHTAWIJVT586ddX3Lli1q8+bNzjwCAoACRLP/5JNPdH3BggVq/fr12tJCQkSYSnxyX90amdOx5DCPY6wD0u8gunf//v0d38KFC/V+w/6cOXOm9pv7JDMg5zaioE+dOlXXse0Iykm8M39tmBo203WOjJl7WRVrtEeFnH6o3i+z2TGSOii0SKYkJCTEdumbCG4s7du3V998842u4waLNC4QZsEotABu6kg3A9asWaNu3rxpdss0yOiQjD4mx65duxxRJeJH9pWMkkJoFSxYUBvAKJOQmJioyz179jg+U5jLPDgWRGghpU1KID2JzFe3bl1nvUwgtNLCxIkTtZkCy7T7S5o6fQSsO8QqsgNAWADZR8iRiJx34hMbNWqU2z7JqGBbcH43bdrUTWhh1LJXr15OH+LOsXOPtG07eM9u0nxYbosWWVv2e28nyeOX0MLOLdlkr+1+6YRHPlPbTiX4NELA6tWr9SMfE1xgcSMEuAFBaIEiRYo4QgsJi4PhhmMKrZIlS2rxiNGYzCq0cufOrUeIAHLw1axZU9ePHz+uH9shAfLZs2e16BGhVaxYMV1idAaPBZEuBtgjWgI+Q8QT9q+M3hQtWtTrPDL61LhxY2fdypUrp2bMmKHrGHGsU6eO0x9AtOGxNZDk18AUWqtWrdLHHNKg+IMtrmyzsUe0ABIGL126VNchSCHasW5hYWE6TQ9Kc59kVMaOHatLfGcYJcV2Yj/jBxZE/M6dO9W+ffusuciVG0+05ay2Xc1eec3xn7/62KlvPxTu1In/+BRaol5rdw5RI2ZfslqTkFyHoEXXGapm87Fq9NR1qnqz0don0yZht+6rjz77Wtf/8HFzVahSH/1ooNtA1wXp99mamd0dIbVs130PcWUbCU7wvgxuojDkZCOEEEIyMz6FVvHGSb/syzRP/lcAhNahY0nPvQtW6u3U3/20ldv0X/K0Ub/9qImKiYlT7Xu7HolIP+RIhA+JnEVwCabQGjHnuK53Gr5Z1Wo3V3UbvV0Nmn6IQosQQgghQUOKQqtAvV267D7W9Tw/JSC0Ll2540xjdArMWrRDzViwzZk2gahat9U1qtGxz1wttBp3nKJHvr7beNjq7X1EK0/FgR6jWRRahBBCCAkGUhRakxZdsV0Bwx69Sg5bUNl2NNT1oishhBBCyKsmRaFFCCGEEELSDoUWIYQQQkiAoNAihBBCCAkQFFqEEEIIIQGCQouQTEC+fPl0wE4gQTwPHDigpk+f7gRozeiBKF8WCNr59OlTJ9J7/fr1dQofBI0F2J937txhdHFCiF9QaBGSSbhyxfUv4b59++oSOeyQhgTRyc+dO6fz2UkOPpI8EFCDBg1y8v+JT4B/woQJFFqEEL+g0CIkk4CkwQBpR4AptICkhSEpAwGFdDuff/65mx/iChQuXFjva8mrSAghKeGX0Hr2zPYQQsjrxejRrnRihBCSGnwKrS+7HVartroeN5Rvud9qTUJyHSLae2pIbX9CCCGEkIyCT6EFxs67rAVXlopb7SYHCC3kLoRw2nXgrKrXeoJOGn389FWVkJCo3vqggbp+8776a962aubC7XqeTTtP6P7e0vMQQgghhGR0fAqtSm0OOPXRcy4ZLe5AaEFAQTjBxkxb57TVbzdRl/kr9FJ9RizV9WffP49E30nfbnb6kszNhQsXbBdJBwYPHmy7UkX+/PnV8ePHbTdJI0WKFLFdrx2TJ09Wp06dst0+sa8RQ4cOdZvO7MifLI4cOWK1kIyKT6EF4uITVc5qrlEofxEhFZ/A3IMkCbmIdu/eXb+o3aVLF6cNYqFy5crONEkdw4YN0yX24/z589Xjx4/19IwZMxw/ePLkiX6Z2xRnEFpE6ZfgBdzwsI9CQ0P19BdffKGaNGnitF+7ds3ph+O2Y8eOTtuIESOc+usK9ov5z8xq1ao59Rw5cjh1AX/WiI2Nda4R48aNUy1atNBCS45hULVqVaeeGaHQynz4JbR6jD1juwhJE/avVQgt/DsO9O/fX3Xo0MGtnfjHjRs31PDhw3X96NGj6uTJk6p69eqqa9euehpxoERYYX/HxMSoiIgIPQ3RRaHlIjw83KkvW7ZMJSQk6Pr169fVxx9/7PwjEYjQKlu2rD5uz5w5o+NvgZo1a7oW8hqDES2TcuXKqbCwMHX//n3nnDePu7Vr16rSpUvrHwIQt9j3EK8QVnIMAxy3pUqVcubLbIjQwv64evWq1UoyIn4JLULSm1mzZunSHNEiaQc3qBcFowfEXWyZIHCpv5gjYyR5pk2bZrsIyXRQaBFCCCGEBAgKLfJSOX36tO0i3xMZGZmipfQYgfs1dSQXbHT16tW2i1jcvn1bP8YTw6NAQkjyUGgREiTYwsqbdevWzZ7Nq4+kH5UqVbJd+h03eb8ob968KmvWrNoAIscj5ZGN3UdA7sTNm73/81re10GeReRXNNcFy2rWrJkzvWHDBp3z0kTeGatQoYKb/0Wg0CIkdVBoERIk2KLKm3Xq1MmezauPvDghISF6pFDEDf6sASZOdIWrkRQ98g9DCC6wfv16nSvRG4cPH9alCKjdu3er7NmzO0Lr4cOHTl98jvRDiZfDTaEFkWWGPsBy8G/v8ePH62lZT5CeeRlFaGHdIf58Ca2u1XKpsAEfq5uDcnjY0a5ZdLu/JD6NVM9iHtluN7Ctvv6ZKN+lDf/pRwIBhRYhQYItqryZN1HlzUdeHPmHrIibnj176nLXrl1ufxzwFsrBm7CRXImbNm3S5Z49e9SJEydUzpw5HaEFESPg34zmcjCCZY+ume0QeEBe5sd6Ct7WJ61gHSGytm/fri0loQURZYsrWMS6fh79fHFrZE7H1DPfYYMQw2vu3Lm6LsJL/nm7YsUKXbZr187VWbn+AUmhRQIBhRZ5ZZQvX16X/fr1UyVKlND12rVru5XSZ+TIkTpkQWbGFlXezJuo8uYDiKNVpUoV203SgITCMJFYgTaXL1+2XR5IjDOT+Ph42+WAkSNBPtfb58ujwkACoSUiy5fQwoiVKbBujymqSxv0s8GIHMwUWKYd2LjI6WOCnJQFChRwpmfPnq1LBJHFiGHFihWdfz0jbIcZ7ytYhFaPHj1UsWLFdB2hMBAuBEIaBsGd3IgcCU78FlrVOhxS75fx/h4BIWkBsXEExCCCKIDAwq92+y/2uIFAaJ09e1bduuXKvZnZsEUVfoXj0VVqhFbRokWdOkZOcMNJz9EMQlIjtBY2+JPHaJY3oYV+XnmW6CGw3Ea2/ABBUHfu3Kn27dunsmXLphYsWOAILYg0/Jjr06ePjt0VLEIL56z82ITINwM5IxYZ8PYeIAlO/BJaYbej1d/Lb1G1O4fYTW5EPY6xXQ7hD6JsF3nNwYUkMTFRjRo1Sgst8cnF7rvvvtOlPC6B0EJgTvPxSmbCFlreLDVCCyMreAmaQittYL/Jy+t4GR3HHh5FyUvw5svtuGFjtARBOQHe1+rbt6+bT8AIhTnigs/B6BQ+wxtLl7rSlqUX3kbnUkNqhNalU4c9RJYttvAYEf2S41n8U6We27O4mOf2RCXGPFKJ0Q+VSnQFk81s4DwXEI8NI3ESAPeTTz7R7//JaBfJGKQotCCsqnU4qOulm+3T0/0mJ6+iIbQuX3UNca/edFjnMRSQcLpF1xmq26CFji/YgTg0t4GQQGKLKm/mS2iR9AP/DETEd7xHBSCIUBfhOnXqVFWwYEFdl8dXImIw6lDr/7d3599R1Wkex+efmDNn5kzPcqZn5ofpPqdZhEZAgbgA0goqjVESBcIugsiqLEqiyKKggqzKLosGAQWRdFiChh2MEdIEEEGWsBgICRAIyXd8nvRzuXUTspjcriK+X+c853vv965VRVkfb6q+NzExos+/XuPGjb1ltj9/IJbvfcm6P/zwg9cn3xFr1aqV3ppGSOArKChwu3ffvh+tXCWWQGiB2/6UKB/QcosbuUJq5yP/kyOmTp2qrXyJ3ka7F3caBLe2X4YHfu2qDFom63BBja9oWdDatDXL/WfzQd4yC1r/1nSg25tV/XcYgF+bYKiqrCoLVZX1oe527dqlrX0pXn7VZyHHxi2zL8gHg5awP0/5++QL6nafPwk6ElgkYPXs2VP7bL/z58/Xdf1BS25HI7dVsi94S9CS7eXP6UauAssVYQta9mcmOYacv7DzsdsLbdy4Udtly5ZFBC3/1VE/hncAaqdGQUs8NWwP39ECQhQMVZXV3Llzg5tFfNcNNWP3zQtat25dxLxd9Tl27JjXJ9/5EZV9od3IOFtVsf36+b/wHiRXroyFr169enl9orIvx1fHwlZtELSA2qlx0AIAAEDtELQAAABCQtACGgj/92sqI6OSHz161PuzWXDwy5q40y/jGorBgwcHu/6uqvrzIYC7E0ELaCAsaA0fPlxbGeS1devWOi0/EY+Li9PWbh0jI4/bNqmpqRVutCzL/caMGfOrCFoyoriRUNq7d2/fGuVspHEZQiMtLU2/J9epUyf38ccfe/cxlOf+7bff1ml57oX8sm/NmjU6betNmDBB2w4dOmjQktHLbTsbwdzIl+6TkvglNHA3IWgBDYB8gEuIEvZl7fXr13vj73z66aeusLBQx+WxL2JbkJJfpsnYTjNnztR5I9tLiaysLJebmxtxE+OGSIKWjBYu5HmU4RtkmAb/XQnkebMvwltoFfbrPRkTbty4cS45OVl/IZifn69fjpeAZa3wBy3pX7FihQatAQMG6HYSsuQc7FeC8hrINpUFPwCxi6AF/ErISNKfffZZsBtVkCt4Yf850a5oAWiYCFoAAAAhIWgBAACEpNqgde5isZu5vHwk90/TzwSWRrp2vfy7IfXBRpgXO/aVj8ws8i/feZBAAACAWFLjoCVVVdAaPHahN/3Yc1PdhLdT3X80G+R+PPOT+/bQCXf85Hn3f22GuX9p1N+dv1jg9mcf99b/p9/38aZXrMnU9rO0fe53bYe537YY7Ea9/pG7WXLLrf1yrzuUW35LCQAAgFhXbdDyqypomeLi27e2iOuaou2kmevcghVb3bAJS92OvbnuT89OdjM+LP+FjpEwZrIOntCg9e/3PK/bStCatShNlxG0AADA3aLeglbe+cvuH39X/rPjR7q/6S5fueoFrX/+Qz8NWqPfWO6GvrpE+2xd8dJrS1yTh2/fGNeC1udp+92I5GUatMT/tBxC0LrLbdiwgaIoiroLC79MrYIWAAAAao6gBQAAEBKCFgAAQEgIWgAAACEhaAEAAISEoAUAABASghai7uLFi+7mzdvjr/0SU6ZMiZiXfValrKwsoq2rgsslMVEAgNhC0ELULV68WNtGjRoFltRccNu+fft60+vXr/ctiVRaWhrs8pw7d/s2UFWxkHMwq8jt31XodmYURISfNr//2LVvtrpCKKpLbdl4qkKfFQAgdhC0EHX+oFVcXKxXtyw4HTt2zA0aNEinJTBt2rTJFRYWuubNm7tbt27perm5uRFBq6CgwJsW/qB148YNd+TI7XtntmrVyl2/ft117tzZ28eMGTO0taB15coVPZ4oKSlxXbp0Kd/4byzgjOx7XNuFM/Migo8ELZtO/+JH90LiVpf2+Uk3vO92t2TOX93El/e4P7Vc41r+70oX94dU1/WBz907E79xf/ztcjei33Z3Pu+6bnv29DVtL1644bZvPuPmTP9O+/o+le7u/Z8VBC0AiEFVBq0rRSUu88BP3vwfnthye2HAb5oMdBk7c3T6z32me/0ywrvp1ne6+33b4d48ICRoWbgZMWKE2717t9u5c6f74YcfAmuWe/HFF93w4eX/juxPhv6gJdMSjKwvGLT88vLytJV1Kwtaq1at0vPZtWuX9m3fvr3C1bNg0AqWBK3vvsnX6SM5l13Cnza6nOzLrv/T6a7Xk2katGa/le0Sf+5v/l/L3UNNVutyCVDZ+8u3O3m8KGKfErQ2rj2h/Tsz8tzQpAyCFgDEoCqDVn5B+fdmOg3c4TZlnncnz14LrHHbwcOn9N6EA0Z/4J5MmuaKrha7mQu+9ILWxfxCbb8/UbM/x+DXw65oNW3aVNuFCxd6Yea1115z999/v04nJCS4Hj166LQFrTZt2rjx48dHhJ+cnPLAn5qaqu0DDzyg7ZAhQyoErQEDBujx33rrLdetWze3cuVK17VrV10WHx+vrQQvu6K1ZMkS73xMMFhFuwAAsaPKoCUeTPpa28+35bnqvjd85Puzbsi4RfoF49LS8pUtaJ3Oy3cHvjvups/jfkloeIJhJxpVeOVW8LQAAFFWbdASzeO3uR5j9ge7AQAAUIUaBS0AAADUHkELAAAgJAQtRJ0NtxD8NV992LZtW7DrF7NfHvrZOdsX6Gvr8uXLVW7rH4qitoqKirTt06ePDoUhQ1nItMjPz9cfHYh27dp524RBHuPd6vTp08GuOqnv/QGIfQQtRF0waB08eFBbG3RUgogsmz17dvkGfzNr1iy3evVqnZZfGiYmJupQDv5fFkrQevXVV12LFi10XgKH6N27t7ZNmjTR1ra59957I+YtJMgPPCZNmuRatmyp87afYNCy4SLk15APP/ywN/K8/WqxdevW+ivIlJQUPa/KgtbevXvde++955KSkiKCVnp6uv7i0c7ZyK815biPPfaYF56EP2gJeUwy3a9fPw1aonHjxjoumfAHXXnu5Vee8nw+//zzXr+Q7f0DvcpzsWjRIm9eHrM9D/IayWO059+O1alTp4j9yvhk9vrKqP4bNmxwJ0+e9JZ//fXXEaP9z50715s2cix7/fw2b96srfzqNDMzM2LZgQMH9NjJycmVbitDjNhjlaDqf36F7NteKxvvze+VV17Rds6cOfpvwj9kiTzHsk977ubNm6fnIvsKHkfOE8DdiaCFqAsGrT179mhrIcWC1pYtW3TeLFu2zAta165d06EaMjIyIj6E5YNQAs19992n8xagOnTooO0999yj7dWrV7Vt1qyZtuLQoUPetJAgYOubYNCyoSXk3CVo2a2F7LGsXbtWA4Sc652ClqwjH7QyXlgwaD3zzDMR5ygkLEnIbN++vX5YGxu41YKWBCeblkFXZVgMYQHDH7TkfCVQyeOxbYwNseEXDME2VIe8RvIY4+LidN4ftIL7HTt2rLYSSuV19Qet7OxsHVg2yB+q5Vjy/Ph98skn3vS0adNcVlaWb2k5CaQSiILbCglGEn5E27Zt9d+XsX3bayVjwAVZ0Fq+fLk+p/6gJfNyzOBzJ2PD+V9HAHc3ghZ+1Sr7cEXDIYE2LGHuG0DD8Q+HDx92VO0LAACgOlzRAgAACAlBCwAAICQELQAAgJAQtAAAAEJC0AIAAAgJQQsAACAkBC1EXWpqaoXBK2srPj4+2AUAQNQRtBBVNnq5kNG/hdzSZenSpe7dd9/1buViZPRyuQ2Of9T24uJighYAICYRtBB1cpuaffv2aQm5rYrc6uSDDz7QW+v4SdAK3gbn1KlTrlu3bhF9AADEAoIWAABASAhaAAAAISFoAQAAhISghaj65ptvgl0AgCjjv831h6CFqBo9enSwCwAQZfy3uf4QtBBVvJkBIPbw3+b6Q9BCVDXUN/P+/fuDXbV27ty5YBcA/F001P82RwNBC1HlfzOnpKR404sXL9bvCNhApMOGDdM2MzNTWxnQVLz//vtuxIgROvDpnDlzyjf+mX+6V69e2g4cONCVlZW5nTt36mCo1v/666+769evu6ysLF0+ePBgd+HCBTdmzBhXVFTkEhMTvX3Jecg6hYWFLj093e3YsUP3O3XqVJeTk+OtJ9LS0txrr73mlixZEjHy/dNPP61t7969vT7Znzh69KgrLS31+keOHKmtnMeQIUN02s5B9n/y5Ek3fvx4fSxyHlevXtV1ly9f7iZOnKjb2LnJecj+X3zxRZefnx/xfAOAH0Gr/hC0EFXBoHXkyBFvftSoURq0zp496xYtWqSh4k5XisaNG+cmTZoU7Hb9+/fXVkKRkJAj+1izZo1/Nd1WjiUlx7IR6QcNGuRefvnliHWnTJmi68n5bd68WftkGwuFEnaEHEO2nzFjhretkIBl5zVt2rSIZf6QJWyflZ2DDNQqyyVgCTmezAfXtXOz87DljKYP4E4IWvWHoIWoCgYt/4e/BJmVK1fqdI8ePbT1By1bV5adOXNGr0qtXr3aW27sClJCQoLerqeqoCWj0sv6/qA1efLkiHUl5Ej4kythFrReeuklt2DBgoj1kpOTvaBl5yDkiprw95lZs2a5sWPH6vRzzz3n9Z8/f951797dm5dzkGP6g5Ys79evn64ry4Ucw87NgpasJ4H2nXfe0StjABBE0Ko/BC1E1S99M8v9EGPZrl27vOk7XYUDgFj1S//bjIoIWogq3swAEHv4b3P9IWghqhgUDwBiD/9trj8ELQAAgJAQtAAAAEJC0AIAAAgJQQsAACAkBC0AAICQELQAAABCQtACAAAICUELAAAgJAQtAACAkBC0AAAAQkLQAgAACAlBCwAAICQELQAAgJAQtAAAAEJC0AIAAAgJQQsAACAkBC0AAICQELQQc67nfOty/viv9Vp/vf+3wcMAQFQcPny43uv06dMRxxg1+6ZrNaC4zvXXE2UR+0XtEbQQc4Ihqb6KsAUg2nJzcyuEpPoqv2BgqkuhbghaiDnBgFSfBQDRFAxH9Vl+wbBUl0LdELQQc4LhqD4LAKIpGI6ssrKyKvRVVVu2bKnQ5xcMS1L3PV+xryaFuiFoIeYEw5HUtW92a1t2o9jltPiNu/TpUm9Zybkz2pZeK4qYr6wAIJqC4UgqISFB2wcffNDt3LnTxcXFua1bt7qkpCQNVEOHDnUjR47UdTp37qztggULKuzHLxiWrNoNLm8Lim6vk3OizM3/vMSt3X6rwvoErbojaCHmBMORP2iZCx9M85bJtNSVbRvd+fcnevPBfRC0AERbMBxJdezYUds9e/a4Ll26uOTkZJeWlqZ9jRo1cgcPHtTpyZMnu5SUFLd79+5fHLTsS/KLN95yT4654W6WlK+bmV3qFqwnaIWBoIWYEwxHFrSKv8/V6cNt/jviqlXZrRL3fXxbd+zJVhHzwX0QtABEWzAcWTVt2tRlZ2e78ePHu549e0YErW7durn4+HhvXtpfErRKbjn305UynT6RV6ZBa+Ss8uC1ZX+pO3+pfFmwUDcELcScYDiqzwKAaAqGo/osv2BYqkuhbghaiDnBcFSfBQDRFAxH9Vl+wbBUl0LdELQQc4LhqL7qx+E9g4cCgL8rGVg0GJDqq/yCYakuhbohaAEAAISEoAUAABASghYAAEBICFoAAAAhIWgBAACEhKAFAAAQEoIWAABASAhaCE1wjBeKoiiqYRWqR9ACAAAICUELAAAgJAQtAACAkBC0AAAAQkLQAgAACAlBCwAAICQELQAAgJAQtAAAAEJC0AIAAAgJQQsAACAkBC0AAICQELQAAABCQtACAAAICUELAAAgJAQtAACAkBC0AAAAQkLQQtRdunTJNWrUyA0ZMkTn27ZtG1ijch9//LErKipyTZo0ce3bt9d91Ebjxo21PXHihEtLSwssBQCg7ghaiDoLSEuXLnXff/+9F7QWLVrkcnNzdTo5OTmi7d+/vxe0ZPt58+Z5+5kyZYp75513dHr27NluwIAB7uzZs27QoEHaZ2x9C1rS9urVS/vkOJs3b3Zz5sxxL7/8ssvJydF+2a+FsoULF7rx48frtH//su2ePXvc8ePHdV7MnTvXbdq0yc2YMcNbZ8KECTrdo0cPV1pa6q0LAGg4CFqIur59+0bMS9B66aWXNEQ9/PDD2mehSNrevXvr9AMPPOAFLVtm4Wbr1q0R27Vp0yZi3qalLGhZgLJ+//r+7TIyMvS4I0aM0HkJYf79S5WUlOi8BK6JEyfq9L59+7SNj4/39vf0009rK/sEADQ8BC1EnYWOZcuWuSNHjmjQkqs8omfPnhHrSNutWzedlnWCQSsYXJ588kltu3btqq2ffzu52iTBzeZtWdOmTb2+4uLi8g19fvrpJ/f4449H7N8fyvz7unz5ckS/X3AeANAwELQQddeuXdOg8corr+i8/enwsccec5MnT9bpwYMHu7fffjsiHPn/dGh9ZWVl2n777bfaZ0FL/swn/RLkjG0nV50kaMn3vCS83SloiXbt2rm4uDidlvXl+2HCv39/aBo4cKD74osvdFqu3PnP1cj03r17vXkAQMNB0AIAAAgJQQsAACAkBC0AAICQELQAAABCQtACAAAICUELUef/lV9NyfpPPPGETj/11FMRy2SA0trur6a6dOlS45HrH3nkEXfhwgU3ffp098ILLwQXV2CPp6ZsfK6asIFV60IGW7VfR9qvOxcsWOA++eSTwJqVs19ohiGs1xsA6oqghZhhQePgwYM6cvp7772nAaFTp04acMSNGze0/fLLL7WVMCNBy8bWsu39ZFvbXgYLFTLsgwzPINasWaMDpJrU1FT32Wef6fS0adO8Y/rZOF0yNMQzzzzj9dugpK1atdJzk7G5ZIgHCVqvvvqqhkBh433ZQKfi9ddf18FPZVgL49+mZcuW2sq2GzZscGvXrtV5OZYMHSEl00JCkY08L8fwBy15ntPT0715Gz/Mngf/8yXkcZw7d06Dkj1u26awsNALWsOGDXO3bt1yQ4cO9ZbJ6Pp9+vTRaTt/0aFDB3fs2DE9Z9mXHNueO7N7927dlzwncicAYcN1yOvy5ptv6rQ8PgtaNp6ZHPfo0aM6LeTx2jnbsWV4kNatW+vQHjJ8iLDXY9SoUW706NERfXLslStX6rQETfl3Yf+GvvvuOz1e586ddR4ADEELMcmucvXr10/n5QPcxtkSFrTkw9auaMmAp8Y/OKix4CH7tfGxbN5/RUQ+QP3bP/TQQ950UGZmZsT8V1995e1LPoTtg1mClgQcC0LCjmv7kHVkzC1/+LJt5BY9sq60ckVNxu1avHixdywJDxKIhATF4GMaM2aMN92sWbOIZXbPx+A24tFHH9V20qRJEVfy/MFUgpYEMSFBx16PgoKCiOfHApiQY8qx7Jzt2BcvXtSxyvwOHDigrbwm99xzj9cvz8H69et12raXevfddyu8LuvWrfOm7djz58+PWO7fh4RDCY6yf+uzfzMyQK3tQyoxMdGdOXNGlzVv3tzbJwAIghaiTq6S2JUS++CV29vIVS0LWsKuYAj5c5zcx1CCjH2w2/0IExISIq5q3XfffXprHglQ27dvdzt37owIWhJk5AqIkSsv/g9Mf9CSq2wfffSRNy9XebZt26bTcm/Gjh07uo0bN+oVE9mPP2jJVSa7kiXkmLKe3edQ1pEPcOkzto1ckZLBWSVc7Nq1S8OYBC0JM88++6xe9fMHrT//+c/eeWZnZ3uP4ebNm3rlRa7kiNOnT2tIkH7/82BXvE6dOqXby62Q/EFr5syZ+hzfe++93hWt999/X18vue+k3NZIgpEEkb/85S+63IKWBCc5rj9oybGDgbZ79+5u3LhxXtCSfds9LIU8F0JeW9mXnKsMGCutzEsgMv6gZccOBi17PWS7N954w7sK+OOPP7qUlBTv34xcSZTHJs/7yZMn9fWW51Cuch0+fFifL7t6BgAELdwV8vLygl342apVqzToSdACAMQeghYAAEBICFoAAAAhIWgBAACEhKAFAAAQEoIWAABASAhaAAAAISFoAQAAhISgBQAAEBKCFgAAQEgIWgAAACEhaCE0GzZsoCiKohpwoXoELYTmypUrFEVRVAMuVI+ghdAE35AURVFUwypUj6CF0ATfkBRFUVTDKlSPoIXQBN+QFEVRVMMqVI+ghdAE35DRrJEjR2qbkZER0d+oUaMK61IURVE1K1SPoIXQBN+Q1VVmZqa206dP9wKQf7qu9dFHH+m+ZsyYofMtWrTQ+eHDh+t8x44dXZMmTXT67NmzFbanKIqiIgvVI2ghNME3ZE3KQo+Fq6SkJJ0+duxYhXVrW7Kfli1basDav3+/zkuNGDFCl0vQsmOdO3euwvYURVFUZKF6BC2EJviGrElZwJL20qVLXn99BC1/+fctdeTIEW/66NGjFdanKIqiKhaqR9BCaIJvSIqiKKphFapH0EJogm9IiqIoqmEVqkfQQmiCb0iKoiiqYRWqR9BCaIJvSIqiKKphFapH0AIAAAgJQQsAACAkBC0AAICQELQQc3JycoJdMUvG46pKamqqtosXL652XQBAw0PQQtTJ4KSibdu2LiUlJaIvbBKA6uLQoUPBrghNmzbVAVBFdesCABoeghaibvz48e7ixYsatNLT093999/vCgsLg6vVSb9+/bSVY1iIW7FihQatkpISnR82bFhEwBs8eLC2Fy5c0Hbt2rXa9unTR9vLly9reGrXrp3Ly8vTvmvXrrmEhAS3Y8cOnZd7J/qDljw+v3nz5mkrI9OLNm3aaCv3fRRTp04tX/FncosgAMDdhaCFqJOgJeFEQoYEIVHfV7QsaHXq1EnDj5AANHv2bFdUVKTzBw4ciDjuunXrtM3Pz9fWliUmJmprQUv2afuQgCT3U9y8ebPOB4PWzJkzddpMmzZN2/Pnz2vbvn17beVchP+Km9wHEgBwdyFoIebIDZ/rmwQtu3Il7CpVTUmoEqWlpRHzVbHwVZ3gvmu6HQAg9hG0AAAAQkLQAgAACAlBCwAAICQELQAAgJAQtAAAAEJC0AIAAAgJQQsAACAkBC0AAICQELQQdf57HcqI6C1atPBGiG/o5LEfP3482A0AaCAIWog6f9Dq27evN0J6fZJ7Bn799deuoKDAffjhh3prHLtnoRzvjTfecAsXLtSbQG/cuFH7H3/8cTd27Fidtm2M3DZo06ZNLj4+Xlsjt9+x+xWK+fPnu9atW7stW7bo/OTJk90LL7zgkpKSdN4ftPznsGrVKte9e3cXFxf3tz0517VrVz2nnJwct2HDBpecnOw6d+7sJk6cqPtcuXKlty4AIDYQtBB1ElokcEhAOXz4sPbV970OxVdffeX27dvnsrOzXc+ePbXP7oEoyx599FENOcbudXjr1q2IbcxDDz2kra0nVq9e7U2b5cuXRwStDh06eMv8QSt4DmVlZVpBzZs311aeL3nujD0WAEDsIGgh6iwsSHCQYCHhIyMjI3KlOsrNzfXCm7S9e/fWabtps/TdKWjZctnmkUce8fokaKWlpXlXuuRK09q1a/XqkmnVqpVbv369TktAkqB16tQpN3r0aO3zB63gOTz77LN6Bcv06NFDr2rZuleuXCFoAUCMI2gBAACEhKAFAAAQEoIWAABASAhaAAAAISFoAQAAhISgBQAAEBKCFgAAQEgIWgAAACEhaAEAAISEoAUAABASghYAAEBI/h+ipmM5IxvgagAAAABJRU5ErkJggg==
