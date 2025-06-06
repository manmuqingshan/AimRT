# Parameter Plugin

## Related Links

Protocol File:
- {{ '[parameter.proto]({}/src/protocols/plugins/parameter_plugin/parameter.proto)'.format(code_site_root_path_url) }}

Reference Example:
- {{ '[parameter_plugin]({}/src/examples/plugins/parameter_plugin)'.format(code_site_root_path_url) }}

## Plugin Overview

The **parameter_plugin** registers an RPC based on protobuf protocol definition, providing some management interfaces for Parameters. Please note that **parameter_plugin** does not provide any communication backend, so this plugin is typically used in combination with RPC backends from other communication plugins, such as the http RPC backend in [net_plugin](./net_plugin.md).

The plugin configuration items are as follows:

| Node                              | Type          | Optional | Default  | Purpose |
| ----                              | ----          | ----     | ----     | ----    |
| service_name                      | string        | Yes      | ""       | RPC Service Name, if empty the default value generated from the protocol will be used |

Here is a simple configuration example combining **parameter_plugin** with the http RPC backend in **net_plugin**:

```yaml
aimrt:
  plugin:
    plugins:
      - name: net_plugin
        path: ./libaimrt_net_plugin.so
        options:
          thread_num: 4
          http_options:
            listen_ip: 127.0.0.1
            listen_port: 50080
      - name: parameter_plugin
        path: ./libaimrt_parameter_plugin.so
  rpc:
    backends:
      - type: http
    servers_options:
      - func_name: "(pb:/aimrt.protocols.parameter_plugin.*)"
        enable_backends: [http]
```

## ParameterService

In the {{ '[parameter.proto]({}/src/protocols/plugins/parameter_plugin/parameter.proto)'.format(code_site_root_path_url) }} protocol file, a `ParameterService` is defined, providing the following interfaces:
- **Set**: Set parameters;
- **Get**: Get parameters;
- **List**: List all parameters;
- **Dump**: Export all parameters;
- **Load**: Load a set of parameters, can directly load parameters previously dumped;

### Set

The `Set` interface is used to set/update a Key-Val parameter pair for a module, with the following interface definition:
```proto
message SetParameterReq {
  string module_name = 1;
  string parameter_key = 2;
  string parameter_value = 3;
}

message SetParameterRsp {
  uint64 code = 1;
  string msg = 2;
}

service ParameterService {
  // ...
  rpc Set(SetParameterReq) returns (SetParameterRsp);
  // ...
}
```

Here is an example using curl tool via Http to call this interface through the http RPC backend in **net_plugin**:
```shell
curl -i \
    -H 'content-type:application/json' \
    -X POST 'http://127.0.0.1:50080/rpc/aimrt.protocols.parameter_plugin.ParameterService/Set' \
    -d '{"module_name": "ParameterModule", "parameter_key": "key-1", "parameter_value": "val-abc"}'
```

This example command adds/updates a key-val parameter pair for the `ParameterModule` module, with parameter Key as `key-1` and parameter Val as `val-abc`. If successful, the command returns:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 21

{"code":0,"msg":""}
```

### Get

The `Get` interface is used to retrieve a Key-Val parameter pair from a module, with the following interface definition:
```proto
message GetParameterReq {
  string module_name = 1;
  string parameter_key = 2;
}

message GetParameterRsp {
  uint32 code = 1;
  string msg = 2;
  string parameter_value = 3;
}

service ParameterService {
  // ...
  rpc Get(GetParameterReq) returns (GetParameterRsp);
  // ...
}
```

Here is an example using curl tool via Http to call this interface through the http RPC backend in **net_plugin**:
```shell
curl -i \
    -H 'content-type:application/json' \
    -X POST 'http://127.0.0.1:50080/rpc/aimrt.protocols.parameter_plugin.ParameterService/Get' \
    -d '{"module_name": "ParameterModule", "parameter_key": "key-1"}'
```

This example command retrieves the value of parameter with Key `key-1` from the `ParameterModule` module. If successful, the command returns:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 47

{"code":0,"msg":"","parameter_value":"val-abc"}
```

### List

The `List` interface is used to retrieve all parameter Keys from a module, with the following interface definition:
```proto
message ListParameterReq {
  string module_name = 1;
}

message ListParameterRsp {
  uint32 code = 1;
  string msg = 2;
  repeated string parameter_keys = 3;
}

service ParameterService {
  // ...
  rpc List(ListParameterReq) returns (ListParameterRsp);
  // ...
}
```

Here is an example using curl tool via Http to call this interface through the http RPC backend in **net_plugin**:
```shell
curl -i \
    -H 'content-type:application/json' \
    -X POST 'http://127.0.0.1:50080/rpc/aimrt.protocols.parameter_plugin.ParameterService/List' \
    -d '{"module_name": "ParameterModule"}'
```

This example command lists all key values from the `ParameterModule` module. If successful, the command returns:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 62

{"code":0,"msg":"","parameter_keys":["key-3","key-2","key-1"]}
```

### Dump

The `Dump` interface is used to retrieve all parameter Keys and Vals from a module, with the following interface definition:
```proto
message ParameterMap {
  map<string, string> value = 1;
}

message DumpParameterReq {
  repeated string module_names = 1;
}

message DumpParameterRsp {
  uint32 code = 1;
  string msg = 2;

  map<string, ParameterMap> module_parameter_map = 3;  // key: module_name
}

service ParameterService {
  // ...
  rpc Dump(DumpParameterReq) returns (DumpParameterRsp);
  // ...
}
```

This interface supports dumping all data from multiple modules at once. Here is an example using curl tool via Http to call this interface through the http RPC backend in **net_plugin**:
```shell
curl -i \
    -H 'content-type:application/json' \
    -X POST 'http://127.0.0.1:50080/rpc/aimrt.protocols.parameter_plugin.ParameterService/Dump' \
    -d '{"module_names": ["ParameterModule"]}'
```

This example command exports all parameters from the `ParameterModule` module. If successful, the command returns:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 218

{
	"code": 0,
	"msg": "",
	"module_parameter_map": {
		"ParameterModule": {
			"value": {
				"key-3": "val-3",
				"key-9": "val-9",
				"key-8": "val-8",
				"key-4": "val-4",
				"key-5": "val-5",
				"key-2": "val-2",
				"key-1": "val-1",
				"key-7": "val-7",
				"key-6": "val-6"
			}
		}
	}
}
```

### Load

The `Load` interface is used to import a parameter package into some modules, which can be exported by the `Dump` interface, with the following interface definition:
```proto
message ParameterMap {
  map<string, string> value = 1;
}

message LoadParameterReq {
  map<string, ParameterMap> module_parameter_map = 1;  // key: module_name
}

message LoadParameterRsp {
  uint32 code = 1;
  string msg = 2;
}

service ParameterService {
  // ...
  rpc Load(LoadParameterReq) returns (LoadParameterRsp);
  // ...
}
```

This interface supports importing data into multiple modules at once. Here is an example using curl tool via Http to call this interface through the http RPC backend in **net_plugin**:
```shell
#!/bin/bash

data='{
	"module_parameter_map": {
		"ParameterModule": {
			"value": {
				"key-a": "val-111",
				"key-b": "val-222",
				"key-c": "val-333",
				"key-d": "val-444",
				"key-e": "val-555",
				"key-f": "val-666",
				"key-g": "val-777"
			}
		}
	}
}'


curl -i \
    -H 'content-type:application/json' \
    -X POST 'http://127.0.0.1:50080/rpc/aimrt.protocols.parameter_plugin.ParameterService/Load' \
    -d "$data"
```

This example command imports a set of parameters into the `ParameterModule` module. If successful, the command returns:
```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 19

{"code":0,"msg":""}
```