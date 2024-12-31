## Hi there 👋

from fastapi import FastAPI, HTTPException
from sqlalchemy import create_engine
import pandas as pd
import requests
from bs4 import BeautifulSoup
import redis
import time
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.interval import IntervalTrigger

# Initialize FastAPI app
app = FastAPI()

# Database connection
DATABASE_URL = "postgresql://username:password@localhost/dfs_tool"
engine = create_engine(DATABASE_URL)

# Redis caching setup
cache = redis.Redis(host='localhost', port=6379, db=0)

# Player stats scraping and updating database
def scrape_and_update_stats():
    url = "https://www.pro-football-reference.com/years/2023/fantasy.htm"
    response = requests.get(url)
    if response.status_code == 200:
        soup = BeautifulSoup(response.content, "html.parser")
        table = soup.find("table", {"id": "fantasy"})
        if table:
            rows = table.find("tbody").find_all("tr")
            players = []
            for row in rows:
                cols = row.find_all("td")
                if cols:
                    players.append({
                        "player_name": cols[0].text.strip(),
                        "team": cols[1].text.strip(),
                        "position": cols[2].text.strip(),
                        "games_played": int(cols[3].text.strip()),
                        "fantasy_points": float(cols[-1].text.strip()),
                        "rushing_yards": float(cols[9].text.strip()),
                        "receiving_yards": float(cols[12].text.strip()),
                        "touchdowns": int(cols[15].text.strip()),
                        "salary": int(cols[15].text.strip()) * 100,
                        "value": float(cols[-1].text.strip()) / (int(cols[15].text.strip()) * 100)
                    })
            df = pd.DataFrame(players)
            df.to_sql("player_stats", engine, if_exists="replace", index=False)
            print("Database updated with new stats.")
        else:
            print("Stats table not found.")
    else:
        print(f"Failed to fetch stats. Status code: {response.status_code}")

# Schedule scraping every hour
scheduler = BackgroundScheduler()
scheduler.add_job(scrape_and_update_stats, IntervalTrigger(hours=1))
scheduler.start()

# API Endpoints
@app.get("/get_player_stats")
def get_player_stats(player_name: str):
    # Check cache first
    cached_data = cache.get(player_name)
    if cached_data:
        return pd.read_json(cached_data).to_dict(orient="records")
    
    # Query database
    query = f"SELECT * FROM player_stats WHERE player_name ILIKE '%{player_name}%'"
    data = pd.read_sql(query, engine)
    if data.empty:
        raise HTTPException(status_code=404, detail="Player not found")
    
    # Cache the data
    cache.set(player_name, data.to_json(), ex=3600)
    return data.to_dict(orient="records")

@app.post("/generate_lineup")
def generate_lineup(salary_cap: int):
    query = "SELECT * FROM player_stats ORDER BY value DESC"
    players = pd.read_sql(query, engine)

    lineup = {"MVP": None, "FLEX": [], "Total Salary": 0}

    # Select MVP
    mvp = players.iloc[0]
    lineup["MVP"] = mvp["player_name"]
    total_salary = mvp["salary"]

    # Select FLEX players
    for _, player in players.iterrows():
        if total_salary + player["salary"] <= salary_cap and len(lineup["FLEX"]) < 4:
            lineup["FLEX"].append(player["player_name"])
            total_salary += player["salary"]

    lineup["Total Salary"] = total_salary
    return lineup

@app.get("/dynamic_metrics")
def dynamic_metrics(game_script: str):
    weights = {
        "high-scoring": {"MVP": 1.8, "FLEX": 1.2},
        "defensive": {"MVP": 1.2, "FLEX": 1.0},
        "balanced": {"MVP": 1.5, "FLEX": 1.1}
    }
    selected_weights = weights.get(game_script, {"MVP": 1.5, "FLEX": 1.1})

    query = "SELECT * FROM player_stats"
    players = pd.read_sql(query, engine)
    players["dynamic_score"] = (
        players["fantasy_points"] * selected_weights["MVP"]
        + players["value"] * selected_weights["FLEX"]
    )
    return players.sort_values(by="dynamic_score", ascending=False).to_dict(orient="records")

# Run the server
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
    
**AImatrixdfs/AiMatrixDFS** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:

- 🔭 I’m currently working on ...
- 🌱 I’m currently learning ...
- 👯 I’m looking to collaborate on ...
- 🤔 I’m looking for help with ...
- 💬 Ask me about ...
- 📫 How to reach me: ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...
-->
