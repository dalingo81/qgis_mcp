# QGISMCP - QGIS Model Context Protocol Integration

QGISMCP connects [QGIS](https://qgis.org/) to [Claude AI](https://claude.ai/chat) through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/docs/getting-started/intro), allowing Claude to directly interact with and control QGIS. This integration enables prompt assisted project creation, layer loading, code execution and more.

This project is strongly based on the [BlenderMCP](https://github.com/ahujasid/blender-mcp/tree/main) project by [Siddharth Ahuja](https://x.com/sidahuj)

> **This is a fork:** [`dalingo81/qgis_mcp`](https://github.com/dalingo81/qgis_mcp), forked from [`jjsantos01/qgis_mcp`](https://github.com/jjsantos01/qgis_mcp).
> It keeps the upstream design and adds bug fixes plus quality-of-life improvements — see [Fork modifications](#fork-modifications) for the full list.

## Features

- **Two-way communication**: Connect Claude AI to QGIS through a socket-based server.
- **Project manipulation**: Create, load and save projects in QGIS.
- **Layer manipulation**: Add and remove vector or raster layers to a project.
- **Execute processing**: Execute processing algorithms ([Processing Toolbox](https://docs.qgis.org/3.40/en/docs/user_manual/processing/toolbox.html)).
- **Code execution**: Run arbitrary Python code in QGIS from Claude. Very powerful, but also be very cautious using this tool.

## Components

The system consists of two main components:

1. **[QGIS plugin](/qgis_mcp_plugin/)**: A QGIS plugin that creates a socket server within QGIS to receive and execute commands.
2. **[MCP Server](/src/qgis_mcp/qgis_mcp_server.py)**: A Python server that implements the Model Context Protocol and connects to the QGIS plugin.

## Installation

### Prerequisites

- QGIS 3.X (only tested on 3.22)
- Claude desktop
- Python 3.10 or newer
- uv package manager:

If you're on Mac, please install uv as

```bash
brew install uv
```

On Windows Powershell

```bash
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Otherwise installation instructions are on their website: [Install uv](https://docs.astral.sh/uv/getting-started/installation/)

**⚠️ Do not proceed before installing UV**

### Download code

Download this repo to your computer. You can clone it with:

```bash
git clone git@github.com:jjsantos01/qgis_mcp.git
```

### QGIS plugin

You need to copy the folder [qgis_mcp_plugin](/qgis_mcp_plugin/) and its content on your QGIS profile plugins folder.

You can get your profile folder in QGIS going to menu `Settings` -> `User profiles` -> `Open active profile folder` Then, go to `Python/plugins` and paste the folder `qgis_mcp_plugin`.

> On a Windows machine the plugins folder is usually located at: `C:\Users\USER\AppData\Roaming\QGIS\QGIS3\profiles\default\python\plugins` 

and on MacOS: `~/Library/Application\ Support/QGIS/QGIS3/profiles/default/python/plugins`

 Then close QGIS and open it again. Go to the menu option `Plugins` > `Installing and Managing Plugins`, select the `All` tab and search for "QGIS MCP", then mark the QGIS MCP checkbox.

### Claude for Desktop Integration

Go to `Claude` > `Settings` > `Developer` > `Edit Config` > `claude_desktop_config.json` to include the following:

> If you can't find the "Developers tab" or the `claude_desktop_config.json` look at this [documentation](https://modelcontextprotocol.io/quickstart/user#2-add-the-filesystem-mcp-server).

```json
{
    "mcpServers": {
        "qgis": {
            "command": "uv",
            "args": [
                "--directory",
                "/ABSOLUTE/PATH/TO/PARENT/REPO/FOLDER/qgis_mcp/src/qgis_mcp",
                "run",
                "qgis_mcp_server.py"
            ]
        }

    }
}
```

## Usage

### Starting the Connection

1. In QGIS, go to `plugins` > `QGIS MCP` > `QGIS MCP`
    ![plugins menu](/assets/imgs/qgis-plugins-menu.png)
2. Click "Start Server"
    ![start server](/assets/imgs/qgis-mcp-start-server.png)

### Using with Claude

Once the config file has been set on Claude, and the server is running on QGIS, you will see a hammer icon with tools for the QGIS MCP.

![Claude tools](assets/imgs/claude-available-tools.png)

#### Tools

- `ping` - Simple ping command to check server connectivity
- `get_qgis_info` - Get QGIS information about the current installation
- `load_project` - Load a QGIS project from the specified path
- `create_new_project` - Create a new project and save it
- `get_project_info` - Get current project information
- `add_vector_layer` - Add a vector layer to the project
- `add_raster_layer` - Add a raster layer to the project
- `get_layers` - Retrieve all layers in the current project
- `remove_layer` - Remove a layer from the project by its ID
- `zoom_to_layer` - Zoom to the extent of a specified layer
- `get_layer_features` - Retrieve features from a vector layer with an optional limit
- `execute_processing` - Execute a processing algorithm with the given parameters
- `save_project` - Save the current project to the given path
- `render_map` - Render the current map view to an image file
- `execute_code` - Execute arbitrary PyQGIS code provided as a string

### Example Commands

This is the example I used for the [demo](https://x.com/jjsantoso/status/1900293848271667395):

```plain
You have access to the tools to work with QGIS. You will do the following:
	1. Ping to check the connection. If it works, continue with the following steps.
	2. Create a new project and save it at: "C:/Users/USER/GitHub/qgis_mcp/data/cdmx.qgz"
	3. Load the vector layer: ""C:/Users/USER/GitHub/qgis_mcp/data/cdmx/mgpc_2019.shp" and name it "Colonias".
	4. Load the raster layer: "C:/Users/USER/GitHub/qgis_mcp/data/09014.tif" and name it "BJ"
	5. Zoom to the "BJ" layer.
	6. Execute the centroid algorithm on the "Colonias" layer. Skip the geometry check. Save the output to "colonias_centroids.geojson".
	7. Execute code to create a choropleth map using the "POB2010" field in the "Colonias" layer. Use the quantile classification method with 5 classes and the Spectral color ramp.
	8. Render the map to "C:/Users/USER/GitHub/qgis_mcp/data/cdmx.png"
	9. Save the project.
```

## Fork modifications

Changes in this fork relative to upstream `jjsantos01/qgis_mcp`, verified on QGIS 3.44.12 "Solothurn" with Python 3.12. All 15 MCP tools were exercised end-to-end; the five bugs below were reproduced, fixed, and re-verified through the MCP channel.

### Bug fixes

| # | Problem | Fix |
|---|---------|-----|
| 1 | `get_project_info` hardcoded a 10-layer limit, silently truncating larger projects (a 23-layer project reported 10) | Returns every layer |
| 2 | `get_layers` / `get_project_info` called `findLayer(...).isVisible()` without a null check — crashed when a layer exists in the project but not in the layer tree | Null-safe; such layers report `visible: false` |
| 3 | `create_new_project` only called `clear()` when `project.fileName()` was set, so layers leaked into the "new" project | Always clears before creating |
| 4 | `execute_processing` stringified output layers (`"<QgsVectorLayer: 'output' (memory)>"`) — the result was unusable and the layer never entered the project | Returns structured `{id, name, type, feature_count}` and adds the layer to the project |
| 5 | `render_map` used `QgsProject.mapLayers().values()`, ignoring layer-tree visibility and draw order — hidden layers were still rendered | Walks the layer tree, renders only visible layers in tree order |

Layer types are also reported as `vector (Point)` / `vector (Line)` / `vector (Polygon)` / `raster` instead of opaque enum values (`vector_0`, `vector_1`, ...).

### Improvements

- **Auto-start the socket server** — the plugin starts it in `initGui()`, so QGIS listens on port 9876 as soon as it opens. No need to click "Start Server".
- **Closing the panel no longer stops the server** — the dock widget hides; only "Stop Server" stops it.
- **Explicit IPv4 connection** — the MCP server connects to `127.0.0.1` instead of `localhost`. On machines where `localhost` resolves to IPv6 `::1` first while the plugin listens on IPv4 only, `localhost` fails with an unhelpful *"Could not connect to Qgis"*.
- **Fixed the heartbeat check** — it referenced a non-existent attribute, and `sendall(b'')` always succeeds on TCP, so a dead peer was never detected.

### Note on the license

Upstream ships **no LICENSE file**, and this fork inherits that state. Check with the upstream author before redistributing or shipping this in a product.
