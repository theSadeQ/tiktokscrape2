name: TikTok Bulk Download & Save to Repo

on:
  push:
    branches:
      - "**"

concurrency:
  group: tiktok-download-${{ github.ref }}
  cancel-in-progress: false

jobs:
  tiktok-download:
    # Prevent the workflow's own commit from triggering another run.
    if: "!contains(github.event.head_commit.message, '[skip ci]')"

    runs-on: ubuntu-latest

    permissions:
      contents: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Install dependencies
        run: |
          set -euo pipefail

          sudo apt-get update
          sudo apt-get install -y ffmpeg zip python3-pip

          python3 -m pip install --upgrade pip
          python3 -m pip install --upgrade yt-dlp curl_cffi

          echo "yt-dlp version:"
          yt-dlp --version

      - name: Extract URLs and download TikTok files
        shell: bash
        env:
          # Optional secret.
          # If TikTok blocks public downloads, add a repo secret named:
          # TIKTOK_COOKIES_B64
          #
          # It should contain base64-encoded cookies.txt content.
          TIKTOK_COOKIES_B64: ${{ secrets.TIKTOK_COOKIES_B64 }}
        run: |
          set -u
          set -o pipefail

          mkdir -p downloads tmp_downloads
          touch downloaded_archive.txt

          MSG_FILE="$(mktemp)"
          git log -1 --pretty=%B > "$MSG_FILE"

          echo "===== Commit message ====="
          cat "$MSG_FILE"
          echo "=========================="

          COMMAND="$(
            grep -Eim1 '^[[:space:]]*(tiktok-zip|download-zip|tiktok-download|download):' "$MSG_FILE" \
              | sed -E 's/^[[:space:]]*([^:]+):.*/\1/I' \
              | tr '[:upper:]' '[:lower:]' || true
          )"

          if [ -z "$COMMAND" ]; then
            echo "❌ No supported download command found."
            echo ""
            echo "Use one of:"
            echo "  tiktok-download:"
            echo "  tiktok-zip:"
            echo "  download:"
            echo "  download-zip:"
            exit 1
          fi

          case "$COMMAND" in
            tiktok-zip|download-zip)
              MODE="zip"
              ;;
            tiktok-download|download)
              MODE="normal"
              ;;
            *)
              echo "❌ Unsupported command: $COMMAND"
              exit 1
              ;;
          esac

          echo "Detected command: $COMMAND"
          echo "Download mode: $MODE"

          URLS="$(
            awk '
              BEGIN { capture = 0 }

              /^[[:space:]]*(tiktok-zip|download-zip|tiktok-download|download):[[:space:]]*/ && capture == 0 {
                capture = 1
                line = $0
                sub(/^[[:space:]]*(tiktok-zip|download-zip|tiktok-download|download):[[:space:]]*/, "", line)
                print line
                next
              }

              capture == 1 {
                print $0
              }
            ' "$MSG_FILE" \
              | grep -Eo 'https?://[^[:space:]<>"]+' \
              | sed 's/[),.;]*$//' \
              | sort -u || true
          )"

          if [ -z "$URLS" ]; then
            echo "❌ No URLs found after '$COMMAND:'."
            echo ""
            echo "Correct example:"
            echo "tiktok-download:"
            echo "https://www.tiktok.com/@user/video/123"
            echo "https://www.tiktok.com/@user/video/456"
            exit 1
          fi

          echo "===== URLs to download ====="
          echo "$URLS"
          echo "============================"

          COOKIES_ARGS=()
          if [ -n "${TIKTOK_COOKIES_B64:-}" ]; then
            echo "Using TikTok cookies from TIKTOK_COOKIES_B64 secret."
            echo "$TIKTOK_COOKIES_B64" | base64 -d > cookies.txt
            chmod 600 cookies.txt
            COOKIES_ARGS=(--cookies cookies.txt)
          else
            echo "No TikTok cookies secret provided. Trying public download."
          fi

          count_files() {
            find tmp_downloads -type f | wc -l
          }

          YTDLP_HAS_ERRORS=0
          SUCCESSFUL_URLS=0

          while IFS= read -r URL; do
            [ -z "$URL" ] && continue

            echo ""
            echo "⬇️ Downloading:"
            echo "$URL"

            BEFORE_COUNT="$(count_files)"

            # Important:
            # We do NOT use --impersonate chrome here.
            #
            # Why?
            # Your runner showed:
            #   Impersonate target "chrome" is not available
            #
            # That means yt-dlp could not use that impersonation target.
            # Removing it makes the workflow portable and prevents this crash.
            if yt-dlp \
              --ignore-errors \
              --no-abort-on-error \
              --no-mtime \
              --download-archive downloaded_archive.txt \
              --format "bv*+ba/b" \
              --merge-output-format mp4 \
              "${COOKIES_ARGS[@]}" \
              --paths "tmp_downloads" \
              -o "%(uploader|unknown_uploader)s/%(upload_date>%Y-%m-%d|unknown_date)s_%(title).80B_%(id)s.%(ext)s" \
              "$URL"; then

              EXIT_CODE=0
            else
              EXIT_CODE=$?
            fi

            AFTER_COUNT="$(count_files)"

            if [ "$EXIT_CODE" -ne 0 ]; then
              echo "⚠️ yt-dlp returned exit code $EXIT_CODE for:"
              echo "$URL"
              YTDLP_HAS_ERRORS=1
            fi

            if [ "$AFTER_COUNT" -gt "$BEFORE_COUNT" ]; then
              echo "✅ New file downloaded for this URL."
              SUCCESSFUL_URLS=$((SUCCESSFUL_URLS + 1))
            else
              echo "⚠️ No new file appeared for this URL."
              echo "Possible reasons:"
              echo "  1. It was already downloaded and listed in downloaded_archive.txt"
              echo "  2. TikTok blocked public access"
              echo "  3. The URL is invalid or unavailable"
              echo "  4. Cookies are required"
            fi
          done <<< "$URLS"

          TOTAL_FILES="$(count_files)"
          echo ""
          echo "Total files currently in tmp_downloads: $TOTAL_FILES"

          if [ "$TOTAL_FILES" -eq 0 ]; then
            if [ "$YTDLP_HAS_ERRORS" -eq 1 ]; then
              echo "❌ No files were downloaded and at least one yt-dlp error occurred."
              exit 1
            fi

            echo "ℹ️ No new files downloaded. They may already be archived."
            exit 0
          fi

          if [ "$MODE" = "zip" ]; then
            echo "📦 Creating split ZIP archive..."

            ARCHIVE_BASE="tiktok_bulk_$(date +%Y%m%d_%H%M%S)"

            (
              cd tmp_downloads
              zip -r -s 90m "../downloads/${ARCHIVE_BASE}.zip" .
            )

            echo "Created archive parts:"
            ls -lh downloads/${ARCHIVE_BASE}* || true

          else
            echo "📁 Copying downloaded files to downloads/..."
            cp -a tmp_downloads/. downloads/
          fi

          rm -rf tmp_downloads

          if [ "$YTDLP_HAS_ERRORS" -eq 1 ]; then
            echo "⚠️ Some URLs had errors, but at least one file was downloaded."
            echo "Continuing so successful downloads can be committed."
          fi

      - name: Commit & Push changes
        shell: bash
        run: |
          set -euo pipefail

          BRANCH="${GITHUB_REF_NAME}"

          git config user.name "github-actions"
          git config user.email "github-actions@github.com"

          git add downloads/ downloaded_archive.txt

          if git diff --cached --quiet; then
            echo "Nothing to commit."
            exit 0
          fi

          git commit -m "Add downloaded TikTok files [skip ci]"
          git push origin HEAD:"$BRANCH"
