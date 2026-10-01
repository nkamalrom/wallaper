name: wallpaper

on:
  schedule:
    - cron: "5 21 * * *"
  workflow_dispatch:

permissions:
  contents: write

jobs:
  render:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Fonts
        run: sudo apt-get install -y fonts-dejavu-core
      - name: Render
        run: |
          npm init -y
          npm i @napi-rs/canvas
          cat > render.js <<'EOF'
          const { createCanvas } = require("@napi-rs/canvas");
          const fs = require("fs");

          const W = 1290, H = 2796;
          const TZ = "Europe/Moscow";
          const CAPTION = "ΓΙΝΕ ΑΥΤΟΣ ΠΟΥ ΕΙΣΑΙ";
          const SANS = '"DejaVu Sans", sans-serif';
          const SERIF = '"DejaVu Serif", serif';
          const HAIR = String.fromCharCode(8202);

          const MONTHS = ["ЯНВАРЬ","ФЕВРАЛЬ","МАРТ","АПРЕЛЬ","МАЙ","ИЮНЬ","ИЮЛЬ","АВГУСТ","СЕНТЯБРЬ","ОКТЯБРЬ","НОЯБРЬ","ДЕКАБРЬ"];
          const DAYS = ["ПН","ВТ","СР","ЧТ","ПТ","СБ","ВС"];

          const ACCENT = "#E8731A";
          const GOLD = "#F2A65A";
          const TEXT = "#F5E6D6";
          const DIM = "rgba(245,230,214,0.28)";
          const DARK = "#1A0A03";

          const ymd = new Intl.DateTimeFormat("en-CA", {
            timeZone: TZ, year: "numeric", month: "2-digit", day: "2-digit"
          }).format(new Date());
          const [year, mon, today] = ymd.split("-").map(Number);
          const month = mon - 1;
          const daysInMonth = new Date(Date.UTC(year, mon, 0)).getUTCDate();
          const firstDow = (new Date(Date.UTC(year, month, 1)).getUTCDay() + 6) % 7;

          const canvas = createCanvas(W, H);
          const ctx = canvas.getContext("2d");

          // background
          const bg = ctx.createLinearGradient(0, 0, 0, H);
          bg.addColorStop(0, "#0C0603");
          bg.addColorStop(0.55, "#1B0B04");
          bg.addColorStop(1, "#3E1804");
          ctx.fillStyle = bg;
          ctx.fillRect(0, 0, W, H);

          // glow
          const glow = ctx.createRadialGradient(W / 2, H, 0, W / 2, H, W * 1.0);
          glow.addColorStop(0, "rgba(232,115,26,0.35)");
          glow.addColorStop(1, "rgba(232,115,26,0)");
          ctx.fillStyle = glow;
          ctx.fillRect(0, 0, W, H);

          function spaced(s) { return s.split("").join(HAIR); }

          function text(str, x, y, w, h, size, color, family, weight, align) {
            ctx.font = weight + " " + size + "px " + family;
            ctx.fillStyle = color;
            ctx.textBaseline = "middle";
            ctx.textAlign = align;
            const px = align === "left" ? x : align === "right" ? x + w : x + w / 2;
            ctx.fillText(str, px, y + h / 2);
          }

          const m = W * 0.11;
          const cw = (W - 2 * m) / 7;
          const rh = W * 0.105;
          let y = H * 0.40;

          text(spaced(MONTHS[month]), m, y, W - 2 * m, W * 0.07, W * 0.05, GOLD, SANS, "bold", "left");
          text(String(year), m, y, W - 2 * m, W * 0.07, W * 0.05, DIM, SANS, "normal", "right");
          y += W * 0.09;

          ctx.fillStyle = "rgba(232,115,26,0.6)";
          ctx.fillRect(m, y, W - 2 * m, 2);
          y += W * 0.03;

          for (let i = 0; i < 7; i++) {
            text(DAYS[i], m + i * cw, y, cw, W * 0.05, W * 0.026, i >= 5 ? ACCENT : DIM, SANS, "bold", "center");
          }
          y += W * 0.065;

          const fs_ = W * 0.04;
          for (let d = 1; d <= daysInMonth; d++) {
            const idx = firstDow + d - 1;
            const col = idx % 7, row = Math.floor(idx / 7);
            const cx = m + col * cw, cy = y + row * rh;

            if (d === today) {
              const r = Math.min(cw, rh) * 0.44;
              ctx.fillStyle = ACCENT;
              ctx.beginPath();
              ctx.arc(cx + cw / 2, cy + rh / 2, r, 0, Math.PI * 2);
              ctx.fill();
              text(String(d), cx, cy, cw, rh, fs_, DARK, SANS, "bold", "center");
            } else if (d < today) {
              text(String(d), cx, cy, cw, rh, fs_, DIM, SANS, "normal", "center");
              const lw = fs_ * (d > 9 ? 1.25 : 0.7);
              ctx.fillStyle = "rgba(232,115,26,0.75)";
              ctx.fillRect(cx + cw / 2 - lw / 2, cy + rh / 2 - 1, lw, 3);
            } else {
              text(String(d), cx, cy, cw, rh, fs_, TEXT, SANS, "normal", "center");
            }
          }

          ctx.fillStyle = "rgba(232,115,26,0.6)";
          ctx.fillRect(W / 2 - W * 0.06, H * 0.925, W * 0.12, 2);
          text(spaced(CAPTION), 0, H * 0.935, W, W * 0.07, W * 0.038, GOLD, SERIF, "normal", "center");

          fs.writeFileSync("wallpaper.png", canvas.toBuffer("image/png"));
          console.log("done", ymd);
          EOF
          node render.js
      - name: Commit
        run: |
          git config user.name "wallpaper-bot"
          git config user.email "bot@users.noreply.github.com"
          git add wallpaper.png
          git commit -m "update wallpaper" || echo "no changes"
          git push
# wallaper