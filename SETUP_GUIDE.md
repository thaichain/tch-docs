# Tempo Validator Network Setup Guide

This guide documents the complete setup process for a 3-node Tempo validator network.

## Overview

- **Network**: Tempo Blockchain (v1.0.2-dev)
- **Chain ID**: 1337
- **Validators**: 3 nodes
- **Consensus**: HotStuff with DKG (Distributed Key Generation)

## Nodes

| Node | IP Address | Role |
|------|-----------|------|
| Node 1 | 46.225.53.92 | Validator + Bootnode |
| Node 2 | 89.167.25.42 | Validator |
| Node 3 | 89.167.21.24 | Validator |

## Prerequisites

- 3 Ubuntu servers (ARM64)
- Tempo binary (`/root/tempo-network/tempo`)
- Validator keys for each node
- Open ports: 3000 (consensus), 3001 (P2P), 8545 (RPC)

---

## Step 1: Generate Genesis File

The genesis file must be generated with all validators pre-configured. Validators cannot be added after genesis.

```bash
cd /root/tempo

cargo run -p tempo-xtask -- generate-genesis \
  --output /root/tempo-network/genesis-final \
  --chain-id 1337 \
  --validators "46.225.53.92:3000,89.167.25.42:3000,89.167.21.24:3000" \
  --accounts 10 \
  --epoch-length 100
```

This creates:
- `genesis.json` - Chain specification with 3 validators embedded
- Validator keys in respective directories

### Genesis Structure

The genesis file contains:
- **Chain ID**: 1337
- **extraData**: DKG public polynomial (3 validators)
- **ValidatorConfig Contract**: Pre-deployed at `0x20C0000000000000000000000000000000000001`
- **Pre-deployed contracts**: Permit2, pathUSD, TIP20Factory, FeeManager, Nonce, AccountKeychain

---

## Step 2: Prepare Validator Keys

Each validator needs these files:

```
/root/tempo-network/<ip>:3000/
├── signing.key      # Ed25519 private key (66 bytes)
├── signing.share    # BLS threshold share (68 bytes)
├── enode.key        # P2P private key (64 bytes hex)
└── enode.identity   # P2P public key (128 chars hex)
```

### Get Enode Identities

```bash
# Node 1
cat /root/tempo-network/46.225.53.92:3000/enode.identity
# e9c5df3d4cd678d9ea8b81767807c088107bd0c57b050e8146ad5aaf0ec355202992fedee32604618c09605518839a1605e4c5a3a59b23d5dc321554cf14cc31

# Node 2
cat /root/tempo-network/89.167.25.42:3000/enode.identity
# d6400afd5ba5ea9c0c222b81c952b8a1754dd9f9eb2d041caf3d655a8e0b5f409dd2048fd00b1568f1d045dcfd43aada21f8abdfab4d0d3bdbaba65e5c62175e

# Node 3
cat /root/tempo-network/89.167.21.24:3000/enode.identity
# 43f2384cebf7ebfe1faf62e3819cddad6a24f95f8184630768df67ddb0ad2b7cb71745ec389f4fe0266808b34c4d5f2c9acc6635b9765cc1135375ce4d3fc95f
```

---

## Step 3: Deploy to All Nodes

Copy files to each server:

```bash
# Copy genesis and binary
scp /root/tempo-network/genesis.json root@<node-ip>:/root/tempo-network/
scp /root/tempo-network/tempo root@<node-ip>:/root/tempo-network/

# Copy validator keys
scp -r /root/tempo-network/<node-ip>:3000 root@<node-ip>:/root/tempo-network/
```

---

## Step 4: Clean Data Directories

**Important**: Always clean old data before starting to avoid genesis hash mismatches.

```bash
# On each node
pkill -9 -f tempo
rm -rf /root/tempo-network/data
rm -rf /root/.cache/reth
rm -rf /root/.local/share/reth
mkdir -p /root/tempo-network/data
```

---

## Step 5: Start Node 1 (Bootnode)

```bash
cd /root/tempo-network
export RUST_LOG=info

nohup ./tempo node \
  --consensus.signing-key ./46.225.53.92:3000/signing.key \
  --consensus.signing-share ./46.225.53.92:3000/signing.share \
  --consensus.listen-address 0.0.0.0:3000 \
  --chain ./genesis.json \
  --datadir ./data \
  --port 3001 \
  --p2p-secret-key ./46.225.53.92:3000/enode.key \
  --consensus.fee-recipient 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 \
  --trusted-peers "enode://d6400afd5ba5ea9c0c222b81c952b8a1754dd9f9eb2d041caf3d655a8e0b5f409dd2048fd00b1568f1d045dcfd43aada21f8abdfab4d0d3bdbaba65e5c62175e@89.167.25.42:3001,enode://43f2384cebf7ebfe1faf62e3819cddad6a24f95f8184630768df67ddb0ad2b7cb71745ec389f4fe0266808b34c4d5f2c9acc6635b9765cc1135375ce4d3fc95f@89.167.21.24:3001" \
  --http --http.port 8545 --http.addr 0.0.0.0 \
  > validator.log 2>&1 &
```

**Key parameters:**
- `--consensus.signing-key`: Ed25519 private key
- `--consensus.signing-share`: BLS threshold share
- `--consensus.fee-recipient`: Address to receive fees
- `--trusted-peers`: Other validators' enode URLs (full 128-char identity)

---

## Step 6: Start Node 2

```bash
cd /root/tempo-network
export RUST_LOG=info

nohup ./tempo node \
  --consensus.signing-key ./89.167.25.42:3000/signing.key \
  --consensus.signing-share ./89.167.25.42:3000/signing.share \
  --consensus.listen-address 0.0.0.0:3000 \
  --chain ./genesis.json \
  --datadir ./data \
  --port 3001 \
  --p2p-secret-key ./89.167.25.42:3000/enode.key \
  --consensus.fee-recipient 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 \
  --trusted-peers "enode://e9c5df3d4cd678d9ea8b81767807c088107bd0c57b050e8146ad5aaf0ec355202992fedee32604618c09605518839a1605e4c5a3a59b23d5dc321554cf14cc31@46.225.53.92:3001,enode://43f2384cebf7ebfe1faf62e3819cddad6a24f95f8184630768df67ddb0ad2b7cb71745ec389f4fe0266808b34c4d5f2c9acc6635b9765cc1135375ce4d3fc95f@89.167.21.24:3001" \
  --http --http.port 8545 --http.addr 0.0.0.0 \
  > validator.log 2>&1 &
```

---

## Step 7: Start Node 3

```bash
cd /root/tempo-network
export RUST_LOG=info

nohup ./tempo node \
  --consensus.signing-key ./89.167.21.24:3000/signing.key \
  --consensus.signing-share ./89.167.21.24:3000/signing.share \
  --consensus.listen-address 0.0.0.0:3000 \
  --chain ./genesis.json \
  --datadir ./data \
  --port 3001 \
  --p2p-secret-key ./89.167.21.24:3000/enode.key \
  --consensus.fee-recipient 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC \
  --trusted-peers "enode://e9c5df3d4cd678d9ea8b81767807c088107bd0c57b050e8146ad5aaf0ec355202992fedee32604618c09605518839a1605e4c5a3a59b23d5dc321554cf14cc31@46.225.53.92:3001,enode://d6400afd5ba5ea9c0c222b81c952b8a1754dd9f9eb2d041caf3d655a8e0b5f409dd2048fd00b1568f1d045dcfd43aada21f8abdfab4d0d3bdbaba65e5c62175e@89.167.25.42:3001" \
  --http --http.port 8545 --http.addr 0.0.0.0 \
  > validator.log 2>&1 &
```

---

## Step 8: Verify Network

### Check Block Numbers

```bash
# Node 1
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://46.225.53.92:8545

# Node 2
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://89.167.25.42:8545

# Node 3
curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://89.167.21.24:8545
```

### Check Logs

```bash
# Look for these success indicators:
# - "total: 3" (in DKG ceremony)
# - "as_dealer=true as_player=true" (participating as signer)
# - "connected_peers=2" (connected to other validators)
# - "Block added to canonical chain"

tail -f /root/tempo-network/validator.log | grep -E "(Status|Block added|connected_peers)"
```

---

## Troubleshooting

### Error: "genesis hash in the storage does not match the specified chainspec"

**Cause**: Data directory contains old genesis state

**Solution**:
```bash
pkill -9 -f tempo
rm -rf /root/tempo-network/data /root/.cache/reth /root/.local/share/reth
mkdir -p /root/tempo-network/data
# Restart node
```

### Error: "Failed to parse id" in --trusted-peers

**Cause**: Using shortened Ed25519 public key instead of full 128-char enode identity

**Solution**: Use the full 128-character enode identity from `enode.identity` file:
```bash
# Correct
--trusted-peers "enode://e9c5df3d4cd678d9ea8b81767807c088107bd0c57b050e8146ad5aaf0ec355202992fedee32604618c09605518839a1605e4c5a3a59b23d5dc321554cf14cc31@46.225.53.92:3001"

# Incorrect (too short)
--trusted-peers "enode://2c2049266b544bef7a8a7a96bcf178092ce33d44d209ec5ceda3feddb80b2908@46.225.53.92:3001"
```

### Error: "we don't have a share for this epoch, participating as a verifier"

**Cause**: Only 2 validators in genesis, but node expects to participate

**Solution**: Regenerate genesis with all 3 validators:
```bash
cargo run -p tempo-xtask -- generate-genesis \
  --validators "46.225.53.92:3000,89.167.25.42:3000,89.167.21.24:3000" \
  ...
```

### connected_peers=0

**Cause**: 
1. Wrong enode format in --trusted-peers
2. Network/firewall issues
3. Nodes not started in correct order

**Solution**:
1. Verify enode identities are 128 characters
2. Check ports are open: `netstat -tlnp | grep tempo`
3. Ensure all nodes use the same genesis file
4. Check logs for connection errors

---

## Important Notes

### Validator Addition

- **Validators CANNOT be added after genesis**
- ValidatorConfig contract is a precompile that reads from genesis extraData
- Must specify all validators during `generate-genesis`

### Key Files

- `signing.key` + `signing.share`: Required for consensus participation
- `enode.key` + `enode.identity`: Required for P2P networking
- Keep these files secure and backed up

### DKG Ceremony

- Runs automatically at epoch 0 when nodes connect
- Requires 2/3 validators online for threshold signing
- If DKG fails, nodes will only participate as verifiers

---

## Maintenance

### Restarting a Node

```bash
# 1. Kill process
pkill -9 -f tempo

# 2. Optional: Clean data if genesis changed
rm -rf /root/tempo-network/data /root/.cache/reth /root/.local/share/reth
mkdir -p /root/tempo-network/data

# 3. Start node (use original start command)
```

### Viewing Logs

```bash
# Real-time
tail -f /root/tempo-network/validator.log

# Recent only
tail -100 /root/tempo-network/validator.log

# Search for specific events
grep "Block added" /root/tempo-network/validator.log
grep "Status" /root/tempo-network/validator.log
```

---

## Network Status

Check this guide's accompanying `status.sh` script for automated health checks.
