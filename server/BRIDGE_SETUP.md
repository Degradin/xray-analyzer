# Bridge Architecture Setup Guide

## Problem: Bridge Correlation Not Working

If the bridge users page shows no activity despite configuration, check the following.

## How Bridge Correlation Works with Remnawave

**Remnawave assigns different synthetic IDs on different nodes for the same real user:**
- Bridge node (4vps-ru-01): `email: 2` (synthetic ID for real user)
- Exit node (waicore-de-01): `email: 15` (different synthetic ID for same real user)

**The analyzer resolves synthetic IDs to real user UUIDs via the remna_users table, then correlates by matching timestamps within ±15s window.**

## Critical Requirements

1. **Bridge nodes** (4vps-ru-01, servhost-ru-02) must send logs to the server
2. **Exit nodes** (waicore-de-01) must send logs with bridge inbound tags (e.g., `BRIDGE_DE_IN`)
3. **Remnawave sync** must be working to populate remna_users table for ID resolution

### 1. Verify Bridge Nodes Have Agents Running

Check that xray-log-agent is running on your bridge nodes:

```bash
# On bridge node (4vps-ru-01, servhost-ru-02)
docker ps | grep xray-log-agent
docker compose -f docker-compose.agent.yml logs --tail 30 xray-log-agent
```

### 2. Verify BRIDGE_INBOUND_PATTERN

The `BRIDGE_INBOUND_PATTERN` regex must match the **actual Xray inbound tag names** from your exit node logs.

**Incorrect:**
```bash
BRIDGE_INBOUND_PATTERN=^Russia → .+$
```

**Correct:**
```bash
BRIDGE_INBOUND_PATTERN=^BRIDGE_.*_IN(_\d+)?$
```

This matches tags like:
- `BRIDGE_DE_IN`
- `BRIDGE_DE_IN_2`
- `BRIDGE_NL_IN`

### 3. Verify BRIDGE_NODE_IDS

The `BRIDGE_NODE_IDS` must match the **exact `node_id` values** your bridge node agents send in their log batches.

**Check your bridge node agent configuration:**
```bash
# On bridge node
cat /opt/xray-analyzer/.env | grep NODE_ID
```

**Example:**
```bash
# In bridge node agent config
NODE_ID=4vps-ru-01

# In server .env
BRIDGE_NODE_IDS=4vps-ru-01,servhost-ru-02
```

### 4. Verify Remnawave Sync is Working

The analyzer needs remna_users table to resolve synthetic IDs to real UUIDs:

```bash
# Check if remna_users has data
docker exec analyzer-postgres psql -U xray_analyzer -d xray_analyzer \
  -c "SELECT COUNT(*) FROM remna_users;"

# Check if synthetic IDs map to real UUIDs
docker exec analyzer-postgres psql -U xray_analyzer -d xray_analyzer \
  -c "SELECT id, username, uuid FROM remna_users WHERE id IN (2, 15) LIMIT 5;"
```

### 5. Verify user_ip_history Has Bridge Node Data

Check the database for recent records from bridge nodes:

```bash
docker exec analyzer-postgres psql -U xray_analyzer -d xray_analyzer \
  -c "SELECT node_id, COUNT(*) FROM user_ip_history WHERE last_seen > NOW() - INTERVAL '1 hour' GROUP BY node_id;"
```

You should see records for your bridge node IDs (4vps-ru-01, servhost-ru-02).

## Common Issues

### Issue: "No bridge candidates found"

**Cause:** No user_ip_history records exist for the bridge nodes within the correlation window.

**Solution:**
1. Ensure bridge nodes have agents running and sending logs
2. Ensure users are actively connecting to bridge nodes
3. Check database for recent user_ip_history records from bridge nodes
4. Verify Remnawave sync is working (remna_users table populated)

### Issue: "Pattern matches nothing"

**Cause:** Regex pattern doesn't match actual inbound tags.

**Solution:** Check exit node logs for actual inbound tag values and adjust pattern accordingly.

### Issue: Wrong node IDs

**Cause:** BRIDGE_NODE_IDS don't match bridge node agent NODE_ID values.

**Solution:** Verify exact node_id values from bridge node agent configs and update server .env.

### Issue: Synthetic IDs not resolving to real UUIDs

**Cause:** Remnawave sync not working or remna_users table empty.

**Solution:**
1. Check REMNAWAVE_ENABLED=true and REMNAWAVE_URL/REMNAWAVE_API_TOKEN are set
2. Check server logs for Remnawave sync errors
3. Manually trigger sync: `docker restart analyzer-server`
