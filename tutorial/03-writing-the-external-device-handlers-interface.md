# The External Device Handler's ubus Interface

Any program connected to ubus that provides the following methods for the device handler interface can act as an external device handler:

| Method           | Purpose                                       |
| ---------------- | --------------------------------------------- |
| `create`         | Create a device.                              |
| `reload`         | Reload the device with another configuration. |
| `dump_info`      | Dump information to a provided buffer according to the `info` field in the JSON description. See below for details. |
| `dump_stats`     | Dump information to a `blob_buffer` according to the `info` field in the JSON description. See below for details. |
| `free`           | Remove a device.                              |

## `create`, `reload` and `free`

The `create` and `free` methods are the bare minimum of operations an external device handler should be able to execute.
`create` takes the entire UCI configuration in the form of a `blob_attr` and parses it to create the device accordingly. `Reload` takes the same input but assumes that the device already exists and has to be reconfigured. This *can* mean deletion and re-creation with another config.
`Free` just receives the device's name and is expected to delete it and all the state associated with the device, entirely.

## `dump_info` and `dump_stats`

If a device handler can provide useful information or statistics about its devices, these methods can be used to fetch it and add it to the data netifd can obtain itself.
To enable this feature, the necessary fields have to be put in the JSON description. In the request, netifd sends the buffer for the information or statistics. The external device can then add the data to it using the appropriate  `blobmsg_add_*` functions.
When called from the shell with

```bash
ubus call network.device status "{'name': 'device_name'}"
```
netifd will print all the data in JSON format.

## Notifying netifd of Finished Operations

Handling of the messages from netifd happens asynchronously at the external device handler. When it has finished the operation successfully, it has to notify netifd via the ubus subscription mechanism. Otherwise, netifd will retry the method call three times before giving up and removing all the state for the devices.
These are the available notifications and their associated messages and semantics:

| Notification | Format                  | Purpose                                                     |
| ------------ | ----------------------- | ----------------------------------------------------------- |
| `create`     | { "name" : *string* }   | Signal successful device creation                           |
| `reload`     | { "name" : *string* }   | Signal successful reloading of a device                     |
| `free`       | { "name" : *string* }   | Signal successful device removal                            |
| `prepare`    | { "bridge" : *string* } | Signal successful preparation of bridge for adding a member |
| `add`        | { "bridge" : *string*, "member" : *string* } | Signal successful adding of a bridge member                 |
| `remove`     | { "bridge" : *string*, "member" : *string* } | Signal successful removal of a bridge member                |

## Example

Here is some example code detailing how to write an external device handler's ubus interface. It is built on the "ACL Bridge" example from the [description file tutorial](INSERT LINK).
To create the interface to ubus, a `blobmsg_policy` is needed per method to describe the format of the method's arguments:

```c
// create and reload
const struct blobmsg_policy create_policy[] = {
  {"name", BLOBMSG_TYPE_STRING},
  {"ifname", BLOBMSG_TYPE_ARRAY},
  {"empty", BLOBMSG_TYPE_BOOL},
  {"acl", BLOBMSG_TYPE_TABLE}
};

// free, info and hotplug prepare only need the bridge's name
const struct blobmsg_policy free_prep_info_policy[] = {
  {"name", BLOBMSG_TYPE_STRING}
};

// hotplug add and remove
const struct blobmsg_policy hotplug_add_remove_policy[] = {
  {"bridge", BLOBMSG_TYPE_STRING},
  {"member", BLOBMSG_TYPE_STRING}
};
```

With these policies in place, and the necessary `ubus_handler_t` functions, the ubus object for the device handler can be created:

```c
struct ubus_method ubus_methods[] = {
  // device handler interface
  UBUS_METHOD("create", _handle_create, create_policy),
  UBUS_METHOD("reload", _handle_reload, create_policy),
  UBUS_METHOD("dump_info", _handle_dump_info, free_prep_info_policy),
  UBUS_METHOD("free", _handle_free, free_prep_info_policy),

  // hotplug ops
  UBUS_METHOD("add", _handle_hotplug_add, hotplug_add_remove_policy),
  UBUS_METHOD("remove", _handle_hotplug_remove,
		hotplug_add_remove_policy),
  UBUS_METHOD("prepare", _handle_hotplug_prepare, free_prep_info_policy),
};

struct ubus_object_type aclbridge_obj_type =
	UBUS_OBJECT_TYPE("aclbridge", ubus_methods);

static struct ubus_object abr_obj = {
	.name = "aclbr",
	.type = &aclbridge_obj_type,
	.methods = ubus_methods,
	.n_methods = 7,
};
```

Here is what `_handle_create` could look like

```c
int
_handle_create(struct ubus_context *ctx, struct ubus_object *obj,
  struct ubus_request_data *req, const char *method, struct blob_attr *msg)
{
  struct blob_attr *tb[4];
  struct ubus_notification_request *notify = malloc(sizeof(struct ubus_notification_request));

  // attach callback to free memory for request after notification is done
  notify->complete_cb = free_req;

  // parse the configuration blob
  blobmsg_parse(create_policy, 4, tb, blob_data(msg), blob_len(msg));

  if (!tb[0] || !tb[1] || !tb[2] || !tb[3])
    return UBUS_STATUS_INVALID_ARGUMENT;

  // create the device
  if (acl_bridge_create(blobmsg_get_string(tb[0]), // name
                    tb[1], // ifname
                    blobmsg_get_bool(tb[2]), // empty
                    tb[3] // acl
                    )) {
    return UBUS_STATUS_UNKNOWN_ERROR;
  }

  // notify netifd
  blob_buf_init(&blob_buffer, 0);
  blobmsg_add_string(&blob_buffer, "name", blobmsg_get_string(tb[0]));

  ubus_notify_async(ubus_ctx, &abr_obj, "create", blob_buffer.head, notify);

  return 0;
}
```

Another example, `_handle_hotplug_add`:

```c
int
_handle_hotplug_add(struct ubus_context *ctx, struct ubus_object *obj,
  struct ubus_request_data *req, const char *method, struct blob_attr *msg)
{
  struct blob_attr *tb[2];
  struct ubus_notification)_request *notify = malloc(sizeof(struct ubus_notification_request));

  // free notification request memory afterwards
  notify->complete_cb = free_req;

  // parse the message
  blobmsg_parse(hotplug_add_remove_policy, 2, tb, blob_data(msg), blob_len(msg));

  if (!tb[0] || !tb[1])
    return UBUS_STATUS_INVALID_ARGUMENT;

  if (!acl_bridge_exists(blobmsg_get_string(tb[0])))
    return UBUS_STATUS_NOT_FOUND;

  // add the port
  acl_bridge_add_port(blobmsg_get_string(tb[0]), blobmsg_get_string(tb[1]));

  // notify netifd
  blob_buf_init(&blob_buffer, 0);
  blobmsg_add_string(&blob_buffer, "bridge", blobmsg_get_string(tb[0]));
  blobmsg_add_string(&blob_buffer, "member", blobmsg_get_string(tb[1]));

  ubus_notify_async(ubus_ctx, &abr_obj, "add", blob_buffer.head, notify);

  return 0;
}
```
