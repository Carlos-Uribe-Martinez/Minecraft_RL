# Núcleo Arena — Arquitectura (Fabric 26.2, nombres de Mojang sin ofuscar)

Mod de fútbol-con-vehículos al estilo Rocket League: **Nomad** (cuña baja con alerón), **Caribe** (hatchback cuadrado) y **Núcleo** (balón). Lore y diseños propios.

## 0. Cómo compilar

```
gradle wrapper --gradle-version 9.5.1      # el wrapper .jar no se incluye
./gradlew runClient                        # Java 25 obligatorio (26.1+)
```
Versiones en `gradle.properties` (Loom 1.17, Loader 0.19.3, Fabric API 0.161.0+26.2): **confírmalas en fabricmc.net/develop**. Plugin `net.fabricmc.fabric-loom` (sin `-remap`, sin `mappings`, `implementation` en vez de `modImplementation`).

**Probarlo en 1 minuto** (mundo superflat o vacío): `/nucleo arena build small` → `/nucleo join blue` → `/nucleo start`. Práctica suelta: `/nucleo car`, `/nucleo ball`, `/nucleo pad large`. Bajar del coche: Mayús.

## 1. Estructura

```
src/main/java/dev/nucleoarena/
├─ NucleoArena.java              tipos de entidad, red, comandos, tick de partido
├─ physics/  PhysicsConstants · VecUtil · CarState · CarPhysics · WorldCollision · CarPredictor
├─ input/    VehicleInput (record + payload C2S con seq)
├─ net/      CarSnapshotPayload (S2C, estado autoritativo por tick)
├─ entity/   VehicleEntity · CustomBallEntity · BoostPadEntity
└─ arena/    ArenaConfig · ArenaBuilder · ArenaSpawnRegistry · MatchManager · ArenaCommands
src/client/java/dev/nucleoarena/
├─ client/   NucleoArenaClient · ClientInputHandler · ArenaCamera
├─ client/render/  Boxes · VehicleRenderer · BallRenderer · BoostPadRenderer (+ RenderState)
└─ mixin/    CameraMixin
```

## 1b. Servidor autoritativo + predicción en cliente

`CarPhysics.step(level, CarState, VehicleInput, steerMult)` es una función **determinista**; no usa `Entity.move` (usa `WorldCollision`: barrido AABB propio). Servidor y cliente ejecutan el mismo código.

```
Cliente (cada tick)                          Servidor (cada tick)
 1 lee teclado → VehicleInput(seq=n)          1 saca UNA entrada de la cola (o repite la última sin flanco de salto)
 2 CarPredictor.step → simula y guarda        2 CarPhysics.step → balón/demos/pads modifican el estado
 3 envía Payload(seq=n)  ───────────────►     3 envía CarSnapshotPayload(ackSeq, estado completo) al conductor
 4 recibe snapshot ◄────────────────────────
 5 compara con lo predicho para ackSeq;
   si difiere: parte del estado del servidor y RE-SIMULA las entradas > ackSeq
 6 el salto visual se absorbe con un desplazamiento que decae (×0.8/tick); >3 bloques = corrección seca
```
- La **cola** (máx. 4) absorbe jitter de red; si se llena descarta la más vieja sin perder el flanco de salto.
- Golpes de balón, demoliciones y pads solo existen en el servidor: llegan al cliente como discrepancia y se reconcilian.
- Los demás coches se ven con posición vanilla interpolada (`getPosition(pt)`) + orientación sincronizada (slerp).
- `CarState` incluye TODO lo que afecta al siguiente tick (temporizadores de flip, normales, boost…); si añades estado a la física, añádelo también a `CarState.set/write/read/differsFrom`.

## 1c. Partido (`MatchManager`)

`IDLE → COUNTDOWN (3 s, congelado) → PLAYING (5:00) → GOAL (3 s) → COUNTDOWN…`; empate a 0:00 ⇒ `OVERTIME` (gol de oro). Gol: centro del balón a `halfL + radio` de la línea, dentro del ancho/alto de la portería. AZUL defiende −Z, NARANJA +Z. Saque en las 5 posiciones estándar escaladas a la cancha; 34 pads (6 grandes, 28 pequeños) con el reparto del juego original. Marcador/tiempo/boost en la barra de acción (cada 10 ticks).

`ArenaBuilder` construye con bloques vanilla: suelo bicolor, techo y paredes de cristal, porterías de color. Medidas `full` ≈ Rocket League (82×102×20 m). **No limpia el interior**: úsalo en mundo plano/vacío.

## 2. Unidades — corrección importante

Minecraft integra en **bloques/tick** (velocidad) y **bloques/tick²** (aceleración). Conversión a 20 TPS: `v = m/s ÷ 20`, `a = m/s² ÷ 400`.

| Parámetro | Especificación | Valor interno |
|---|---|---|
| Gravedad | 6.5 m/s² | **0.01625 b/t²** |
| Balón máx. | 60 m/s | 3.0 b/t |
| Coche máx. | 23 m/s | 1.15 b/t |
| Supersónico | 22 m/s | 1.10 b/t |
| Boost (aceleración) | 9.91 m/s² | 0.024775 b/t² |
| Boost (consumo) | 33.3 /s | 1.665 /tick |
| Ventana de flip | 1.25 s | 25 ticks |
| Respawn | 3 s | 60 ticks |
| Pared: velocidad mínima | 5 m/s | 0.25 b/t |

La especificación indica "6.5 m/s² (0.325 m/tick)". **0.325 es 6.5/20** (cambio de velocidad en m/s por tick), no la unidad que usa `deltaMovement`. Aplicarlo tal cual daría una gravedad 20× mayor, así que el código usa 0.01625.

## 3. Pipeline por tick (`CarPhysics.step`)

```
1 sondas de ruedas ─► onSurface, normal        (4 raycasts hacia −UP del coche)
2 aterrizaje / despegue                         (resetea o concede ventana de flip)
3 orientación: alinear a superficie │ control aéreo (pitch/yaw/roll, flip anim)
4 salto: 1.º salto │ 2.º salto/dodge │ salto variable │ temporizador
5 gravedad (con regla de pared)
6 tracción: steer → throttle/freno → agarre lateral
7 boost
8 clamp 23 m/s → move() → respuesta a colisión/esquinas → demos → sync
```

**Orientación.** Minecraft solo tiene yaw/pitch; el coche necesita roll (paredes, techo, air roll, dodge). Se guarda un `Quaternionf` (local→mundo) autoritativo en servidor y se sincroniza con `EntityDataSerializers.QUATERNION`. El cliente interpola con slerp.

**Hitbox.** Minecraft no rota AABBs. Se usa un **cubo 1.6³** (igual en todas las orientaciones, así suelo/pared/techo se comportan igual) y una **OBB** interna (1.1 × 0.7 × 1.6, desplazada −0.45 en Y) solo para la colisión con el balón. El modelo se dibuja con las ruedas en la cara inferior del cubo.

**Detección de superficie.** Cuatro raycasts desde las posiciones de las ruedas a lo largo de `−UP` local (largo 1.0 = 0.8 + 0.2 de tolerancia). Con ≥ 3 aciertos hay superficie; la normal es la media de las caras golpeadas. Al ser relativo a la orientación del coche, funciona igual en suelo, pared y techo. Constante `WHEELS_REQUIRED` (3 por defecto; 4 si quieres ser literal).

## 4. Máquina de estados del salto (temporizadores)

| `doubleJumpAvailable` | `flipWindowTicks` | Significado |
|---|---|---|
| false | 0 | En superficie, o 2.º salto ya usado |
| true | 25 → 1 | 1.er salto hecho; 2.º salto disponible |
| true | **−1** | **Flip Reset**: sin límite hasta volver a saltar |

```java
canDoubleJump = available && (window == FLIP_UNLIMITED || window > 0);
// cada tick en el aire, DESPUÉS de procesar el botón:
if (!onSurface && window > 0 && --window == 0) available = false;
```

- El botón se evalúa **antes** de decrementar, así el tick T+25 todavía es válido (25 ticks = 1.25 s exactos).
- `jumpGraceTicks = 3` ignora el contacto tras saltar: sin esto, las sondas aún tocan el suelo el tick siguiente y el aterrizaje cancelaría el timer.
- **Flip Reset:** `CustomBallEntity.collideWithCar` detecta contacto con la cara inferior (`nLocal.y < −0.85`, centro inferior = 4 ruedas) y llama a `grantFlipReset()`. Tocar suelo, pared o techo ejecuta `onLanded()` (cancela el timer).
- `FALL_OFF_GRANTS_FLIP_WINDOW` (true): caer de un borde sin saltar también da la ventana de 1.25 s, como el original. Pon `false` para seguir la especificación al pie de la letra.
- **Dodge:** dirección = `fwd·throttle + left·steer`, impulso 5 m/s, anima 13 ticks girando alrededor de `(throttle, 0, −steer)`. La velocidad extra sale de sumar impulsos; no hay "bonus" artificial: 2–3 flips seguidos superan el tope pasivo de 14.1 m/s.

## 5. Paredes y techo

- `normal.y < 0.7` ⇒ pared/techo.
- Velocidad tangencial ≥ 5 m/s ⇒ gravedad = 0 (adherido).
- Por debajo: `g_ef = g · (1 − e^(−t/τ))` con `t` = ticks bajo el umbral y τ = 10 ⇒ arrastre exponencial.
- **Transiciones** (suelo→pared, pared→techo): si el AABB choca contra un plano nuevo con velocidad contra él > 2 m/s, esa componente se redirige a lo largo de la normal anterior (×0.97) y se fuerza `transitionNormal` 6 ticks. El coche "trepa" en vez de frenarse en seco.

## 6. Conducción y power-slide

- Pasivo: `+THROTTLE_ACCEL` por tick hasta 14.1 m/s. **Ojo:** **Decisión tomada:** `THROTTLE_ACCEL = accel(16.0)` (valor del juego original). Con el 1.6 literal de la especificación tardaba ~9 s en llegar a 14.1 m/s; se interpretó como errata. Para revertir: `accel(1.6)`.
- Agarre lateral: normal elimina el 90 %/tick de la velocidad lateral; power-slide el 4 %. Efecto emergente: con freno de mano al aterrizar se conserva la velocidad del aire, sin él el agarre la corrige.
- Giro: proporcional a velocidad (no gira parado) y ×1.7 con power-slide (media vuelta rápida).
- Air Roll: eje Z local; sin él, A/D es yaw.

## 7. Balón

- Sub-pasos (×4): coche 1.15 + balón 3.0 b/t = 4.15 b/t relativos; sin sub-pasos atravesaría un coche.
- Rebote por eje contra bloques: `v' = −v · 0.6`; bajo 0.04 b/t no hay rebote (evita temblor).
- Coche↔balón: esfera vs OBB en espacio local. Impulso `j = −(1 + e_ef) · vn`, con `e_ef = 0.6 · transfer(zona)`:

| Zona | transfer | Efecto |
|---|---|---|
| Defensa / techo | 1.00 | transferencia total |
| Lados / trasera | 0.70 | parcial |
| Ruedas / bajos | 0.00 | amortigua (+ amortiguación tangencial) |

- **Dribbling:** contacto sobre el techo con velocidad normal relativa < 0.06 b/t ⇒ sin rebote + fricción que iguala velocidades (el balón "se pega").
- **Estela (cliente):** > 20 m/s. `t = (v − 20)/40` controla densidad (1–3 partículas/punto), puntos (2–10) y largo (2–14 bloques); se ensancha hacia atrás. Tipo sincronizado (`setTrail(0..3)`).

## 8. Demoliciones y respawn

`resolvePair` (cada pareja una vez, por id): el atacante es quien va a ≥ 22 m/s, se acerca (`closing > 0`) y apunta a la víctima (`forward·n > 0.35`). Si ambos cumplen, gana el mayor cierre. Si no hay demo, choque elástico suave.

`demolish()`: explosión (emisor + llamas + humo + chispas), vehículo invisible y sin física, input ignorado. Tras 60 ticks, `ArenaSpawnRegistry.pick()` elige un punto libre del equipo, reinicia orientación, velocidad y boost (33.3).

## 9. BoostPad

Entidad sin física. Servidor: escanea coches a ≤ 1.5 bloques (`car.position()` = base del coche). **No se consume con el boost lleno.** Pequeño: +12, cooldown 4 s (la especificación no lo pide, pero sin él recargaría cada tick). Grande: 100 y cooldown 10 s. El cooldown se sincroniza para atenuar el render.

## 10. Cámaras

| Modo | Comportamiento |
|---|---|
| **RIGID** | Pegada a la orientación completa del coche (incluye roll): `pos = centro + q·(0, 1.7, −5.5)`, `rot = q · Ry(π) · Rx(−8°)`. El coche se ve igual siempre. |
| **BALL** | Mira siempre al balón, sin roll. `k = smoothstep(25°, 60°, elevación)`. Con `k` alto la cámara queda **debajo** del coche (línea coche→balón hacia atrás + altura −2.2). |

- **Coche translúcido:** `ArenaCamera.vehicleAlpha()` (1 → 0.22 según `k`; 0.35 si la cámara está a < 2.2 bloques). El renderer lo usa con `RenderType.entityTranslucent`.
- **Balón X-Ray:** raycast cámara→balón; si lo tapa un bloque o el coche (opaco), `setClientXray(true)` y `isCurrentlyGlowing()` devuelve true. El contorno *glowing* de vanilla se dibuja a través de bloques y entidades.
- **Suelo transparente:** alterar el alfa de bloques por jugador exige reconstruir chunks (inviable cada frame). Solución: `ArenaFloorBlock` con modelo de **una sola cara** (solo `up`) sobre una cavidad de aire ≥ 8 bloques. Desde arriba parece suelo; desde abajo la cara superior se descarta por *back-face culling* y no hay cara inferior, así que se ve la cancha a través. La colisión sigue siendo completa.
- `CameraMixin` aplica la pose tras `Camera.setup`, incluyendo el cuaternión completo (roll) y los ejes `forwards/up/left`.

## 11. Cámara y render — notas

- Modelos: cajas procedurales (`Boxes`) con una textura blanca de 4×4 px; color por equipo. Sin Blockbench, sin `ModelPart`.
- El coche del conductor local se dibuja en la posición/orientación **predichas**; `extractRenderState` calcula el desplazamiento respecto a la posición vanilla de la entidad.
- Coche translúcido con la cámara de balón: `ArenaCamera.vehicleAlpha()` → `RenderTypes.entityTranslucent`.
- **X-Ray del balón**: vanilla dibuja el contorno *glowing* solo en modelos propios; con geometría custom puede no aparecer. Se añadió una pasada extra con `RenderTypes.debugQuads()`, que solo atraviesa bloques si esa pipeline no hace test de profundidad (⚠ sin verificar).
- No se implementó el suelo transparente (`ArenaFloorBlock` de una cara) descrito en la versión anterior.

## 12. Verificado vs. NO verificado (léelo antes de compilar)

No he podido compilar contra 26.2 (sin red en el entorno). El código pasa el **análisis de sintaxis de javac**, pero no la resolución de símbolos.

**Contrastado con documentación oficial de 26.2/26.1:** plugin Loom y versiones; `EntityType.Builder.of(...).build(key)`; `EntityRenderers.register`; `defineSynchedData(Builder)`; `ValueInput/ValueOutput` (`getInt(..).orElse`); `PayloadTypeRegistry.serverboundPlay()/clientboundPlay()`; `KeyMappingHelper.registerKeyMapping` y `KeyMapping.Category.register(Identifier)`; `submitCustomGeometry(PoseStack, RenderType, (pose, consumer) -> …)`; Java 25.

**Puntos de ajuste probables (por orden de riesgo):**
1. `mixin/CameraMixin` — firma de `Camera.setup(BlockGetter, Entity, boolean, boolean, float)` y campos `rotation/forwards/up/left`. Solo la he visto confirmada hasta 1.21.5; 26.x refactorizó cámara/render. Si falla al arrancar, es esto.
2. Renderers — imports `state.level.CameraRenderState`, `rendertype.RenderTypes`, `EntityRenderer.submit(...)`, y setters de vértice en `Boxes` (26.2 reescribió `VertexFormat`).
3. `WorldCollision` — `Level.getBlockCollisions(null, AABB)`, `Shapes.collide(Axis, AABB, Iterable, double)`, `ClipContext(.., CollisionContext)`.
4. `VehicleEntity.demolish` — `playSound` con `GENERIC_EXPLODE` (Holder).
5. `Entity.interact(Player, InteractionHand)` y `startRiding(Entity)`.
6. Comandos **sin control de permisos** (cambió en 1.21.11+).
7. Obtención de `ctx.client()` en el receptor cliente y `Player.isLocalPlayer()`.

## 13. Limitaciones conocidas

- Paredes verticales: sin curvas; el coche las trepa por la regla de esquina. Rampas diagonales se aproximan por escalones (las sondas usan normales de cara de bloque).
- Balón: sub-pasos ×4; a 60 m/s contra esquinas muy finas puede haber micro-penetraciones.
- La predicción **no** modela golpes de balón: el cliente los ve como corrección (esperable y es lo que hace Rocket League).
- Un solo partido global; sin persistencia (el estado se pierde al reiniciar, las entidades quedan: `/nucleo clean`).
- Balance (agarre, steer, pad pequeño 12 boost/4 s) son valores de partida.

