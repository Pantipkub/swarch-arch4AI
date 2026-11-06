# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ: นาย ปิยปาณชนก มีบุญสล้าง**

**Student ID / รหัสนิสิต: 6630196021**

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 504 | 612 | 93.5% | 100% |
| t0-small | 33 | 85 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> large ซื้อ accuracy ได้เยอะมาก จาก 72.5% (`t0-small`) ขึ้นเป็น 93.5% (`t0-large`) แต่แลกกับ latency ที่ช้าลงประมาณ 15 เท่า (p50 33 → 504 ms, p95 85 → 612 ms) แถมทุก call เสียเงิน และรับได้แค่ 20 calls/s

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 30.3 | 247 | 414 | 21 | 0 | 92.7% |
| t1-after | 9.8 | 402 | 1018 | 0 | 28 | 93.0% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> p50 คิดจากเฉพาะ request ที่สำเร็จ ส่วน 21 request ที่ค้างจนโดน timeout ถูกนับเป็น errors และไม่ได้อยู่ใน p50 เลย ตัวที่ค้างก็ไม่ได้แย่ง large model กับคนอื่นด้วย ตัวที่เหลือเลยดูเร็วขึ้น column ที่บอกความจริงคือ `errors` (21 จาก 200) กับ run time ที่ยืดจาก ~10 s เป็น 30.3 s

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> request ที่ค้างไม่ได้แค่ช้าคนเดียว มันจอง connection กับ coroutine ใน review service (timeout=None) และ gateway ไว้นานถึง 20 วินาที ทำให้ slot ที่มีจำกัด (concurrency 10) ถูกกินไปเรื่อยๆ request ปกติก็ต้องรอคิวตามไปด้วย เลยกลายเป็นทั้งระบบช้า (run time 30.3 s) ไม่ใช่แค่ 10% ที่ช้า

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> ควรเก็บรีวิว pending ไว้ แล้วให้ batch job มาดึงไปติด label ทีหลังตอนที่ large model กลับมาปกติ ก็คือ topology แบบ Asynchronous / batch (T4) ที่เอาเข้าคิวหรือเก็บไว้แล้วค่อย score ทีหลังเป็นชุด

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> ถ้า 0.1 s มันสั้นกว่าเวลาปกติของ large (~200 ms) ไปแล้ว เกือบทุกรีวิวจะกลายเป็น pending ทั้งที่ model ไม่ได้มีปัญหาอะไร ถ้า 10 s ก็จะได้ผลเกือบเหมือน `t1-before` คือ request ค้างนานจนกิน connection หมด และผิด QR2 ที่ให้รอไม่เกิน 1 วินาที

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 10 | 555 | 91.5% | 38% |
| t2-0.7 | 0.7 | 396 | 622 | 93.0% | 53% |
| t2-0.8 | 0.8 | 503 | 716 | 93.5% | 64% |
| t2-0.9 | 0.9 | 483 | 590 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก 0.6 เพราะ accuracy 91.5% ยังผ่าน QR3 (≥ 90%) และเรียก large แค่ 38% ซึ่งน้อยสุดตาม QR4 แถม p50 เหลือ 10 ms กับ p95 555 ms ก็ผ่าน QR1 สิ่งที่ยอมเสียคือ accuracy ประมาณ 2% เทียบกับ large ล้วน (93.5%)

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> ที่ 0.6 มีแค่ 38% ที่ไป large เกินครึ่งเลยได้คำตอบจาก small ทันที p50 เลยลงเหลือ 10 ms แต่ p95 ดู 5% ที่ช้าสุด ซึ่งก็ยังเป็นพวกที่ไป large อยู่ดี p95 เลยค้างอยู่แถว 555 ms เท่ากับ latency ของ large (ดูได้จาก 0.7 ที่ large calls 53% เกินครึ่ง p50 ก็กระโดดไป 396 ms ทันที)

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> small model เป็นแค่ bag-of-words เลยอ่านพวกปฏิเสธหรือประโยคขัดแย้งพลาด เช่น "not bad" หรือ "food was good but delivery was slow" แต่ยังให้ confidence สูง cascade ก็จะเชื่อแล้วไม่ส่งต่อ large ผลคือร้านได้ label ผิด เช่นรีวิวที่ติจริงๆ กลับนับเป็น positive

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache | 10 | 565 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> ได้ประโยชน์คือรีวิวสั้นๆ ที่คนพิมพ์ซ้ำกันบ่อย เช่น "Delicious!", "Too salty" (หลัง lowercase และตัด space แล้วต้องตรงกันเป๊ะ) ส่วนรีวิวยาวๆ ที่เขียนไม่เหมือนใคร จะไม่มีวัน hit เลย เพราะ key ต้องตรงทั้งข้อความ

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้อง invalidate cache ทิ้ง ไม่งั้นจะยังตอบด้วยผลจาก model เก่าอยู่ ออกแบบโดยใส่ model version ไว้ใน key เช่น `(model_version, text)` พอ deploy ตัวใหม่ key ก็เปลี่ยนเอง ของเก่าก็โดน LRU ไล่ออกไปเอง หรือจะตั้ง TTL เสริมไว้ด้วยก็ได้

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> p50 ไม่ได้ค่อยๆ ลดตาม large calls แต่มันกระโดด พอ large calls ต่ำกว่า 50% (0.6 = 38%) p50 ลงจาก ~400 เหลือ 10 ms เลย แต่ p95 แทบไม่ขยับ ถ้าดูแค่ p50 อย่างเดียวจะหลงคิดว่าระบบเร็วขึ้นสำหรับทุกคน
