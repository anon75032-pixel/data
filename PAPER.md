# Snake Game for Game of Prompts (GoP)

This repository contains an implementation of the classic Snake game, designed to function as a **Game Service** within the Game of Prompts (GoP) platform. It allows Solver Services (bots or AIs) to compete to achieve the highest possible score.

## General Description

The service implements the Snake game where the objective is to control a snake to eat apples while avoiding collisions and poisonous apples. The game ends if the snake collides with the board's walls or with itself.

As a GoP Game Service, this implementation is designed to:
1.  Receive a **Solver Service** provided by a player.
2.  Run a game of Snake where the Solver Service makes the movement decisions.
3.  Evaluate the Solver's performance based on the final score.
4.  Generate the necessary data for the player to register their game on the Ergo blockchain through the GoP platform.

## Game Operation

### Interaction with Solver Services
* When a game starts, this Game Service loads a **Solver Service** (a `.celaut.bee` file) provided by the player.
* The Game Service manages the game state (snake's position, normal apple, poisonous apples, score) and, at each game step, sends the current state to the Solver Service.
* Communication with the solver is done via an HTTP API. The Game Service sends a POST request to the solver's `/move` endpoint with the current game state in JSON format:
    ```json
    {
        "snake": [[y,x], [y,x], ...], // Coordinates of the snake segments (head first)
        "apple": [y,x],             // Apple coordinates
        "poison_apples": [[y,x], ...], // Poison apple coordinates
        "board_rows": 60,
        "board_cols": 120
    }
    ```
* The Solver Service must respond with a move in JSON format:
    ```json
    {
        "move": "UP" // Options: "UP", "DOWN", "LEFT", "RIGHT"
    }
    ```
* The game progresses according to the received move. If the solver does not respond in time, returns an error, or an invalid move, the game ends.

### Game Parameters
* **Board Size:** Configured by default to 60 rows by 120 columns.
* **Seed:** The game uses a seed (an integer or a string) for random number generation (initial position of the snake and apples). This ensures that a game can be **reproducible** if the same seed and the same solver are used. The seed can be provided by the user when starting the game.

### Scoring
* Score is decoupled from snake length:
  * Eating a normal apple adds `+1`.
  * Eating a poisonous apple subtracts `-2` (minimum score `0`).
  * Poison apples on board: `floor(score / 10)`.

## Integration with Game of Prompts

At the end of each game (whether by victory, defeat, or solver error), this Game Service generates a set of data that the player can use to participate in a GoP competition. This data includes:

* **`solverId`**: The unique identifier of the Solver Service that played the game.
* **`true_score`**: The final score obtained in the game (not necessarily snake length).
* **`hash_logs_hex`**: A Blake2b256 hash of the complete game history (sequence of states and moves), ensuring the integrity of the logs.
* **`commitment_c_hex`**: The cryptographic commitment that links `solverId`, `true_score`, `hash_logs_hex`, and the Game Service's secret (`SECRET_S_HEX`). This is the fundamental data that is registered on-chain.
* **`score_list`**: A list of scores (including the `true_score` and several decoy scores) to obfuscate the real score in the `ParticipationBox` until the secret is revealed.
* **`seed`**: The seed used for that specific game, allowing its reproducibility.

This data can be downloaded from the game's web interface in a standardized JSON format.

## Game Service Web Interface

This service includes an interactive web interface built with Flask that allows:
* **Configure and Start a Game:**
    * Upload a `.celaut.bee` file corresponding to the Solver Service.
    * Optionally, specify a seed for the game.
    * Start the game execution, where the Game Service will call the Solver Service.
* **Live Visualization:** Watch the Snake game in real-time on an HTML canvas. The score, last move, and current seed are displayed.
* **Stop the Game:** Interrupt an ongoing game.
* **GoP Participation Data:**
    * Once the game is finished, the interface displays the generated participation data.
    * Allows the user to review the `score_list` (with the real score highlighted) and edit the decoy scores.
    * Allows annotating an "indeterminism index" for the solver.
    * Download the JSON data package ready to be used on the GoP platform to register the participation on-chain.
* **History and Playback:**
    * Download the complete history of the last game played (detailed logs).
    * Upload a JSON history file (previously downloaded) to visually replay a past game, with speed, pause, and frame-by-frame advance controls.

## Guía práctica para desarrollar y empaquetar un Solver Service

Esta sección está orientada a desarrolladores que quieren construir su propio solver desde cero y dejarlo listo para distribución (`.celaut`).

### 1) Cómo empezar a implementar el solver

Tu solver debe exponer un endpoint `POST /move` que reciba este estado:

```json
{
  "snake": [[y, x], [y, x]],
  "apple": [y, x],
  "poison_apples": [[y, x]],
  "board_rows": 60,
  "board_cols": 120
}
```

Y responder siempre con:

```json
{
  "move": "UP"
}
```

Donde `move` solo puede ser una de: `UP`, `DOWN`, `LEFT`, `RIGHT`.

Punto de partida recomendado:
1. Implementa primero un solver simple y estable, no el más inteligente.
2. Prioriza evitar movimientos inválidos y colisiones por encima de perseguir la manzana.
3. Asegura latencia baja y estable por tick (evita lógica pesada en cada decisión).
4. Valida el payload de entrada y maneja errores de forma explícita.
5. Si tu estrategia avanzada falla, usa fallback seguro (nunca devuelvas un string inválido).

### 2) Estructura mínima del proyecto y carpeta `.service`

Estructura recomendada:

```text
my-solver-project/
├── app.py (o binario/entrada principal)
└── .service/
    ├── Dockerfile
    ├── service.json
    └── pack_config.json
```

Qué debe contener cada archivo:

1. `.service/service.json` (especificación del servicio)
   - `tag`: nombre del servicio (ej. `my-solver`).
   - `api.port`: puerto HTTP donde escucha tu solver.
   - `entrypoint`: ejecutable/script dentro del contenedor.
   - `architecture`: normalmente `linux/amd64`.
   - `resources`: límites de memoria/disco.

2. `.service/pack_config.json` (configuración de empaquetado)
   - `workdir`: normalmente `service`.
   - `include`: lista exacta de archivos/carpetas a empaquetar.
   - Mantén `include` mínimo para reducir tamaño y tiempos.

3. `.service/Dockerfile`
   - Construye runtime reproducible del solver.
   - Instala solo dependencias necesarias.
   - Garantiza que el entrypoint exista y sea ejecutable.
   - Si usas compilación (ej. Rust), usa multi-stage build.

Bloque de referencia mínima (solver + `.service`):

```python
# app.py (solver HTTP mínimo y seguro)
from flask import Flask, request, jsonify
app = Flask(__name__)
DIRS = {"UP": (-1, 0), "DOWN": (1, 0), "LEFT": (0, -1), "RIGHT": (0, 1)}
OPPOSITE = {"UP": "DOWN", "DOWN": "UP", "LEFT": "RIGHT", "RIGHT": "LEFT"}

def step(pos, move):
    dy, dx = DIRS[move]
    return [pos[0] + dy, pos[1] + dx]

@app.post("/move")
def move():
    s = request.get_json(silent=True) or {}
    snake = s.get("snake") or []
    apple = s.get("apple") or [0, 0]
    rows = int(s.get("board_rows", 60))
    cols = int(s.get("board_cols", 120))
    last = s.get("last_move")
    if not snake:
        return jsonify({"move": "RIGHT"})

    head = snake[0]
    body = {tuple(p) for p in snake[1:]}
    candidates = []
    for m in ("UP", "DOWN", "LEFT", "RIGHT"):
        if last and OPPOSITE.get(last) == m:
            continue
        nxt = step(head, m)
        y, x = nxt
        if 0 <= y < rows and 0 <= x < cols and (y, x) not in body:
            dist = abs(apple[0] - y) + abs(apple[1] - x)
            candidates.append((dist, m))

    move = min(candidates)[1] if candidates else "RIGHT"
    return jsonify({"move": move})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
```

```json
// .service/service.json
{
  "tag": "my-solver",
  "api": [{ "port": 8080, "protocol": ["http"] }],
  "architecture": "linux/amd64",
  "entrypoint": "/service/app.py",
  "envs": [],
  "network": [],
  "resources": { "at_init": { "mem_limit": 250000000, "disk_space": 1000000000 } }
}
```

```json
// .service/pack_config.json
{
  "workdir": "service",
  "include": ["app.py"]
}
```

```dockerfile
# .service/Dockerfile
FROM python:3.11-slim
RUN pip3 install --no-cache-dir Flask
COPY service /service
RUN chmod +x /service/app.py
```

### 3) Cómo probar el solver antes de empaquetar

Prueba integrada con el Game Service (modo Test):
1. Inicia el Game Service local:
   ```bash
   nodo execute snake-game
   ```
2. Abre la interfaz web del servicio (URL mostrada en consola) o usa su API.
3. Selecciona `Test Mode`.
4. Carga tu solver empaquetado (`.celaut` / `.celaut.bee`) o usa el modo de prueba local que permita apuntar al solver en ejecución.
5. Lanza partidas de prueba con y sin seed fija para validar:
   - Reproducibilidad (misma seed => mismo comportamiento).
   - Robustez (sin errores HTTP, sin timeouts, sin respuestas inválidas).
   - Rendimiento (latencia estable de decisión por tick).

Checklist básico antes de hacer `pack`:
1. Levanta el solver localmente y verifica que escuche en el puerto declarado en `service.json`.
2. Prueba el endpoint:
   ```bash
   curl -sS -X POST http://127.0.0.1:<PUERTO>/move \
     -H 'Content-Type: application/json' \
     -d '{"snake":[[10,10],[10,9]],"apple":[12,20],"poison_apples":[[11,20]],"board_rows":60,"board_cols":120}'
   ```
3. Confirma que siempre devuelve JSON válido y `move` permitido.
4. Ejecuta varias semillas/estados para detectar loops, bloqueos o timeouts.
5. Si vas a competir, mide latencia media y peor caso por decisión.

### 4) Empaquetar el servicio en microVM

Cuando el solver ya esté estable:

```bash
nodo pack /home/use/my-solver-project
```

Nota: reemplaza la ruta por la real de tu máquina (por ejemplo `/home/user/my-solver-project`).

### 5) Exportar el `.celaut` para distribución

Luego de empaquetar, exporta el servicio:

```bash
nodo export my-solver /home/user/
```

Esto generará un archivo `*.celaut` (o `*.celaut.bee`, según configuración de nodo) listo para compartir y probar en otros entornos.

### 6) Obtener el hash del servicio para blockchain

Obtén el identificador/hash del servicio y úsalo en el registro on-chain:

```bash
nodo servicies | egrep my-solver
```

El hash mostrado para `my-solver` es el que debes agregar en la cadena de bloques.

### 7) Validación en Real Mode antes de competir

1. Ejecuta el Game Service en modo real y sube tu `solver.celaut` (o `solver.celaut.bee`) desde UI o API.
2. Si es una prueba exploratoria, puedes ejecutar sin seed explícita.
3. Revisa el resultado de la partida (score, estabilidad y consistencia del solver).
4. Si el resultado te convence, registra tu `solver_id` en la competición.

### 8) Flujo después de la Seed Ceremony

1. Cuando la Seed Ceremony haya finalizado, participa usando la misma microVM/servicio que subiste previamente.
2. Ejecuta de nuevo el Game Service en `Real Mode`.
3. Carga tu `solver.celaut` (UI o API), indica el seed oficial de la partida y tu `ergotree`.
4. Ejecuta la partida en real mode: el Game Service correrá internamente tu solver y te devolverá `true_score`, `hash_logs_hex` y `commitment_c_hex`.
5. Si la puntuación te convence, finaliza la participación pagando la tarifa correspondiente.
6. Si quedas como candidato ganador, comparte cuanto antes tu archivo `solver.celaut` completo para validación por parte de los jueces.
