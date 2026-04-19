# QP Teardown Crash: BUG_ON(qp_state != RESET) with 32 QPs

## Summary

write_bw loopback with 32 QPs crashes during QP teardown on F1D1 silicon.
All 32 QPs complete data transfer successfully, but the disconnect/destroy
sequence hits a BUG_ON assertion in `roce_qp_fini`.

## Crash Signature

```
Abort: ("BUG_ON(rflow->qp_state != FUN_RDMA_QP_STATE_RESET)") at networking/rdma/roce/roce_qp.c:243
```

## Stack Trace

```
roce_qp_fini + 0x1b0
__channel__rdma_admin_ulp_fini_done + 0x108
_wdi_pmk_handler + 0x50
_wdi_prefcheck_handler + 0x7c
_wdi_wurx_handler + 0xd0
```

Crash VP: `5.2.1`

## FoD Job

- **Job ID:** 5518059
- **URL:** http://palladium-jobs.fungible.local/job/5518059
- **Hardware:** F1D1 silicon (demand-053:f1_0)
- **Build:** debug, from stable branch (commit e624bc7a4eb)
- **Binary:** funos-f1d1.signed (local build, SIGN=1 SKIP_CSR_OVERRIDE=0)

## Steps to Reproduce

### 1. Create dpcsh script

```bash
mkdir -p /tmp/xdata_wb/scripts
cat > /tmp/xdata_wb/scripts/write_bw_loopback.dpcsh << 'EOF'
serial [modcfg set rdma_cm/ip_addr "29.1.1.2"] [modcfg set rdma/dcqcn_force_init_rate 200] [modcfg set rdma/dcqcn_max_rate 200] [execute write_bw {rdma_test_async:true,server:1,addr:"29.1.1.2",size:262144,conn_count:32,run_time:120,send_depth:16,stag0:1}] [sleep 2] [execute write_bw {rdma_test_async:true,server:0,addr:"29.1.1.2",size:262144,conn_count:32,run_time:120,send_depth:16,stag0:1}] [sleep 10] [debug sample vp_4_4_0 5] [sleep 80] [rdma_test results_all] [debug sample_cleanup] [peek stats/vppkts]
EOF
echo "scripts/write_bw_loopback.dpcsh" > /tmp/xdata_wb/write_bw_xdata.list
```

### 2. Build

```bash
make MACHINE=f1d1 SIGN=1 SKIP_CSR_OVERRIDE=0 FUNBLOCKDEV=0 \
    XDATA_LISTS=/tmp/xdata_wb/write_bw_xdata.list -j16
```

### 3. Submit to FoD

```bash
python3 ~/ws/PalladiumOnDemand/run_f1/run_f1.py \
    --hardware-model F1D1 --duration 30 --priority normal_priority \
    --email suresh.nedunchezhian@microsoft.com --remoteuser snedunchezhian \
    --note "write_bw 32 QPs loopback" \
    build/funos-f1d1.signed \
    -- app=load_mods --csr-replay --test-exit-fast syslog=3 \
    --dpcsh 'script /scripts/write_bw_loopback.dpcsh'
```

### Note on dcqcn_force_init_rate

The `modcfg set rdma/dcqcn_force_init_rate 200` and `modcfg set rdma/dcqcn_max_rate 200`
are not required to reproduce. The crash is in QP teardown and occurs regardless of
rate limiting. These can be removed from the repro script. The crash is specific to
**32 QPs** — 4 and 16 QPs do not reproduce it.

## Timeline from UART Log

```
t=216.8s  "All client nodes done: created=32 failed=0 completed=32"
t=217.2s  "All server nodes done: created=32 failed=0 completed=32"
t=217.8s  EMERG: errant WU rdma_cm_cmpl_disconnect detected (multiple)
t=217.8s  EMERG: errant WU rdma_cm_disconnect_done detected (multiple)
t=217.8s  EMERG: errant WU wb_cm_disconnect_rsp detected (multiple)
t=217.8s  EMERG: errant WU wb_disconnect_node__reactivate detected (multiple)
t=220.2s  Abort: BUG_ON(rflow->qp_state != FUN_RDMA_QP_STATE_RESET)
          at roce_qp.c:243, in roce_qp_fini called from rdma_admin_ulp_fini_done
```

## Analysis

- All 32 QPs complete data transfer successfully (0 failures)
- During shutdown, WDI quiesce detects "errant WUs" — disconnect WUs arriving
  after quiesce has started
- Multiple disconnect completions are in flight across different VPs simultaneously
  (`0.2.1`, `2.5.0`, `2.5.1`, `1.4.3`, `4.5.2`)
- `roce_qp_fini` is called (via `rdma_admin_ulp_fini_done`) while a QP is not
  yet in `RESET` state — the disconnect hasn't completed for that QP
- This is a race between QP teardown and the async CM disconnect completion path

## Reproduction Rate

- **32 QPs:** crashed 1/1 attempt
- **16 QPs:** no crash (3+ successful runs)
- **4 QPs:** no crash (5+ successful runs)

## Workaround

Use `conn_count:16` or fewer to avoid the crash.
