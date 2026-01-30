# Disaster Recovery Guide

คู่มือการกู้คืนระบบเมื่อเกิดปัญหา Key Loss ใน Tempo Validator Network

---

## ⚠️ สิ่งสำคัญที่ต้องเข้าใจก่อน

### ความสัมพันธ์ของ Keys

```
signing.key (Ed25519 private key)
    ↓ derive
public key (Ed25519 public key) ← ใช้เป็น identity ใน DKG
    ↓
Validator Address (Ethereum address) ← ใช้ใน contract
```

### DKG ทำงานยังไง?

```
1. DKG อ่าน validators จาก ValidatorConfig contract
   → ได้ list: [(public_key, address, status, inbound, outbound)]

2. Dealers (validators ปัจจุบัน) สร้าง signing shares
   → สำหรับแต่ละ public_key ที่ active=true

3. ส่ง shares ผ่าน P2P
   → โดยใช้ public_key เป็น node identifier

4. Validators รับ shares และรวมเป็น group key
```

### ถ้า `signing.key` หาย?

```
✗ สร้าง signing.key ใหม่ → ได้ public key ใหม่
✗ public key ใหม่ ≠ public key ใน contract
✗ Dealers ส่ง shares ไปที่ public key เก่า (ที่หายไป)
✗ Validator ใหม่ไม่ได้รับ shares

ดังนั้น: ต้อง update public key ใน contract!
```

### แต่... update ยังไง?

| Function | ใช้ key อะไร sign | ใช้ได้เมื่อไหร่ |
|----------|------------------|---------------|
| `updateValidator` | **Validator's own key** | ถ้ามี key อยู่ |
| `changeValidatorStatus` | **Admin key** | ตลอด (ถ้า network ทำงาน) |
| `addValidator` | **Admin key** | ตลอด (ถ้า network ทำงาน) |

**สรุป:** ถ้า key หายหมด → ใช้ `updateValidator` ไม่ได้! ต้องใช้ **admin** หรือ **genesis ใหม่**

---

## 🔴 กรณีที่ 1: Key 1 Node หาย (มี Backup)

### สถานการณ์
- ✅ Validator 1, 2: ทำงานปกติ
- ⚠️ Validator 3: `signing.key` และ `signing.share` หาย
- 🟡 Network: ยังทำงาน (เหลือ 2 signers > threshold)

### วิธีแก้: Restore from Backup

```bash
# 1. Copy signing.share จาก backup
scp /backup/validator3/signing.share root@89.167.21.24:/root/tempo-network/89.167.21.24:3000/

# 2. Restart validator
ssh root@89.167.21.24 '
  cd /root/tempo-network
  pkill -f tempo
  ./tempo node \
    --consensus.signing-key ./89.167.21.24:3000/signing.key \
    --consensus.signing-share ./89.167.21.24:3000/signing.share \
    --consensus.listen-address 0.0.0.0:3000 \
    --chain ./genesis.json \
    --datadir ./data \
    --port 3001 \
    --p2p-secret-key ./89.167.21.24:3000/enode.key \
    --consensus.fee-recipient 0x90F79bf6EB2c4f870365E785982E1f101E93b906 \
    --trusted-peers "enode://<V1_ENODE>@46.225.53.92:3001,enode://<V2_ENODE>@89.167.25.42:3001" \
    --http --http.port 8545 --http.addr 0.0.0.0
'
```

**ผลลัพธ์:** ✅ กลับมาทำงานทันที (มี signing share เดิม)

---

## 🔴 กรณีที่ 2: Key 1 Node หาย (ไม่มี Backup) + Network ทำงาน

### สถานการณ์
- ✅ Validator 1, 2: ทำงานปกติ (สร้าง blocks ได้)
- ❌ Validator 3: `signing.key` และ `signing.share` หาย ไม่มี backup
- 🟡 Network: ยังทำงาน

### วิธีแก้: ใช้ Admin เพิ่มใหม่

```bash
export PATH="$HOME/.foundry/bin:$PATH"

# 1. Deactivate validator เก่า (ใช้ admin key)
cast send 0xCccCcCCC00000000000000000000000000000000 \
"changeValidatorStatus(address validator, bool active)" \
"0x90F79bf6EB2c4f870365E785982E1f101E93b906" \
"false" \
--private-key <ADMIN_KEY> \
-r http://localhost:8545

# 2. สร้าง key ใหม่สำหรับ Validator 3
cd /root/tempo-network
./tempo consensus generate-private-key --output ./89.167.21.24:3000/signing.key
NEW_PUBKEY=$(./tempo consensus calculate-public-key --private-key ./89.167.21.24:3000/signing.key | grep "public key:" | awk '{print $3}')

echo "New public key: $NEW_PUBKEY"

# 3. Add validator ใหม่ด้วย public key ใหม่ (ใช้ admin key)
cast send 0xCccCcCCC00000000000000000000000000000000 \
"addValidator(address newValidatorAddress, bytes32 publicKey, bool active, string calldata inboundAddress, string calldata outboundAddress)" \
"0x90F79bf6EB2c4f870365E785982E1f101E93b906" \
"$NEW_PUBKEY" \
"true" \
"89.167.21.24:3000" \
"89.167.21.24:3000" \
--private-key <ADMIN_KEY> \
-r http://localhost:8545

# 4. Start validator รอ DKG
ssh root@89.167.21.24 '
  cd /root/tempo-network
  ./tempo node \
    --consensus.signing-key ./89.167.21.24:3000/signing.key \
    --consensus.listen-address 0.0.0.0:3000 \
    --chain ./genesis.json \
    --datadir ./data \
    --port 3001 \
    --p2p-secret-key ./89.167.21.24:3000/enode.key \
    --consensus.fee-recipient 0x90F79bf6EB2c4f870365E785982E1f101E93b906 \
    --trusted-peers "enode://<V1_ENODE>@46.225.53.92:3001,enode://<V2_ENODE>@89.167.25.42:3001" \
    --http --http.port 8545 --http.addr 0.0.0.0
'

# 5. รอ DKG ceremony (1-2 epochs)
# Validator 3 จะได้รับ signing share ใหม่
```

**ผลลัพธ์:** ✅ Validator 3 กลับมาเป็น signer ใน 1-2 epochs

**ทำไมใช้ `addValidator` ได้:** เพราะใช้ **admin key** sign ไม่ต้องใช้ key ของ Validator 3!

---

## 🔴 กรณีที่ 3: Key 1 Node หาย (ไม่มี Backup) + Network Halt

### สถานการณ์
- ❌ Validator 1: มี signing share แต่ไม่ถึง threshold
- ❌ Validator 2, 3: key หาย (ไม่มี backup)
- 🛑 Network: **Halt** (ไม่สร้าง blocks)

### ⚠️ ปัญหาใหญ่

```
Network Halt → ไม่มี blocks ใหม่
            → Admin transactions ค้างใน mempool
            → ไม่สามารถ deactivate/add validator ได้!
```

### วิธีแก้: Genesis ใหม่ (เท่านั้น!)

```bash
# 1. Stop ทุก node
ssh root@46.225.53.92 "pkill -9 -f tempo"
ssh root@89.167.25.42 "pkill -9 -f tempo"  
ssh root@89.167.21.24 "pkill -9 -f tempo"

# 2. Generate genesis ใหม่ (เหลือแค่ Validator 1)
cd /root/tempo
cargo run -p tempo-xtask -- generate-genesis \
  --output /root/tempo-network/genesis-recovery \
  --chain-id 1338 \
  --validators "46.225.53.92:3000" \
  --accounts 10 \
  --epoch-length 50 \
  --validator-admin 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

# 3. Deploy ไปทุก node
scp /root/tempo-network/genesis-recovery/genesis.json root@89.167.25.42:/root/tempo-network/
scp /root/tempo-network/genesis-recovery/genesis.json root@89.167.21.24:/root/tempo-network/

# 4. Clean data ทุก node
for ip in 46.225.53.92 89.167.25.42 89.167.21.24; do
  ssh root@$ip "rm -rf /root/tempo-network/data /root/.cache/reth /root/.local/share/reth"
done

# 5. Start Validator 1
ssh root@46.225.53.92 'cd /root/tempo-network && ./tempo node ...'

# 6. เพิ่ม Validator 2, 3 ใหม่ตามขั้นตอน dynamic addition
```

**ข้อเสีย:** ❌ สูญเสียข้อมูลทั้งหมด (blocks, transactions, state)

---

## 🔴 กรณีที่ 4: Key 2 Nodes หาย

### สถานการณ์ A: มี Backup
- ✅ Restore backup ของทั้ง 2 nodes
- ✅ Network กลับมาทำงานทันที

```bash
# Restore Validator 2
scp /backup/validator2/signing.share root@89.167.25.42:/root/tempo-network/89.167.25.42:3000/
ssh root@89.167.25.42 "cd /root/tempo-network && ./tempo node ..."

# Restore Validator 3  
scp /backup/validator3/signing.share root@89.167.21.24:/root/tempo-network/89.167.21.24:3000/
ssh root@89.167.21.24 "cd /root/tempo-network && ./tempo node ..."
```

### สถานการณ์ B: ไม่มี Backup + Network ทำงาน
- 🟡 Validator 1: เหลือ signer เดียว (ยังทำงานได้)
- ❌ Validator 2, 3: key หาย

```bash
# ใช้วิธีเดียวกับกรณีที่ 2 (admin add new)
# แต่ทำทั้ง 2 nodes

# Deactivate ทั้ง 2
cast send ... "changeValidatorStatus" "0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc" "false" ...
cast send ... "changeValidatorStatus" "0x90f79bf6eb2c4f870365e785982e1f101e93b906" "false" ...

# สร้าง key ใหม่ทั้ง 2
# Add ใหม่ด้วย admin
# รอ DKG
```

### สถานการณ์ C: ไม่มี Backup + Network Halt
- 🛑 Validator 1: คนเดียวไม่ถึง threshold
- ❌ Validator 2, 3: key หาย
- 🛑 Network: **Halt**

**ผล:** ใช้ admin ไม่ได้ (ไม่มี blocks) → ต้อง **Genesis ใหม่** เท่านั้น!

---

## 📊 สรุป: Recovery Matrix

| กรณี | Network | มี Backup | วิธีแก้ | เวลา | ข้อมูลสูญหาย |
|------|---------|----------|--------|------|-------------|
| 1 node หาย | ✅ Running | ✅ Yes | Restore backup | 5 min | 0 |
| 1 node หาย | ✅ Running | ❌ No | Admin deactivate + add new | 10 min | 0 |
| 1 node หาย | 🛑 Halt | ❌ No | Genesis ใหม่ | 30 min | **ทั้งหมด** |
| 2 nodes หาย | ✅ Running | ✅ Yes | Restore both | 10 min | 0 |
| 2 nodes หาย | ✅ Running | ❌ No | Admin add ใหม่ 2 nodes | 15 min | 0 |
| 2 nodes หาย | 🛑 Halt | ❌ No | Genesis ใหม่ | 30 min | **ทั้งหมด** |
| 3 nodes หาย | - | ✅ Yes | Restore all | 10 min | 0 |
| 3 nodes หาย | - | ❌ No | Genesis ใหม่ | 30 min | **ทั้งหมด** |

---

## 🛡️ Best Practices: ป้องกัน Key Loss

### 1. Backup อย่างน้อย 3 ที่

```bash
#!/bin/bash
# backup-validator.sh

VALIDATOR="89.167.21.24:3000"
DATE=$(date +%Y%m%d_%H%M%S)

# 1. Local encrypted
mkdir -p /secure/backup/$VALIDATOR
cp /root/tempo-network/$VALIDATOR/signing.share /secure/backup/$VALIDATOR/signing.share.$DATE
cp /root/tempo-network/$VALIDATOR/signing.key /secure/backup/$VALIDATOR/signing.key.$DATE
gpg --symmetric --cipher-algo AES256 /secure/backup/$VALIDATOR/signing.share.$DATE

# 2. Remote server
scp /secure/backup/$VALIDATOR/signing.share.$DATE.gpg backup-server:/backups/tempo/

# 3. Offline (USB/Hardware wallet)
# คัดลอกไฟล์ .gpg ลง USB แล้วเก็บในที่ปลอดภัย
```

### 2. ตรวจสอบสุขภาพระบบ

```bash
#!/bin/bash
# health-check.sh

export PATH="$HOME/.foundry/bin:$PATH"

# Check block height
for ip in 46.225.53.92 89.167.25.42 89.167.21.24; do
  BLOCK=$(curl -s -m 5 http://$ip:8545 \
    -X POST -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
    | grep -o '0x[0-9a-f]*' | tail -1)
  
  if [ -z "$BLOCK" ]; then
    echo "ALERT: Validator $ip is DOWN!"
    # Send notification
    curl -X POST -H 'Content-Type: application/json' \
      -d '{"text":"Validator '$ip' is DOWN!"}' \
      $SLACK_WEBHOOK
  fi
done

# Check validator count
COUNT=$(cast call 0xCccCcCCC00000000000000000000000000000000 \
  "validatorCount()" \
  -r http://localhost:8545 2>/dev/null)

echo "Active validators: $COUNT"
```

### 3. Key Rotation Policy

```bash
# หมุนเวียน keys ทุก 90 วัน
# 1. Generate new keys
# 2. Add new validator (ผ่าน admin)
# 3. รอ DKG (1 epoch)
# 4. Deactivate validator เก่า
# 5. Archive keys เก่า (เก็บไว้อย่างน้อย 1 ปี)
```

---

## 🔑 สิ่งที่ต้อง Backup

| ไฟล์ | ความสำคัญ | เก็บที่ไหน | หมายเหตุ |
|------|-----------|-----------|----------|
| `signing.share` | ⭐⭐⭐ Critical | Encrypted, 3+ locations | ไม่มี = ไม่สามารถ sign ได้ |
| `signing.key` | ⭐⭐⭐ Critical | Encrypted, 3+ locations | Identity ของ validator |
| `genesis.json` | ⭐⭐⭐ Critical | Git, Multiple servers | Chain specification |
| `enode.key` | ⭐⭐ Important | Local | สร้างใหม่ได้ แต่จะเปลี่ยน enode URL |
| `data/` | ⭐ Not critical | - | Resync ได้ |

---

## ❌ วิธีที่ใช้ไม่ได้ (และเหตุผล)

### ❌ `updateValidator` ถ้า key หาย
```bash
# ใช้ไม่ได้! เพราะต้องใช้ private key ของ validator sign

--private-key <VALIDATOR_3_OLD_PRIVATE_KEY>
# ↑ ถ้า key หาย = ไม่มี private key = sign ไม่ได้!
```

### ❌ Admin transactions ถ้า network halt
```bash
# ใช้ไม่ได้! เพราะ transaction ต้องรอ include ใน block
# ถ้า network halt = ไม่มี block = transaction ค้างตลอดกาล

cast send ... "changeValidatorStatus" ...
# Transaction submitted... (รอ confirm ไม่มีวันได้)
```

### ❌ DKG "รู้เอง"
```
DKG ไม่ใช่ AI!
DKG อ่านจาก contract ตามที่บันทึกไว้เท่านั้น
ถ้า public key ไม่ตรง = ไม่ได้รับ shares
```

---

## 📞 Emergency Contacts & Commands

### ตรวจสอบสถานะด่วน

```bash
# Check blocks
curl -s http://localhost:8545 -X POST \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# Check validators
cast call 0xCccCcCCC00000000000000000000000000000000 \
  "validatorCount()" \
  -r http://localhost:8545

# Check DKG
grep "enter_epoch" /root/tempo-network/validator.log | tail -3
```

### Emergency Restore

```bash
# 1. Stop node
pkill -f tempo

# 2. Restore from backup
cp /backup/validator/signing.share /root/tempo-network/<ip>:3000/

# 3. Start node
./tempo node ...
```

---

*สร้างเมื่อ: January 30, 2026*  
*อัปเดตล่าสุด: January 30, 2026*  
*เวอร์ชัน: 2.0 (แก้ไขข้อผิดพลาดครั้งใหญ่)*
