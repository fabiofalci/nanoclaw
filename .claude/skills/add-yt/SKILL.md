---
name: add-yt
description: Add @Name yt <URL> command to download YouTube/web videos via yt-dlp and send them back to WhatsApp. Requires yt-dlp installed on the host machine.
---

# Add yt-dlp Video Download Command

Adds a `@Name yt <URL>` command that downloads a video using yt-dlp on the host machine and sends it back to WhatsApp. Owner-only (triggered only by `is_from_me` messages).

## Pre-flight

1. Verify `yt-dlp` is installed on the host: `which yt-dlp` — if missing, install it (`brew install yt-dlp` on macOS, `pip install yt-dlp` or your package manager on Linux)
2. Check if this skill is already applied: `grep -n "handleYtDownload" src/index.ts` — if found, skip to Verify

## Apply Changes

### 1. `src/types.ts` — add `sendFile` to the Channel interface

Find the Channel interface and add `sendFile?` after `syncGroups?`:

```typescript
  // Optional: send a local file (video, image, document) to a chat.
  sendFile?(jid: string, filePath: string, mimetype: string, caption?: string): Promise<void>;
```

### 2. `src/channels/whatsapp.ts` — implement `sendFile`

Add this method to `WhatsAppChannel` after the `sendMessage` method:

```typescript
  async sendFile(jid: string, filePath: string, mimetype: string, caption?: string): Promise<void> {
    const buffer = fs.readFileSync(filePath);
    await this.sock.sendMessage(jid, { video: buffer, mimetype, caption });
    logger.info({ jid, mimetype }, 'File sent');
  }
```

### 3. `src/index.ts` — add imports, handler function, and intercept

**Add imports** at the top (after existing imports):

```typescript
import { execFile } from 'child_process';
import os from 'os';
```

**Add `handleYtDownload` function** inside `main()`, right before the `// Channel callbacks` comment:

```typescript
  // Handle "@Name yt <URL>" command: download video with yt-dlp and send back
  async function handleYtDownload(url: string, chatJid: string): Promise<void> {
    const channel = findChannel(channels, chatJid);
    if (!channel) return;

    const tmpFile = path.join(os.tmpdir(), `nanoclaw-yt-${Date.now()}.mp4`);
    try {
      await channel.sendMessage(chatJid, 'Downloading video...');
      await new Promise<void>((resolve, reject) => {
        execFile(
          'yt-dlp',
          ['-f', 'bestvideo[ext=mp4]+bestaudio[ext=m4a]/mp4', '-o', tmpFile, url],
          (err, _stdout, stderr) => {
            if (err) reject(new Error(stderr || err.message));
            else resolve();
          },
        );
      });
      if (channel.sendFile) {
        await channel.sendFile(chatJid, tmpFile, 'video/mp4');
      } else {
        await channel.sendMessage(chatJid, 'Video downloaded but this channel does not support file sending.');
      }
    } catch (err) {
      await channel.sendMessage(chatJid, `yt failed: ${err instanceof Error ? err.message : String(err)}`);
    } finally {
      if (fs.existsSync(tmpFile)) fs.unlinkSync(tmpFile);
    }
  }
```

**Add the intercept** inside the `onMessage` callback, right after the `/remote-control` block:

```typescript
      // @Name yt <URL> — download video with yt-dlp and send back (owner only)
      const ytMatch = trimmed.match(
        new RegExp(`^@${ASSISTANT_NAME}\\s+yt\\s+(\\S+)$`, 'i'),
      );
      if (ytMatch && msg.is_from_me) {
        handleYtDownload(ytMatch[1], chatJid).catch((err) =>
          logger.error({ err, chatJid }, 'yt download error'),
        );
        return;
      }
```

## Build and Restart

```bash
npm run build
# macOS:
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
# Linux:
systemctl --user restart nanoclaw
```

## Verify

Send `@YourAssistantName yt https://www.youtube.com/watch?v=dQw4w9WgXcQ` from your own WhatsApp in any registered chat. You should see:
1. "Downloading video..." reply
2. The video sent back shortly after

## Notes

- **Owner-only**: only messages sent from your own WhatsApp account (`is_from_me`) trigger the download — other group members cannot use it
- **Host yt-dlp**: the command runs on the machine hosting NanoClaw, not inside the agent container
- **Temp files**: downloaded to `/tmp/nanoclaw-yt-<timestamp>.mp4` and deleted after sending
- **Format**: tries best mp4 video + m4a audio; falls back to any available mp4
