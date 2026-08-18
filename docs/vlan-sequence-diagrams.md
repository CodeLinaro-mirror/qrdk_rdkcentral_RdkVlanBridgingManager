# VlanManager Components & Sequence Diagrams

This document describes the runtime components of VlanManager and the
create / delete / marking sequences for tagged and untagged VLAN interfaces.

- [1. Components](#1-components)
- [2. Trigger & threading model](#2-trigger--threading-model)
- [3. Tagged VLAN create (incl. VLAN tag-0)](#3-tagged-vlan-create-incl-vlan-tag-0)
- [4. Untagged VLAN create (bridge / macvlan)](#4-untagged-vlan-create-bridge--macvlan)
- [5. Delete sequences](#5-delete-sequences)
- [6. Markings (QoS)](#6-markings-qos)

---

## 1. Components

```mermaid
flowchart LR
    subgraph WM[WanManager]
        WMK[Marking table<br/>SKBMark / EthPriority / DSCP]
    end
    subgraph VM[VlanManager]
        VDML[VLANTermination DML<br/>vlan_dml.c]
        VAPI[VLAN entry logic<br/>vlan_apis.c]
        EDML[EthLink DML<br/>ethernet_dml.c]
        EAPI[EthLink logic<br/>ethernet_apis.c]
    end
    K[(Linux kernel<br/>ip / brctl / ifconfig)]
    PSM[(PSM)]

    PSM --> VAPI
    PSM --> EAPI
    VDML --> VAPI
    VAPI -->|Vlan_SetEthLink| EDML
    EDML --> EAPI
    WMK -->|EthLink_AddMarking / GetMarking| EAPI
    VAPI -->|popen: ip link / QoS| K
    EAPI -->|popen: ip link / brctl / ifconfig| K
    EAPI -->|VlanStatus Up/Down| WM
```

| Component | File | Responsibility |
|-----------|------|----------------|
| VLANTermination DML | `vlan_dml.c` | TR-181 get/set, spawns worker threads on Enable |
| VLAN entry logic | `vlan_apis.c` | Tagged create/delete, QoS apply, status poll |
| EthLink DML | `ethernet_dml.c` | TR-181 get/set for `Device.X_RDK_Ethernet.Link` |
| EthLink logic | `ethernet_apis.c` | Untagged create/delete, marking table, VlanStatus |
| Command runner | `ethernet_apis.c` `EthLink_RunCmd` / `EXEC_CMD` | `popen()` wrapper that logs the command + all stdout/stderr and the exit code |

**Key architectural point:** the *config* (type, vlanid) lives on the **VLAN
entry**, but untagged interface *creation* happens in the **EthLink** object.
`Vlan_SetEthLink()` bridges the two — `EthLink_GetEntry()` returns the live
`DML_ETHERNET` struct, so the VLAN thread writes `UnTaggedVlanType` directly onto
it before committing.

---

## 2. Trigger & threading model

Enabling an entry (from PSM at boot, or a DM `Enable=true`) spawns a detached
worker thread. Tagged and untagged both start the same way but diverge on
`vlanid` / `UnTaggedVlanType`.

```mermaid
flowchart TD
    E[Vlan Enable = true] --> C[Vlan_SetParamBoolValue]
    C --> P[pthread_create]
    P --> EN[Vlan_Enable thread]
    EN --> D{vlanid &gt; 0<br/>OR type == VlanTag0?}
    D -->|yes| TAG[Tagged path]
    D -->|no| UNT[Untagged path]
```

---

## 3. Tagged VLAN create (incl. VLAN tag-0)

Both a normal tagged VLAN (`vlanid > 0`) and VLAN tag-0 (`UntaggedVlanType =
VlanTag0`) take this path — a tag-0 interface is a real 802.1Q device, so
create / QoS / delete are identical.

```mermaid
sequenceDiagram
    participant T as Vlan_Enable (VLAN thread)
    participant SE as Vlan_SetEthLink
    participant EE as EthLink_Enable
    participant CI as Vlan_CreateTaggedInterface
    participant QM as Vlan_ApplyQoSMarking
    participant K as Kernel
    participant WM as WanManager

    T->>SE: enable=TRUE, PriTag=TRUE
    SE->>SE: sync UnTaggedVlanType to EthLink struct
    SE->>EE: EthLink Enable=TRUE, PriorityTagging=TRUE
    EE->>EE: EthLink_AddMarking populate table
    Note over EE: PriorityTagging=TRUE so skip untagged create
    T->>K: if exists, ip link set down and delete
    T->>CI: create
    CI->>K: ip link add link base name ifname type vlan id N
    CI->>K: ip link set ifname up
    CI->>K: ip link set dev ifname address mac
    T->>QM: apply QoS
    QM->>EE: EthLink_GetMarking alias
    QM->>K: ip link set ifname type vlan egress-qos-map SKB colon PCP
    T->>K: poll IFF_RUNNING 10x 2s
    T->>WM: VlanStatus = Up
```

---

## 4. Untagged VLAN create (bridge / macvlan)

For `vlanid < 0` with a bridge or macvlan type, creation is delegated to the
EthLink object. `Vlan_SetEthLink` sets `PriorityTagging=FALSE`, which makes
`EthLink_Enable` run the untagged creation switch.

```mermaid
sequenceDiagram
    participant T as Vlan_Enable (VLAN thread)
    participant SE as Vlan_SetEthLink
    participant EE as EthLink_Enable
    participant CU as EthLink_CreateUnTaggedInterface
    participant K as Kernel
    participant WM as WanManager

    T->>SE: enable=TRUE, PriTag=FALSE
    SE->>SE: sync UnTaggedVlanType to live EthLink struct
    SE->>EE: EthLink Enable=TRUE, PriorityTagging=FALSE
    EE->>EE: EthLink_AddMarking
    EE->>CU: create untagged
    alt UNTAGGED_SIMPLE_BRIDGE default
        CU->>K: brctl addbr ifname
        CU->>K: brctl addif ifname base
        CU->>K: ifconfig ifname up
    else UNTAGGED_MACVLAN private/vepa/bridge/passthru/source
        CU->>K: ip link add link base name ifname address mac type macvlan mode m
        CU->>K: ip link set ifname allmulticast multicast on, mtu 1500, up
    end
    EE->>K: poll IFF_RUNNING 10x 2s
    EE->>WM: VlanStatus = Up
    T->>WM: VlanStatus = Up untagged status re-check
```

---

## 5. Delete sequences

Delete mirrors create. The `vlanid > 0 || type == VlanTag0` test again routes
tagged/tag-0 to the VLAN thread and bridge/macvlan to the EthLink object.

```mermaid
sequenceDiagram
    participant T as Vlan_Disable (VLAN thread)
    participant SE as Vlan_SetEthLink
    participant ED as EthLink_Disable
    participant DU as EthLink_DeleteUnTaggedInterface
    participant K as Kernel
    participant WM as WanManager

    T->>SE: enable=FALSE
    SE->>ED: EthLink Enable=FALSE
    alt tagged OR VlanTag0
        Note over T: handled in VLAN thread
        T->>K: ip link set ifname down
        T->>K: ip link delete ifname
    else untagged bridge/macvlan
        ED->>DU: delete untagged
        alt UNTAGGED_SIMPLE_BRIDGE
            DU->>K: brctl delif ifname base
            DU->>K: ifconfig ifname down
            DU->>K: brctl delbr ifname
        else UNTAGGED_MACVLAN modes
            DU->>K: ip link set ifname down
            DU->>K: ip link delete ifname
        end
    end
    T->>WM: VlanStatus = Down
```

> Interfaces are always brought **down before delete** (`ip link set … down`
> then `ip link delete …`, or `ifconfig … down` then `brctl delbr …`) so the
> kernel does not reject removal of a running device.

---

## 6. Markings (QoS)

WanManager owns the marking table; VlanManager only *reads* it and applies the
result to tagged / tag-0 interfaces via the kernel VLAN `egress-qos-map`.

### 6.1 Data flow

```mermaid
flowchart LR
    subgraph WanManager
        DM["Device.X_RDK_WanManager…Marking.{i}<br/>Alias / SKBPort / SKBMark / EthernetPriorityMark"]
    end
    AM[EthLink_AddMarking] -->|CCSP get| DM
    AM --> TBL[DML_ETHERNET.pstDataModelMarking&#91;&#93;]
    GM[EthLink_GetMarking] --> TBL
    GM --> CFG["vlan_configuration_t.skb_config&#91;&#93;<br/>(vlan_skb_config_t)"]
    CFG --> QOS[egress-qos-map]
    QOS --> K["ip link set &lt;if&gt; type vlan<br/>egress-qos-map SKBMark:EthPriority"]
```

### 6.2 `vlan_skb_config_t` fields

| Field | Meaning | Used in egress-qos-map? |
|-------|---------|--------------------------|
| `alias` | Label only (`DATA` / `VOICE`) | No — no kernel equivalent |
| `skbMark` | Kernel skb priority (fwmark class) | **Yes** — left side of the map |
| `skbEthPriorityMark` | Outgoing 802.1p PCP (0–7) | **Yes** — right side of the map |
| `skbPort` | Port-based classification | No — would require `tc filter` rules |

The applied command is:

```
ip link set <ifname> type vlan egress-qos-map <skbMark>:<skbEthPriorityMark>
```

### 6.3 Which types get QoS

| Interface type | egress-qos-map | Reason |
|----------------|:--------------:|--------|
| Tagged VLAN | ✅ | 802.1Q device carries a PCP field |
| VlanTag0 | ✅ | also a 802.1Q device (priority-tagged) |
| Bridge | ❌ | not a VLAN device |
| Macvlan (all modes) | ❌ | no 802.1Q tag at this layer |

For bridge / macvlan untagged types there is no L2 PCP to set; QoS for those is
handled elsewhere in the stack (WanManager L2/L3 marking framework).

### 6.4 When QoS is (re)applied

```mermaid
flowchart TD
    A[Tagged/Tag0 create] --> Q1[Vlan_ApplyQoSMarking]
    R[EthLink refresh<br/>X_RDK_Refresh=true] --> Q2[EthLink_TriggerVlanRefresh]
    Q2 --> Q3[EthLink_SetEgressQoSMap]
    Q1 --> MAP[egress-qos-map applied]
    Q3 --> MAP
```
