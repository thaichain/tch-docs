# DKG Ceremony Deep Dive

อธิบายลึกว่า DKG ทำงานยังไง และความสัมพันธ์ระหว่าง signing.key กับ signing.share

---

## 🔑 ความเข้าใจผิดที่พบบ่อย

### ❌ ความเข้าใจผิด
```
"DKG สร้าง signing.key ให้ validator"
"ถ้า signing.key หาย DKG จะสร้างใหม่ให้"
```

### ✅ ความจริง
```
signing.key (Ed25519)    → สร้างเอง ใช้เป็น identity
signing.share (BLS12-381) → ได้จาก DKG ใช้ sign blocks

DKG ไม่ได้สร้าง signing.key!
DKG สร้าง BLS "threshold shares" สำหรับ sign blocks ร่วมกัน
```

---

## 🎭 บทบาทของแต่ละ Key

### 1. `signing.key` (Ed25519 Private Key)

```
สร้างโดย: Validator สร้างเอง (./tempo consensus generate-private-key)
ใช้สำหรับ:
  - สร้าง public key (identity ในระบบ)
  - Sign P2P messages (ระหว่าง validators)
  - Sign transactions (ถ้าใช้ updateValidator)

ไม่ใช้สำหรับ: Sign blocks ใน consensus!
```

### 2. `signing.share` (BLS12-381 Share)

```
สร้างโดย: DKG Ceremony (ได้รับจาก dealers)
ใช้สำหรับ:
  - Sign blocks ใน consensus (threshold signature)
  - ร่วมกับ validators อื่นเพื่อสร้าง group signature

ไม่ใช่: Private key เต็ม! เป็นส่วนหนึ่งของ threshold scheme
```

---

## ⚙️ DKG ทำงานยังไง?

### ขั้นตอนการทำงาน

```
Epoch N (เริ่มต้น)
├── 1. อ่าน validators จาก contract
│   → ได้: [(public_key_1, addr_1), (public_key_2, addr_2), ...]
│   → public_key คือ Ed25519 public key ที่ derive จาก signing.key
│
├── 2. Dealers (validators ปัจจุบัน) สร้าง BLS polynomial
│   → แต่ละ dealer สร้าง polynomial ของตัวเอง
│   → f(x) = a₀ + a₁x + a₂x² + ... + aₜxᵗ
│   → โดยที่ a₀ = random secret (ไม่ใช่ signing.key!)
│
├── 3. คำนวณ shares
│   → Dealer คำนวณ f(1), f(2), f(3), ... สำหรับแต่ละ player
│   → f(1) = share สำหรับ validator 1
│   → f(2) = share สำหรับ validator 2
│   → ...
│
├── 4. ส่ง shares ผ่าน P2P
│   → ส่งไปยัง public_key ที่ระบุใน contract
│   → ใช้ Ed25519 signing.key ของตัวเอง sign P2P message
│   → แต่ส่ง BLS shares!
│
└── 5. Players รวม shares
    → รวม shares จากทุก dealers → group polynomial
    → ได้ signing.share (BLS) สำหรับ sign blocks
```

### สิ่งสำคัญ

```
Ed25519 signing.key  ≠  BLS signing.share

 signing.key (Ed25519)              signing.share (BLS)
       ↓                                   ↓
  ใช้ sign P2P messages             ใช้ sign blocks
  ใช้เป็น identity                   ใช้ใน consensus
  สร้างเอง                           ได้จาก DKG
  1 key ต่อ validator                1 share ต่อ validator ต่อ epoch
```

---

## 🔬 เปรียบเทียบ Cryptography

### Ed25519 ( signing.key )

```
ประเภท: Digital Signature Algorithm
ใช้สำหรับ: Identity + P2P communication
Key format: 64 bytes private, 32 bytes public
สร้างโดย: Validator (random)
```

### BLS12-381 ( signing.share )

```
ประเภท: Threshold Signature Scheme
ใช้สำหรับ: Consensus block signing
Share format: 96 bytes (BLS12-381 G1)
สร้างโดย: DKG Ceremony (polynomial evaluation)
คุณสมบัติ: Aggregatable (รวม signatures ได้)
```

---

## 📋 ตัวอย่างการทำงานจริง

### สร้าง Validator ใหม่

```bash
# 1. Validator สร้าง identity ของตัวเอง
./tempo consensus generate-private-key --output signing.key
# ได้: signing.key (Ed25519)

./tempo consensus calculate-public-key --private-key signing.key  
# ได้: public key (Ed25519) เช่น 0x14902d2fa87f570c2a48a397bc144a2805c20e4a7c62c64457af0f08dcf408cd

# 2. Admin add validator เข้า contract
cast send ... "addValidator" \
  "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC" \
  "0x14902d2fa87f570c2a48a397bc144a2805c20e4a7c62c64457af0f08dcf408cd" \
  ...

# 3. Validator เริ่มทำงาน
./tempo node \
  --consensus.signing-key signing.key \
  --consensus.signing-share ???  # ← ยังไม่มี!

# 4. รอ DKG ceremony
# Dealers สร้าง BLS polynomial
# ส่ง BLS share มาให้ (ผ่าน P2P)
# ได้: signing.share (BLS12-381)

# 5. ตอนนี้ validator มีทั้งสองอย่าง:
#    - signing.key (Ed25519) = identity + sign P2P
#    - signing.share (BLS) = sign blocks
```

### DKG Logs ที่เห็น

```
# Validator 1 (Dealer + Player)
run_dkg_loop: entering a new DKG ceremony
  me=85ebe227deab39e7d75139e64e82f5e075f4aceca44ae4bdb044a2e2a78c5ddc
  dealers=[Validator 1, Validator 2]
  players=[Validator 1, Validator 2, Validator 3]
  as_dealer=true, as_player=true

# Validator 3 (Player อย่างเดียวตอนแรก)  
run_dkg_loop: entering a new DKG ceremony
  me=14902d2fa87f570c2a48a397bc144a2805c20e4a7c62c64457af0f08dcf408cd
  dealers=[Validator 1, Validator 2]
  players=[Validator 1, Validator 2, Validator 3]
  as_dealer=false, as_player=true  ← ได้รับ shares จาก dealers
  
# Validator 3 หลังได้รับ shares
enter_epoch: we have a share for this epoch, participating as a signer
  public=Sharing { total: 3, poly: Poly(...) }
```

---

## ❓ คำถามที่พบบ่อย

### Q: DKG สร้าง signing.key ให้ไหม?

**A:** ไม่! signing.key สร้างเองก่อนเข้าร่วม network ใช้เป็น identity

### Q: ถ้า signing.key หาย DKG ช่วยได้ไหม?

**A:** ไม่ได้! DKG ไม่ได้เก็บหรือสร้าง signing.key ต้องสร้างใหม่เอง และ update public key ใน contract

### Q: signing.share ใช้แทน signing.key ได้ไหม?

**A:** ไม่ได้! ใช้งานคนละอย่าง:
- signing.key → Ed25519 → P2P identity
- signing.share → BLS → Block signing

### Q: ทำไมต้องมี 2 ระบบ?

**A:** 
- Ed25519: เร็ว ใช้ P2P communication
- BLS: Aggregatable ใช้ consensus (รวม signatures จากหลาย validators ได้)

### Q: DKG รู้ private key ของเราไหม?

**A:** ไม่รู้! DKG:
- รู้แค่ public key (Ed25519) จาก contract
- สร้าง BLS polynomial แยกต่างหาก
- ส่ง BLS shares ผ่าน P2P (ที่ sign ด้วย Ed25519)

---

## 🔄 สรุป Flow

```
สร้าง Validator:
  1. สร้าง signing.key (Ed25519) → ได้ public key
  2. Admin add เข้า contract ด้วย public key
  3. Start node รอ DKG
  4. DKG สร้าง BLS polynomial
  5. ได้รับ signing.share (BLS) จาก dealers
  6. ตอนนี้มีทั้ง 2 อย่าง:
     - signing.key → identity
     - signing.share → sign blocks

Key หาย:
  signing.key หาย → สร้างใหม่ + update public key ใน contract
  signing.share หาย → รอ DKG รอบถัดไป (จะได้รับใหม่)
```

---

*สร้างเมื่อ: January 30, 2026*
