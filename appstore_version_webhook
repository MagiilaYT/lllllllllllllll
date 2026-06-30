#!/usr/bin/env python3
"""
Discord Webhook Script for App Store Version Tracking
Only sends a webhook when a NEW version is detected.

Usage:
    python appstore_version_webhook.py --app-id 1580330718 --webhook-url URL
"""

import argparse
import json
import os
import sys
import urllib.request
import urllib.error

STATE_FILE = "last_version.json"


def load_last_version() -> str | None:
    """Load the last known version from state file."""
    if not os.path.exists(STATE_FILE):
        return None
    try:
        with open(STATE_FILE, "r") as f:
            data = json.load(f)
            return data.get("version")
    except Exception:
        return None


def save_version(version: str, app_name: str) -> None:
    """Save the current version to state file."""
    with open(STATE_FILE, "w") as f:
        json.dump({"version": version, "app_name": app_name}, f)


def get_app_store_info(app_id: str) -> dict:
    """Fetch app info from the iTunes/App Store API."""
    url = f"https://itunes.apple.com/lookup?id={app_id}"
    req = urllib.request.Request(
        url,
        headers={"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.0"}
    )
    try:
        with urllib.request.urlopen(req, timeout=15) as response:
            data = json.loads(response.read().decode("utf-8"))
    except urllib.error.HTTPError as e:
        print(f"HTTP Error {e.code}: {e.reason}")
        sys.exit(1)
    except Exception as e:
        print(f"Error fetching App Store data: {e}")
        sys.exit(1)

    if data.get("resultCount", 0) == 0:
        print(f"No app found with ID: {app_id}")
        sys.exit(1)

    return data["results"][0]


def send_discord_webhook(webhook_url: str, app_info: dict, previous_version: str | None) -> None:
    """Send app version info to a Discord webhook."""
    app_name = app_info.get("trackName", "Unknown App")
    version = app_info.get("version", "Unknown")
    developer = app_info.get("artistName", "Unknown Developer")
    release_notes = app_info.get("releaseNotes", "No release notes available.")
    app_url = app_info.get("trackViewUrl", "")
    icon_url = app_info.get("artworkUrl100", "")

    if len(release_notes) > 1000:
        release_notes = release_notes[:997] + "..."

    # Build embed fields
    fields = [
        {"name": "🔢 New Version", "value": f"**{version}**", "inline": True},
        {"name": "👤 Developer", "value": developer, "inline": True},
    ]
    
    if previous_version:
        fields.append({"name": "📉 Previous Version", "value": previous_version, "inline": True})
    
    fields.append({"name": "📝 Release Notes", "value": release_notes, "inline": False})

    payload = {
        "username": "App Store Tracker",
        "avatar_url": "https://developer.apple.com/assets/elements/icons/app-store/app-store-128x128.png",
        "embeds": [
            {
                "title": f"🆕 {app_name} Updated!",
                "description": f"A new version of **{app_name}** is now available on the App Store!",
                "url": app_url,
                "color": 0x34C759,
                "thumbnail": {"url": icon_url},
                "fields": fields,
                "footer": {"text": "App Store Version Tracker"},
                "timestamp": app_info.get("currentVersionReleaseDate", "")
            }
        ]
    }

    req = urllib.request.Request(
        webhook_url,
        data=json.dumps(payload).encode("utf-8"),
        headers={"Content-Type": "application/json", "User-Agent": "AppStoreWebhook/1.0"}
    )

    try:
        with urllib.request.urlopen(req, timeout=15) as response:
            if response.status in (200, 204):
                print(f"Successfully posted new version {version} of '{app_name}' to Discord!")
            else:
                print(f"Unexpected response status: {response.status}")
    except urllib.error.HTTPError as e:
        print(f"Discord webhook failed: HTTP {e.code} - {e.read().decode('utf-8')}")
        sys.exit(1)
    except Exception as e:
        print(f"Error sending webhook: {e}")
        sys.exit(1)


def main():
    parser = argparse.ArgumentParser(description="Fetch App Store version and notify Discord only on updates.")
    parser.add_argument("--app-id", required=True, help="App Store app ID")
    parser.add_argument("--webhook-url", required=True, help="Discord webhook URL")
    args = parser.parse_args()

    print(f"Fetching App Store info for app ID: {args.app_id}...")
    app_info = get_app_store_info(args.app_id)

    current_version = app_info.get("version", "Unknown")
    app_name = app_info.get("trackName", "Unknown App")
    last_version = load_last_version()

    print(f"Current version: {current_version}")
    if last_version:
        print(f"Last known version: {last_version}")
    else:
        print(f"No previous version recorded (first run)")

    if last_version == current_version:
        print(f"Version unchanged ({current_version}). No notification sent.")
        sys.exit(0)

    print(f"New version detected! {last_version or 'N/A'} -> {current_version}")
    print(f"Sending Discord notification...")

    send_discord_webhook(args.webhook_url, app_info, last_version)
    save_version(current_version, app_name)
    print(f"Saved version {current_version} to state file.")


if __name__ == "__main__":
    main()
