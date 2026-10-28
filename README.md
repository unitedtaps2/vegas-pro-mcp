# VEGAS Pro MCP

Control VEGAS Pro 2026 from an MCP-compatible AI assistant on Windows. Inspect a
project, edit the timeline, work with audio and effects, and render through nine
tools with 100 actions.

**Public beta: 2.1.0-beta.1.** Developed by United Taps. This is an independent
community project; it is not an official VEGAS, Boris FX, or MAGIX product.

The server runs locally over MCP stdio. A C# application extension runs inside
VEGAS and accepts authenticated named-pipe requests from your Windows account.
No cloud service, API key, or paid connector subscription is required. Your AI
client may send tool inputs and results, including project paths and metadata,
to its model provider under that client's settings.

## Requirements

- Windows 10 or 11, 64-bit.
- A licensed installation of VEGAS Pro 2026. The local integration was checked
  with Build 143; other versions are unverified.
- Node.js 22 or newer (22 and 24 are the CI targets).
- Windows PowerShell 5.1 and .NET Framework 4.8, including its C# compiler.
- An MCP client that supports local stdio servers.
- Optional: FFmpeg on the VEGAS process PATH, or an explicit `ffmpeg_path`, for
  LUFS measurement. Other tools do not require FFmpeg.

VEGAS, its scripting DLL, FFmpeg, media, credentials, and compiled binaries are
not included. The extension is compiled against your installed VEGAS scripting
API; no separate Visual Studio installation is needed.

## Install from source

Run these commands in PowerShell:

```powershell
git clone https://github.com/unitedtaps2/vegas-pro-mcp.git
cd vegas-pro-mcp
npm ci --ignore-scripts
npm run check
npm run smoke
```

**Save your work and close every VEGAS process before installing the extension.**
The installer refuses to replace the extension while VEGAS is running.

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\install-extension.ps1
```

The installer builds `dist\UnitedTaps.VegasMcp.dll`, creates a local random token,
restricts its permissions, and installs under the Windows Documents Known Folder:
`Vegas Application Extensions`. This supports redirected Documents folders.
An existing DLL is copied to a unique directory under `backups` before replacement.
Existing tokens are preserved and are never uploaded.

For a nonstandard VEGAS installation, pass its directory:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\install-extension.ps1 -VegasInstallPath 'D:\Apps\Vegas Pro 2026'
```

Then start VEGAS. Under Tools > Extensions, the **United Taps VEGAS MCP Status**
command should report that the connector is running. The extension's internal
DLL, pipe and menu names retain the United Taps prefix for compatibility.

## Connect your AI client

Register a local stdio server using `node` and the **absolute path** to
`src/server.mjs`. In clients that use an `mcpServers` JSON object, the entry looks
like this; replace the example path with your clone location:

```json
{
  "mcpServers": {
    "vegas-pro": {
      "command": "node",
      "args": ["C:\\Tools\\vegas-pro-mcp\\src\\server.mjs"]
    }
  }
}
```

Follow your client's configuration format and restart or reconnect it. The
server must have the same Windows account as VEGAS. Do not run either program
as administrator unless you have a specific need and understand the implications.
You can also launch `run-vegas-mcp.cmd`; it locates the source relative to itself.

The default installation directory is detected automatically. If you deliberately
installed elsewhere using `-InstallRoot`, set `VEGAS_MCP_INSTALL_ROOT` in your MCP
client's server environment to the same directory. `VEGAS_MCP_TOKEN_FILE` can
select a different token-file path; its contents must match the extension's
installed token. Never put a token value in client configuration or an issue.

## First use

1. Open and save a disposable VEGAS project.
2. Ask the assistant to call `vegas_capabilities`, then `vegas_status`.
3. Inspect the project and timeline before asking for an edit.
4. Review a small edit in VEGAS, including picture, sound, and Undo, before
   using it on important work.

If more than one VEGAS process is connected, pass `vegas_pid` inside the action's
`params` (or directly to `vegas_status`). Obtain it from status discovery.
A useful first request is: "Inspect my VEGAS project and summarize its tracks.
Do not change anything."

| Tool | Purpose |
| --- | --- |
| `vegas_capabilities` | List tools, actions, and safety guarantees |
| `vegas_status` | Discover running instances and inspect connection status |
| `vegas_project` | Project settings, backups, transport, snapshots, Undo/Redo |
| `vegas_timeline` | Query, insert, move, trim, split, group, fade, and batch |
| `vegas_audio` | Gain, normalization, LUFS, envelopes, buses, and routing |
| `vegas_effects` | Discover and edit supported effects and OFX parameters |
| `vegas_media` | Media inspection, bins, takes, and image sequences |
| `vegas_motion` | Event pan/crop and track-motion keyframes |
| `vegas_render` | Discover templates and start, inspect, or cancel a render |

See [usage examples](docs/USAGE.md) and the
[action and safety contract](docs/VEGAS-MCP-ACTION-CONTRACT.md).

## Editing safeguards and limits

- Content edits require the exact saved active project path. Existing-event
  edits also require the inspected event ID and prior start/length.
- Content edits create a separate `.veg` backup in Documents\VEGAS MCP Backups
  before entering a named VEGAS Undo step. Source media is never edited or deleted.
- Backups contain project state, not copies of source media. Keep your normal
  media and project backups. Connector backups are not automatically pruned.
- Output paths that already exist are refused by snapshots and renders.
- Destructive project operations have additional acknowledgement flags.
- Batch dry runs check routes and supplied target identities; they do not
  simulate every action-specific validation. Apply uses one backup and Undo step.
- The experimental all-track `ripple_delete` action has explicit limits around
  grouping and automation. Read its contract before enabling its acknowledgements.
- If an operation times out, inspect VEGAS before retrying. An operation already
  running inside VEGAS may still finish. Expired queued requests are refused by
  the public beta extension.
- Third-party plug-ins, every render template, and every media type are not
  covered. Use [the validation record](docs/VALIDATION.md) to see what was checked.

## Development and checks

```powershell
npm run check          # syntax checks and 30 focused tests
npm run smoke          # real MCP stdio handshake, catalog, rejected unsafe calls
npm run test:native    # 12 range-edit algorithm scenarios; Windows only
npm run test:installer # isolated install/update/rollback fixture; Windows only
powershell.exe -NoProfile -File .\scripts\build-extension.ps1
npm audit --omit=dev
```

CI runs the Node tests and protocol smoke on Windows and Linux, Node 22 and 24.
Windows also runs the range-edit model scenarios. CI cannot compile against or
run VEGAS because its proprietary API is not shipped in this repository.

Generated output rule: the build replaces the stable generated DLL under `dist`.
Do not hand-edit compiled output. Runtime files, outputs, backups, dependency
folders, and secrets must stay untracked. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Troubleshooting

- **No live instance:** verify installation, fully restart VEGAS after installing,
  and check Tools > Extensions. Inspect the log under the installation directory's
  `UnitedTaps.VegasMcp.runtime` folder. Logs can contain private paths.
- **Wrong installation folder:** set `VEGAS_MCP_INSTALL_ROOT` to the installer result.
- **Active project changed:** inspect again and use the returned exact path and
  event identities. Do not reuse IDs across project reloads or Undo/Redo.
- **FFmpeg not found:** pass `ffmpeg_path` or restart VEGAS after adding FFmpeg to PATH.
- **Compiler/API missing:** check .NET Framework and use `-VegasInstallPath`.

Before reporting an issue, remove tokens, user names, private paths, media names,
and customer information. Do not upload your project or source media by default.
See [SECURITY.md](SECURITY.md) for private vulnerability reporting.

## License and references

MIT license: free to use, change, and share; see [LICENSE](LICENSE). Third-party
dependencies retain their own licenses. VEGAS and other product names belong to
their respective owners.

**Acknowledgment:** During the initial research for this project, we reviewed
[Marco Ravich](https://github.com/MarcoRavich)'s
[VEGAS-AI-control](https://github.com/MarcoRavich/VEGAS-AI-control), an earlier
public exploration of AI control through the VEGAS scripting API, released under
CC0 1.0. United Taps subsequently developed a separate MCP server and C#
application extension.

- [MCP TypeScript SDK: local stdio servers](https://ts.sdk.modelcontextprotocol.io/server)
- [MCP local-server security](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/docs/2026-07-28/tutorials/security/local-server-security.mdx)
- [Official VEGAS scripting and extensions documentation](https://cdn.borisfx.com/borisfx/Documentation/vegas/2026/en/content/topics/11-autoscript/scripting_vegas.htm)
