<p align="center">
  <a href="https://dragonflydb.io">
    <img  src="/.github/images/logo-full.svg"
      width="284" border="0" alt="Dragonfly">
  </a>
</p>

[![ci-tests](https://github.com/dragonflydb/dragonfly/actions/workflows/ci.yml/badge.svg)](https://github.com/dragonflydb/dragonfly/actions/workflows/ci.yml) [![Twitter URL](https://img.shields.io/twitter/follow/dragonflydbio?style=social)](https://twitter.com/dragonflydbio)

> ก่อนไปต่อ ฝากกด GitHub star ⭐️ ให้พวกเราด้วยนะ ขอบคุณ!

ภาษาอื่น: [English](README.md) [简体中文](README.zh-CN.md) [日本語](README.ja-JP.md) [한국어](README.ko-KR.md) [Português](README.pt-BR.md)

[เว็บไซต์](https://www.dragonflydb.io/) • [เอกสาร](https://dragonflydb.io/docs) • [เริ่มต้นใช้งาน](https://www.dragonflydb.io/docs/getting-started) • [Community Discord](https://discord.gg/HsPjXGVH85) • [Dragonfly User Conference](https://www.dragonflydb.io/events/dragonfly-ascent) • [ร่วมเป็นส่วนหนึ่งของ Dragonfly Community](https://www.dragonflydb.io/community)

[GitHub Discussions](https://github.com/dragonflydb/dragonfly/discussions) • [GitHub Issues](https://github.com/dragonflydb/dragonfly/issues) • [แนวทางการ Contribute](https://github.com/dragonflydb/dragonfly/blob/main/CONTRIBUTING.md) • [คู่มือสำหรับ AI Agents](AGENTS.md) • [Dragonfly Cloud](https://www.dragonflydb.io/cloud)

## The world's most efficient in-memory data store

Dragonfly คือ in-memory data store ที่ถูกออกแบบมาเพื่อรองรับ workload ของแอปพลิเคชันยุคใหม่โดยเฉพาะ

Dragonfly เข้ากันได้กับ API ของทั้ง Redis และ Memcached แบบเต็มรูปแบบ คุณจึงนำไปใช้แทนได้เลยโดยไม่ต้องแก้โค้ดสักบรรทัด เมื่อเทียบกับ in-memory data store แบบดั้งเดิม Dragonfly ให้ throughput สูงกว่าถึง 25 เท่า มี cache hit rate ที่สูงขึ้นพร้อม tail latency ที่ต่ำลง และยังใช้ทรัพยากรน้อยลงได้ถึง 80% สำหรับ workload ขนาดเท่ากัน

## Contents

- [Benchmarks](#benchmarks)
- [Quick start](https://github.com/dragonflydb/dragonfly/tree/main/docs/quick-start)
- [Configuration](#configuration)
- [Design decisions](#design-decisions)
- [Background](#background)
- [Build from source](./docs/build-from-source.md)
- [Contributors](#contributors)

## <a name="benchmarks"><a/>Benchmarks

เริ่มต้นเราจะเปรียบเทียบ Dragonfly กับ Redis บนอินสแตนซ์ `m5.large` ซึ่งเป็นสเปกที่นิยมใช้รัน Redis เพราะตัว Redis เองมีสถาปัตยกรรมแบบ single-threaded อยู่แล้ว โดยรัน benchmark จากอินสแตนซ์ load-test อีกตัว (c5n) ใน AZ เดียวกัน ด้วยคำสั่ง `memtier_benchmark  -c 20 --test-time 100 -t 4 -d 256 --distinct-client-seed`

ผลลัพธ์ที่ได้ Dragonfly ทำงานได้ใกล้เคียงกัน:

1. SETs (`--ratio 1:0`):

|  Redis                                   |      DF                                |
| -----------------------------------------|----------------------------------------|
| QPS: 159K, P99.9: 1.16ms, P99: 0.82ms    | QPS:173K, P99.9: 1.26ms, P99: 0.9ms    |
|                                          |                                        |

2. GETs (`--ratio 0:1`):

|  Redis                                  |      DF                                |
| ----------------------------------------|----------------------------------------|
| QPS: 194K, P99.9: 0.8ms, P99: 0.65ms    | QPS: 191K, P99.9: 0.95ms, P99: 0.8ms   |

จากผล benchmark ข้างต้น จะเห็นว่า layer เชิง algorithm ที่ทำให้ DF สเกลแนวตั้ง (vertically) ได้นั้น แทบไม่ได้กินประสิทธิภาพเพิ่มเลยตอนรันแบบ single-threaded

แต่พอเปลี่ยนไปใช้อินสแตนซ์ที่แรงขึ้นอีกหน่อย (m5.xlarge) ช่องว่างระหว่าง DF กับ Redis เริ่มถ่างออก
(`memtier_benchmark  -c 20 --test-time 100 -t 6 -d 256 --distinct-client-seed`):
1. SETs (`--ratio 1:0`):

|  Redis                                  |      DF                                |
| ----------------------------------------|----------------------------------------|
| QPS: 190K, P99.9: 2.45ms, P99: 0.97ms   |  QPS: 279K , P99.9: 1.95ms, P99: 1.48ms|

2. GETs (`--ratio 0:1`):

|  Redis                                  |      DF                                |
| ----------------------------------------|----------------------------------------|
| QPS: 220K, P99.9: 0.98ms , P99: 0.8ms   |  QPS: 305K, P99.9: 1.03ms, P99: 0.87ms |


Throughput ของ Dragonfly ยังเพิ่มขึ้นได้เรื่อย ๆ ตามขนาดอินสแตนซ์ ในขณะที่ Redis แบบ single-threaded ติด bottleneck ที่ CPU แล้วก็ไปถึงจุดสูงสุดของตัวเองในแง่ performance

<img src="http://static.dragonflydb.io/repo-assets/aws-throughput.svg" width="80%" border="0"/>

ถ้าลองเทียบ Dragonfly กับ Redis บนอินสแตนซ์ที่รองรับ network ได้แรงที่สุดอย่าง c6gn.16xlarge Dragonfly ทำ throughput ได้มากกว่า Redis ถึง 25 เท่า ทะลุ 3.8M QPS เลย

ตัวเลข latency ที่ percentile 99 ของ Dragonfly ตอน throughput พีค:

| op    | r6g   | c6gn  | c7g   |
|-------|-------|-------|-------|
| set   | 0.8ms | 1ms   | 1ms   |
| get   | 0.9ms | 0.9ms | 0.8ms |
| setex | 0.9ms | 1.1ms | 1.3ms |

*benchmark ทั้งหมดรันด้วย `memtier_benchmark` (ดูตัวอย่างด้านล่าง) โดยปรับจำนวน thread ให้เหมาะกับแต่ละ server และ instance type ส่วน `memtier` เองรันแยกอยู่บนเครื่อง c6gn.16xlarge อีกตัวหนึ่ง เราตั้งเวลา expiry ไว้ที่ 500 สำหรับ benchmark ของ SETEX เพื่อให้ key ยังอยู่รอดจนจบการทดสอบ*

```bash
  memtier_benchmark --ratio ... -t <threads> -c 30 -n 200000 --distinct-client-seed -d 256 \
     --expiry-range=...
```

ในโหมด pipeline (`--pipeline=30`) Dragonfly ทำ throughput ได้ถึง **10M QPS** สำหรับ SET และ **15M QPS** สำหรับ GET

### Dragonfly vs. Memcached

เราเปรียบเทียบ Dragonfly กับ Memcached บนอินสแตนซ์ c6gn.16xlarge

ด้วย latency ที่ใกล้เคียงกัน throughput ของ Dragonfly เหนือกว่า Memcached ทั้งใน workload แบบ write และ read เลย ส่วนเรื่อง latency ในฝั่ง write นั้น Dragonfly ทำได้ดีกว่าเพราะปัญหาการแย่ง lock ใน [write path ของ Memcached](docs/memcached_benchmark.md)

#### SET benchmark

| Server    | QPS(thousands qps) | latency 99% | 99.9%   |
|:---------:|:------------------:|:-----------:|:-------:|
| Dragonfly |  🟩 3844           |🟩 0.9ms     | 🟩 2.4ms |
| Memcached |   806              |   1.6ms     | 3.2ms    |

#### GET benchmark

| Server    | QPS(thousands qps) | latency 99% | 99.9%   |
|-----------|:------------------:|:-----------:|:-------:|
| Dragonfly | 🟩 3717            |   1ms       | 2.4ms   |
| Memcached |   2100             |  🟩 0.34ms  | 🟩 0.6ms |


ฝั่ง Memcached latency ในการอ่านต่ำกว่า แต่ throughput ก็ต่ำกว่าด้วยเช่นกัน

### Memory efficiency

เพื่อทดสอบ memory efficiency เรายัดข้อมูลลง Dragonfly กับ Redis ราว ๆ 5GB ด้วยคำสั่ง `debug populate 5000000 key 1024` จากนั้นยิง update traffic เข้าไปด้วย `memtier` แล้วสั่ง snapshot ด้วยคำสั่ง `bgsave`

กราฟด้านล่างแสดงให้เห็นว่า server แต่ละตัวใช้หน่วยความจำต่างกันแค่ไหน

<img src="http://static.dragonflydb.io/repo-assets/bgsave-memusage.svg" width="70%" border="0"/>

ตอนอยู่เฉย ๆ (idle) Dragonfly ประหยัดหน่วยความจำกว่า Redis ถึง 30% แล้วก็แทบไม่เห็นหน่วยความจำเพิ่มขึ้นเลยระหว่างช่วง snapshot ต่างจาก Redis ที่พอถึงจุดพีค หน่วยความจำพุ่งขึ้นไปเกือบ 3 เท่าของ Dragonfly

แถม Dragonfly ยัง snapshot เสร็จเร็วกว่ามาก ใช้เวลาแค่ไม่กี่วินาที

ถ้าอยากรู้รายละเอียดเรื่อง memory efficiency ของ Dragonfly มากขึ้น ไปอ่านต่อได้ที่ [เอกสาร Dashtable](/docs/dashtable.md)



## <a name="configuration"><a/>Configuration

Dragonfly รองรับ argument ส่วนใหญ่ของ Redis เหมือนเดิม เช่น คุณสามารถรันคำสั่ง `dragonfly --requirepass=foo --bind localhost` ได้เลย

ตอนนี้ Dragonfly รองรับ argument เฉพาะของ Redis ดังนี้:
 * `port`: พอร์ตสำหรับเชื่อมต่อแบบ Redis (`ค่าเริ่มต้น: 6379`)
 * `bind`: ใช้ `localhost` เพื่อให้เชื่อมต่อได้เฉพาะจากเครื่องตัวเอง หรือใส่ public IP เพื่อให้เชื่อมต่อได้จาก **IP นั้น ๆ** (รวมถึงจากภายนอกด้วย) ส่วน `0.0.0.0` จะเปิดรับทุก IPv4
 * `requirepass`: password สำหรับยืนยันตัวตนผ่าน AUTH (`ค่าเริ่มต้น: ""`)
 * `maxmemory`: เพดานหน่วยความจำสูงสุด (หน่วยเป็น byte แบบอ่านง่าย) ที่ database จะใช้ (`ค่าเริ่มต้น: 0`) ถ้าตั้งเป็น `0` โปรแกรมจะคำนวณเพดานหน่วยความจำให้เองอัตโนมัติ
 * `dir`: Dragonfly เวอร์ชัน Docker ใช้โฟลเดอร์ `/data` สำหรับเก็บ snapshot เป็นค่าเริ่มต้น ส่วน CLI ใช้ `""` คุณใช้ออปชัน `-v` ของ Docker เพื่อ map ไปยังโฟลเดอร์บนเครื่อง host ได้
 * `dbfilename`: ชื่อไฟล์ที่ใช้ save/load database (`ค่าเริ่มต้น: dump`)

นอกจากนี้ยังมี argument เฉพาะของ Dragonfly เองอีกชุดหนึ่ง:
 * `memcached_port`: พอร์ตสำหรับเปิดใช้ API แบบ Memcached-compatible (`ค่าเริ่มต้น: disabled`)
 * `keys_output_limit`: จำนวน key สูงสุดที่คำสั่ง `keys` จะส่งกลับมาให้ (`ค่าเริ่มต้น: 8192`) เพราะ `keys` เป็นคำสั่งที่อันตรายพอสมควร เราเลย truncate ผลลัพธ์ไว้ ป้องกันหน่วยความจำพุ่งตอนดึง key จำนวนมาก ๆ
 * `dbnum`: จำนวน database สูงสุดที่รองรับสำหรับคำสั่ง `select`
 * `cache_mode`: ดูรายละเอียดที่หัวข้อ [novel cache design](#novel-cache-design) ด้านล่าง
 * `hz`: ความถี่ในการตรวจสอบ key ที่หมดอายุ (`ค่าเริ่มต้น: 100`) ยิ่งความถี่ต่ำ ยิ่งกิน CPU น้อยตอน idle แต่ก็จะ evict key ช้าลงตามไปด้วย
 * `snapshot_cron`: cron expression สำหรับตั้งเวลา backup snapshot อัตโนมัติ ใช้ syntax cron มาตรฐานที่ granularity ระดับนาที (`ค่าเริ่มต้น: ""`)
   ตัวอย่าง cron expression ดูได้ด้านล่างนี้ ส่วนรายละเอียดเพิ่มเติมอ่านได้ใน [เอกสารของเรา](https://www.dragonflydb.io/docs/managing-dragonfly/backups#the-snapshot_cron-flag)

   | Cron Schedule Expression | คำอธิบาย                                |
   |--------------------------|--------------------------------------------|
   | `* * * * *`              | ทุกนาที                            |
   | `*/5 * * * *`            | ทุก 5 นาที                        |
   | `5 */2 * * *`            | นาทีที่ 5 ของทุก 2 ชั่วโมง            |
   | `0 0 * * *`              | เที่ยงคืน (00:00) ของทุกวัน              |
   | `0 6 * * 1-5`            | 06:00 น. ของวันจันทร์ถึงศุกร์ |

 * `primary_port_http_enabled`: ถ้าตั้งเป็น `true` จะเข้าถึง HTTP console ผ่านพอร์ต TCP หลักได้เลย (`ค่าเริ่มต้น: true`)
 * `admin_port`: เปิด admin console ผ่านพอร์ตที่กำหนด (`ค่าเริ่มต้น: disabled`) รองรับทั้ง HTTP และ RESP
 * `admin_bind`: กำหนด address ที่จะ bind การเชื่อมต่อ TCP ของ admin console (`ค่าเริ่มต้น: any`) รองรับทั้ง HTTP และ RESP
 * `admin_nopass`: เปิด admin console แบบไม่ต้องใช้ token ยืนยันตัวตน (`ค่าเริ่มต้น: false`) รองรับทั้ง HTTP และ RESP
 * `cluster_mode`: โหมด cluster ที่รองรับ (`ค่าเริ่มต้น: ""`) ตอนนี้รองรับแค่ `emulated`
 * `cluster_announce_ip`: IP ที่คำสั่งของ cluster จะประกาศให้ client รู้
 * `announce_port`: พอร์ตที่คำสั่งของ cluster จะประกาศให้ client และ replication master รู้

### Example start script with popular options:

```bash
./dragonfly-x86_64 --logtostderr --requirepass=youshallnotpass --cache_mode=true -dbnum 1 --bind localhost --port 6379 --maxmemory=12gb --keys_output_limit=12288 --dbfilename dump.rdb
```

Argument พวกนี้ยังส่งผ่านช่องทางอื่นได้ด้วย:
 * `--flagfile <filename>`: ไฟล์นี้ต้องมี flag บรรทัดละหนึ่งตัว ใช้เครื่องหมาย `=` แทน space สำหรับ flag ที่มี key-value ไม่ต้องใส่ quote ครอบค่า
 * ตั้งเป็น environment variable โดยตั้งชื่อ `DFLY_x` โดย `x` คือชื่อ flag ตัวนั้น ๆ เป๊ะ ๆ (case sensitive)

ถ้าอยากรู้ option อื่น ๆ อีก เช่นการจัดการ log หรือการรองรับ TLS ลองรัน `dragonfly --help` ดูได้เลย


## <a name="design-decisions"><a/> Design decisions

### Novel cache design

Dragonfly มี caching algorithm แบบ adaptive ที่รวมเป็นชุดเดียว เรียบง่าย และประหยัดหน่วยความจำ

คุณเปิดโหมด cache ได้ด้วยการส่ง flag `--cache_mode=true` เมื่อเปิดโหมดนี้แล้ว Dragonfly จะ evict item ที่มีโอกาสถูกเรียกใช้ในอนาคตน้อยที่สุดออกไปก่อน แต่จะทำแบบนี้ก็ต่อเมื่อใกล้แตะเพดาน `maxmemory` เท่านั้น

### Expiration deadlines with relative accuracy

ช่วงเวลา expiration รองรับได้สูงสุดประมาณ ~8 ปี

Expiration deadline ที่ละเอียดถึงระดับ millisecond (เช่น PEXPIRE, PSETEX) จะถูกปัดให้เหลือหน่วยวินาทีที่ใกล้ที่สุด **สำหรับ deadline ที่มากกว่า 2^28ms** ซึ่งมี error น้อยกว่า 0.001% ถือว่ายอมรับได้สำหรับช่วงเวลาที่ยาว ๆ ถ้ากรณีของคุณใช้แบบนี้ไม่ได้ ติดต่อเราหรือเปิด issue อธิบาย use case มาได้เลย

ส่วนความแตกต่างอื่น ๆ ระหว่าง expiration deadline ของ Dragonfly กับ Redis [ดูเพิ่มเติมได้ที่นี่](docs/differences.md)

### Native HTTP console and Prometheus-compatible metrics

โดยค่าเริ่มต้น Dragonfly เปิดให้เข้าถึงผ่าน HTTP ได้จากพอร์ต TCP หลัก (6379) เลย พูดอีกอย่างคือคุณเชื่อมต่อ Dragonfly ผ่าน Redis protocol หรือ HTTP protocol ก็ได้ทั้งคู่ — server จะตรวจจับ protocol ให้เองตอนเริ่มเชื่อมต่อ ลองเปิดผ่าน browser ดูได้เลย ตอนนี้หน้า HTTP อาจยังมีข้อมูลไม่เยอะ แต่ในอนาคตจะมีข้อมูลสำหรับ debug และจัดการระบบเพิ่มเข้ามา

เข้า URL `:6379/metrics` เพื่อดู metric ที่รองรับ Prometheus

metric ที่ export ออกมาจาก Prometheus เข้ากันได้กับ Grafana dashboard [ดูตัวอย่างได้ที่นี่](tools/local/monitoring/grafana/provisioning/dashboards/dragonfly.json)


สำคัญ! HTTP console ควรเข้าถึงได้เฉพาะภายใน network ที่ปลอดภัยเท่านั้น ถ้าคุณเปิดพอร์ต TCP ของ Dragonfly ออกสู่ภายนอก แนะนำให้ปิด console นี้ด้วย `--http_admin_console=false` หรือ `--nohttp_admin_console`


## <a name="background"><a/>Background

Dragonfly เริ่มต้นจากการทดลองว่าถ้าจะออกแบบ in-memory datastore ขึ้นมาใหม่ในปี 2022 หน้าตาจะเป็นแบบไหน จากประสบการณ์ที่เราสั่งสมมาทั้งในฐานะผู้ใช้ memory store และวิศวกรที่เคยทำงานให้กับบริษัท cloud เรารู้ดีว่ามีคุณสมบัติหลักสองอย่างที่ Dragonfly ต้องรักษาไว้ให้ได้: การรับประกัน atomicity ของทุก operation และ latency ที่ต่ำระดับต่ำกว่า millisecond ในขณะที่ throughput ยังสูงมาก

โจทย์แรกคือทำยังไงให้ใช้ CPU, memory และ I/O ได้เต็มประสิทธิภาพบน server ที่หาได้ทั่วไปบน public cloud ทุกวันนี้ เราเลยเลือกใช้ [shared-nothing architecture](https://en.wikipedia.org/wiki/Shared-nothing_architecture) ซึ่งช่วยให้เราแบ่ง keyspace ของ memory store ออกเป็นส่วน ๆ ให้แต่ละ thread จัดการ dictionary data ของตัวเองได้ เราเรียกส่วนแบ่งพวกนี้ว่า `shard` ส่วน library ที่ดูแลเรื่อง thread กับ I/O สำหรับ shared-nothing architecture นี้ เรา open source ไว้ [ที่นี่](https://github.com/romange/helio)

สำหรับการรับประกัน atomicity ของ operation แบบ multi-key เรานำงานวิจัยล่าสุดมาปรับใช้ โดยเลือก paper ["VLL: a lock manager redesign for main memory database systems"](https://www.cs.umd.edu/~abadi/papers/vldbj-vll.pdf) มาพัฒนาต่อเป็น transactional framework ของ Dragonfly การเลือกใช้ shared-nothing architecture ร่วมกับ VLL ทำให้เรา compose atomic multi-key operation ได้โดยไม่ต้องพึ่ง mutex หรือ spinlock เลย นี่ถือเป็น milestone สำคัญของ PoC ตอนนั้น แล้ว performance ของมันก็โดดเด่นกว่า solution อื่น ๆ ทั้งฝั่ง commercial และ open-source

โจทย์ที่สองคือการออกแบบ data structure ให้มีประสิทธิภาพมากขึ้นสำหรับ store ตัวใหม่นี้ เพื่อให้ถึงเป้าหมายนี้ เราเอา core hashtable structure มาอิง paper ["Dash: Scalable Hashing on Persistent Memory"](https://arxiv.org/pdf/2003.07302.pdf) ตัว paper เองเน้นเรื่อง persistent memory เป็นหลัก ไม่ได้พูดถึง main-memory store เท่าไหร่นัก แต่ก็ยังเอามาปรับใช้กับโจทย์ของเราได้ดีที่สุด การออกแบบ hashtable ใน paper นี้ช่วยให้เรารักษาคุณสมบัติพิเศษสองอย่างที่มีอยู่ใน Redis dictionary ไว้ได้: ความสามารถในการทำ incremental hashing ระหว่าง datastore ขยายขนาด และความสามารถในการไล่ traverse dictionary ระหว่างที่มันเปลี่ยนแปลงอยู่ด้วย stateless scan operation นอกจาก 2 คุณสมบัตินี้แล้ว Dash ยังประหยัด CPU และ memory กว่าเดิมด้วย พอต่อยอดจาก design ของ Dash เราก็ต่อยอด feature เพิ่มเติมได้อีก:
 * ทำ record expiry สำหรับ TTL record ได้อย่างมีประสิทธิภาพ
 * cache eviction algorithm แบบใหม่ ที่ hit rate สูงกว่า strategy เดิมอย่าง LRU และ LFU โดยไม่มี memory overhead เพิ่มเลยแม้แต่นิดเดียว
 * snapshotting algorithm แบบ **fork-less** ตัวใหม่

พอวางรากฐานของ Dragonfly เสร็จ แล้ว[พอใจกับ performance ที่ได้](#benchmarks) เราก็เดินหน้าต่อไปทำ Redis กับ Memcached functionality จนถึงตอนนี้เราทำ Redis command ไปแล้วประมาณ 185 คำสั่ง (เทียบเท่า Redis 5.0 API โดยประมาณ) และ Memcached command อีก 13 คำสั่ง

แล้วสุดท้าย, <br>
<em>ภารกิจของเราคือสร้าง in-memory datastore ที่ออกแบบมาอย่างดี เร็วสุด ๆ และคุ้มค่าใช้จ่ายสำหรับ cloud workload โดยใช้ประโยชน์จาก hardware รุ่นใหม่ล่าสุดให้เต็มที่ เราตั้งใจแก้ pain point ของ solution ที่มีอยู่ในตอนนี้ พร้อมกับรักษา API และจุดขายเดิมของมันไว้</em>

## <a name="contributors"><a/>Contributors

ขอบคุณผู้ร่วมพัฒนาโปรเจกต์ Dragonfly ทุกคนเลย!

<a href="https://github.com/dragonflydb/dragonfly/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=dragonflydb/dragonfly" />
</a>
