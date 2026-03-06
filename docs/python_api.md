# NIXL Python API

The Python API can be found at `src/api/python/_api.py`. These are the pythonic APIs for NIXL, if more direct access to C++ style methods are desired,
the exact header implementation of `src/api/cpp` is done through pybind11 that can be found in `src/bindings/python`.

## Installation

### From PyPI

```bash
pip install nixl
```

### From Source

To build from source, follow the main build instructions in the README.md, then install the Python bindings:

```bash
# From the root nixl directory
pip install .
```

## API Reference

### `nixl_agent_config`

Configuration class for NIXL agent. Passed to `nixl_agent` constructor.

```python
nixl_agent_config(
    enable_prog_thread: bool = True,
    enable_listen_thread: bool = False,
    listen_port: int = 0,
    capture_telemetry: bool = False,
    num_threads: int = 0,
    backends: list[str] = ["UCX"],
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enable_prog_thread` | `bool` | `True` | Enable the progress thread, if available |
| `enable_listen_thread` | `bool` | `False` | Enable the listener thread for metadata communication |
| `listen_port` | `int` | `0` | Port for the listener thread to listen on |
| `capture_telemetry` | `bool` | `False` | Enable telemetry capture |
| `num_threads` | `int` | `0` | Number of threads for supported multi-threaded backends |
| `backends` | `list[str]` | `["UCX"]` | List of backend names to initialize |

### `nixl_agent`

Main class for creating a NIXL agent and performing transfers.

```python
nixl_agent(
    agent_name: str,
    nixl_conf: Optional[nixl_agent_config] = None,
    instantiate_all: bool = False,
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `agent_name` | `str` | required | Unique name for the agent |
| `nixl_conf` | `nixl_agent_config` | `None` | Agent configuration (uses defaults if None) |
| `instantiate_all` | `bool` | `False` | Instantiate all available backend plugins |

---

#### Plugin/Backend Discovery

##### `get_plugin_list() -> list[str]`

Get the list of available plugins.

##### `get_plugin_mem_types(backend: str) -> list[str]`

Get the memory types supported by a plugin.

- **`backend`**: Name of the plugin.

##### `get_plugin_params(backend: str) -> dict[str, str]`

Get the initialization parameters of a plugin. Returns a dictionary of option names to default values.

- **`backend`**: Name of the plugin.

##### `get_backend_mem_types(backend: str) -> list[str]`

Get the memory types supported by an initialized backend. After a plugin is initialized, the supported memory types might have changed.

- **`backend`**: Name of the backend.

##### `get_backend_params(backend: str) -> dict[str, str]`

Get the parameters of an initialized backend. Available initialization parameters might have changed after initialization.

- **`backend`**: Name of the backend.

##### `create_backend(backend: str, initParams: dict[str, str] = {})`

Initialize a backend with the specified initialization parameters.

- **`backend`**: Name of the backend.
- **`initParams`**: Dictionary of initialization parameters.

---

#### Memory Registration

##### `register_memory(reg_list, mem_type: Optional[str] = None, backends: list[str] = []) -> nixlRegDList`

Register memory regions, optionally with specified backends. Accepts lists of memory region tuples, tensors, or `nixlRegDList`.

- **`reg_list`**: Memory regions to register (tuples, tensors, or `nixlRegDList`).
- **`mem_type`**: Memory type string, required when using tuples.
- **`backends`**: Limit registration to specific backends.
- **Returns**: `nixlRegDList` for use with `deregister_memory`.

##### `deregister_memory(dereg_list: nixlRegDList, backends: list[str] = [])`

Deregister memory regions from backends.

- **`dereg_list`**: `nixlRegDList` from `register_memory` or `get_reg_descs`.
- **`backends`**: Limit deregistration to specific backends.

##### `query_memory(reg_list, backend: str, mem_type: Optional[str] = None) -> list[Optional[dict[str, str]]]`

Query information about memory/storage for a specific backend.

- **`reg_list`**: Memory regions to query (tuples, tensors, or `nixlRegDList`).
- **`backend`**: Backend name for querying.
- **`mem_type`**: Memory type string, required when using tuples.
- **Returns**: List of query results (each is `None` or a dict with info).

##### `get_reg_descs(descs, mem_type: Optional[str] = None) -> nixlRegDList`

Create a `nixlRegDList` from various input types:
- List of 4-element tuples `(address, len, device_id, meta_info)` with `mem_type`
- A single `torch.Tensor`
- A list of `torch.Tensor`
- An Nx3 numpy array with `mem_type`
- Passes through an existing `nixlRegDList`

---

#### Transfer Operations

##### `prep_xfer_dlist(agent_name: str, xfer_list, mem_type: Optional[str] = None, backends: list[str] = []) -> nixl_prepped_dlist_handle`

Prepare a transfer descriptor list for data transfer. Must be done on the initiator agent for both sides of a transfer.

- **`agent_name`**: `"NIXL_INIT_AGENT"` or `""` for local, remote agent name for remote, local agent name for loopback.
- **`xfer_list`**: Transfer descriptors (tuples, tensors, numpy array, or `nixlXferDList`).
- **`mem_type`**: Memory type, required when using tuples.
- **`backends`**: Limit which backends are used during preparation.
- **Returns**: `nixl_prepped_dlist_handle`.

##### `make_prepped_xfer(operation, local_xfer_side, local_indices, remote_xfer_side, remote_indices, notif_msg=b"", backends=[], skip_desc_merge=False) -> nixl_xfer_handle`

Create a transfer request from prepared descriptor list handles.

- **`operation`**: `"WRITE"` or `"READ"`.
- **`local_xfer_side`**: `nixl_prepped_dlist_handle` for local side.
- **`local_indices`**: List or numpy array of indices selecting local descriptors.
- **`remote_xfer_side`**: `nixl_prepped_dlist_handle` for remote side.
- **`remote_indices`**: List or numpy array of indices selecting remote descriptors.
- **`notif_msg`**: Optional notification message (bytes).
- **`backends`**: Limit which backends NIXL can use.
- **`skip_desc_merge`**: Deprecated.
- **Returns**: `nixl_xfer_handle`.

##### `initialize_xfer(operation, local_descs, remote_descs, remote_agent, notif_msg=b"", backends=[]) -> nixl_xfer_handle`

Combined API to create a transfer request from two descriptor lists. NIXL prepares the descriptor lists and then the transfer in one step.

- **`operation`**: `"WRITE"` or `"READ"`.
- **`local_descs`**: Local transfer descriptors (`nixlXferDList`).
- **`remote_descs`**: Remote transfer descriptors (`nixlXferDList`).
- **`remote_agent`**: Name of the remote agent.
- **`notif_msg`**: Optional notification message (bytes).
- **`backends`**: Limit which backends NIXL can use.
- **Returns**: `nixl_xfer_handle`.

##### `transfer(handle: nixl_xfer_handle, notif_msg: bytes = b"") -> str`

Initiate a data transfer. After calling this, check transfer state asynchronously.

- **`handle`**: Transfer handle from `make_prepped_xfer` or `initialize_xfer`.
- **`notif_msg`**: Optional notification message to specify or update per transfer call.
- **Returns**: `"DONE"`, `"PROC"`, or `"ERR"`.

##### `check_xfer_state(handle: nixl_xfer_handle) -> str`

Check the state of a transfer operation.

- **`handle`**: Transfer handle.
- **Returns**: `"DONE"`, `"PROC"`, or `"ERR"`.

##### `estimate_xfer_cost(req_handle: nixl_xfer_handle) -> tuple[int, int, int]`

Estimate the cost of a transfer operation. Times are in microseconds.

- **`req_handle`**: Transfer handle.
- **Returns**: Tuple of `(duration, error_margin, method)`.

##### `query_xfer_backend(handle: nixl_xfer_handle) -> str`

Query the backend that was chosen for a transfer operation.

- **`handle`**: Transfer handle.
- **Returns**: Name of the backend.

##### `get_xfer_descs(descs, mem_type: Optional[str] = None) -> nixlXferDList`

Create a `nixlXferDList` from various input types:
- List of 3-element tuples `(address, len, device_id)` with `mem_type`
- A single `torch.Tensor`
- A list of `torch.Tensor`
- An Nx3 numpy array with `mem_type`
- Passes through an existing `nixlXferDList`

---

#### Handle Management

##### `release_xfer_handle(handle: nixl_xfer_handle)`

Release a transfer handle, freeing associated memory. If the transfer is active, NIXL will attempt to cancel it.

##### `release_dlist_handle(handle: nixl_prepped_dlist_handle)`

Release a descriptor list handle, freeing associated memory.

---

#### Notifications

##### `get_new_notifs(backends: list[str] = []) -> dict[str, list[bytes]]`

Get new notifications that have come to the agent. Returns a dictionary mapping remote agent names to lists of notification messages.

- **`backends`**: Limit which backends are checked.

##### `update_notifs(backends: list[str] = []) -> dict[str, list[bytes]]`

Same as `get_new_notifs`, but returns all unhandled notifications in agent (accumulates).

- **`backends`**: Limit which backends are checked.

##### `check_remote_xfer_done(remote_agent_name: str, lookup_tag: bytes, backends: list[str] = [], tag_is_prefix=True) -> bool`

Check if a remote transfer is done by matching a notification tag. Removes the matched notification.

- **`remote_agent_name`**: Name of the remote agent.
- **`lookup_tag`**: Tag to match against notification messages.
- **`backends`**: Limit which backends are checked.
- **`tag_is_prefix`**: If True, match as prefix; if False, match as substring.
- **Returns**: True if found, False otherwise.

##### `send_notif(remote_agent_name: str, notif_msg: bytes, backend: Optional[str] = None)`

Send a standalone notification to a remote agent, not bound to a transfer.

- **`remote_agent_name`**: Name of the remote agent.
- **`notif_msg`**: Message to send (bytes).
- **`backend`**: Optional specific backend to use.

---

#### Metadata Exchange

##### `get_agent_metadata() -> bytes`

Get the full metadata of the local agent. Used for out-of-band exchange with remote agents.

##### `get_partial_agent_metadata(descs: nixlRegDList, inc_conn_info: bool = False, backends: list[str] = []) -> bytes`

Get partial metadata of the local agent.

- **`descs`**: Descriptors to include (can be empty for connection info only).
- **`inc_conn_info`**: Include connection info.
- **`backends`**: Backends to consider.

##### `add_remote_agent(metadata: bytes) -> str`

Add a remote agent using its metadata. Returns the remote agent's name.

##### `remove_remote_agent(agent: str)`

Remove a remote agent. Disconnects and prevents further transfers to this agent.

##### `send_local_metadata(ip_addr: str = "", port: int = DEFAULT_COMM_PORT)`

Send metadata to a peer or central metadata server.

- **`ip_addr`**: Peer IP address (if empty, sends to central server).
- **`port`**: Peer port (ignored for central server).

##### `send_partial_agent_metadata(descs, inc_conn_info=False, backends=[], ip_addr="", port=DEFAULT_COMM_PORT, label="")`

Send partial metadata to a peer or central metadata server.

- **`descs`**: Descriptors to include.
- **`inc_conn_info`**: Include connection info.
- **`backends`**: Backends to consider.
- **`ip_addr`**: Peer IP address.
- **`port`**: Peer port.
- **`label`**: Label for central metadata server.

##### `fetch_remote_metadata(remote_agent: str, ip_addr: str = "", port: int = DEFAULT_COMM_PORT, label: str = "")`

Request metadata from a central metadata server or peer.

- **`remote_agent`**: Name of the remote agent.
- **`ip_addr`**: Peer IP address.
- **`port`**: Peer port.
- **`label`**: Label for the metadata to fetch.

##### `invalidate_local_metadata(ip_addr: str = "", port: int = DEFAULT_COMM_PORT)`

Invalidate your own metadata in the central metadata server or from a specific peer.

##### `check_remote_metadata(agent: str, descs: nixlXferDList = None) -> bool`

Check if remote metadata for a specific agent is available.

- **`agent`**: Name of the remote agent.
- **`descs`**: Optional descriptor list for partial metadata check.
- **Returns**: True if available.

---

#### Serialization

##### `get_serialized_descs(descs) -> bytes`

Serialize a NIXL descriptor list with pickle.

##### `deserialize_descs(serialized_descs: bytes)`

Deserialize a NIXL descriptor list from pickle bytes.

---

### `nixl_prepped_dlist_handle`

Opaque handle wrapper for a prepared transfer descriptor list. Returned by `nixl_agent.prep_xfer_dlist()`.

##### `release()`

Explicitly free resources associated with this handle. `__del__` performs best-effort cleanup if not called.

### `nixl_xfer_handle`

Opaque handle wrapper for a transfer request. Returned by `nixl_agent.make_prepped_xfer()` and `nixl_agent.initialize_xfer()`.

##### `release()`

Explicitly free resources. If the transfer was not complete, this initiates the abort process (if available) and raises an exception.

## Memory Types

Valid memory type strings used with `mem_type` parameters:

| Value | Description |
|-------|-------------|
| `"DRAM"` | Host DRAM memory |
| `"VRAM"` | GPU video memory |
| `"FILE"` | File storage |
| `"BLOCK"` | Block storage |
| `"OBJ"` | Object store |
| `"cpu"` | Alias for `"DRAM"` |
| `"cuda"` | Alias for `"VRAM"` |

## Operations

Valid operation strings for transfer methods:

| Value | Description |
|-------|-------------|
| `"WRITE"` | Write data to remote |
| `"READ"` | Read data from remote |

## Environment Variables

| Variable | Description |
|----------|-------------|
| `NIXL_LOG_LEVEL` | Set logging level (e.g., `DEBUG`, `INFO`, `WARN`, `ERROR`) |
| `NIXL_DEBUG_LOGGING` | Enable debug logging output |

## Examples

See the [Python examples](../examples/python/) directory for complete working examples including:

- [query_mem_example.py](../examples/python/query_mem_example.py) - QueryMem API demonstration
- [nixl_gds_example.py](../examples/python/nixl_gds_example.py) - GDS backend usage
- [nixl_api_example.py](../examples/python/nixl_api_example.py) - General API usage
- [basic_two_peers.py](../examples/python/basic_two_peers.py) - Basic transfer operations
- [partial_md_example.py](../examples/python/partial_md_example.py) - Partial metadata handling
