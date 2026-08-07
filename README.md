# CronzPH Phone Storage MCP Bridge

A complete, beginner-friendly guide to running a full-access filesystem MCP server on your Android phone (via Termux), exposing it to the internet through a Cloudflare Quick Tunnel, and connecting it to Claude through a permanent (static) URL — so you never have to update your Claude connector settings again, even when the tunnel changes.

If you follow every step below in order, copying and pasting exactly what's shown, this will work even if you've never done anything like this before.

---

## What You're Building

Three pieces working together:

1. **MCP Storage Server** — a small program that runs on your phone (inside Termux) and gives Claude the ability to list, read, write, delete, search, and create files/folders in your phone's storage.
2. **Cloudflare Tunnel** — a free service that gives your phone's local server a public internet address, since your phone doesn't have a fixed public IP.
3. **Cloudflare Worker Bridge** — a small piece of code hosted permanently by Cloudflare that acts as a "forwarding address." You give Claude this one permanent URL, and it automatically forwards every request to whatever tunnel address is currently active — even if that address changes every time you restart your phone or Termux.

Here's how a request travels:

```
Claude
  → https://mcp1.cronzph101.workers.dev/mcp   (this NEVER changes)
  → Cloudflare Worker (looks up the current address)
  → https://random-words.trycloudflare.com    (this DOES change each restart)
  → cloudflared tunnel running on your phone
  → your MCP server on localhost:8787
  → your phone's storage folder
```

You only need to set up the Worker bridge **once**. After that, every time you restart the tunnel, it automatically tells the Worker its new address — so the URL you gave Claude keeps working forever without you touching it again.

---

## What You Need Before Starting

- An Android phone with **Termux** installed (a terminal/command-line app)
- A free **Cloudflare account** (sign up at cloudflare.com if you don't have one)
- About 30–45 minutes the first time you do this

---

# PART 1 — Build the MCP Storage Server

This is the program that actually reads and writes your phone's files. It runs inside Termux.

### Step 1.1 — Install the required packages

Open Termux and type this, then press Enter:

```bash
pkg install nodejs termux-api
```

Wait for it to finish installing.

### Step 1.2 — Give Termux permission to access your phone's storage

Type this and press Enter:

```bash
termux-setup-storage
```

Android will show a permission popup. Tap **"Allow"**. This creates a folder at `~/storage/shared` that links to your phone's normal storage (Downloads, Documents, Pictures, etc.).

### Step 1.3 — Create a folder for the server

```bash
mkdir -p ~/mcp-storage-server
cd ~/mcp-storage-server
```

### Step 1.4 — Create the server file

Copy the entire block below (everything from `cat >` down to the final `EOF`), paste it into Termux as one single paste, and press Enter:

```bash
cat > server.js << 'ENDOFFILE'
#!/usr/bin/env node
/**
 * CronzPH Phone Storage MCP Server
 * Full-access filesystem MCP server for Termux — list, read, write, delete, search.
 */

import express from "express";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { z } from "zod";
import fs from "fs/promises";
import path from "path";
import os from "os";

const ROOT = process.env.STORAGE_ROOT || path.join(os.homedir(), "storage", "shared");
const PORT = process.env.PORT || 8787;

function resolveSafe(relPath) {
  const target = path.resolve(ROOT, relPath || ".");
  if (!target.startsWith(path.resolve(ROOT))) {
    throw new Error(`Access denied: "${relPath}" is outside the allowed storage root.`);
  }
  return target;
}

function fmtEntry(name, stat) {
  return {
    name,
    type: stat.isDirectory() ? "directory" : "file",
    size: stat.size,
    modified: stat.mtime.toISOString(),
  };
}

const server = new McpServer({ name: "phone-storage", version: "1.0.0" });

server.registerTool(
  "storage_list",
  {
    title: "List files and folders",
    description: "List files and subfolders inside a directory relative to the storage root. Use '.' for the root itself.",
    inputSchema: { path: z.string().default(".").describe("Relative path to list, e.g. 'Download' or '.'") },
    annotations: { readOnlyHint: true, openWorldHint: false },
  },
  async ({ path: relPath }) => {
    const dir = resolveSafe(relPath);
    const names = await fs.readdir(dir);
    const entries = await Promise.all(
      names.map(async (name) => {
        try {
          const stat = await fs.stat(path.join(dir, name));
          return fmtEntry(name, stat);
        } catch {
          return { name, type: "unknown", error: "could not stat" };
        }
      })
    );
    return {
      content: [{ type: "text", text: JSON.stringify(entries, null, 2) }],
      structuredContent: { path: relPath, entries },
    };
  }
);

server.registerTool(
  "storage_read",
  {
    title: "Read a file",
    description: "Read the text content of a file relative to the storage root.",
    inputSchema: {
      path: z.string().describe("Relative file path, e.g. 'Documents/notes.txt'"),
      encoding: z.enum(["utf8", "base64"]).default("utf8").describe("Use base64 for binary files like images"),
    },
    annotations: { readOnlyHint: true, openWorldHint: false },
  },
  async ({ path: relPath, encoding }) => {
    const file = resolveSafe(relPath);
    const data = await fs.readFile(file, encoding === "base64" ? "base64" : "utf8");
    return { content: [{ type: "text", text: data }] };
  }
);

server.registerTool(
  "storage_write",
  {
    title: "Write or create a file",
    description: "Write text content to a file relative to the storage root. Creates parent folders if needed.",
    inputSchema: {
      path: z.string().describe("Relative file path to write to"),
      content: z.string().describe("Text content to write"),
      mode: z.enum(["overwrite", "append"]).default("overwrite"),
    },
    annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: false },
  },
  async ({ path: relPath, content, mode }) => {
    const file = resolveSafe(relPath);
    await fs.mkdir(path.dirname(file), { recursive: true });
    if (mode === "append") {
      await fs.appendFile(file, content, "utf8");
    } else {
      await fs.writeFile(file, content, "utf8");
    }
    return { content: [{ type: "text", text: `Wrote ${content.length} chars to ${relPath}` }] };
  }
);

server.registerTool(
  "storage_delete",
  {
    title: "Delete a file or folder",
    description: "Permanently delete a file or folder (recursive) relative to the storage root. Cannot be undone.",
    inputSchema: { path: z.string().describe("Relative path to delete") },
    annotations: { readOnlyHint: false, destructiveHint: true, idempotentHint: true },
  },
  async ({ path: relPath }) => {
    const target = resolveSafe(relPath);
    await fs.rm(target, { recursive: true, force: true });
    return { content: [{ type: "text", text: `Deleted ${relPath}` }] };
  }
);

server.registerTool(
  "storage_mkdir",
  {
    title: "Create a folder",
    description: "Create a directory (and parents if needed) relative to the storage root.",
    inputSchema: { path: z.string().describe("Relative folder path to create") },
    annotations: { readOnlyHint: false, destructiveHint: false, idempotentHint: true },
  },
  async ({ path: relPath }) => {
    const dir = resolveSafe(relPath);
    await fs.mkdir(dir, { recursive: true });
    return { content: [{ type: "text", text: `Created folder ${relPath}` }] };
  }
);

server.registerTool(
  "storage_search",
  {
    title: "Search for files by name",
    description: "Recursively search for files/folders whose name contains the query string, starting from a path.",
    inputSchema: {
      query: z.string().describe("Substring to search for in file/folder names (case-insensitive)"),
      path: z.string().default(".").describe("Relative folder to start searching from"),
      max_results: z.number().int().min(1).max(500).default(50),
    },
    annotations: { readOnlyHint: true, openWorldHint: false },
  },
  async ({ query, path: relPath, max_results }) => {
    const startDir = resolveSafe(relPath);
    const q = query.toLowerCase();
    const results = [];

    async function walk(dir) {
      if (results.length >= max_results) return;
      let names;
      try {
        names = await fs.readdir(dir);
      } catch {
        return;
      }
      for (const name of names) {
        if (results.length >= max_results) return;
        const full = path.join(dir, name);
        let stat;
        try {
          stat = await fs.stat(full);
        } catch {
          continue;
        }
        if (name.toLowerCase().includes(q)) {
          results.push({
            path: path.relative(ROOT, full),
            type: stat.isDirectory() ? "directory" : "file",
            size: stat.size,
          });
        }
        if (stat.isDirectory()) {
          await walk(full);
        }
      }
    }

    await walk(startDir);
    return {
      content: [{ type: "text", text: JSON.stringify(results, null, 2) }],
      structuredContent: { query, results },
    };
  }
);

const app = express();
app.use(express.json({ limit: "25mb" }));

app.post("/mcp", async (req, res) => {
  try {
    const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined });
    res.on("close", () => transport.close());
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  } catch (err) {
    console.error("MCP error:", err);
    if (!res.headersSent) {
      res.status(500).json({ jsonrpc: "2.0", error: { code: -32603, message: "Internal server error" }, id: null });
    }
  }
});

app.get("/health", (req, res) => res.json({ ok: true, root: ROOT }));

app.listen(PORT, "127.0.0.1", () => {
  console.log(`Phone Storage MCP server running on http://127.0.0.1:${PORT}/mcp`);
  console.log(`Storage root: ${ROOT}`);
});
ENDOFFILE
```

After pasting, press Enter. You should see no errors — just a new command prompt.

**What this server can do (its tools):**
| Tool | What it does |
|---|---|
| `storage_list` | Lists files and folders in a directory |
| `storage_read` | Reads the text content of a file |
| `storage_write` | Creates or overwrites a file with text |
| `storage_delete` | Permanently deletes a file or folder |
| `storage_mkdir` | Creates a new folder |
| `storage_search` | Searches for files/folders by name |

All of these are locked inside your phone's shared storage folder — the server cannot touch anything outside of it, for safety.

### Step 1.5 — Create the package.json file

Copy and paste this whole block as one paste:

```bash
cat > package.json << 'ENDOFFILE'
{
  "name": "phone-storage-mcp",
  "version": "1.0.0",
  "type": "module",
  "description": "Full-access filesystem MCP server for phone storage via Termux",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.4",
    "express": "^4.21.1",
    "zod": "^3.23.8"
  }
}
ENDOFFILE
```

### Step 1.6 — Install the dependencies

```bash
npm install
```

This will take a minute or two. Wait until it finishes and you see a new prompt.

### Step 1.7 — Test that the server runs

```bash
node server.js
```

You should see this output:

```
Phone Storage MCP server running on http://127.0.0.1:8787/mcp
Storage root: /data/data/com.termux/files/home/storage/shared
```

If you see this, it's working. Leave it running for now, or press `Ctrl+C` to stop it — we'll start it properly later with the launcher script in Part 4.

---

# PART 2 — Build the Cloudflare Worker Bridge

This is the "permanent forwarding address." You only set this up once, on the Cloudflare website (works fine in your phone's browser).

### Step 2.1 — Create a Worker

1. Go to **dash.cloudflare.com** and log in.
2. In the left sidebar, tap **Compute** → **Workers & Pages**.
3. Tap the blue **"Create application"** button.
4. Tap **"Start with Hello World!"**.
5. You can rename it or leave the default name — either is fine, just remember it.
6. Tap **"Deploy"**.

### Step 2.2 — Replace the code with the bridge script

1. On your new Worker's page, tap **"Edit code"**.
2. Select and delete everything in the code editor.
3. Paste in the entire code block below:

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    if (url.pathname === "/update" && request.method === "POST") {
      const auth = request.headers.get("Authorization") || "";
      if (auth !== `Bearer ${env.UPDATE_SECRET}`) {
        return new Response("Unauthorized", { status: 401 });
      }
      const body = await request.text();
      const params = new URLSearchParams(body);
      const newTarget = params.get("url");
      if (!newTarget || !newTarget.startsWith("https://")) {
        return new Response("Missing or invalid 'url' param", { status: 400 });
      }
      await env.MCP_TARGET.put("target", newTarget.replace(/\/+$/, ""));
      return new Response(`Target updated to: ${newTarget}`, { status: 200 });
    }

    if (url.pathname === "/health") {
      const target = await env.MCP_TARGET.get("target");
      return Response.json({ ok: true, target: target || null });
    }

    const target = await env.MCP_TARGET.get("target");
    if (!target) {
      return new Response("No backend target configured yet. POST to /update first.", { status: 503 });
    }

    const destUrl = target + url.pathname + url.search;
    const proxied = new Request(destUrl, {
      method: request.method,
      headers: request.headers,
      body: ["GET", "HEAD"].includes(request.method) ? undefined : request.body,
    });

    try {
      return await fetch(proxied);
    } catch (err) {
      return new Response(`Bridge could not reach backend: ${err.message}`, { status: 502 });
    }
  },
};
```

4. Tap **"Deploy"** to publish this code.

### Step 2.3 — Create the KV storage (where the current address is remembered)

1. On your Worker's page, tap the **"Bindings"** tab.
2. Tap **"Add binding"**.
3. Choose **"KV namespace"**.
4. If a dropdown appears without a "create new" option:
   - Go to the left sidebar → **"Storage & databases"** → **"KV"**.
   - Tap **"Create a namespace"**, name it `MCP_TARGET`, and create it.
   - Go back to your Worker → Bindings → Add binding → KV Namespace.
5. Fill in:
   - **Variable name:** `MCP_TARGET`
   - **KV namespace:** select the `MCP_TARGET` namespace you just created.
6. Confirm/Add the binding.

### Step 2.4 — Create your secret password

This secret protects your bridge so strangers can't hijack it and redirect it somewhere malicious.

1. On your Worker's page, go to **"Settings"** → **"Variables"**.
2. Tap **"Add variable"**.
3. Set **Type** to **"Secret"**.
4. **Variable name:** `UPDATE_SECRET`
5. **Value:** make up a long random string yourself, for example something like `cronzph-mcp-key-2026-xk9pL4mQ7vN2wR8t`. Write this down somewhere safe (like a notes app) — you will need to type it again in Part 3.
6. Tap **"Add variable"**, then **"Deploy"**.

### Step 2.5 — Find your permanent Worker URL

On your Worker's overview page, you'll see your address. It looks like:

```
https://<your-worker-name>.<your-cloudflare-subdomain>.workers.dev
```

For example: `https://mcp1.cronzph101.workers.dev`

**Write this down** — this is the permanent URL you'll eventually give to Claude.

### Step 2.6 — Test it

Open this in your phone's browser (replace with your actual Worker URL):

```
https://mcp1.cronzph101.workers.dev/health
```

You should see:
```
{"ok":true,"target":null}
```

`target: null` is expected for now — we haven't started the tunnel yet.

---

# PART 3 — Connect the Tunnel to the Bridge

Now we connect your phone's server to the internet, and make it automatically tell the Worker its current address.

### Step 3.1 — Install cloudflared

In Termux:

```bash
pkg install cloudflared
```

### Step 3.2 — Create the launcher script

This script starts the tunnel and automatically sends the new address to your Worker.

**Important:** Copy the ENTIRE block below as one single paste. Before pasting, replace two things in the text you're about to paste:
- `WORKER_URL="https://mcp1.cronzph101.workers.dev"` → change to your actual Worker URL from Step 2.5
- `UPDATE_SECRET="PASTE_YOUR_SECRET_HERE"` → change to your actual secret from Step 2.4

```bash
cat > ~/start-tunnel.sh << 'ENDOFFILE'
#!/data/data/com.termux/files/usr/bin/bash
WORKER_URL="https://mcp1.cronzph101.workers.dev"
UPDATE_SECRET="PASTE_YOUR_SECRET_HERE"
LOCAL_PORT=8787
LOGFILE="/data/data/com.termux/files/home/cloudflared.log"
echo "Starting cloudflared tunnel to localhost:$LOCAL_PORT ..."
cloudflared tunnel --url "http://localhost:$LOCAL_PORT" > "$LOGFILE" 2>&1 &
CF_PID=$!
echo "Waiting for tunnel URL to appear in logs..."
NEW_URL=""
for i in $(seq 1 30); do
  NEW_URL=$(grep -o 'https://[a-zA-Z0-9.-]*\.trycloudflare\.com' "$LOGFILE" | head -n1)
  if [ -n "$NEW_URL" ]; then
    break
  fi
  sleep 1
done
if [ -z "$NEW_URL" ]; then
  echo "Could not detect tunnel URL after 30s. Check $LOGFILE"
  exit 1
fi
echo "Detected tunnel URL: $NEW_URL"
echo "Updating Worker bridge..."
curl -s -X POST "$WORKER_URL/update" -H "Authorization: Bearer $UPDATE_SECRET" -d "url=$NEW_URL"
echo ""
echo "Done. Claude connector stays pointed at: $WORKER_URL/mcp"
echo "cloudflared is running in background (PID $CF_PID). Logs: $LOGFILE"
ENDOFFILE
chmod +x ~/start-tunnel.sh
```

### Step 3.3 — Run it

Make sure your MCP server is still running from Step 1.7 (if you stopped it, run `cd ~/mcp-storage-server && node server.js &` first). Then:

```bash
cd ~
./start-tunnel.sh
```

You should see output like this:

```
Starting cloudflared tunnel to localhost:8787 ...
Waiting for tunnel URL to appear in logs...
Detected tunnel URL: https://random-words-here.trycloudflare.com
Updating Worker bridge...
Target updated to: https://random-words-here.trycloudflare.com

Done. Claude connector stays pointed at: https://mcp1.cronzph101.workers.dev/mcp
cloudflared is running in background (PID 12345). Logs: /data/data/com.termux/files/home/cloudflared.log
```

### Step 3.4 — Verify everything is connected

Open this in your browser (your actual Worker URL):

```
https://mcp1.cronzph101.workers.dev/health
```

You should now see the target filled in:

```
{"ok":true,"target":"https://random-words-here.trycloudflare.com"}
```

If you see this, your entire pipeline is working end to end.

---

# PART 4 — One-Command Launcher (Optional but Recommended)

Instead of running two separate commands every time (start the server, then start the tunnel), use this Python script to do both with a single command: `python pmcp.py`

It will:
1. Start the MCP storage server in the background
2. Start the tunnel and automatically update the Worker bridge
3. Keep everything running until you stop it

**To stop everything cleanly, press `Ctrl+C`.** The script listens for this and shuts down both the server and the tunnel properly before exiting — it won't just leave orphaned processes running.

### Step 4.1 — Install Python

```bash
pkg install python
```

### Step 4.2 — Create the launcher script

```bash
cat > ~/pmcp.py << 'ENDOFFILE'
#!/usr/bin/env python3
"""
CronzPH Phone Storage MCP Launcher
Run everything with one command: python pmcp.py
Stop everything cleanly with: Ctrl+C
"""

import subprocess
import os
import signal
import sys
import time

HOME = os.path.expanduser("~")
SERVER_DIR = os.path.join(HOME, "mcp-storage-server")

server_process = None
tunnel_process = None


def cleanup(signum=None, frame=None):
    print("\nStopping processes...")
    if tunnel_process and tunnel_process.poll() is None:
        tunnel_process.terminate()
    if server_process and server_process.poll() is None:
        server_process.terminate()
    sys.exit(0)


def main():
    global server_process, tunnel_process

    signal.signal(signal.SIGINT, cleanup)
    signal.signal(signal.SIGTERM, cleanup)

    print("Starting MCP storage server...")
    server_process = subprocess.Popen(
        "node server.js",
        shell=True,
        cwd=SERVER_DIR,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True,
        bufsize=1,
    )

    for _ in range(20):
        line = server_process.stdout.readline()
        if line:
            print(f"[server] {line.strip()}")
            if "running on" in line:
                break
        else:
            time.sleep(0.5)

    print("\nStarting tunnel + Worker bridge update...")
    tunnel_process = subprocess.Popen(
        "bash start-tunnel.sh",
        shell=True,
        cwd=HOME,
    )
    tunnel_process.wait()

    print("\nMCP server is still running in the background.")
    print("Press Ctrl+C to stop everything.")

    try:
        while True:
            time.sleep(3600)
    except KeyboardInterrupt:
        cleanup()


if __name__ == "__main__":
    main()
ENDOFFILE
```

### Step 4.3 — Run it

```bash
cd ~
python pmcp.py
```

That's it — one command starts everything. Press `Ctrl+C` any time to stop the server and tunnel together.

---

# PART 5 — Connect to Claude

1. Open the Claude app or claude.ai.
2. Go to **Settings** → **Connectors**.
3. Tap **"Add custom connector"**.
4. Enter your permanent Worker URL with `/mcp` at the end, for example:
   ```
   https://mcp1.cronzph101.workers.dev/mcp
   ```
5. Tap **"Add"**.

You should now be able to ask Claude things like "list the files in my Downloads folder" and it will use your phone's storage tools.

---

# Every Time You Restart Your Phone or Close Termux

Everything stops when Termux closes or your phone restarts. To bring it back up, just run:

```bash
cd ~
python pmcp.py
```

**You do NOT need to change anything in Claude's connector settings** — the URL you gave Claude never changes. Only the address behind the scenes changes, and the script updates that automatically.

To double check it reconnected properly, visit:
```
https://<your-worker-url>/health
```

### Optional: Prevent Android from killing Termux in the background

```bash
termux-wake-lock
```

Run this once per session if you want the server/tunnel to keep running even when you switch to other apps.

---

# Troubleshooting

**"Error: listen EADDRINUSE: address already in use 127.0.0.1:8787"**
This means the server is already running from before. This is not a real error — you don't need to start it again.

**Script gives "command not found" errors on every line**
This usually means hidden characters got introduced when typing the script manually (for example, through a text editor like `nano`). The fix: always use the `cat > filename << 'ENDOFFILE' ... ENDOFFILE` method shown in this guide, pasted as one single block, instead of typing line by line into `nano`.

**The tunnel URL keeps changing and Claude stops working**
This should not happen if you're using the Worker bridge URL (`.workers.dev/mcp`) in Claude, not the `.trycloudflare.com` URL directly. Double-check your Claude connector is set to the Worker URL, not the tunnel URL.

**`/health` shows `"target":null` even after running the tunnel**
The tunnel script may not have successfully reached the Worker. Check that `WORKER_URL` and `UPDATE_SECRET` inside `start-tunnel.sh` exactly match what you set up in Cloudflare (Step 2.4 and 2.5).

---

# Technical Notes

- Termux does not have `systemd`, so standard daemon/service modes for `tailscaled` or `cloudflared service install` do not work. This setup uses manual/foreground process mode instead, which is why the tunnel needs to be relaunched after every restart.
- Tailscale on Termux has the same systemd limitation, which is why this setup uses Cloudflare Quick Tunnel instead.
- Cloudflare's "Published application" tunnel routes require you to own a custom domain — they don't work with the free `.workers.dev` subdomain. That's why this setup uses a Quick Tunnel (no domain required) combined with a Worker bridge (which does work on the free `.workers.dev` subdomain).
- All storage operations are locked inside `STORAGE_ROOT` (default: `~/storage/shared`) as a safety measure — the MCP server cannot access anything outside your shared storage folder.
