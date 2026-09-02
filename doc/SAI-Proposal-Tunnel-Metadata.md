# [SAI] SAI Metadata Support for Tunnel Objects

---

| Title | SAI Metadata Support for Tunnel Objects |
|-------------|--------------------------------------------------------------|
| Authors | Wadood A Khan (Marvell) |
| Status | In review |
| Type | Standards track |
| Created | 2026-09-02 |
| SAI-Version | 1.x |

---

## 1.0 Overview

SAI supports user-assigned metadata on several forwarding objects (Port, VLAN,
Route, Neighbor, NextHop) that can be matched in ACL entries. The established
pattern of group-tagging forwarding objects for ACL policy allows a single ACL
entry to cover a set of objects sharing the same metadata value. Tunnel objects
have no equivalent support today. This proposal adds a scalar metadata attribute
to Tunnel, following the same established pattern, so that ACL policy can be
expressed through metadata-based tunnel grouping rather than individual tunnel
endpoint addresses.

## 2.0 Specification

One new object attribute, one new switch range attribute, and two new ACL match
fields are introduced.

### 2.1 `saitunnel.h`

```c
/**
 * @brief User-based metadata for Tunnel
 *
 * Value Range #SAI_SWITCH_ATTR_TUNNEL_USER_META_DATA_RANGE
 *
 * @type sai_uint32_t
 * @flags CREATE_AND_SET
 * @default 0
 */
SAI_TUNNEL_ATTR_META_DATA,
```

### 2.2 `saiswitch.h`

```c
/**
 * @brief Tunnel user-based metadata range
 *
 * @type sai_u32_range_t
 * @flags READ_ONLY
 */
SAI_SWITCH_ATTR_TUNNEL_USER_META_DATA_RANGE,
```

### 2.3 `saiacl.h`

ACL table match field:

```c
/**
 * @brief Tunnel user metadata
 *
 * @type bool
 * @flags CREATE_ONLY
 * @default false
 */
SAI_ACL_TABLE_ATTR_FIELD_TUNNEL_USER_META = SAI_ACL_TABLE_ATTR_FIELD_START + 0x16a,
```

ACL entry match field:

```c
/**
 * @brief Tunnel user metadata
 *
 * Value must be in the range defined in
 * #SAI_SWITCH_ATTR_TUNNEL_USER_META_DATA_RANGE
 *
 * @type sai_acl_field_data_t sai_uint32_t
 * @flags CREATE_AND_SET
 * @default disabled
 */
SAI_ACL_ENTRY_ATTR_FIELD_TUNNEL_USER_META = SAI_ACL_ENTRY_ATTR_FIELD_START + 0x16a,
```

### 2.4 Alignment with existing SAI metadata pattern

This proposal follows the identical pattern of existing SAI metadata support:

| Object | Metadata Attr | Switch Range Attr | ACL Match Field |
|--------|--------------|-------------------|-----------------|
| Port | `SAI_PORT_ATTR_META_DATA` | `SAI_SWITCH_ATTR_PORT_USER_META_DATA_RANGE` | `SAI_ACL_*_ATTR_FIELD_PORT_USER_META` |
| VLAN | `SAI_VLAN_ATTR_META_DATA` | `SAI_SWITCH_ATTR_VLAN_USER_META_DATA_RANGE` | `SAI_ACL_*_ATTR_FIELD_VLAN_USER_META` |
| Route | `SAI_ROUTE_ENTRY_ATTR_META_DATA` | `SAI_SWITCH_ATTR_ROUTE_DST_USER_META_DATA_RANGE` | `SAI_ACL_*_ATTR_FIELD_ROUTE_DST_USER_META` |
| Neighbor | `SAI_NEIGHBOR_ENTRY_ATTR_META_DATA` | `SAI_SWITCH_ATTR_NEIGHBOR_DST_USER_META_DATA_RANGE` | `SAI_ACL_*_ATTR_FIELD_NEIGHBOR_DST_USER_META` |
| NextHop | `SAI_NEXT_HOP_ATTR_META_DATA` | `SAI_SWITCH_ATTR_NEXT_HOP_USER_META_DATA_RANGE` | `SAI_ACL_*_ATTR_FIELD_NEXT_HOP_USER_META` |
| **Tunnel** | `SAI_TUNNEL_ATTR_META_DATA` | `SAI_SWITCH_ATTR_TUNNEL_USER_META_DATA_RANGE` | `SAI_ACL_*_ATTR_FIELD_TUNNEL_USER_META` |

## 3.0 API Example

### 3.1 Network OS initialization — query range support

A Network OS follows the standard three-step pattern used for all SAI metadata
range attributes. The same pattern is already applied by Network OS stacks for
`SAI_SWITCH_ATTR_PORT_USER_META_DATA_RANGE` and
`SAI_SWITCH_ATTR_VLAN_USER_META_DATA_RANGE`.

**Step 1** — At initialization, query whether the switch implements the range
attribute:

```c
sai_attr_capability_t cap;
sai_query_attribute_capability(switch_id, SAI_OBJECT_TYPE_SWITCH,
    SAI_SWITCH_ATTR_TUNNEL_USER_META_DATA_RANGE, &cap);
```

**Step 2** — If supported, retrieve the hardware min/max and store for later
validation:

```c
if (cap.get_implemented)
{
    sai_attribute_t attr;
    attr.id = SAI_SWITCH_ATTR_TUNNEL_USER_META_DATA_RANGE;
    sai_switch_api->get_switch_attribute(switch_id, 1, &attr);

    tunnel_meta_min = attr.value.u32range.min;
    tunnel_meta_max = attr.value.u32range.max;
}
```

**Step 3** — Before setting `SAI_TUNNEL_ATTR_META_DATA` on any Tunnel object,
validate the meta value falls within the discovered range and reject out-of-range
values without invoking SAI.

Without `SAI_SWITCH_ATTR_TUNNEL_USER_META_DATA_RANGE`, Steps 1 and 2 are
impossible and the Network OS is forced to hardcode a range assumption, which
results in failure on platforms with a narrower hardware range.

### 3.2 Create Tunnels with metadata

```c
/* Tunnel-1 and Tunnel-2 assigned to the same workload group (meta = 1). */
...(Existing Attributes)
sai_attr_list[attr_count].id = SAI_TUNNEL_ATTR_META_DATA;
sai_attr_list[attr_count++].value.u32 = 1;
sai_create_tunnel_fn(&tunnel1_id, switch_id, attr_count, sai_attr_list);

...(Existing Attributes)
sai_attr_list[attr_count].id = SAI_TUNNEL_ATTR_META_DATA;
sai_attr_list[attr_count++].value.u32 = 1;
sai_create_tunnel_fn(&tunnel2_id, switch_id, attr_count, sai_attr_list);
```

### 3.3 Create ACL table with Tunnel metadata match field

```c
sai_attr_list[attr_count].id = SAI_ACL_TABLE_ATTR_ACL_STAGE;
sai_attr_list[attr_count++].value.u32 = SAI_ACL_STAGE_EGRESS;

sai_attr_list[attr_count].id = SAI_ACL_TABLE_ATTR_FIELD_TUNNEL_USER_META;
sai_attr_list[attr_count++].value.booldata = true;

sai_create_acl_table_fn(&acl_table_id, switch_id, attr_count, sai_attr_list);
```

### 3.4 Create ACL entry matching on Tunnel metadata

```c
/* Apply egress policy to all tunnels in group 1 (tunnel_meta == 1). */
sai_attr_list[attr_count].id = SAI_ACL_ENTRY_ATTR_TABLE_ID;
sai_attr_list[attr_count++].value.oid = acl_table_id;

sai_attr_list[attr_count].id = SAI_ACL_ENTRY_ATTR_FIELD_TUNNEL_USER_META;
sai_attr_list[attr_count].value.aclfield.enable = true;
sai_attr_list[attr_count].value.aclfield.data.u32 = 1;
sai_attr_list[attr_count++].value.aclfield.mask.u32 = 0xffffffff;

sai_attr_list[attr_count].id = SAI_ACL_ENTRY_ATTR_ACTION_SET_DSCP;
sai_attr_list[attr_count].value.aclaction.enable = true;
sai_attr_list[attr_count++].value.aclaction.parameter.u8 = 46;

sai_create_acl_entry_fn(&acl_entry_id, switch_id, attr_count, sai_attr_list);
```

## 4.0 Backward Compatibility

- All new attributes are additive. No existing attribute is changed, renumbered,
  or retyped.
- Default value for `SAI_TUNNEL_ATTR_META_DATA` is `0`, preserving existing
  behavior. ACL table and entry match fields default to disabled.
- Support is discoverable via `sai_query_attribute_capability()`. Implementations
  that do not support these attributes return `SAI_STATUS_ATTR_NOT_SUPPORTED`.
- No change is made to `sai_attribute_value_t` or any other public union or
  struct layout.
