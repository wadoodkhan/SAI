# [SAI] Prefix Compression Entry Meta Data Range

---

| Title       | SAI Prefix Compression Entry Meta Data Range                |
|-------------|-------------------------------------------------------------|
| Authors     | Wadood A Khan (Marvell)                                     |
| Status      | In review                                                   |
| Type        | Standards track                                             |
| Created     | 2026-09-04                                                  |
| SAI-Version | 1.x                                                         |

---

## 1.0 Overview

`SAI_PREFIX_COMPRESSION_ENTRY_ATTR_META` is a mandatory attribute on every
prefix compression entry. Its valid value range is hardware-dependent and varies
across ASIC implementations. Without a corresponding switch attribute exposing
this range, a Network OS cannot validate meta values before programming entries,
leading to failures at `create_prefix_compression_entry` time.

This proposal adds `SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE`
(`sai_u32_range_t`, `READ_ONLY`) following the established SAI pattern for all
other object types that carry a user-assigned meta data value.

## 2.0 Specification

### 2.1 `saiswitch.h`

```c
/**
 * @brief Prefix compression entry user-based meta data range
 *
 * @type sai_u32_range_t
 * @flags READ_ONLY
 */
SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE,
```

### 2.2 `saiprefixcompression.h` — Updated entry attr comment

Updated comment on `SAI_PREFIX_COMPRESSION_ENTRY_ATTR_META` to reference the
range attribute:

```c
/**
 * @brief Prefix Compression entry meta data
 *
 * Value Range #SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE
 *
 * @type sai_uint32_t
 * @flags MANDATORY_ON_CREATE | CREATE_AND_SET
 */
SAI_PREFIX_COMPRESSION_ENTRY_ATTR_META = SAI_PREFIX_COMPRESSION_ENTRY_ATTR_START,
```

### 2.3 `saiacl.h` — Updated ACL entry attr comments

Updated comments on `SAI_ACL_ENTRY_ATTR_FIELD_SRC_PREFIX_META` and
`SAI_ACL_ENTRY_ATTR_FIELD_DST_PREFIX_META` to reference the range attribute:

```c
/**
 * @brief SRC META data
 *
 * Value must be in the range defined in
 * #SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE
 *
 * @type sai_acl_field_data_t sai_uint32_t
 * @flags CREATE_AND_SET
 * @default disabled
 */
SAI_ACL_ENTRY_ATTR_FIELD_SRC_PREFIX_META = SAI_ACL_ENTRY_ATTR_FIELD_START + 0x160,

/**
 * @brief DST META data
 *
 * Value must be in the range defined in
 * #SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE
 *
 * @type sai_acl_field_data_t sai_uint32_t
 * @flags CREATE_AND_SET
 * @default disabled
 */
SAI_ACL_ENTRY_ATTR_FIELD_DST_PREFIX_META = SAI_ACL_ENTRY_ATTR_FIELD_START + 0x161,
```

### 2.4 Alignment with existing SAI meta data range pattern

Every SAI object type that carries a user-assigned or hardware-constrained
scalar meta data value has a companion `SAI_SWITCH_ATTR_*_META_DATA_RANGE`
attribute. This proposal fills the missing entry for prefix compression:

| Object | Meta Data Attr | Switch Range Attr |
|--------|---------------|-------------------|
| FDB entry | `SAI_FDB_ENTRY_ATTR_META_DATA` | `SAI_SWITCH_ATTR_FDB_DST_USER_META_DATA_RANGE` |
| Route entry | `SAI_ROUTE_ENTRY_ATTR_META_DATA` | `SAI_SWITCH_ATTR_ROUTE_DST_USER_META_DATA_RANGE` |
| Neighbor entry | `SAI_NEIGHBOR_ENTRY_ATTR_META_DATA` | `SAI_SWITCH_ATTR_NEIGHBOR_DST_USER_META_DATA_RANGE` |
| Port | `SAI_PORT_ATTR_META_DATA` | `SAI_SWITCH_ATTR_PORT_USER_META_DATA_RANGE` |
| VLAN | `SAI_VLAN_ATTR_META_DATA` | `SAI_SWITCH_ATTR_VLAN_USER_META_DATA_RANGE` |
| ACL entry | `SAI_ACL_ENTRY_ATTR_ACTION_SET_ACL_META_DATA` | `SAI_SWITCH_ATTR_ACL_USER_META_DATA_RANGE` |
| NextHop | `SAI_NEXT_HOP_ATTR_META_DATA` | `SAI_SWITCH_ATTR_NEXT_HOP_USER_META_DATA_RANGE` |
| **Prefix compression entry** | `SAI_PREFIX_COMPRESSION_ENTRY_ATTR_META` | **`SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE`** |

## 3.0 Use Case

A prefix compression table maps IP prefixes to meta values. An ACL table
references the prefix compression table via
`SAI_ACL_TABLE_ATTR_SRC_PREFIX_COMPRESSION_TABLE` or
`SAI_ACL_TABLE_ATTR_DST_PREFIX_COMPRESSION_TABLE`, and ACL entries match on
the resulting meta value. The valid range of meta values is determined by the
hardware implementation of the prefix compression in the pipeline.

A Network OS populates prefix compression entries with meta values chosen from
operator configuration. Before programming any entry, it must confirm that the
chosen meta values fall within the hardware-supported range. Without
`SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE`, the Network OS has no
standard way to discover this range.

## 4.0 Network OS Query Pattern

A Network OS follows the standard three-step pattern used for all SAI meta
data range attributes. The same pattern is already applied by Network OS stacks
for `SAI_SWITCH_ATTR_ACL_USER_META_DATA_RANGE`.

**Step 1** — At initialization, query whether the switch implements the range
attribute:

```c
sai_attr_capability_t capability;
sai_query_attribute_capability(switch_id, SAI_OBJECT_TYPE_SWITCH,
    SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE, &capability);
```

**Step 2** — If supported, retrieve the hardware min/max and store for later
validation:

```c
if (capability.get_implemented)
{
    sai_attribute_t attr;
    attr.id = SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE;
    sai_switch_api->get_switch_attribute(switch_id, 1, &attr);

    meta_min = attr.value.u32range.min;
    meta_max = attr.value.u32range.max;
}
```

**Step 3** — Before programming any prefix compression entry, validate the
meta value falls within the discovered range and reject out-of-range values
without invoking SAI.

Without `SAI_SWITCH_ATTR_PREFIX_COMPRESSION_META_DATA_RANGE`, Steps 1 and 2
are impossible and the Network OS is forced to hardcode a range assumption,
which results in failure on platforms with a narrower hardware range.

## 5.0 Backward Compatibility

- The new attribute is additive. No existing attribute is changed, renumbered,
  or retyped.
- Implementations that do not support prefix compression return
  `SAI_STATUS_NOT_IMPLEMENTED` or `SAI_STATUS_ATTR_NOT_SUPPORTED` when the
  attribute is queried; the Network OS treats absence of capability as
  "feature not available on this platform."
- No change is made to `sai_attribute_value_t` or any other public union or
  struct layout.
