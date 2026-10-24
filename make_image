"""Fetch new GitHub repos for your topics and render them as a PNG table."""
import os
import sys
from datetime import datetime, timedelta, timezone

import requests
from PIL import Image, ImageDraw, ImageFont

# ---- Settings (override with env vars in the workflow) ----
TOPICS = [t.strip() for t in os.getenv("TOPICS", "machine-learning,llm,rust").split(",") if t.strip()]
DAYS = int(os.getenv("DAYS", "7"))          # "new" = created in the last N days
PER_TOPIC = int(os.getenv("PER_TOPIC", "8"))  # rows per topic
OUT = os.getenv("OUT", "output/latest.png")

# ---- Colours ----
BG, CARD, HEADER = "#0d1117", "#161b22", "#58a6ff"
TEXT, MUTED, STAR, LINE = "#e6edf3", "#8b949e", "#e3b341", "#30363d"


def font(size, bold=False):
    name = "DejaVuSans-Bold.ttf" if bold else "DejaVuSans.ttf"
    for path in (f"/usr/share/fonts/truetype/dejavu/{name}", name):
        try:
            return ImageFont.truetype(path, size)
        except OSError:
            continue
    return ImageFont.load_default(size)


F_TITLE, F_SUB = font(30, True), font(15)
F_TOPIC, F_NAME, F_DESC, F_META = font(20, True), font(16, True), font(14), font(14)


def fetch(topic):
    since = (datetime.now(timezone.utc) - timedelta(days=DAYS)).strftime("%Y-%m-%d")
    headers = {"Accept": "application/vnd.github+json", "User-Agent": "topic-digest"}
    if os.getenv("GH_TOKEN"):
        headers["Authorization"] = f"Bearer {os.environ['GH_TOKEN']}"
    r = requests.get(
        "https://api.github.com/search/repositories",
        params={"q": f"topic:{topic} created:>{since}", "sort": "stars",
                "order": "desc", "per_page": PER_TOPIC},
        headers=headers, timeout=30,
    )
    r.raise_for_status()
    return r.json()["items"]


def fit(text, fnt, max_w):
    """Trim text with an ellipsis so it fits within max_w pixels."""
    text = (text or "No description").replace("\n", " ")
    # drop emoji / CJK etc. that the font can't draw (they'd show as empty boxes)
    text = "".join(c for c in text if ord(c) < 0x2000 or 0x2010 <= ord(c) <= 0x2122)
    text = " ".join(text.split()) or "No description"
    if fnt.getlength(text) <= max_w:
        return text
    while text and fnt.getlength(text + "…") > max_w:
        text = text[:-1]
    return text.rstrip() + "…"


def stars(n):
    return f"{n/1000:.1f}k" if n >= 1000 else str(n)


def main():
    data = {}
    for t in TOPICS:
        try:
            data[t] = fetch(t)
        except Exception as e:  # keep going if one topic fails
            print(f"Topic {t} failed: {e}", file=sys.stderr)
            data[t] = []

    W, PAD, ROW_H = 1100, 40, 56
    height = 120 + sum(70 + max(len(v), 1) * ROW_H + 20 for v in data.values()) + 30
    img = Image.new("RGB", (W, height), BG)
    d = ImageDraw.Draw(img)

    today = datetime.now(timezone.utc).strftime("%d %b %Y")
    d.text((PAD, 30), "New GitHub repos for your topics", font=F_TITLE, fill=TEXT)
    d.text((PAD, 74), f"Created in the last {DAYS} days  ·  sorted by stars  ·  {today} (UTC)",
           font=F_SUB, fill=MUTED)

    y = 120
    for topic, repos in data.items():
        block_h = 70 + max(len(repos), 1) * ROW_H + 10
        d.rounded_rectangle((PAD - 10, y, W - PAD + 10, y + block_h), radius=12, fill=CARD)
        d.text((PAD + 10, y + 18), f"# {topic}", font=F_TOPIC, fill=HEADER)
        ry = y + 60
        if not repos:
            d.text((PAD + 10, ry + 14), "No new repos found", font=F_DESC, fill=MUTED)
        for i, r in enumerate(repos, 1):
            d.line((PAD + 10, ry, W - PAD - 10, ry), fill=LINE)
            d.text((PAD + 10, ry + 8), f"{i}", font=F_NAME, fill=MUTED)
            d.text((PAD + 45, ry + 6), fit(r["full_name"], F_NAME, 640), font=F_NAME, fill=TEXT)
            d.text((PAD + 45, ry + 30), fit(r["description"], F_DESC, 700), font=F_DESC, fill=MUTED)
            d.text((W - PAD - 330, ry + 8), f"★ {stars(r['stargazers_count'])}", font=F_META, fill=STAR)
            d.text((W - PAD - 230, ry + 8), fit(r["language"] or "-", F_META, 190), font=F_META, fill=TEXT)
            ry += ROW_H
        y += block_h + 20

    os.makedirs(os.path.dirname(OUT), exist_ok=True)
    img.save(OUT)
    print(f"Saved {OUT}")


if __name__ == "__main__":
    main()
