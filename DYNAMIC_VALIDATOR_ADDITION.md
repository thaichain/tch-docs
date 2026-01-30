# Dynamic Validator Addition Guide

การเพิ่ม Validator เข้าไปในเครือข่าย Tempo แบบ dynamic (หลังจาก genesis แล้ว)

## ภาพรวม

Tempo รองรับการเพิ่ม validator แบบ dynamic ผ่านกลไก:
- **ValidatorConfig Precompile Contract** (`0xCccCcCCC00000000000000000000000000000000`)
- **DKG (Distributed Key Generation) Ceremony** ที่รันทุก epoch
- On-chain validator set management

---

## ผลการทดสอบ

| Node | IP Address | สถานะ | บทบาท |
|------|-----------|--------|--------|
| Validator 1 | 46.225.53.92 | ✅ Active | Dealer + Player + Signer |
| Validator 2 | 89.167.25.42 | ✅ Active | Dealer + Player + Signer |
| Validator 3 | 89.167.21.24 | ✅ Active | Player + Signer |

**จำนวน Validators:** 3 ตัว  
**Block Height:** ~1,900+ blocks  
**สถานะ:** ทุก node sync กันและผลิต block ปกติ

---

## ขั้นตอนการทำ

### Phase 1: สร้าง Genesis ด้วย 1 Validator

```bash
cd /root/tempo

# สร้าง genesis พร้อม 1 validator และกำหนด admin
cargo run -p tempo-xtask -- generate-genesis \
  --output /root/tempo-network/genesis-1val \
  --chain-id 1337 \
  --validators "46.225.53.92:3000" \
  --accounts 10 \
  --epoch-length 50 \
  --validator-admin 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266

# Copy ไฟล์ที่สร้าง
cp /root/tempo-network/genesis-1val/genesis.json /root/tempo-network/genesis-dynamic.json
cp -r /root/tempo-network/genesis-1val/46.225.53.92:3000 /root/tempo-network/
```

**หมายเหตุ:** 
- `--epoch-length 50` = 50 blocks ต่อ epoch (~4-5 นาที)
- `--validator-admin` = address ที่มีสิทธิ์เพิ่ม/ลบ validator

---

### Phase 2: รัน Validator 1 (Bootnode)

```bash
cd /root/tempo-network
export RUST_LOG=info

# Clean data
rm -rf /root/tempo-network/data /root/.cache/reth /root/.local/share/reth
mkdir -p /root/tempo-network/data

# Start Validator 1
./tempo node \
  --consensus.signing-key ./46.225.53.92:3000/signing.key \
  --consensus.signing-share ./46.225.53.92:3000/signing.share \
  --consensus.listen-address 0.0.0.0:3000 \
  --chain ./genesis-dynamic.json \
  --datadir ./data \
  --port 3001 \
  --p2p-secret-key ./46.225.53.92:3000/enode.key \
  --consensus.fee-recipient 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 \
  --http --http.port 8545 --http.addr 0.0.0.0
```

**ตรวจสอบ:**
```bash
curl -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:8545
```

---

### Phase 3: เพิ่ม Validator 2

#### 3.1 สร้าง Keys สำหรับ Validator 2

```bash
cd /root/tempo-network

# สร้าง signing key
./tempo consensus generate-private-key --output ./89.167.25.42:3000/signing.key

# ดู public key
./tempo consensus calculate-public-key --private-key ./89.167.25.42:3000/signing.key
# Output: public key: 14902d2fa87f570c2a48a397bc144a2805c20e4a7c62c64457af0f08dcf408cd

# สร้าง enode key (64 hex chars)
head -c 32 /dev/urandom | xxd -p -c 64 | tr -d "\n" > ./89.167.25.42:3000/enode.key
```

#### 3.2 สร้างคำสั่ง addValidator

```bash
cd /root/tempo

cargo run -p tempo-xtask -- generate-add-peer \
  --public-key 14902d2fa87f570c2a48a397bc144a2805c20e4a7c62c64457af0f08dcf408cd \
  --inbound-address 89.167.25.42:3000 \
  --rpc-endpoint http://46.225.53.92:8545 \
  --admin-index 0 \
  --validator-index 2
```

**Output:**
```bash
cast send 0xCccCcCCC00000000000000000000000000000000 \
"addValidator(address newValidatorAddress, bytes32 publicKey, bool active, string calldata inboundAddress, string calldata outboundAddress)" \
"0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC" \
"14902d2fa87f570c2a48a397bc144a2805c20e4a7c62c64457af0f08dcf408cd" \
"true" \
"89.167.25.42:3000" \
"89.167.25.42:3000" \
--private-key ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
-r http://46.225.53.92:8545
```

#### 3.3 Execute addValidator

```bash
export PATH="$HOME/.foundry/bin:$PATH"

cast send 0xCccCcCCC00000000000000000000000000000000 \
"addValidator(address newValidatorAddress, bytes32 publicKey, bool active, string calldata inboundAddress, string calldata outboundAddress)" \
"0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC" \
"14902d2fa87f570c2a48a397bc144a2805c20e4a7c62c64457af0f08dcf408cd" \
"true" \
"89.167.25.42:3000" \
"89.167.25.42:3000" \
--private-key ac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
-r http://46.225.53.92:8545
```

#### 3.4 รัน Validator 2

```bash
# On Node 2 (89.167.25.42)
export PATH="$HOME/.foundry/bin:$PATH"

# Clean data
rm -rf /root/tempo-network/data /root/.cache/reth /root/.local/share/reth
mkdir -p /root/tempo-network/data

# Copy genesis and keys from Node 1
scp root@46.225.53.92:/root/tempo-network/genesis-dynamic.json /root/tempo-network/genesis.json

# Start Validator 2
./tempo node \
  --consensus.signing-key ./89.167.25.42:3000/signing.key \
  --consensus.listen-address 0.0.0.0:3000 \
  --chain ./genesis.json \
  --datadir ./data \
  --port 3001 \
  --p2p-secret-key ./89.167.25.42:3000/enode.key \
  --consensus.fee-recipient 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC \
  --trusted-peers "enode://<VALIDATOR_1_ENODE>@46.225.53.92:3001" \
  --http --http.port 8545 --http.addr 0.0.0.0
```

#### 3.5 รอ Sync และ DKG

Validator 2 จะ:
1. Sync chain จาก block 0
2. เข้าร่วมเป็น **syncer** ใน epoch ปัจจุบัน
3. กลายเป็น **player** ใน epoch ถัดไป
4. ได้รับ **signing share** จาก DKG ceremony
5. กลายเป็น **signer** (สามารถ propose/verify blocks ได้)

**ตรวจสอบสถานะ:**
```bash
# ดู logs
tail -f /root/tempo-network/validator.log | grep -E "(enter_epoch|run_dkg|as_player|we have a share)"

# ตรวจสอบ block number
curl -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://89.167.25.42:8545
```

---

### Phase 4: เพิ่ม Validator 3

ทำตามขั้นตอนเดียวกับ Validator 2:

#### 4.1 สร้าง Keys

```bash
./tempo consensus generate-private-key --output ./89.167.21.24:3000/signing.key
./tempo consensus calculate-public-key --private-key ./89.167.21.24:3000/signing.key
head -c 32 /dev/urandom | xxd -p -c 64 | tr -d "\n" > ./89.167.21.24:3000/enode.key
```

#### 4.2 Generate และ Execute addValidator

```bash
cargo run -p tempo-xtask -- generate-add-peer \
  --public-key <PUBLIC_KEY> \
  --inbound-address 89.167.21.24:3000 \
  --rpc-endpoint http://46.225.53.92:8545 \
  --admin-index 0 \
  --validator-index 3

# Execute คำสั่งที่ได้
```

#### 4.3 รัน Validator 3

```bash
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
```

---

## กลไกการทำงาน

### DKG Ceremony Timeline

```
Epoch N (50 blocks = ~4-5 mins)
├── Block N*50: DKG ceremony เริ่มต้น
├── Block N*50 + 25: Dealers ส่ง key shares ให้ players
├── Block (N+1)*50 - 1: สรุปผล DKG
└── Block (N+1)*50: Validator ใหม่ได้รับ signing share

Validator ใหม่จะสามารถ:
- Propose blocks (เป็น leader)
- Sign/verify blocks
- มีสิทธิ์ vote ใน consensus
```

### Validator States

| State | คำอธิบาย |
|-------|---------|
| **Syncer** | กำลัง sync chain ยังไม่มีสิทธิ์ใน DKG |
| **Player** | มีชื่อใน DKG ceremony รอรับ key shares |
| **Dealer** | Validator ปัจจุบัน ที่ generate และส่ง key shares |
| **Signer** | มี signing share สามารถ propose/verify blocks |

---

## คำสั่งตรวจสอบสถานะ

### ตรวจสอบจำนวน Validators

```bash
cast call 0xCccCcCCC00000000000000000000000000000000 \
  "validatorCount()" \
  -r http://<node-ip>:8545
```

### ดูรายการ Validators

```bash
cast call 0xCccCcCCC00000000000000000000000000000000 \
  "getValidators()" \
  -r http://<node-ip>:8545
```

### ตรวจสอบ DKG Status ใน Logs

```bash
# ดู epoch และ role
grep -E "(enter_epoch|run_dkg_loop)" /root/tempo-network/validator.log | tail -5

# ดู dealers/players/syncers
grep -E "(dealers|players|syncers)" /root/tempo-network/validator.log | tail -3

# ดูสถานะ signing share
grep "we have a share for this epoch" /root/tempo-network/validator.log | tail -3
```

---

## Troubleshooting

### Error: "genesis hash in the storage does not match"

**สาเหตุ:** Data directory มี genesis เก่าอยู่

**แก้ไข:**
```bash
rm -rf /root/tempo-network/data /root/.cache/reth /root/.local/share/reth
mkdir -p /root/tempo-network/data
```

### Error: "ValidatorAlreadyExists"

**สาเหตุ:** Validator address หรือ public key ซ้ำ

**แก้ไข:** เปลี่ยน `--validator-index` เป็นค่าอื่น

### Validator ใหม่ sync ช้า

**สาเหตุ:** Network latency หรือมี blocks จำนวนมาก

**แก้ไข:** 
- รอให้ sync เสร็จ
- ตรวจสอบ `connected_peers` ใน logs
- ตรวจสอบ firewall (ports 3000, 3001, 8545)

### Validator ไม่ได้รับ signing share

**สาเหตุ:** 
1. ยังไม่ถึง epoch boundary
2. Node ไม่ได้เป็น player ใน DKG

**ตรวจสอบ:**
```bash
# ดูว่าเป็น player หรือไม่
grep "as_player" /root/tempo-network/validator.log | tail -5

# ตรวจสอบ validator มีชื่อใน contract หรือยัง
cast call 0xCccCcCCC00000000000000000000000000000000 \
  "validators(address)" \
  "0x<VALIDATOR_ADDRESS>" \
  -r http://<node>:8545
```

---

## สรุป

✅ **Tempo รองรับ Dynamic Validator Addition:**
- ไม่ต้อง redeploy chain
- Validator ใหม่ได้รับ signing share ผ่าน DKG ceremony
- กระบวนการอัตโนมัติ ไม่ต้อง manual intervention

⏱️ **ระยะเวลา:**
- เพิ่ม validator เข้า contract: ทันที
- ได้รับ signing share: 1-2 epochs (~5-10 นาที)
- Sync chain: ขึ้นกับ block height

🔒 **ความปลอดภัย:**
- เฉพาะ admin เท่านั้นที่เพิ่ม validator ได้
- DKG ceremony ใช้ BLS12-381 threshold signatures
- Validator ใหม่ต้อง sync chain ก่อนจึงจะมีสิทธิ์

---

## References

- `/root/tempo/docs/internal/network-reconfiguration.md`
- `/root/tempo/crates/contracts/src/precompiles/validator_config.rs`
- ValidatorConfig Address: `0xCccCcCCC00000000000000000000000000000000`

---

*สร้างเมื่อ: January 30, 2026*  
*ทดสอบบน: Tempo v1.0.2-dev*
