# Import
import csv #Used to read flashcards from a .csv file
import random #Used to randomly select flashcards
import json #Used to save learning progress into a file
import os #Used to check if a file exists
import time #Used for countdown timer
import threading #Used to run input and timer simultaneously
from datetime import datetime, timedelta #Used for spaced repetition scheduling

PROGRESS_FILE = "progress.json" #where learning progress is saved

#defining flashcards
class Flashcard:
    def __init__(self, polish, spanish, category):
        self.polish = polish.strip() #self. is used to access variables that belongs to the class, self means the current object
        self.spanish = spanish.strip() #strip to remove whitespaces
        self.category = category.strip()

#handles saving and updating learning progress
class ProgressManager:
    def __init__(self): #initializing (setting up) the object
        self.data = {} #for knowing in how many days this flashcards should appear again
        self.load() #Loads data from progress.json

#loading saved progress from a file into the self.data
    def load(self):
        if os.path.exists(PROGRESS_FILE):
            try:
                with open(PROGRESS_FILE, "r", encoding="utf-8") as f: #readmode
                    self.data = json.load(f)
            except:
                self.data = {} #if any error occurs initialize self.data as an empty dictionary
        else:
            self.data = {} #initialize self.data as an empty dictionary

# saves the current self.data dictionary to file
    def save(self):
        with open(PROGRESS_FILE, "w", encoding="utf-8") as f:
            json.dump(self.data, f, indent=4)

#retrieves progress for a specific item, or gives default starting values if it doesn't exist yet
    def get_progress(self, key): #key here are the spanish words
        return self.data.get(key, {
            "correct_streak": 0,
            "next_due": str(datetime.today().date())
        })

    #update progress: x2 days after good response
    def update_progress(self, key, correct):
        progress = self.get_progress(key)

        if correct:
            progress["correct_streak"] += 1
            interval = 1 + (progress["correct_streak"] - 1) * 2
        else:
            progress["correct_streak"] = 0
            interval = 1

        next_due = datetime.today().date() + timedelta(days=interval)
        progress["next_due"] = str(next_due)

        self.data[key] = progress
        self.save()

#reads flashcards from a CSV and returns them as a list of flashcards objects
def load_flashcards(filename):
    flashcards = []
    try:
        with open(filename, newline='', encoding='latin-1') as csvfile:
            reader = csv.DictReader(csvfile) #reads each row as a dictionary
            for row in reader:
                flashcards.append(
                    Flashcard(row["polish"], row["spanish"], row["category"])
                )
    except FileNotFoundError:
        print("CSV file doesn't exist.")
        exit()
    except Exception as e:
        print("Error trying to load the file:", e)
        exit()

    return flashcards


def timed_input(prompt, timeout):
    print(f"You have {timeout} seconds to answer.")
    answer = [None]

    def get_input():
        answer[0] = input(prompt)

    thread = threading.Thread(target=get_input)
    thread.daemon = True
    thread.start()

    thread.join(timeout)


    if thread.is_alive():
        print("\nTime is up!")
        return None

    return answer[0]


def filter_due_cards(flashcards, progress_manager):
    today = datetime.today().date()
    due = []

    for card in flashcards:
        key = card.polish + "|" + card.spanish
        progress = progress_manager.get_progress(key)
        if datetime.strptime(progress["next_due"], "%Y-%m-%d").date() <= today:
            due.append(card)

    return due


def play_game(flashcards, num_cards, time_limit, multiplayer=False):
    progress_manager = ProgressManager()
    due_cards = filter_due_cards(flashcards, progress_manager)

    if not due_cards:
        print("No new flashcards for today.")
        return

    selected = random.choices(due_cards, k=num_cards)

    scores = {"Player 1": 0}
    if multiplayer:
        scores["Player 2"] = 0

    players = list(scores.keys())
    current_player_index = 0

    for card in selected:
        player = players[current_player_index]
        print(f"\n{player}, translate: {card.polish}")

        answer = timed_input("Twoja odpowiedź: ", time_limit)

        key = card.polish + "|" + card.spanish

        if answer and answer.strip().lower() == card.spanish.lower():
            print("Poprawna odpowiedź!")
            scores[player] += 1
            progress_manager.update_progress(key, True)
        else:
            print(f"Błędna. Poprawna odpowiedź: {card.spanish}")
            progress_manager.update_progress(key, False)

        if multiplayer:
            current_player_index = (current_player_index + 1) % len(players)

    print("\n=== WYNIKI ===")
    for player, score in scores.items():
        print(f"{player}: {score} punktów")

    if multiplayer:
        if scores["Player 1"] > scores["Player 2"]:
            print("Wygrywa Player 1!")
        elif scores["Player 2"] > scores["Player 1"]:
            print("Wygrywa Player 2!")
        else:
            print("Remis!")

#choice for type of the game, how many flashcards you want to learn and how many time you need for the response in seconds
def main():
    flashcards = load_flashcards("flashcards.csv")

    while True:
        print("\n1. Individual game")
        print("2. Multiplayer game")
        print("3. Exit")

        choice = input("Choose an option: ")

        if choice == "1":
            num = int(input("How many flashcards you would like to learn? "))
            time_limit = int(input("Time limit for response (seconds): "))
            play_game(flashcards, num, time_limit, multiplayer=False)

        elif choice == "2":
            num = int(input("How many flashcards you would like to learn? "))
            time_limit = int(input("Time limit for response (seconds): "))
            play_game(flashcards, num, time_limit, multiplayer=True)

        elif choice == "3":
            break

        else:
            print("Incorrect choice. Try again")

        replay = input("\nWould you like to play again? (yes/no): ")
        if replay.lower() != "yes":
            break


if __name__ == "__main__":
    main()

