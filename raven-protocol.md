# Bot-Game Sequence (authoritative sequence):

1. **Open WebSocket** using IP/port from `.env` (e.g., `ws://localhost:50101`).
2. **Register Bot** with JSON:
   ```json
   { "team_id": "1", "api_key": "splapikey1" }
   ```
   **Server Response:**
   ```json
   { "request": "success" }
   ```
3. **Wait** for Room Server to start the game (e.g., required players joined).
4. On **Game Start**, server sends:

   ```json
   {
     "matchId": "c4c435cc5wqvcw5cc3453c4vcvc345",
     "gameId": "ho3n288945nmc3c939c294cw",
     "yourId": "P1",
     "type": "game-start",
     "ravenCount": 3,
     "detectiveCount": 1,
     "doctorCount": 1,
     "villagerCount": 5,
     "yourRole": "Villager"
   }
   ```

   Based on number of `game-start` received, bot program will create it own parallel thread for processing individual games by tracking the `gameId`.

5. **Gameplay Loop:**
   This below set of comments between server and bot will vary for each game (singham, wordle, etc.,)
   - **Server → Bot (command):**
     ```json
     { "type": "player-status", ... }
     ```
   - **Bot → Server (predicted move):** _(schema example, see below)_
      ```json
      {
        "gameId": "ho3n288945nmc3c939c294cw",
        "type": "raven-comment",
        "day": 1,
        "phase": "night",
        "otp": "123456789",
        "playerId": "P1",
        "comment": "I believe P2 is acting suspiciously and might be the werewolf. We should consider voting against them.",
        "votes": ["P1", "P5"]
      }
      ```
6. **Repeat** until game win/lose/draw condition is met.
7. **Game Result** from server:
   ```json
   {
     "matchId": "c4c435cc5wqvcw5cc3453c4vcvc345",
     "gameId": "ho3n288945nmc3c939c294cw",
     "yourId": "P1",
     "type": "game-result",
     "result": "Villager Won"
   }
   ```
8. Bot will wait for all N games `game-result` message.
9. **Bot closes** the WebSocket connection.

---

# Singham Game Play

## Player Status

All Players Status wiil be sent after every Vote Finalization (End of Morning & End of Night)
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "yourId": "P1",
  "type": "player-status",
  "day": 0,
  "phase": "morning",
  "allPlayers": [
    { "id": "P1", "isAlive": true, "lynchedBy": null, "lynchedDay": null },
    { "id": "P2", "isAlive": true, "lynchedBy": null, "lynchedDay": null },
    {
      "id": "P3",
      "isAlive": false,
      "lynchedBy": "Villager",
      "lynchedDay": 1
    },
    { "id": "P4", "isAlive": true, "lynchedBy": null, "lynchedDay": null },
    { "id": "P5", "isAlive": false, "lynchedBy": "Raven", "lynchedDay": 1 }
  ]
}
```

## Phase Result
Server Publishes Phase Voting Result to All Players.
If playerLynched is Empty no one was Eliminated/Killed.
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "type": "phase-result",
  "day": 1,
  "phase": "night",
  "playerLynched": "P5"
}
```

## Ack from Server
Ack will be sent for every response received from bot by the Server.

requestStatus can be Invalid OTP, Invalid Vote, Invalid JSON, or Success
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "yourId": "P1",
  "type": "ack",
  "otp": "123456789",
  "requestStatus": "Success",
}
```

## Night Discussion

## Raven Communication

## 1. Server to Raven

```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "yourId": "P1",
  "type": "night-discussion",
  "day": 1,
  "phase": "night",
  "timeout": 40,
  "otp": "123456789",
  "villagersAlive": ["P1", "P5"]
}
```

## 2. Raven to Server
llmModelUsed - Is an optional Response.
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "yourId": "P1",
  "type": "vote",
  "otp": "123456789",
  "comment": "I believe P2 is acting suspiciously and might be the werewolf. We should consider voting against them.",
  "votes": ["P1", "P5"],
  "llmModelUsed": "gpt-nano"
}
```

## 3. Server to All Ravens (During Converstaion)
Server publishes the latest Raven Comment to all Ravens
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "type": "raven-comment",
  "day": 1,
  "phase": "night",
  "timeRemaining": 20,
  "otp": "123456789",
  "discussions":[
    {"playerId": "P1",
    "comment": "I believe P2 is acting suspiciously and might be the werewolf. We should consider voting against them.",
    "votes": ["P1", "P5"]},
    {"playerId": "P3",
    "comment": "I believe P5 is acting suspiciously and might be the werewolf. We should consider voting against them.",
    "votes": ["P5", "P1"]}
    ]
}
```

# Detective Communication

## 1. Server to Detective

```json
{
  "gameId": "asdfgh",
  "yourId": "P1",
  "type": "night-investigation",
  "day": 1,
  "phase": "night",
  "timeout": 40,
  "otp": "123456789",
  "playersAlive": ["P2", "P3"],
  "identifiedRaven": ["P5"],
  "identifiedVillager": ["P3"]
}
```

## 2. Detective to Server
llmModelUsed - Is an optional Response.
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "yourId": "P1",
  "type": "vote",
  "otp": "123456789",
  "comment": "I believe P2 is acting suspiciously and might be the werewolf. We should consider voting against them.",
  "votes": ["P1"],
  "llmModelUsed": "gpt-nano"
}
```
Note: The Result of the Investigation will be published to Detective during Morning Discussion Request
## 3. Server to Detective

```json
{
  "gameId": "asdfgh",
  "type": "ack-night-investigation",
  "investigated": ["P1"],
  "isRaven": [true]
}
``` 

# Doctor Communication

## 1. Server to Doctor

```json
{
  "gameId": "asdfgh",
  "yourId": "P1",
  "type": "night-protection",
  "day": 1,
  "phase": "night",
  "timeout": 40,
  "otp": "123456789",
  "playersAlive": ["P2", "P3"]
}
```

## 2. Doctor to Server
llmModelUsed - Is an optional Response.
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "yourId": "P1",
  "type": "vote",
  "otp": "123456789",
  "comment": "I believe P2 is acting suspiciously and might be the werewolf. We should consider voting against them.",
  "votes": ["P2"],
  "llmModelUsed": "gpt-nano"
}
```

# Morning Discussion

# Villager Communication

## 1. Server to Villager
```json
{
  "gameId": "asdfgh",
  "yourId": "P1",
  "type": "morning-discussion",
  "day": 1,
  "phase": "morning",
  "timeout": 115,
  "firstVoteTimeout": 40,
  "otp": "123456789",
  "playersAlive": ["P2", "P3"],
}
```

## 2. Villager to Server
If doneVoting is false the players can add votes with comment to be discussed.
If doneVoting is true further conversations will not be considered.
Player vote count is limited as per game rule.

llmModelUsed - Is an optional Response.
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "yourId": "P1",
  "type": "vote",
  "otp": "123456789",
  "comment": "I believe P2 is acting suspiciously and might be the werewolf. We should consider voting against them.",
  "votes": ["P2"],
  "llmModelUsed": "gpt-nano"
}
```

## 2. Server to All Villager (Morning Conversation)
Server publishes the latest Villager Comment to all Villagers
```json
{
  "gameId": "ho3n288945nmc3c939c294cw",
  "type": "morning-player-comment",
  "day": 1,
  "phase": "morning",
  "timeRemaining": 20,
  "otp": "123456789",
  "discussions":[
    {"playerId": "P1",
    "comment": "I believe P2 is acting suspiciously and might be the werewolf. We should consider voting against them.",
    "votes": ["P2"]},
    {"playerId": "P3",
    "comment": "I believe P5 is acting suspiciously and might be the werewolf. We should consider voting against them.",
    "votes": ["P5"]}
    ]
}
```
