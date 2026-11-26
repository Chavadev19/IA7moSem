# Sanctified – Documentación de la IA (Kanban de tareas)

Juego de terror en primera persona. Un cura tiene que juntar 5 objetos malditos en una casa mientras un fantasma lo caza. El fantasma usa un **Behavior Tree** conectado a un **Blackboard**, se mueve con **NavMesh** y detecta al jugador con **AI Perception** (vista y oído).

Todo lo del proyecto está en `Content/Sanctified/`:

```
Sanctified/
├── Blueprints/
│   ├── BP_MainCharacter        (jugador)
│   ├── BP_GameModeSanctified   (game mode)
│   └── BP_ObjetoMaldito        (objeto que se recolecta)
├── IA/
│   ├── BP_Fantasma             (el NPC, es un Character)
│   ├── AIC_Fantasma            (AI Controller)
│   ├── BB_Fantasma             (Blackboard)
│   ├── BT_Fantasma             (Behavior Tree)
│   ├── M_Fantasma              (material translúcido)
│   └── Tasks/
│       ├── BTT_PuntoAleatorio
│       ├── BTT_PuntoLejano
│       ├── BTT_CambiarBool
│       └── BTT_CambiarVelocidad
├── Input/
│   └── IA_AguaBendita          (Input Action)
└── Maps/
    └── Lvl_Casa                (nivel principal)
```

---

## Tarea #2 – Crear el main level con el NavMesh

**Qué se hizo:** se creó el nivel `Lvl_Casa`.

- Es una casa de **un solo piso** con **6 cuartos** (3 × 2) de 1000 × 1000 unidades cada uno. Las paredes miden 350 de alto.
- Los cuartos se conectan con **puertas** (huecos de 220 de ancho) para que el fantasma pueda ir de uno a otro. Se hizo de un solo piso para no batallar con escaleras en la navegación, como dice el One Pager.
- Hay un **NavMeshBoundsVolume** que cubre toda la casa. Ese volumen le dice a Unreal en qué zona generar el NavMesh, que es la "malla" por donde la IA sabe caminar.
- En *Project Settings → Navigation Mesh → Runtime Generation* está en **Dynamic**, así el NavMesh también se actualiza mientras se juega.
- Se puso como mapa por defecto: *Project Settings → Maps & Modes → Editor Startup Map / Game Default Map = Lvl_Casa*.

**Cómo enseñarlo:**
- En el viewport, presiona **P** para ver el NavMesh (el área verde). Se ve que pasa por las puertas.
- Si un mueble tapa el paso, el NavMesh le hace un hueco alrededor. Así la IA no se queda atorada.

**Posibles preguntas:**
- *¿Por qué NavMesh?* Porque el nodo **Move To** del Behavior Tree necesita el NavMesh para calcular el camino (pathfinding). Sin NavMesh el fantasma no se mueve.
- *¿Qué pasa si quito el NavMeshBoundsVolume?* El fantasma se queda quieto: no encuentra puntos ni caminos.

---

## Tarea #3 – Setear el main character (BP_MainCharacter)

**Qué se hizo:** se duplicó el personaje del template de terror (`BP_HorrorCharacter`) y se le puso de nombre **BP_MainCharacter**. Ya trae la cámara en primera persona, la **linterna** y el **sprint** con barra de stamina, que va con un juego de terror.

**Variables nuevas:**

| Variable | Tipo | Valor inicial | Para qué |
|---|---|---|---|
| `AguaBendita` | Integer | 3 | Botellas de agua bendita que le quedan (Instance Editable) |
| `ObjetosRecolectados` | Integer | 0 | Cuántos objetos malditos lleva |

**Input de agua bendita:**
- Se creó el Input Action `IA_AguaBendita` y se mapeó en el `IMC_Horror` con **clic izquierdo** y la tecla **F**.

**Lógica del agua bendita (Event Graph, abajo de los nodos del template):**

```
EnhancedInputAction IA_AguaBendita (Started)
 └─ Branch (AguaBendita > 0)
     ├─ True:
     │   Set AguaBendita = AguaBendita - 1
     │   Get Actor Of Class (BP_Fantasma)
     │   Branch (Get Distance To fantasma <= 400)
     │     ├─ True:  Asustar (evento del fantasma) → Print "Agua bendita restante: X"
     │     └─ False: Print "Agua bendita restante: X"   (la gastaste y fallaste)
     └─ False: Print "Ya no tienes agua bendita!"
```

**Game Mode:** `BP_GameModeSanctified` (copia del GameMode de terror) tiene **Default Pawn Class = BP_MainCharacter**. También quedó como GameMode por defecto del proyecto y del nivel (World Settings → GameMode Override).

---

## Tarea #4 – Crear los personajes: que tengan BT, BB, Controller y que sean Character

### BP_Fantasma (el NPC)
- **Clase padre:** `Character`. Por eso ya trae Capsule, Mesh y **Character Movement**, que es lo que usa el Move To para moverlo.
- **Mesh:** `SKM_Manny_Simple` con el material `M_Fantasma`, que es translúcido, azulado y emisivo para que parezca espíritu.
- **Anim Class:** `ABP_Unarmed`, la animación del maniquí (idle / caminar / correr según la velocidad).
- **AI Controller Class:** `AIC_Fantasma`.
- **Auto Possess AI:** *Placed in World or Spawned*. Así el controller lo posee en cuanto empieza el juego.
- **Character Movement:** Max Walk Speed 180, *Orient Rotation to Movement* activado y *Use Controller Rotation Yaw* desactivado, para que voltee hacia donde camina.
- **Componente `ZonaAtaque`:** una Sphere Collision de radio 70 (OverlapAllDynamic). Es la zona con la que "toca" al jugador.
- **Variable `VelocidadExtra`** (Float): sirve para el sistema de agresividad.

### AIC_Fantasma (AI Controller)
- **Event On Possess → Run Behavior Tree (BT_Fantasma)**: arranca el árbol cuando posee al fantasma.
- Tiene el componente **AIPerception** con los sentidos de la tarea #6.

### BB_Fantasma (Blackboard) – la "memoria" del fantasma

| Key | Tipo | Para qué |
|---|---|---|
| `Jugador` | Object (Actor) | Referencia al jugador que detectó |
| `VeJugador` | Bool | Si ahora mismo lo está percibiendo |
| `Asustado` | Bool | Si le echaron agua bendita |
| `Despierto` | Bool | Si ya terminó el estado Idle del inicio |
| `PuntoDestino` | Vector | A dónde camina (patrulla o huida) |

### BT_Fantasma (Behavior Tree)
Usa el `BB_Fantasma` como Blackboard. La explicación completa está en la tarea #6.

---

## Tarea #5 – Hacer un nivel con sus assets

Dentro de `Lvl_Casa` se usaron los assets de *LevelPrototyping* y del template de terror:

- **Estructura:** piso, techo y paredes con `SM_Cube` y materiales de grid.
- **Muebles** (carpeta *Casa/Muebles* en el Outliner): cama, mesa, sillón, librero, altar, ropero, columna y barril.
- **Iluminación de terror:**
  - Point Lights naranjas y tenues en 5 cuartos. Un cuarto se dejó **totalmente oscuro**.
  - SkyLight muy bajo (0.3), Exponential Height Fog y un PostProcessVolume con **viñeta**.
  - El jugador trae la linterna del template.
- **Actores de juego:**
  - `PlayerStart` en el primer cuarto.
  - `BP_Fantasma` en el cuarto más lejano.
  - 5 `BP_ObjetoMaldito` repartidos por la casa (carpeta *Objetos*).

### BP_ObjetoMaldito
Componentes: un cubo pequeño (Static Mesh) más una **Point Light roja** para que se vea en la oscuridad.

```
Event ActorBeginOverlap (Other Actor)
 └─ Cast To BP_MainCharacter
     Set ObjetosRecolectados = ObjetosRecolectados + 1
     Report Noise Event (en la posición del objeto, instigator = jugador)   ← el fantasma lo puede OÍR
     Get Actor Of Class (BP_Fantasma) → AumentarAgresividad                ← se vuelve más rápido
     Print "Objetos malditos: X / 5"
     Branch (ObjetosRecolectados >= 5)
       ├─ True:  Print "La casa fue santificada. GANASTE!" → Set Game Paused (true) → Destroy Actor
       └─ False: Destroy Actor
```

---

## Tarea #6 – Modificar el BT para que tenga Sentidos, Movimiento, Animaciones e Interacciones

### Sentidos (AI Perception en AIC_Fantasma)

| Sentido | Configuración |
|---|---|
| **Sight (vista)** | Sight Radius 1500, Lose Sight Radius 1800, ángulo de visión 70°, detecta Enemies, Neutrals y Friendlies |
| **Hearing (oído)** | Hearing Range 1500, Max Age 3 s (después de 3 s "olvida" el ruido) |

Se marca **Detect Neutrals** porque el jugador no tiene equipo asignado y Unreal lo toma como *neutral*. Si solo se detectan enemigos, el fantasma no lo ve.

**Evento On Target Perception Updated (Actor, Stimulus):**

```
On Target Perception Updated
 └─ Cast To BP_MainCharacter (Actor)
     Get Blackboard → Set Value as Object  (Key "Jugador",   valor = Actor)
     Get Blackboard → Set Value as Bool    (Key "VeJugador", valor = Break AIStimulus → Successfully Sensed)
```

Para el nombre de la key se usa *Make Literal Name*. **Successfully Sensed** es *true* cuando lo ve o lo oye y *false* cuando lo pierde de vista. Así el BT sabe cuándo perseguir y cuándo dejar de hacerlo.

### Estructura del Behavior Tree

```
ROOT
└── SELECTOR "Estados del Fantasma"
    ├── 1) SEQUENCE "Evade Main Character"      [Decorator: Asustado Is Set — Observer aborts: Lower Priority]
    │     ├─ BTT_CambiarVelocidad (500)          "Correr rapido"
    │     ├─ BTT_PuntoLejano (Jugador → PuntoDestino)
    │     ├─ Move To (PuntoDestino)              "Huir"
    │     ├─ Wait 5 s                            "Wait 5 segs"
    │     ├─ BTT_CambiarBool (Asustado = false)  "Ya no esta asustado"
    │     └─ BTT_CambiarBool (VeJugador = false) "Olvidar al jugador"
    │
    ├── 2) SEQUENCE "Idle"                       [Decorator: Despierto Is Not Set — Observer aborts: None]
    │     ├─ Wait 8 s                            "Esperar al inicio"
    │     └─ BTT_CambiarBool (Despierto = true)  "Despertar"
    │
    ├── 3) SEQUENCE "Follow Main Character"      [Decorator: VeJugador Is Set — Observer aborts: Both]
    │     ├─ BTT_CambiarVelocidad (330)          "Velocidad de caceria"
    │     └─ Move To (Jugador, Acceptable Radius 10)
    │
    └── 4) SEQUENCE "Move Random Point"          (sin decorator: es lo que hace "por defecto")
          ├─ BTT_CambiarVelocidad (180)          "Velocidad normal"
          ├─ BTT_PuntoAleatorio (→ PuntoDestino)
          ├─ Move To (PuntoDestino)
          └─ Wait 2 s
```

**Cómo funciona:**
- El **Selector** prueba sus hijos de izquierda a derecha y ejecuta el **primero que puede**. Por eso el orden es la prioridad: Huir > Idle > Perseguir > Patrullar.
- Los **Decorators de Blackboard** son las condiciones de cada rama.
- El **Observer Aborts** hace que el árbol reaccione al instante cuando cambia una key:
  - **Huir (Lower Priority):** si le echan agua bendita mientras persigue o patrulla, cancela eso y huye.
  - **Perseguir (Both):** si lo ve mientras patrulla, cancela la patrulla (*lower priority*). Si lo pierde mientras persigue, cancela la persecución (*self*).
  - **Idle (None):** no cancela nada. Solo corre una vez al inicio y le da al jugador tiempo de explorar.

**Los 5 estados del One Pager:**

| Estado del PDF | Rama del BT |
|---|---|
| Idle | Sequence "Idle" (espera 8 s al inicio) |
| Move Random Point | Sequence "Move Random Point" |
| Follow Main Character | Sequence "Follow Main Character" |
| Evade Main Character | Sequence "Evade Main Character" |
| Wait 5 segs | Wait 5 s dentro de la rama de huida |

### Movimiento
- Todo el movimiento lo hace el task **Move To** del BT, que usa el NavMesh.
- La velocidad cambia según el estado con `BTT_CambiarVelocidad`: patrulla 180, persecución 330 y huida 500.

### Animaciones
- El fantasma usa `ABP_Unarmed`, que lee la velocidad del Character Movement y mezcla idle, caminar y correr con un Blend Space.
- Por eso, al cambiar la velocidad en el BT, la animación cambia sola: camina despacio al patrullar y corre al perseguir o huir.

### Interacciones (en BP_Fantasma)

**1. Atrapar al jugador (Game Over):**
```
Event ActorBeginOverlap (Other Actor)   ← la ZonaAtaque toca al jugador
 └─ Cast To BP_MainCharacter
     Branch (Blackboard "Asustado")      ← si está asustado NO te mata
       └─ False: Print "El fantasma te atrapo... GAME OVER"
                 Delay 2 s → Get Current Level Name → Open Level   (reinicia el nivel)
```

**2. Asustarse con agua bendita** (lo llama el jugador):
```
Custom Event Asustar
 └─ Set Value as Bool   (Asustado = true)
    Set Value as Object (Jugador = Get Player Character)   ← para saber de quién huir aunque no lo haya visto
    Print "El fantasma huye!"
```

**3. Sistema de agresividad** (lo llama cada objeto recolectado):
```
Custom Event AumentarAgresividad
 └─ VelocidadExtra = VelocidadExtra + 60
    Print "El fantasma se vuelve mas agresivo..."
```
Esa `VelocidadExtra` se suma en `BTT_CambiarVelocidad`. Con los 5 objetos, el fantasma va 300 unidades más rápido que al inicio.

---

## Tarea #7 – Hacer Tasks personalizadas

Todas heredan de **BTTask_BlueprintBase**. Usan el evento **Event Receive Execute AI** (te da el *Owner Controller* y el *Controlled Pawn*) y siempre terminan con **Finish Execute**. Si no se llama a Finish Execute, el árbol se queda trabado en esa task.

Las variables tipo **Blackboard Key Selector** están marcadas como *Instance Editable*. Por eso, al seleccionar la task en el BT, se puede elegir qué key del Blackboard usa. Así una misma task sirve para varias cosas.

### BTT_PuntoAleatorio (patrullar)
- Variables: `Destino` (BB Key Selector), `Radio` (Float, 1500).
```
Receive Execute AI
 └─ Get Random Reachable Point in Radius (Origin = ubicación del fantasma, Radius = Radio)
    Set Blackboard Value as Vector (Destino = Random Location)
    Finish Execute (Success = si encontró punto)
```

### BTT_PuntoLejano (huir del jugador)
- Variables: `Jugador` (BB Key), `Destino` (BB Key), `Distancia` (Float, 1200).
- Calcula la dirección **jugador → fantasma**, avanza `Distancia` en esa dirección y busca un punto del NavMesh cerca de ahí (radio 1500).
```
Dirección = Normalize(PosFantasma - PosJugador)
PuntoLejos = PosFantasma + Dirección * Distancia
Get Random Location in Navigable Radius (PuntoLejos, 1500) → Set Blackboard Value as Vector (Destino)
Finish Execute
```

### BTT_CambiarBool (task genérica)
- Variables: `Clave` (BB Key), `Valor` (Bool).
- Pone una key bool del Blackboard en true o false. Se usa 3 veces en el árbol: despertar, quitar el susto y olvidar al jugador.

### BTT_CambiarVelocidad
- Variable: `Velocidad` (Float).
- Hace Cast a BP_Fantasma y pone `Max Walk Speed = Velocidad + VelocidadExtra`. Así se conecta el sistema de agresividad con el BT.

---

## Tarea #8 – Probar el juego

**Flujo esperado al darle Play:**
1. Apareces en el primer cuarto. Durante ~8 s el fantasma está en **Idle** y no hace nada.
2. Después **patrulla** caminando a puntos random de la casa.
3. Si te **ve** (dentro de 1500 y a 70° de su frente) o te **oye** (cuando recoges un objeto), te **persigue** corriendo.
4. Si te toca → **GAME OVER** y se reinicia el nivel.
5. Si le das **clic o F** cuando está a 400 unidades o menos → huye rápido a un punto lejano, espera 5 s y vuelve a patrullar. Solo tienes 3 botellas.
6. Cada objeto que recoges → el fantasma se vuelve más rápido y oye dónde estás.
7. Con los 5 objetos → "La casa fue santificada. GANASTE!" y el juego se pausa.

**Herramientas útiles para la presentación:**
- **Tecla P** en el editor: muestra el NavMesh.
- **Debug del Behavior Tree:** abre `BT_Fantasma` mientras juegas (Play) y se ve en vivo qué rama está activa, además de los valores del Blackboard.
- **Tecla ' (apóstrofe)** durante el Play: activa el *AI Debugger*. Con el numpad se ven el Behavior Tree, la Perception (el cono verde de visión) y el NavMesh.

**Ya probado:** idle → patrulla, detección por vista, persecución, Game Over con reinicio, recolectar los 5 objetos con agresividad y victoria, y la huida con espera de 5 s.

**⚠️ Prueba esto antes de presentar:** el clic o F del agua bendita en una partida real.

---

## Preguntas que te pueden hacer

- **¿Qué diferencia hay entre Blackboard y Behavior Tree?** El Blackboard es la **memoria** (datos: a quién vio, a dónde va). El Behavior Tree es la **lógica** (qué hacer según esos datos).
- **¿Para qué sirve el AI Controller?** Es el "cerebro" que posee al Character. Corre el BT y tiene la percepción. El Character solo es el cuerpo.
- **¿Por qué un Selector y no un Sequence en la raíz?** El Selector elige **una** opción (el primer estado que se puede hacer). El Sequence hace **todos** sus hijos en orden y se detiene si uno falla, por eso cada estado por dentro es un Sequence.
- **¿Qué es Observer Aborts?** Hace que un decorator revise su condición todo el tiempo y cancele ramas cuando cambia la key. Sin eso, el fantasma terminaría lo que estaba haciendo antes de reaccionar.
- **¿Por qué el fantasma puede perderte?** Por el *Lose Sight Radius* y el ángulo de visión. Cuando sales de su vista, Perception manda *Successfully Sensed = false*, `VeJugador` pasa a false y el decorator aborta la persecución.
- **¿Cómo funciona el oído?** `Report Noise Event` genera un ruido en una posición. Si está dentro del Hearing Range del fantasma, se dispara la percepción igual que con la vista.
- **¿Cómo funciona la agresividad?** Cada objeto suma 60 a `VelocidadExtra`, y la task de velocidad siempre la suma a la velocidad base del estado.
- **¿Por qué la casa es de un piso?** Para evitar problemas de navegación con escaleras (está en el plan de mitigación del One Pager).
