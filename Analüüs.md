import requests
import sqlite3

API_KEY = 'RGAPI-25c26be9-2cb8-4c1c-8813-46c798c36f5c'
REGION = 'eun1'  # Regioon, kus asub Rioti mänguserver ('na1', 'euw1', 'eun1')
DATABASE_NAME = 'players.db'  # Andmebaasi nimi, kus hoitakse mängijate andmeid

# Loo andmebaas, kui see puudub
def create_database():
    """
    Loob andmebaasi, kui see puudub, koos tabeli defineerimisega, kus hoitakse mängijate nimesid.
    """
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute('''CREATE TABLE IF NOT EXISTS players (
                        summoner_name TEXT PRIMARY KEY
                    )''')
    connection.commit()
    connection.close()

# Lisa mängija andmebaasi
def add_player_to_database(summoner_name):
    """
    Lisab mängija andmebaasi, eeldusel et mängijat pole juba olemas.
    """
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute('''INSERT OR IGNORE INTO players (summoner_name) VALUES (?)''', (summoner_name,))
    connection.commit()
    connection.close()

# Hangi mängijad andmebaasist
def get_players_from_database():
    """
    Hangib kõik mängijad andmebaasist.
    """
    connection = sqlite3.connect(DATABASE_NAME)
    cursor = connection.cursor()
    cursor.execute('''SELECT summoner_name FROM players''')
    players = cursor.fetchall()
    connection.close()
    return [player[0] for player in players]

# Hangi summoneerija ID
def get_summoner_id(summoner_name):
    """
    Hangib summoneerija ID Rioti API abil.
    """
    url = f'https://{REGION}.api.riotgames.com/lol/summoner/v4/summoners/by-name/{summoner_name}'
    headers = {'X-Riot-Token': API_KEY}
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        return response.json()['id']
    else:
        return None

# Hangi praegune mäng
def get_current_game(summoner_id):
    """
    Hangib summoneerija praeguse mängu Rioti API abil.
    """
    url = f'https://{REGION}.api.riotgames.com/lol/spectator/v4/active-games/by-summoner/{summoner_id}'
    headers = {'X-Riot-Token': API_KEY}
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        return response.json()
    else:
        return None

# Prindi mängijad mängus ja kontrolli, kas mängijad on andmebaasis
def print_players_in_game(current_game, players_from_database):
    """
    Prindib mängus olevad mängijad ning märgib, kui mõni neist on andmebaasis.
    """
    if current_game:
        print("Players in the current game:")
        for player in current_game['participants']:
            player_name = player['summonerName']
            print(player_name)
            if player_name in players_from_database:
                print(f"PLAYER FLAGGED! ---> {player_name}")
    else:
        print("Summoner is not currently in a game.")

if __name__ == "__main__":
    create_database()
    while True:
        summoner_name = input("Enter summoner name to add to the database (press Enter to skip): ")
        if not summoner_name:
            break
        add_player_to_database(summoner_name)
    players_from_database = get_players_from_database()

    summoner_name = input("Enter the summoner name of the player you want to check: ")
    summoner_id = get_summoner_id(summoner_name)
    if summoner_id:
        current_game = get_current_game(summoner_id)
        print_players_in_game(current_game, players_from_database)
    else:
        print("Summoner not found.")
