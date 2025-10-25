# SoundCloud Archiver Webhook

A Rust application that monitors SoundCloud users for new track uploads and sends them to a Discord webhook with complete metadata and audio files.

## Quick Start

### Prerequisites

- [ffmpeg](https://ffmpeg.org/download.html) installed and available in your PATH
- A Discord webhook URL ([create one here](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks))

### Download Binary

Download the latest release for your platform from the [Releases page](https://github.com/scarchives/archiver_webhook/releases).

Or download from GitHub Actions artifacts (requires GitHub login):
- [Linux (Debian 12)](https://github.com/scarchives/archiver_webhook/actions)
- [Windows](https://github.com/scarchives/archiver_webhook/actions)

### Generate Configuration

The easiest way to get started is to use the `--generate-config` command, which automatically creates both `config.json` and `users.json` based on a SoundCloud user's followings:

```bash
./archiver_webhook --generate-config https://soundcloud.com/your-username
```

This will:
1. Fetch all users that the specified user follows
2. Interactively prompt you for configuration values (Discord webhook URL, poll interval, etc.)
3. Create `config.json` and `users.json` files automatically
4. Display each followed user with their track count for reference

All prompts have sensible defaults - just press Enter to accept them.

### Run the Application

After generating your configuration:

```bash
./archiver_webhook
```

The application will:
- Monitor all configured SoundCloud users for new tracks
- Download track metadata, audio files, and artwork
- Send everything to your Discord webhook
- Save progress to `tracks.json` to avoid reposting

## Building from Source

### Requirements

- Rust 1.70 or later
- ffmpeg (runtime dependency)

### Build

```bash
git clone https://github.com/scarchives/archiver_webhook.git
cd archiver_webhook
cargo build --release
```

The compiled binary will be in `target/release/archiver_webhook`.

## Docker

### Using Pre-built Image

```bash
# Create configuration files first (run locally or in a temporary container)
docker run --rm -it \
  -v "$(pwd):/data" \
  ghcr.io/scarchive/archiver_webhook:latest \
  --generate-config https://soundcloud.com/your-username

# Run the archiver
docker run -d --name archiver_webhook \
  -v "$(pwd)/config.json:/app/config.json:ro" \
  -v "$(pwd)/users.json:/app/users.json:rw" \
  -v "$(pwd)/tracks.json:/app/tracks.json:rw" \
  -v "$(pwd)/temp:/app/temp:rw" \
  ghcr.io/scarchive/archiver_webhook:latest
```

### Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3'

services:
  archiver_webhook:
    image: ghcr.io/scarchive/archiver_webhook:latest
    container_name: archiver_webhook
    restart: unless-stopped
    volumes:
      - ./config.json:/app/config.json:ro
      - ./users.json:/app/users.json:rw
      - ./tracks.json:/app/tracks.json:rw
      - ./temp:/app/temp:rw
```

Then run:

```bash
# Generate config (one-time)
docker-compose run --rm archiver_webhook --generate-config https://soundcloud.com/your-username

# Start the service
docker-compose up -d

# View logs
docker-compose logs -f
```

## Configuration

The `config.json` file is automatically created by `--generate-config`, but you can edit it manually:

```json
{
  "discord_webhook_url": "YOUR_DISCORD_WEBHOOK_URL",
  "log_level": "info",
  "poll_interval_sec": 60,
  "users_file": "users.json",
  "tracks_file": "tracks.json",
  "max_tracks_per_user": 500,
  "pagination_size": 50,
  "temp_dir": null,
  "max_soundcloud_parallelism": 2,
  "max_discord_parallelism": 4,
  "max_processing_parallelism": 4,
  "scrape_user_likes": false,
  "max_likes_per_user": 500,
  "auto_follow_source": null,
  "auto_follow_interval": 24,
  "db_save_interval": 1,
  "db_save_tracks": 5,
  "show_ffmpeg_output": false,
  "log_file": "latest.log"
}
```

### Key Configuration Options

- **discord_webhook_url** (required): Discord webhook URL for notifications
- **poll_interval_sec**: How often to check for new tracks (default: 60 seconds)
- **max_soundcloud_parallelism**: Parallel SoundCloud API requests (default: 2, keep low to avoid rate limits)
- **max_discord_parallelism**: Parallel Discord webhook requests (default: 4)
- **max_processing_parallelism**: Parallel processing tasks like ffmpeg (default: 4)
- **scrape_user_likes**: Monitor users' liked tracks in addition to their uploads (default: false)
- **auto_follow_source**: User URL/ID to auto-follow their followings (optional)

See the full [Configuration Reference](#configuration-reference) below for all options.

## Command Line Options

```bash
# Run in watcher mode (monitors for new tracks continuously)
./archiver_webhook

# Generate config.json and users.json interactively
./archiver_webhook --generate-config URL

# Resolve a SoundCloud URL and display information
./archiver_webhook --resolve URL

# Initialize tracks database with existing tracks from watched users
./archiver_webhook --init-tracks

# Post a specific track to Discord (bypasses database check)
./archiver_webhook --post-track TRACK_ID_OR_URL

# Look up a track by Discord message ID
./archiver_webhook --lookup-discord-id DISCORD_MESSAGE_ID

# Show help
./archiver_webhook --help
```

## Features

- **Automatic Discovery**: Use `--generate-config` to automatically set up monitoring for all users that a SoundCloud account follows
- **Complete Archival**: Downloads all available audio formats, high-resolution artwork, and complete JSON metadata
- **Discord Integration**: Sends rich embeds with track details and media files to Discord
- **Efficient Monitoring**: Tracks database prevents duplicate posts
- **Parallelism Controls**: Separate limits for SoundCloud API, Discord webhooks, and processing tasks
- **Likes Monitoring**: Optionally monitor users' liked tracks in addition to their uploads
- **Auto-follow**: Automatically add new followings from a source user
- **Flexible Logging**: Configurable log levels with both console and file output

## What Gets Archived

For each new track detected:

1. **Audio Files**: All available formats (MP3, AAC, Opus, etc.) - preserves maximum quality
2. **Artwork**: Original high-resolution cover art
3. **Metadata**: Complete JSON snapshot of track information
4. **Discord Post**: Rich embed with track details, artist info, and all files attached

The bot handles Discord's limits automatically (8MB per file, max 10 attachments).

## Finding SoundCloud User IDs

To find a user's ID or get information about any SoundCloud URL:

```bash
./archiver_webhook --resolve https://soundcloud.com/username
```

This displays the user ID, username, and other details.

## Configuration Reference

<details>
<summary>Click to expand full configuration options</summary>

### Required

- **discord_webhook_url** (string): Discord webhook URL for sending notifications

### General Settings

- **log_level** (string): Logging verbosity - `trace`, `debug`, `info`, `warn`, or `error` (default: `info`)
- **log_file** (string): Path to log file (default: `latest.log`)
- **poll_interval_sec** (number): How often to check for new tracks in seconds (default: `60`)

### File Paths

- **users_file** (string): Path to JSON file with user IDs to monitor (default: `users.json`)
- **tracks_file** (string): Path to JSON database of known tracks (default: `tracks.json`)
- **temp_dir** (string or null): Directory for temporary files (default: system temp directory)

### SoundCloud API Settings

- **max_tracks_per_user** (number): Maximum total tracks to fetch per user (default: `500`)
- **pagination_size** (number): Tracks to fetch per API request (default: `50`)
- **max_soundcloud_parallelism** (number): Concurrent SoundCloud API requests - keep low (1-2) to avoid rate limiting (default: `2`)

### Likes Monitoring

- **scrape_user_likes** (boolean): Whether to monitor users' liked tracks (default: `false`)
- **max_likes_per_user** (number): Maximum likes to fetch per user when enabled (default: `500`)

### Auto-follow

- **auto_follow_source** (string or null): User ID or URL whose followings to automatically add (default: `null`)
- **auto_follow_interval** (number): How often to check for new followings in poll cycles (default: `24`)

### Performance Settings

- **max_discord_parallelism** (number): Concurrent Discord webhook requests (default: `4`)
- **max_processing_parallelism** (number): Concurrent processing tasks like ffmpeg - tune based on CPU cores (default: `4`)

### Database Settings

- **db_save_interval** (number): How often to save database in poll cycles (default: `1`)
- **db_save_tracks** (number): Number of new tracks before auto-saving database (default: `5`)

### Debugging

- **show_ffmpeg_output** (boolean): Show ffmpeg output in console logs (default: `false`)

</details>

## Troubleshooting

### ffmpeg not found

Ensure ffmpeg is installed and in your system PATH:

```bash
# Test if ffmpeg is available
ffmpeg -version

# Linux (Debian/Ubuntu)
sudo apt-get install ffmpeg

# macOS
brew install ffmpeg

# Windows
# Download from https://ffmpeg.org/download.html
```

### Rate Limiting

If you encounter SoundCloud rate limiting:
- Reduce `max_soundcloud_parallelism` to 1
- Increase `poll_interval_sec`
- Reduce `max_tracks_per_user`

### Discord Upload Failures

If Discord rejects uploads:
- Check file size limits (8MB for regular servers, 50MB for boosted)
- Verify webhook URL is correct and active
- Reduce `max_discord_parallelism` if getting rate limited

## License

MIT License - see [LICENSE](LICENSE) file for details.
