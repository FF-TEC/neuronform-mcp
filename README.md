# NeuronForm MCP server

NeuronForm turns OpenStreetMap data into print-ready, multi-colour 3D map
models. Its public MCP server lets AI assistants such as Claude or ChatGPT
design these models in a chat: find a place, create and change a design,
show it as an image and, where the chat supports it, as a rotatable 3D view,
and hand the user a link that opens exactly that design in the NeuronForm
configurator.

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
| Request body | at most 2 MiB; a larger request is answered with HTTP 413 before any tool runs |
| Authentication | none; no account, no API key |
| Cost | free of charge |
| Server name | `NeuronForm`, title “NeuronForm 3D map models” |
| MCP Apps | extension `io.modelcontextprotocol/ui`; the 3D viewer is the resource `ui://neuronform/viewer` |
| Resources | the viewer, the design schema `neuronform://schema/design`, and one guide `neuronform://guide/{topic}` per topic of `get_design_guide` |

Agents never export print files and never buy. Exports and purchases happen
only in the configurator, with the user's own NeuronForm account.

## Setup

### Claude (claude.ai, desktop, and mobile apps)

Custom connectors are available on every Claude plan; the free plan allows
one.

1. Open “Customize” › “Connectors”.
2. Choose “+ Add” and then “Add custom connector”.
3. Enter the name `NeuronForm` and the MCP address, and select “Continue”.
4. Review the detected authentication settings and select “Continue”.
5. Under “Authentication”, choose “No sign in”.
6. Select “Add”.

In a chat, switch the connector on under “+” › “Connectors”. In Team and
Enterprise organizations an owner adds the connector under “Organization
settings” › “Connectors” with “Add” › “Custom” › “Web”, and members connect
it under “Customize” › “Connectors” with “Connect”. Source: [Anthropic help article on custom connectors](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp).

### ChatGPT

Whether you can add your own MCP servers depends on the ChatGPT plan and, in
a workspace, on its permissions and security settings.

1. Open [ChatGPT Plugins](https://chatgpt.com/plugins) in ChatGPT on the web.
2. Choose the plus button and then “Add custom MCP server”.
3. Enter the name `NeuronForm` and, optionally, a short description.
4. Under “Connection”, choose the public endpoint and enter the MCP address.
5. Choose “No authentication”, review the risk warning, and select “I
   understand and want to continue”.
6. Select “Create as a plugin”, review the tools the server lists, and
   install the plugin.

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

Run “MCP: Add Server” from the Command Palette, choose “HTTP (HTTP or
Server-Sent Events)”, enter the MCP address and the server ID `neuronform`,
and choose a destination:

- “Global” › “Copilot Global” writes `~/.copilot/mcp-config.json`, or
  `$COPILOT_HOME/mcp-config.json` when `COPILOT_HOME` is set, for every
  workspace.
- `.mcp.json` at the root of the workspace shares the server with a team;
  the guided flow offers it with the experimental setting
  `chat.mcp.workspaceRootConfig.enabled`.
- `.vscode/mcp.json` in the workspace, which the guided flow labels
  deprecated.

The entry in `.vscode/mcp.json`:

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

The portable files `~/.copilot/mcp-config.json` and `.mcp.json` list the
same entry under `mcpServers` instead of `servers`. Microsoft describes the
destinations in its [documentation of MCP servers in VS Code](https://code.visualstudio.com/docs/copilot/customization/mcp-servers).

### Other clients

Every client that connects to remote MCP servers over Streamable HTTP needs
only the MCP address.

## Tools

Every tool takes the optional argument `lang` (`de` or `en`, default `en`).
It sets the language of the summaries, the notices, and the lead sentences
and hints of error messages; the guide texts and the detail texts of the
design validation are English. A successful call returns a text block with a short
summary followed by the JSON result, and the same result as
`structuredContent`; `render_design` returns the JPEG image before the text
block, and the text block of `get_design_guide` holds the guide text. A
failed call returns `isError: true` with an actionable message and, in `structuredContent`, the `message` and the list
`errors` with one entry per refused field (`field`, `message`, `hint`); an
unknown tool name is a JSON-RPC error. The `design` argument is the complete
design JSON; its schema is the resource `neuronform://schema/design`.

| Tool | Purpose | Main arguments | Read-only |
| --- | --- | --- | --- |
| `search_places` | Searches places, streets, and house numbers in the place directory of the server; returns name, lat, lon, and kind of each candidate | `query` (2 to 200 characters), `limit` (1 to 10, default 5) | yes |
| `create_design` | Compiles a short brief into a complete design with the configurator defaults; invalid values fall back to their defaults with a notice naming the brief field | `brief` | yes |
| `edit_design` | Changes a design with brief fields (`changes`) and JSON Patch operations `add`, `replace`, `remove` (`patch`, at most 100, applied after `changes`) and returns the changed design; at least one of the two is required; the service stores no design, so every call carries the complete design | `design`, `changes`, `patch` | yes |
| `get_design_guide` | Explains one topic of the design JSON with the JSON schema of its fields | `topic` (default `overview`) | yes |
| `find_map_elements` | Lists stretches of roads, paths, railways, and waterways between junctions (`line`), buildings, areas, or ground regions of the design with a ready-to-insert fragment and its target path; the fragments carry the colour `D8493F`, and a building also carries the placement of a custom house that replaces it | `design`, `kind` (`line`, `building`, `area`, `ground`), `name`, `class`, `near` (`lat`, `lon`), `limit` (1 to 50, default 10), `tile` | yes |
| `edit_tiles` | Adds a tile at a compass side of a tile, or removes the tile there, and returns the changed design; tile 0 always stays | `design`, `action` (`add`, `remove`), `tile` (default 0), `direction` (`n`, `ne`, `e`, `se`, `s`, `sw`, `w`, `nw`) | yes |
| `validate_design` | Checks a design against the request rules, the limits of anonymous use, and the restricted areas without generating it, and lists every tile with its centre and its free edges and every custom house with its printed size and the details its print size leaves out | `design` | yes |
| `render_design` | Renders a JPEG of 1600 × 800 pixels with views from the north, east, south, and west beside the top view, or one view of 800 × 800 pixels (`view` `top`, or `oblique` with `direction`, default `sw`), twice the width and height with `resolution` `high`; returns a summary with dimensions, triangles, colour groups, warnings, and tile positions, the attribution, and a viewer link, and opens the 3D viewer in hosts that show MCP Apps or ChatGPT apps | `design`, `view` (`overview`, `top`, `oblique`; default `overview`), `direction`, `resolution` (`standard`, `high`) | yes |
| `share_design` | Stores the design as an anonymous snapshot and returns a link, valid for 90 days, that opens it in the configurator | `design` | no (creates a snapshot, deletes nothing) |
| `list_templates` | Lists up to 50 published gallery templates with slug, title, description, tile count, and image URL; with a slug it returns the design of that template | `slug` | yes |

Guide topics of `get_design_guide`: `overview`, `footprint`, `tiles`,
`colors_and_layers`, `terrain`, `frame`, `hanging`, `paths`,
`texts_and_shapes`, `inserts`, `edge_labels`, `element_takeover`,
`details`, `focus_scaling`, `custom_houses`, `limits`.

Besides the fields of the brief, a design takes every field of the
configurator through `edit_design`: among them texts and shapes sized in
millimetres of the print or in metres on the map (topic
`texts_and_shapes`), the generic details (roofs, windows, doors, and
trees, with a facade style and a detail level, up to a map width of 3 km;
topic `details`), focus scaling that
magnifies one place and shrinks its surroundings toward the edge (topic
`focus_scaling`), and up to five custom houses on free ground or in place
of a building of the map (topic `custom_houses`).

A compass direction of `edit_tiles` selects the edge whose outward normal
lies within 22.5° of it, so the directions follow the shape and the rotation:
a hexagon without rotation offers `ne`, `e`, `se`, `sw`, `w`, and `nw`.
`validate_design` and `render_design` list the free edges of every tile.

### Brief

The brief of `create_design` and the `changes` of `edit_design`; in a
brief only the place, or `lat` and `lon`, is required, and in `changes` an
absent field keeps the design.

| Field | Meaning | Default |
| --- | --- | --- |
| `place` | Place or address, 2 to 200 characters; the first match of the place search becomes the centre; with `lat` and `lon` only its name | required unless `lat` and `lon` |
| `lat`, `lon` | Map centre in degrees, latitude −85.05 to 85.05, longitude −180 to 180; win over `place` | |
| `width` | Map width in metres, 100 to 30,000 | 1500 |
| `shape` | `hexagon`, `circle`, `square`, `rectangle` (0.75 times as high as wide), `polygon` | `hexagon` |
| `sides` | Sides of `polygon`, 3 to 12; without `shape` it selects `polygon` | 6 |
| `rotation` | Clockwise rotation in degrees, −360 to 360 | 0 |
| `tiles` | Tile count, arranged compactly around the first tile; at most 3 for agents; a circle takes one tile | 1 |
| `model_width` | Printed width of one tile in millimetres, 50 to 300 | 200 |
| `colors` | Colour template `single`, `two`, `four`, or an object with an optional `template` and colours `RRGGBB` or `#RRGGBB` by group (`ground`, `green`, `water`, `traffic`), by layer kind (`terrain`, `paved_areas`, `bare_ground`, `vegetation`, `agriculture`, `water_areas`, `sea`, `waterways`, `major_roads`, `local_roads`, `paths`, `railways`, `buildings`), `frame`, or `label`; a layer kind wins over its group, and every distinct colour is one filament | `four` |
| `relief` | `true` real relief, `false` flat | automatic (flat below 300 m elevation difference) |
| `exaggeration` | Vertical exaggeration, 0.1 to 10 | 1 |
| `frame` | `none`, `integrated`, `separate` | `none` |
| `label` | Edge text on the south-facing free edge of the first tile, or on its first free edge, up to 64 characters; a design with an edge text takes the new text into its first edge text | none |
| `lang` | Language of the notices, `de` or `en`; when absent, the `lang` argument of the call applies | `en` |

### Workflow

1. `search_places` finds the place; `create_design` turns a brief into a
   complete design. The centre of the brief is the centre of tile 0.
2. `edit_design` changes brief fields and patches every other field;
   `edit_tiles` adds or removes tiles at the free edges that
   `validate_design` lists, and `find_map_elements` returns ready-to-insert
   roads, buildings, areas, and ground regions.
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
the tool calls in their order, and the start link of its brief. The steps list
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
`relief`, `exaggeration`, `frame`, and `label`; `colors` takes a template
name only, and the language of the notices is that of the configurator.
Only `place`, or `lat` and `lon`, is required. Values are URL-encoded, such
as `%20` for a space. Numbers are decimal numbers, and `relief` is `true` or
`false`. The configurator builds the design with the compiler of
`create_design` and names every value it adjusted; a link with a malformed
value, or with a place the place search does not find, opens the
configurator without the design and names the reason.

## Limits and fair use

| Rule | Value |
| --- | --- |
| Policy | limits of a visitor without an account: up to 3 tiles, up to 20 custom lines, and the ranges of the brief; an unchanged gallery template of `list_templates` keeps its own tile count |
| Generic details | roofs, windows, doors, and trees up to a map width of 3 km; above it, a design with a detail switched on is refused on the field `details` |
| Custom houses | up to 5 per design; one house within its detail budget of 2 million triangles, which the service may set otherwise |
| Renders | 2,000 newly generated models per hour across all agents; a render whose model is still cached does not count, and rendered designs stay cached for up to 24 hours |
| Per calling address | 600 render calls and 1,200 lookups (searches, briefs, design edits, element searches, tile edits, validations, template listings, viewer downloads) per hour; 500 new share links per 24 hours |
| Priority | visitors of the website take precedence; while they occupy the service, an agent receives “busy, retry in 10 s” |
| Capacity | place searches, design validations, and images run within a bounded capacity that the website shares; a call beyond it fails with an error that names “the service accepts no further requests of this kind right now; try again in a moment” |
| Model size | models above about one million triangles are refused for agents; reduce the width or the tile count, or share the design and open it in the configurator |
| Share links | anonymous snapshots valid for 90 days; a repeated call for an unchanged design returns the link issued before while it has at least one day of validity left and the service still holds it in its cache |
| Viewer | each render issues a viewer link valid for one hour |
| Restricted areas, countries | as for visitors of the website: military restricted areas cannot be printed, and the service is not offered in some countries |

Use is free of charge. Bulk retrieval and circumventing the limits are not
permitted; see the terms below.

## Maintenance

While the operator puts the configurator, for visitors or for accounts, or
the whole application into maintenance, the MCP address answers every
request with HTTP 503 instead of a JSON-RPC response, so no tool runs and
MCP clients report a failed request; the downloads of the 3D viewer answer
the same way. The answer carries:

- the header `Retry-After` with the seconds until the expected end of the
  maintenance, at least 60 and at most 86,400, or 300 without an expected
  end;
- a problem JSON body of the type `tag:neuronform.com,2026:maintenance`
  with the code `maintenance.configurator` or `maintenance.site` and the
  object `maintenance` with `scope`, the messages of the operator
  `message_de` and `message_en`, and the expected end `expected_end` (RFC
  3339, UTC), each `null` when the operator set none.

The switches in force are public at
[neuronform.com/map/api/v1/maintenance](https://neuronform.com/map/api/v1/maintenance).
Retry after the time that `Retry-After` names.

## Privacy and terms

A request contains the tool arguments, such as the place and the design, and
the IP address of the calling system: the platform of the chat for Claude or
ChatGPT, the user's own computer for Claude Code, Cursor, or VS Code. When a
client connects, it names its client software and version, which NeuronForm
counts for its usage statistics. NeuronForm receives no account data and no
other content of the chat. Place searches are answered by the place
directory of the NeuronForm server, which it builds from OpenStreetMap data;
the search terms reach no further service.
The providers of the assistants are responsible for the chat contents on
their platforms.

- Privacy policy: [English](https://neuronform.com/privacy/en#agents), [Deutsch](https://neuronform.com/privacy/de#agents)
- Terms, section on AI agents: [English](https://neuronform.com/terms/en#agents), [Deutsch](https://neuronform.com/terms/de#agents)

## Map data and attribution

The service names its data sources from one data source registry. Every
result of `render_design` carries, in the field `attribution`, the
attribution lines of the data sources of its model in the language of the
call, and its image carries the lines of the map data and terrain sources;
the sea-floor depths are named exactly when the model contains them. Keep the
attribution when you show or publish images of a design.

The registry lists every source with its role, attribution line, licence,
the notice its licence requires, and its citation:

- as JSON: [neuronform.com/map/api/v1/data-sources](https://neuronform.com/map/api/v1/data-sources)
- as a page: [neuronform.com/licenses/en](https://neuronform.com/licenses/en)

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
- ihn als Bild und, wo der Chat das unterstützt, als drehbare 3D-Ansicht zeigen;
- einen Link übergeben, der genau diesen Entwurf im Konfigurator öffnet.

Exportieren und kaufen können Agenten nicht; das geht nur im Konfigurator
mit einem eigenen Konto.

- **MCP-Adresse:** `https://neuronform.com/mcp` (Streamable HTTP, Anfragen bis 2 MiB, ohne Konto, ohne API-Schlüssel, kostenlos).
- **Einrichtung:** In Claude unter „Customize“ › „Connectors“ › „+ Add“ › „Add custom connector“, Authentifizierung „No sign in“. In Claude Code mit `claude mcp add --transport http neuronform https://neuronform.com/mcp`. Für ChatGPT, Cursor und VS Code siehe die Abschnitte oben oder die Seite [KI-Integration](https://neuronform.com/agents/de).
- **Werkzeuge:** `search_places`, `create_design`, `edit_design`, `get_design_guide`, `find_map_elements`, `edit_tiles`, `validate_design`, `render_design`, `share_design`, `list_templates`. Jedes Werkzeug nimmt den optionalen Parameter `lang` (`de` oder `en`, Standard `en`) für Zusammenfassungen, Hinweise und Fehlermeldungen.
- **Startlinks ohne MCP:** `https://neuronform.com/map/app?place=Hallstatt&width=3000&shape=hexagon&relief=true&exaggeration=1.5`
- **Grenzen:**
  - Es gelten die Grenzen eines Besuchs ohne Konto (bis zu 3 Kacheln und 20 eigene Linien).
  - Generische Details (Dächer, Fenster, Türen und Bäume) gibt es bis 3 km Kartenbreite; einen breiteren Entwurf mit eingeschalteten Details lehnt der Server ab. Eigene Häuser gibt es bis zu fünf je Entwurf mit einem Detailbudget von 2 Millionen Dreiecken je Haus.
  - Alle Agenten zusammen können pro Stunde 2.000 Modelle neu berechnen lassen; Ansichten eines noch zwischengespeicherten Modells zählen nicht.
  - Besucherinnen und Besucher der Website haben Vorrang.
  - Ortssuchen, Entwurfsprüfungen und Bilder laufen in einer begrenzten Kapazität, die die Website mitnutzt; ist sie ausgeschöpft, lehnt der Server weitere Aufrufe dieser Art vorübergehend ab.
  - Modelle mit mehr als etwa einer Million Dreiecken lehnt der Server für Agenten ab.
- **Wartung:** Solange der Betreiber den Konfigurator oder die ganze Anwendung in Wartung schaltet, beantwortet die MCP-Adresse jede Anfrage mit HTTP 503 statt mit einer JSON-RPC-Antwort. Die Antwort nennt in `Retry-After` die Sekunden bis zum erwarteten Ende (ohne Ende 300) und im Objekt `maintenance` die Nachricht und das erwartete Ende; den Stand zeigt [neuronform.com/map/api/v1/maintenance](https://neuronform.com/map/api/v1/maintenance).
- **Datenschutz und Bedingungen:** [Datenschutzerklärung](https://neuronform.com/privacy/de#agents), [Allgemeine Geschäftsbedingungen, Abschnitt „Nutzung durch KI-Agenten“](https://neuronform.com/terms/de#agents).
- **Kartendaten und Quellen:** Jedes Ergebnis von `render_design` trägt im Feld `attribution` die Namensnennung der Datenquellen seines Modells, das Bild die der Karten- und Geländedaten. Alle Quellen mit Lizenz, Hinweis und Zitat listet das Quellenverzeichnis des Dienstes: [als JSON](https://neuronform.com/map/api/v1/data-sources) und auf der Seite [Lizenzen und Attribution](https://neuronform.com/licenses/de).
