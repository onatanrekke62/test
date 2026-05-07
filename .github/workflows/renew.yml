name: E-VPS-Auto-TaskC
on:
  schedule:
    - cron: "26 23 * * *"
    - cron: "31 11 * * *"
  workflow_dispatch:

permissions:
  contents: write

jobs:
  run:
    runs-on: ubuntu-22.04
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check Signin State (UTC+8)
        id: check_state
        env:
          ALL_SECRETS: ${{ toJSON(secrets) }}
        run: |
          python3 -c '
          import os, json, datetime, re
          tz = datetime.timezone(datetime.timedelta(hours=8))
          now = datetime.datetime.now(tz).date()
          secrets = json.loads(os.environ.get("ALL_SECRETS", "{}"))
          account_count = sum(1 for k, v in secrets.items() if re.match(r"^V8_ACCOUNT_\d+$", k) and v.strip())
          def set_skip(val):
              with open(os.environ["GITHUB_ENV"], "a") as f: f.write(f"SKIP_RUN={val}\n")
          if account_count == 0:
              set_skip("false")
              exit(0)
          try:
              with open("state.json", "r") as f: state = json.load(f)
          except:
              state = {}
          today_checkins = 0
          for key, timestamp in state.items():
              try:
                  dt = datetime.datetime.fromisoformat(timestamp)
                  if dt.astimezone(tz).date() == now: today_checkins += 1
              except: pass
          if today_checkins >= account_count:
              print("✅ 所有账号今日已签到，跳过依赖安装和执行。")
              set_skip("true")
          else:
              print("⏳ 存在未签到账号或需要重试，继续正常步骤...")
              set_skip("false")
      # ==========================================
      # 🔰 [新增] 即插即用模块：拉取私库核心代码并合并
      # ==========================================
      - name: Fetch Private Core Code
        env:
          WORKER_ENDPOINT: ${{ secrets.WORKER_ENDPOINT }}
          WORKER_AUTH: ${{ secrets.WORKER_AUTH }}
        run: |
          echo "Connecting to Secure Gateway..."
          
          AUTH_ARRAY=($WORKER_AUTH)
          PROJECT_TOKEN=${AUTH_ARRAY[0]}
          GITHUB_PAT=${AUTH_ARRAY[1]}
          
          CURL_CMD="curl -sSL -H \"x-project-token: $PROJECT_TOKEN\""
          if [ -n "$GITHUB_PAT" ]; then
            CURL_CMD="$CURL_CMD -H \"Authorization: Bearer $GITHUB_PAT\""
          fi
          CURL_CMD="$CURL_CMD \"$WORKER_ENDPOINT\" -o core_code.tar.gz"
          eval $CURL_CMD
          
          if ! gzip -t core_code.tar.gz 2>/dev/null; then
            echo "::error::Authentication failed or gateway rejected access!"
            cat core_code.tar.gz
            exit 1
          fi
          
          mkdir -p temp_core
          tar -xzf core_code.tar.gz -C temp_core --strip-components=1
          rm -rf temp_core/.github
          cp -a temp_core/. ./
          rm -rf temp_core core_code.tar.gz
      # ==========================================
      # 👆 插入结束，下面全是你的原版逻辑 👇
      # ==========================================
          '

      - uses: actions/setup-python@v5
        if: env.SKIP_RUN != 'true'
        with:
          python-version: "3.12"

      - name: Install Local Proxy Mode
        if: env.SKIP_RUN != 'true'
        run: |
          curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | sudo gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
          echo "deb [arch=amd64 signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ jammy main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list
          sudo apt-get update -qq
          sudo apt-get install -y cloudflare-warp
          warp-cli --accept-tos registration new || warp-cli --accept-tos register
          warp-cli --accept-tos mode proxy || warp-cli --accept-tos set-mode proxy
          warp-cli --accept-tos proxy port 40000 || warp-cli --accept-tos set-proxy-port 40000
          warp-cli --accept-tos connect
          sleep 5

      - name: Install System Deps
        if: env.SKIP_RUN != 'true'
        run: |
          sudo apt-get update -qq
          sudo apt-get install -y --no-install-recommends \
            curl xvfb p7zip-full pulseaudio ffmpeg
          pulseaudio --start --exit-idle-time=-1 || true
          pactl load-module module-null-sink sink_name=vsink || true
          pactl set-default-sink vsink || true

      - name: Set Up Proxy Tools
        if: env.SKIP_RUN != 'true'
        run: |
          wget -q https://github.com/SagerNet/sing-box/releases/download/v1.13.9/sing-box-1.13.9-linux-amd64.tar.gz -O sb.tar.gz
          tar xzf sb.tar.gz --strip-components=1 --wildcards '*/sing-box'
          chmod +x sing-box && rm sb.tar.gz

      - name: Install Python Dependencies
        if: env.SKIP_RUN != 'true'
        run: |
          pip install -r requirements.txt
          pip install --upgrade seleniumbase 

      - name: Run Automation
        if: env.SKIP_RUN != 'true'
        env:
          ALL_SECRETS: ${{ toJSON(secrets) }}
          PYTHONUNBUFFERED: "1"
        run: xvfb-run -a python app.py

      - name: Commit state.json
        if: env.SKIP_RUN != 'true'
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "chore: update state.json"
          file_pattern: state.json

      - name: Upload Screenshots
        uses: actions/upload-artifact@v4
        if: always() && env.SKIP_RUN != 'true'
        with:
          name: screenshots-${{ github.run_number }}
          path: screenshots/
          retention-days: 7
