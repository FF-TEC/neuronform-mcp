# NeuronForm MCP server

NeuronForm turns OpenStreetMap data into print-ready, multi-colour 3D map
models. Its public MCP server lets AI assistants such as Claude or ChatGPT
design these models in a chat: find a place, create and change a design,
show it as an image and a rotatable 3D view, and hand the user a link that
opens exactly that design in the NeuronForm configurator.

This repository holds the documentation of the server, example briefs with
the prompts that produce them, and the `server.json` for the MCP registry.
It holds no code of the NeuronForm service.

German version: [Deutsch](#deutsch). Website: [neuronform.com/agents/en](https://neuronform.com/agents/en).

## MCP address

```
https://neuronform.com/mcp
```

| Item | Value |
| --- | --- |
| Transport | Streamable HTTP, stateless (one JSON response per request, no session) |
| Authentication | none; no account, no API key |
| Cost | free of charge |
| Server name | `NeuronForm` |
| MCP Apps | extension `io.modelcontextprotocol/ui`; the 3D viewer is the resource `ui://neuronform/viewer` |

Agents never export print files and never buy. Exports and purchases happen
only in the configurator, with the user's own NeuronForm account.

## Setup

### Claude (claude.ai, desktop, and mobile apps)

Custom connectors are available on every Claude plan; the free plan allows
one.

1. Open “Customize” › “Connectors”.
2. Choose “+ Add” and then “Add custom connector”.
3. Enter the name `NeuronForm` and the MCP address.
4. Choose “No sign in” as authentication.
5. Select “Add”.

In a chat, switch the connector on under “+” › “Connectors”. In Team and
Enterprise organizations an owner adds the connector under “Organization
settings” › “Connectors”, and members connect it under “Customize” ›
“Connectors”. Source: [Anthropic help article on custom connectors](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp).

### ChatGPT

Whether you can add your own MCP servers depends on the ChatGPT plan and, in
a workspace, on its settings; on some plans developer mode must be switched
on first.

1. Open ChatGPT Plugins.
2. Choose the plus button and then “Add custom MCP server”.
3. Enter the name `NeuronForm`, a short description, and the MCP address
   under “Connection”.
4. Choose no authentication and accept the risk notice.
5. Select “Create as a plugin”.

In a new conversation, type `@` and choose NeuronForm. OpenAI changes these
labels from time to time; its [developer documentation](https://developers.openai.com/apps-sdk/deploy/connect-chatgpt)
holds the current steps.

### Claude Code

```sh
claude mcp add --transport http neuronform https://neuronform.com/mcp
```

With `--scope project` the entry goes into the `.mcp.json` of the project.

### Cursor

`.cursor/mcp.json` in the project, or `~/.cursor/mcp.json` for every project:

```json
{
  "mcpServers": {
    "neuronform": {
      "url": "https://neuronform.com/mcp"
    }
  }
}
```

### VS Code

Run “MCP: Add Server” from the Command Palette, or add the server to
`.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "neuronform": {
      "type": "http",
      "url": "https://neuronform.com/mcp"
    }
  }
}
```

Current versions of VS Code also read a portable `.mcp.json` at the root of
the project, which lists the same entry under `mcpServers`.

### Other clients

Every client that connects to remote MCP servers over Streamable HTTP needs
only the MCP address.

## Tools

Every tool takes the optional argument `lang` (`de` or `en`, default `en`)
for its summary and error texts. A failed call returns `isError: true` with
an actionable message and one entry per refused field. The `design` argument
is the complete design JSON; its schema is the resource
`neuronform://schema/design`.

| Tool | Purpose | Main arguments | Read-only |
| --- | --- | --- | --- |
| `search_places` | Finds places and addresses; returns name, lat, lon, and kind of each candidate | `query`, `limit` (1 to 10) | yes |
| `create_design` | Compiles a short brief into a complete design with the configurator defaults; invalid values fall back to their defaults with a notice | `brief` | yes |
| `get_design_guide` | Explains one topic of the design JSON with the JSON schema of its fields | `topic` | yes |
| `find_map_elements` | Lists roads and paths, buildings, areas, or ground regions of the design with a ready-to-insert fragment and its target path | `design`, `kind`, `name`, `class`, `near`, `limit`, `tile` | yes |
| `edit_tiles` | Adds a tile at a compass side of a tile, or removes the tile there, and returns the changed design | `design`, `action`, `tile`, `direction` | yes |
| `validate_design` | Checks a design against the request rules, the limits of anonymous use, and the restricted areas without generating it | `design` | yes |
| `render_design` | Renders a PNG of 1600 × 800 pixels with an oblique view and a top view, returns a summary with dimensions, triangles, and colour groups, and opens the 3D viewer where the host supports MCP Apps | `design` | yes |
| `share_design` | Stores the design as an anonymous snapshot and returns a link, valid for 90 days, that opens it in the configurator | `design` | no (creates a snapshot, deletes nothing) |
| `list_templates` | Lists the gallery templates; with a slug it returns the design of that template | `slug` | yes |

Guide topics of `get_design_guide`: `overview`, `footprint`, `tiles`,
`colors_and_layers`, `terrain`, `frame`, `hanging`, `paths`,
`texts_and_shapes`, `inserts`, `edge_labels`, `element_takeover`, `limits`.

### Brief

The brief of `create_design`; only the place, or `lat` and `lon`, is
required.

| Field | Meaning | Default |
| --- | --- | --- |
| `place` | Place or address; with `lat` and `lon` only its name | required unless `lat` and `lon` |
| `lat`, `lon` | Map centre in degrees; win over `place` | |
| `width` | Map width in metres, 100 to 30,000 | 1500 |
| `shape` | `hexagon`, `circle`, `square`, `rectangle` (0.75 times as high as wide), `polygon` | `hexagon` |
| `sides` | Sides of `polygon`, 3 to 12 | 6 |
| `rotation` | Clockwise rotation in degrees, −360 to 360 | 0 |
| `tiles` | Tile count, arranged compactly around the first tile; at most 3 for agents | 1 |
| `model_width` | Printed width of one tile in millimetres, 50 to 300 | 200 |
| `colors` | Colour template `single`, `two`, `four` | `four` |
| `relief` | `true` real relief, `false` flat | automatic (flat below 300 m elevation difference) |
| `exaggeration` | Vertical exaggeration, 0.1 to 10 | 1 |
| `frame` | `none`, `integrated`, `separate` | `none` |
| `label` | Edge text on the free edge of the first tile that faces south, up to 64 characters | none |

### Workflow

1. `search_places` finds the place; `create_design` turns a brief into a
   complete design.
2. The agent changes the design JSON directly; `edit_tiles` adds or removes
   tiles, and `find_map_elements` returns ready-to-insert roads, buildings,
   areas, and ground regions.
3. `validate_design` checks a changed design without generating it.
4. `render_design` shows the design as an image and an interactive 3D view.
5. `share_design` returns the link for the user.

## Example prompts

- “Create a 3 km wide hexagonal relief model of Hallstatt with 1.5-fold
  exaggeration and show it to me in 3D.”
- “Make a square city map of central Vienna, 2 km wide, in two colours, with
  the edge text ‘Wien’, and give me a link to open it in the configurator.”
- “Build a wall map of Salzburg from three hexagonal tiles, add an integrated
  frame, and highlight the Getreidegasse in red.”
- “Show me the Großglockner as a round relief, 8 km wide, and compare 1.2-fold
  with 2-fold exaggeration.”

The folder [examples](examples) holds the first three as JSON files. Each
file names the prompt in English and German, the brief for `create_design`,
the tool calls in their order, and the equivalent start link. The steps list
the arguments besides `brief` and `design`: `create_design` receives the
brief of the file, and every later call receives the current design.

## Start links for chats without MCP

A chat without MCP can hand over a design as a start link of the
configurator:

```
https://neuronform.com/map/app?place=Hallstatt&width=3000&shape=hexagon&relief=true&exaggeration=1.5
```

The query parameters are the fields of the brief: `place`, `lat`, `lon`,
`width`, `shape`, `sides`, `rotation`, `tiles`, `model_width`, `colors`,
`relief`, `exaggeration`, `frame`, and `label`. Only `place`, or `lat` and
`lon`, is required. Values are URL-encoded, such as `%20` for a space.
Numbers are decimal numbers, and `relief` is `true` or `false`. The
configurator builds the design with the compiler of `create_design` and
names every value it adjusted.

## Limits and fair use

| Rule | Value |
| --- | --- |
| Policy | limits of a visitor without an account: up to 3 tiles, 20 custom paths, the ranges of the free offer |
| Renders | 2,000 new renders per hour across all agents; renders of an unchanged design come from a 24-hour cache and do not count |
| Per calling address | 600 render calls and 1,200 lookups (searches, briefs, element searches, tile edits, validations, template listings, viewer downloads) per hour; 500 new share links per 24 hours |
| Priority | visitors of the website take precedence; while they occupy the service, an agent receives “busy, retry in 10 s” |
| Model size | models above about one million triangles are refused for agents; reduce the width or the tile count, or share the design and open it in the configurator |
| Share links | anonymous snapshots valid for 90 days; the same design returns the same link while it has at least one day left |
| Viewer | each render issues a viewer link valid for one hour |
| Restricted areas, countries | as for visitors of the website: military restricted areas cannot be printed, and the service is not offered in some countries |

Use is free of charge. Bulk retrieval and circumventing the limits are not
permitted; see the terms below.

## Privacy and terms

A request contains the tool arguments, such as the place and the design, and
the IP address of the calling system: the platform of the chat for Claude or
ChatGPT, the user's own computer for Claude Code, Cursor, or VS Code.
NeuronForm receives no account data and no other content of the chat. Place
searches go from the NeuronForm server to Nominatim of the OpenStreetMap
Foundation. The providers of the assistants are responsible for the chat
contents on their platforms.

- Privacy policy: [English](https://neuronform.com/privacy/en#agents), [Deutsch](https://neuronform.com/privacy/de#agents)
- Terms, section on AI agents: [English](https://neuronform.com/terms/en#agents), [Deutsch](https://neuronform.com/terms/de#agents)

## Map data and attribution

Every image and every result of the server carries the attribution of its
data:

```
Map data © OpenStreetMap contributors, ODbL 1.0 · Contains modified Copernicus DEM data
```

The map data comes from [OpenStreetMap](https://www.openstreetmap.org/copyright)
under the Open Database License (ODbL) 1.0. Keep the attribution when you
show or publish images of a design. The licenses of all data sources are
listed at [neuronform.com/licenses/en](https://neuronform.com/licenses/en).

## MCP registry

[server.json](server.json) describes the server for the official MCP registry
under the name `com.neuronform/mcp`, a namespace that the registry verifies
against the domain neuronform.com. It follows the schema
`https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json`.

## Contact

Questions and feedback: [office@neuronform.com](mailto:office@neuronform.com).

## License

The documentation and the examples of this repository are licensed under the
[MIT License](LICENSE). The license does not cover the NeuronForm service or
its results, such as images, 3D views, and share links; the
[terms](https://neuronform.com/terms/en) of NeuronForm govern them.

## Deutsch

NeuronForm macht aus OpenStreetMap-Daten druckfertige, mehrfarbige
3D-Kartenmodelle. Über den öffentlichen MCP-Server können KI-Assistenten wie
Claude oder ChatGPT diese Modelle im Chat gestalten:

- einen Ort finden;
- einen Entwurf erstellen und ändern;
- ihn als Bild und drehbare 3D-Ansicht zeigen;
- einen Link übergeben, der genau diesen Entwurf im Konfigurator öffnet.

Exportieren und kaufen können Agenten nicht; das geht nur im Konfigurator
mit einem eigenen Konto.

- **MCP-Adresse:** `https://neuronform.com/mcp` (Streamable HTTP, ohne Konto, ohne API-Schlüssel, kostenlos).
- **Einrichtung:** In Claude unter „Customize“ › „Connectors“ › „+ Add“ › „Add custom connector“ mit „No sign in“. In Claude Code mit `claude mcp add --transport http neuronform https://neuronform.com/mcp`. Für ChatGPT, Cursor und VS Code siehe die Abschnitte oben oder die Seite [Für KI-Agenten](https://neuronform.com/agents/de).
- **Werkzeuge:** `search_places`, `create_design`, `get_design_guide`, `find_map_elements`, `edit_tiles`, `validate_design`, `render_design`, `share_design`, `list_templates`. Jedes Werkzeug nimmt den optionalen Parameter `lang` (`de` oder `en`).
- **Startlinks ohne MCP:** `https://neuronform.com/map/app?place=Hallstatt&width=3000&shape=hexagon&relief=true&exaggeration=1.5`
- **Grenzen:**
  - Es gelten die Grenzen eines Besuchs ohne Konto (bis zu 3 Kacheln).
  - Alle Agenten zusammen können 2.000 neue Ansichten pro Stunde berechnen.
  - Besucherinnen und Besucher der Website haben Vorrang.
  - Modelle über etwa einer Million Dreiecken werden für Agenten nicht berechnet.
- **Datenschutz und Bedingungen:** [Datenschutzerklärung](https://neuronform.com/privacy/de#agents), [Verkaufsbedingungen, Abschnitt „Nutzung durch KI-Agenten“](https://neuronform.com/terms/de#agents).
- **Kartendaten:** © OpenStreetMap-Mitwirkende (ODbL 1.0); enthält modifizierte Copernicus-DEM-Daten.
