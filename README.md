# engce301-week6_lab_ntier_redis_loadbalance

Lab Summary – Week 6
N-Tier Architecture, Redis Caching and Load Balancing

โปรเจกต์นี้เป็นส่วนหนึ่งของรายวิชา **ENGSE207 Software Architecture**
โดยมีวัตถุประสงค์เพื่อศึกษาการออกแบบระบบที่รองรับผู้ใช้จำนวนมากด้วย

* N-Tier Architecture
* Redis Caching
* Load Balancing
* Horizontal Scaling
* Docker Containers

---

# Architecture Overview

ระบบนี้ใช้สถาปัตยกรรมแบบ **4-Tier Architecture** โดยแยกแต่ละส่วนออกเป็น Docker Containers

| Tier    | Component   | Description                             |
| ------- | ----------- | --------------------------------------- |
| Tier 1  | Nginx       | Web Server and Load Balancer            |
| Tier 2  | Node.js API | Application Server (Multiple Instances) |
| Tier 3a | Redis       | In-Memory Cache                         |
| Tier 3b | PostgreSQL  | Persistent Database                     |

```
Client (Browser)
      │
      ▼
┌────────────────────┐
│  Nginx LoadBalancer │
└──────────┬─────────┘
           │
     ┌─────┼─────┬─────┐
     ▼     ▼     ▼
   App1   App2   App3
     │      │      │
     └──────┼──────┘
            │
      ┌─────┴─────┐
      ▼           ▼
    Redis      PostgreSQL
   (Cache)      (Database)
```

ทุก Tier ทำงานเป็น Docker Container และเชื่อมต่อกันผ่าน Docker Network

---

# Caching Strategy

ระบบนี้ใช้ **Cache-Aside Pattern (Lazy Loading)** เพื่อเพิ่มประสิทธิภาพของระบบ

## Read Operation

ขั้นตอนการอ่านข้อมูล

1. ตรวจสอบข้อมูลใน Redis ด้วย key `tasks:all`
2. หากพบข้อมูล (Cache HIT) จะส่งข้อมูลกลับทันที
3. หากไม่พบข้อมูล (Cache MISS) จะ Query จาก PostgreSQL
4. บันทึกข้อมูลลง Redis พร้อม TTL 60 วินาที

```
Client
   │
   ▼
Application
   │
   ├─ Check Redis
   │
   ├─ HIT  → Return Cache
   │
   └─ MISS → Query PostgreSQL
             │
             ├─ Save to Redis
             └─ Return Response
```

---

## Write Operation

ขั้นตอนการเขียนข้อมูล

1. บันทึกข้อมูลลง PostgreSQL
2. ลบ Cache key `tasks:all`

```
POST /tasks
     │
     ▼
Write to PostgreSQL
     │
DEL tasks:all
```

การลบ Cache ทำให้ Request ครั้งถัดไปดึงข้อมูลใหม่จาก Database

---

# Load Balancing

ระบบใช้ **Nginx** เป็น Load Balancer เพื่อกระจาย Request ไปยัง Application Server หลายตัว

## Example nginx.conf

```
upstream my_app {
    server app:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://my_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Docker DNS จะช่วยให้ service `app` กระจายไปยัง container ทุกตัวที่ถูก scale

---

# How to Run

Start containers

```
docker compose up -d
```

Scale application instances

```
docker compose up -d --scale app=3
```

Check running containers

```
docker compose ps
```

---

# Quality Attributes Impact

| Attribute    | Impact    | Reason                                                |
| ------------ | --------- | ----------------------------------------------------- |
| Performance  | Increased | Redis ลด Response Time จากประมาณ 50ms เหลือประมาณ 2ms |
| Scalability  | Increased | สามารถเพิ่มจำนวน instance ด้วยคำสั่ง scale            |
| Availability | Increased | หาก App instance หนึ่งล่ม ระบบยังสามารถให้บริการได้   |
| Complexity   | Increased | มีหลาย Tier และต้องจัดการ Cache                       |

---

# Key Takeaways

Layer และ Tier มีความแตกต่างกัน

| Layer                             | Tier                           |
| --------------------------------- | ------------------------------ |
| Logical separation                | Physical separation            |
| Controller / Service / Repository | Nginx / App / Redis / Database |

TTL มีความสำคัญในการป้องกันข้อมูลค้างเก่า (Stale Data)

Application ควรเป็น Stateless เพื่อให้ Load Balancer สามารถกระจาย Request ไปยัง Server ใดก็ได้

---

# Summary

Lab นี้ช่วยให้เข้าใจแนวคิดสำคัญของ

* N-Tier Architecture
* Redis Caching
* Load Balancing
* Horizontal Scaling
* Architecture Trade-offs

ซึ่งเป็นพื้นฐานของการออกแบบระบบที่รองรับผู้ใช้จำนวนมากในระบบจริง

---
