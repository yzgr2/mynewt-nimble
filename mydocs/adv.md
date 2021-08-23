
specification:
mesh : MshPRFv1.0.pdf


## adv


### adv format


### adv data

| len | ad type | ad data |
| -- | -- | -- |
| 02 | 01:Flags | 06 |
| 03 | 03: servie uuids | 0x1827: Mesh Provisioning Service |
| 0x15 | 0x16:service data | 0x27, 0x18, + DevieUUID + OOB |

Table 7.3

static const struct bt_data prov_ad[] = {
	BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
	BT_DATA_BYTES(BT_DATA_UUID16_ALL,
		      BT_UUID_16_ENCODE(BT_UUID_MESH_PROV_VAL)),
	BT_DATA(BT_DATA_SVC_DATA16, prov_svc_data, sizeof(prov_svc_data)),
};


code

```C
//host/mesh/src/proxy.c::
int bt_mesh_proxy_adv_start()
{
    //: set @prov_svc_data as uuid + oob
    prov_sd_len = gatt_prov_adv_create(prov_sd);

    //: start broadcast adv
    if (bt_le_adv_start(param, prov_ad, ARRAY_SIZE(prov_ad),
                prov_sd, prov_sd_len) == 0)

    prov_fast_adv: return 60s;
    prov_slow_adv: return K_FOREVER = -1;
}

//host/mesh/src/adv.c::
void mesh_adv_thread(void *args)
{
	static struct ble_npl_event *ev;
	struct os_mbuf *buf;
#if (MYNEWT_VAL(BLE_MESH_PROXY))
	int32_t timeout;
#endif

	BT_DBG("started");

	while (1) {
#if (MYNEWT_VAL(BLE_MESH_PROXY))
		ev = ble_npl_eventq_get(&adv_queue, 0);
		while (!ev) {   //: loop when empty.
            //: prov_fast_adv return 60s. prov_fast_adv->0
            //: prov_slow_adv: K_FOREVER
			timeout = bt_mesh_proxy_adv_start();
			BT_DBG("Proxy Advertising up to %d ms", (int) timeout);

			// FIXME: should we redefine K_SECONDS macro instead in glue?
			if (timeout != K_FOREVER) {
				timeout = ble_npl_time_ms_to_ticks32(timeout);
			}

            //: if queue is empty, will block @timeout ticks
			ev = ble_npl_eventq_get(&adv_queue, timeout);
			bt_mesh_proxy_adv_stop();
		}
#else
		ev = ble_npl_eventq_get(&adv_queue, BLE_NPL_TIME_FOREVER);
#endif

		if (!ev || !ble_npl_event_get_arg(ev)) {
			continue;
		}

		buf = ble_npl_event_get_arg(ev);

		/* busy == 0 means this was canceled */
		if (BT_MESH_ADV(buf)->busy) {
			BT_MESH_ADV(buf)->busy = 0;
			adv_send(buf);  //: will sleep @duration
		} else {
			net_buf_unref(buf);
		}

		/* os_sched(NULL); */
	}
}
```

```C
//: beacon.c
static void beacon_send(struct ble_npl_event *work)
{
    if (IS_ENABLED(BLE_MESH_PB_ADV)) {
		unprovisioned_beacon_send();
		k_delayed_work_submit(&beacon_timer, //: func = beacon_send
							  K_SECONDS(MYNEWT_VAL(BLE_MESH_UNPROV_BEACON_INT)));
	}
}

bt_mesh_beacon_enable() -> submit(&beacon_timer)

static int unprovisioned_beacon_send(void)
{
    net_buf_add_u8(buf, BEACON_TYPE_UNPROVISIONED);
	net_buf_add_mem(buf, prov->uuid, 16);
    net_buf_add_be16(buf, oob_info);
	net_buf_add_mem(buf, uri_hash, 4);
    bt_mesh_adv_send(buf, NULL, NULL); -> net_buf_put(&adv_queue, net_buf_ref(buf));

    if (prov->uri) { ...}
}
```

```C
bt_mesh_init(uint8_t own_addr_type, const struct bt_mesh_prov *prov,
		 const struct bt_mesh_comp *comp)
{
    bt_mesh_prov_init() -> bt_mesh_prov_init();

    host/mesh/src/adv.c:: bt_mesh_adv_init();
}

void bt_mesh_adv_init(void)
{
    ble_npl_eventq_init(&adv_queue);

    os_task_init(&adv_task, "mesh_adv", mesh_adv_thread, NULL,
	             MYNEWT_VAL(BLE_MESH_ADV_TASK_PRIO), OS_WAIT_FOREVER,
	             g_blemesh_stack, MYNEWT_VAL(BLE_MESH_ADV_STACK_SIZE));
}



int bt_mesh_prov_init(const struct bt_mesh_prov *prov_info)
{
	bt_mesh_prov = prov_info;

	if (IS_ENABLED(CONFIG_BT_MESH_PB_ADV)) {
		pb_adv_init();
	}
	if (IS_ENABLED(CONFIG_BT_MESH_PB_GATT)) {
		pb_gatt_init();
	}
	...
}



```