

Let's build a **complete anime streaming website** using LiSA as the backend and animepahe-dl for stream extraction, handling Cloudflare, and deploying locally. Here's the step-by-step plan:

---

### **1. Backend Setup (LiSA + Custom API)**
We'll modify LiSA to act as a **headless API service** that:
- Bypasses Cloudflare
- Extracts stream URLs using animepahe-dl's logic
- Provides REST endpoints for search, episodes, and streaming

#### **Step 1.1: Fork LiSA and Install Dependencies**
```bash
git clone https://github.com/Cosmicoppai/LiSA.git
cd LiSA
# Install dependencies (Python 3.8+ required)
pip install -r requirements.txt
# Install Node.js dependencies (for LiSA's frontend, but we'll ignore it for our backend)
npm install
```

#### **Step 1.2: Create a Backend API (FastAPI)**
Create a new file `backend/app.py` to expose REST endpoints:

```python
# backend/app.py
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse
import subprocess
import cloudscraper
import time

app = FastAPI()

# Use animepahe-dl to extract stream URL (if LiSA doesn't handle this natively)
def get_animepahe_stream_url(episode_url):
    try:
        # Use cloudscraper to bypass Cloudflare
        scraper = cloudscraper.create_scraper()
        # Run animepahe-dl to extract the stream URL
        result = subprocess.run(
            ["python", "animepahe_dl.py", "--url", episode_url, "--format", "best", "--print-url"],
            capture_output=True,
            text=True,
            check=True
        )
        # Parse the output (example: look for ".m3u8" in logs)
        # Adjust this based on actual output of animepahe-dl
        for line in result.stdout.splitlines():
            if line.endswith(".m3u8"):
                return line.strip()
        raise ValueError("No stream URL found")
    except Exception as e:
        print(f"Error extracting stream: {e}")
        return None

@app.get("/api/search")
async def search_anime(q: str):
    # TODO: Implement search logic (LiSA's internal search function)
    # For now, return dummy data
    return [
        {"title": "Demon Slayer", "id": "1"},
        {"title": "Attack on Titan", "id": "2"}
    ]

@app.get("/api/episodes/{anime_id}")
async def get_episodes(anime_id: str):
    # TODO: Implement episode listing
    return [{"season": 1, "episode": 1, "url": "https://animepahe.ru/..."}]

@app.get("/api/episode/{episode_id}/stream")
async def stream_episode(episode_id: str):
    # Get the AnimePahe episode URL (from frontend or internal logic)
    episode_url = f"https://animepahe.ru/play/{episode_id}"  # Example
    stream_url = get_animepahe_stream_url(episode_url)
    if not stream_url:
        raise HTTPException(status_code=404, detail="Stream not found")
    return {"stream_url": stream_url}

# Run the backend
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, reload=True)
```

#### **Step 1.3: Integrate animepahe-dl for Stream Extraction**
1. Place `animepahe_dl.py` from [KevCui's repo](https://github.com/KevCui/animepahe-dl) in the `LiSA` directory.
2. Modify `get_animepahe_stream_url()` to correctly parse the output. Test it:
   ```bash
   python backend/app.py
   # Then visit http://localhost:8000/api/episode/test_id/stream? (Replace test_id with a real AnimePahe episode URL)
   ```

---

### **2. Frontend Setup (HTML/JavaScript)**
Create a frontend to interact with the backend. Use the design from `design.md` or your `index.html`.

#### **Step 2.1: Basic Frontend (index.html)**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Anime Streamer</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/video.js/8.10.0/video-js.css">
    <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
    <script src="https://cdn.jsdelivr.net/npm/video.js@8.10.0/video.js"></script>
</head>
<body>
    <div class="container">
        <h1>Anime Streamer</h1>
        <input type="text" id="search" placeholder="Search anime...">
        <button onclick="searchAnime()">Search</button>
        <div id="episodes"></div>
        <div id="player"></div>
    </div>

    <script>
        async function searchAnime() {
            const query = document.getElementById("search").value;
            try {
                const response = await fetch("http://localhost:8000/api/search?q=" + encodeURIComponent(query));
                const data = await response.json();
                displayEpisodes(data);
            } catch (error) {
                console.error("Search failed:", error);
            }
        }

        function displayEpisodes(episodes) {
            const container = document.getElementById("episodes");
            container.innerHTML = episodes.map(episode => `
                <div>${episode.title} S${episode.season}E${episode.episode}</div>
                <button onclick="playEpisode('${episode.id}')">Play</button>
            `).join('');
        }

        async function playEpisode(episodeId) {
            try {
                const response = await fetch("http://localhost:8000/api/episode/" + episodeId + "/stream");
                const streamUrl = await response.json();
                const player = document.getElementById("player");
                if (Hls.isSupported()) {
                    const hls = new Hls();
                    hls.loadSource(streamUrl.stream_url);
                    hls.attachMedia(player);
                    hls.on(Hls.Events.MANIFEST_PARSED, () => player.play());
                } else {
                    player.src = streamUrl.stream_url;
                    player.play();
                }
            } catch (error) {
                console.error("Playback failed:", error);
                alert("Could not load video");
            }
        }
    </script>
</body>
</html>
```

#### **Step 2.2: CSS (Optional)**
Add basic styling (e.g., `style.css`):
```css
body { font-family: Arial; }
.container { max-width: 1200px; margin: 0 auto; padding: 20px; }
#player { width: 100%; }
button { margin: 5px; padding: 5px; }
```

---

### **3. Run Locally**
#### **Step 3.1: Start the Backend**
```bash
python backend/app.py
# If you get errors, ensure:
# 1. animepahe_dl.py is in the same directory.
# 2. Node.js is installed (LiSA may need it for some dependencies).
```

#### **Step 3.2: Run the Frontend**
Open `index.html` in your browser (e.g., via a local server or directly).

---

### **4. Fix Common Issues**
#### **A. Cloudflare Bypass**
- If streams fail, ensure `cloudscraper` is used. In `app.py`, add:
  ```python
  import cloudscraper
  scraper = cloudscraper.create_scraper()
  response = scraper.get(url)  # Replace normal requests with this
  ```

#### **B. Stream URL Not Found**
- Check `animepahe_dl.py` output. Modify `get_animepahe_stream_url()` to match its output format. For example, if it logs:
  ```
  Downloading video: https://example.com/video.m3u8 [size: 1.2GB]
  ```
  Parse the URL from the log.

#### **C. CORS Errors**
- Add CORS middleware to FastAPI:
  ```python
  from fastapi.middleware.cors import CORSMiddleware

  app.add_middleware(
      CORSMiddleware,
      allow_origins=["*"],
      allow_methods=["*"],
      allow_headers=["*"],
  )
  ```

#### **D. LiSA Dependency Conflicts**
- If LiSA requires old packages, create a virtual environment:
  ```bash
  python -m venv venv
  source venv/bin/activate  # Linux/Mac
  venv\Scripts\activate     # Windows
  pip install -r requirements.txt
  ```

---

### **5. Test End-to-End**
1. Search for an anime (e.g., "Demon Slayer").
2. Click an episode to trigger `playEpisode()`.
3. Check if HLS.js loads the video. If not:
   - Inspect the browser's console for errors.
   - Verify the stream URL from the backend API matches the format: `https://.../video.m3u8`.

---

### **6. Next Steps for Production**
- **Deploy Backend**: Use platforms like **Render**, **Heroku**, or a VPS.
- **Optimize Performance**: Add caching for stream URLs.
- **Add Features**: Implement downloads, user accounts, etc., using LiSA's existing code.

---

### **Full Local Setup Summary**
```bash
# Clone repos
git clone https://github.com/Cosmicoppai/LiSA.git
cd LiSA
git clone https://github.com/KevCui/animepahe-dl.git  # Place animepahe_dl.py here

# Install dependencies
pip install -r requirements.txt

# Create backend API file (backend/app.py)
# Create frontend files (index.html, style.css)

# Run backend: python backend/app.py
# Open frontend in browser: file:///path/to/index.html
```

Let me know if you hit specific errors, and I'll troubleshoot!
