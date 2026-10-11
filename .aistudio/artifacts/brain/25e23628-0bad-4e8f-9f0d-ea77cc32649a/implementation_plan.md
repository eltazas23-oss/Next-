# Sistema de Aviso Táctico de Boxes (BOX Call), Radio F1 y Sincronización de Compuestos

Plan de ingeniería para incorporar el botón táctil de aviso de parada en boxes ("BOX") sobre el control de DRS izquierdo, confirmación acústica realista por radio VHF con locución del ingeniero de pista, penalización de 2 segundos en parada imprevista, bloqueo de estrategia de neumáticos en pit lane y corrección del color de los neumáticos de recambio en la animación de los mecánicos.

> [!IMPORTANT]
> **Decisiones Críticas Confirmadas por el Usuario:**
> 1. **Confirmación por Radio:** Locución de ingeniero mediante síntesis de voz Web Speech combinada con efectos procedurales Web Audio de radio VHF (ruido de squelch, filtro paso banda 400Hz–3.5kHz y roger beeps de transmisión).
> 2. **Comportamiento del Aviso:** Si el piloto activa el botón pero no entra a boxes en esa vuelta, el aviso se cancela automáticamente al cruzar la línea de meta.
> 3. **Estilo del Botón BOX:** Cristal translúcido sutil a juego con los mandos ergonómicos, ubicado exactamente encima del botón de DRS (sin alterar este último), que se ilumina en ámbar brillante con anillo pulsante al estar activado.
> 4. **Penalización por Parada Imprevista:** Si se entra a boxes sin haber pulsado el botón BOX, la parada se extiende exactamente 2 segundos adicionales (de 4.0s a 6.0s), manteniendo la misma coreografía cinemática mediante interpolación temporal normalizada.
> 5. **Bloqueo de Estrategia:** El selector de neumáticos de la HUD queda completamente bloqueado con el distintivo `🔒 BOX · ESTRATEGIA BLOQUEADA` una vez que el vehículo cruza la entrada del pit lane.
> 6. **Fidelidad Visual del Compuesto:** Los neumáticos que montan los mecánicos (repuestos en rack y animados) reflejan de forma inmediata e inequívoca el compuesto seleccionado (Amarillo Medios, Rojo Blandos, Blanco Duros), evitando desincronizaciones con el compuesto previo.

---

## 1. Visión General del Sistema y Flujo

```
┌────────────────────────────────────────────────────────────────────────┐
│                        FLUJO TÁCTICO DE PARADA EN BOXES                │
└────────────────────────────────────────────────────────────────────────┘

 [En Pista]
     │
     ├──► Piloto pulsa botón "BOX" (encima de DRS izquierdo)
     │       │
     │       ├──► Audio Radio F1: Squelch VHF + "Copy, box in this lap" + Roger Beep
     │       ├──► Botón pasa a estado ámbar brillante: "BOX CONFIRMADO"
     │       └──► Estado `isPitRequestedThisLap = true` registrado en el motor
     │
     ├──► Selección de Neumáticos en HUD (Soft / Medium / Hard) PERMITIDA en pista
     │
     ▼
 [Cruce de Línea de Entrada a Boxes / Pit Lane]
     │
     ├──► Selector de Neumáticos en HUD se BLOQUEA: "🔒 BOX · ESTRATEGIA BLOQUEADA"
     │
     ├──► ¿Se pulsó el botón BOX antes de entrar?
     │       │
     │       ├──► SÍ: Duración de parada = 4.0s (Mecánicos preparados, cambio rápido)
     │       └──► NO: Duración de parada = 6.0s (+2.0s retraso, mecánicos desprevenidos)
     │
     ├──► Los mecánicos extraen y colocan neumáticos del color EXACTO seleccionado
     │    (Amarillo si es Medios, Rojo si es Blandos, Blanco si es Duros)
     │
     ▼
 [Salida de Boxes & Nueva Vuelta]
     │
     └──► `isPitRequestedThisLap` se reinicia a `false`
     └──► Si el piloto NO entró y cruzó la meta, el aviso BOX se cancela automáticamente
```

---

## 2. Experiencia de Usuario y Diseño Visual

### 2.1. Botón "BOX" sobre el DRS Izquierdo
- **Ubicación ergonómica exacta:** Directamente encima del botón de DRS en el flanco izquierdo (`TouchControls.tsx`), situado sobre el área de giro, respetando intacto el botón de DRS existente.
- **Estado Inactivo (En Pista):**
  - Contenedor de cristal translúcido oscuro: `bg-neutral-950/60 border-amber-500/40 text-amber-300 backdrop-blur-[2px] shadow-[0_0_10px_rgba(245,158,11,0.2)]`.
  - Icono táctico (llave inglesa / radio) con texto `BOX IN`.
- **Estado Activo (Confirmado para esa vuelta):**
  - Resplandor ámbar vivo: `bg-amber-500/85 border-amber-300 text-neutral-950 ring-2 ring-amber-400 shadow-[0_0_16px_rgba(245,158,11,0.5)] font-black scale-[1.03]`.
  - Texto dinámico: `BOX CONFIRMADO`.
- **Interacción y Disparo:** Activación instantánea mediante pointer events y touch start (`0ms latency`), evitando retrasos de clic móvil.

### 2.2. Bloqueo del Selector de Neumáticos en HUD
- **En Pista:** Los 3 botones de compuestos (S / M / H) permanecen interactivos para planificar la estrategia.
- **En Pit Lane (`isInPit === true` o tras cruzar línea de entrada):**
  - Los botones se atenúan (`opacity-50 pointer-events-none cursor-not-allowed`).
  - Cabecera de parada reemplazada por una insignia de alta visibilidad: `🔒 BOX · ESTRATEGIA BLOQUEADA`.

---

## 3. Arquitectura Técnica y Estrategia de Rendimiento

### 3.1. Arquitectura de Componentes y Estado

```
┌────────────────────────────────────────────────────────────────────────┐
│                        ARQUITECTURA DEL SISTEMA                        │
└────────────────────────────────────────────────────────────────────────┘

             ┌──────────────────────────────────────────────┐
             │            RacingGameEngine.ts               │
             │  - isPitRequestedThisLap: boolean            │
             │  - requestBoxThisLap()                       │
             │  - checkPitStopArea()                        │
             │  - onLapCompleted() -> reset Box if no pit   │
             └──────────────┬────────────────────────┬──────┘
                            │                        │
            ┌───────────────▼────────┐      ┌────────▼──────────────┐
            │   PitStopManager.ts    │      │    TouchControls.tsx  │
            │ - totalDuration: 4s/6s │      │ - Botón BOX sutil     │
            │ - freshPzeroStripeMat  │      │   sobre DRS izq       │
            │ - spareWheels visual   │      │ - Feedback háptico/ui │
            └────────────────────────┘      └───────────────────────┘
                            │
            ┌───────────────▼────────────────────────┐
            │          RadioAudioSystem.ts           │
            │ - Procedural VHF noise burst + beeps   │
            │ - Web Speech API: "Copy, box this lap" │
            │ - Bandpass telephone filter 400-3500Hz │
            └────────────────────────────────────────┘
```

### 3.2. Duración de la Parada (4.0s vs 6.0s) sin Alterar Animaciones
En `PitStopManager.ts`:
- La animación de los mecánicos y la cinemática de las pistolas/gatos se basa en `serviceT = (t - 1.0) / serviceDuration`, donde `serviceDuration = (totalDuration - 0.6) - 1.0`.
- Cuando `isPitRequestedThisLap === false`, configuramos `this.totalDuration = 6.0`.
- Esto alarga la fase de servicio en exactamente 2 segundos sin cortar ni desfasar ningún fotograma ni curva Bézier de los mecánicos, simulando con realismo que los mecánicos tuvieron que preparar los neumáticos con retraso.

### 3.3. Solución Radical al Bug del Color de Neumáticos
- **Causa raíz identificada:**
  1. `this.freshPzeroStripeMat` no actualizaba `needsUpdate = true` de forma atómica en todos los objetos de la rueda de repuesto (`spareWheels`).
  2. Los LEDs indicadores de los calentadores térmicos tenían color rojo rígido.
  3. `CarModel` cambiaba sus texturas dinámicas al comenzar el servicio, pero si `nextTireCompound` no se propagaba inmediatamente al material compartido de `spareWheels`, la rueda física en la cuna térmica y en las manos de los mecánicos conservaba el color por defecto (rojo).
- **Corrección:**
  - Crear un método unificado `updateSpareTireCompoundVisuals(compound: TireCompoundType)` en `PitStopManager.ts`.
  - Recorrer todas las mallas de los 4 `spareWheels` y actualizar simultáneamente:
    - El material de la banda exterior (Pirelli ring) con el color hexadecimal exacto (`soft: 0xef4444`, `medium: 0xfacc15`, `hard: 0xffffff`) y `needsUpdate = true`.
    - La emisión del aro para alta visibilidad bajo cualquier iluminación.
    - Los LEDs digitales del rack térmico de calentadores de neumáticos.
  - Sincronizar inmediatamente al momento en que el usuario pulse cualquier compuesto en pista, asegurando que los neumáticos visibles en los soportes ya sean los correctos antes de entrar.

### 3.4. Confirmación de Radio VHF Realista ("Copy, box in this lap")
- Al pulsar el botón BOX:
  1. Disparo de un chirp de radio F1 (doble tono 1200Hz y 1800Hz de 40ms con distorsión armónica).
  2. Ráfaga de ruido blanco analógico VHF filtrado por paso banda (500Hz a 3200Hz) con ráfaga de squelch.
  3. Locución del ingeniero de boxes vía `window.speechSynthesis` con acento en inglés/español de carreras, tasa acelerada de radio y corte con tono de despedida "Roger Beep".

---

## 4. Fases de Ejecución

1. **Gestión de Audio de Radio F1 (`src/utils/uiAudio.ts` o módulo de audio):**
   - Implementar función de audio procedural para radio de boxes con VHF squelch, beeps y síntesis de voz con frase auténtica "Copy, box in this lap".
2. **Lógica de Motor y Estado de Box en Vuelta (`src/game/RacingGameEngine.ts`):**
   - Añadir estado `isPitRequestedThisLap` y método `toggleBoxCall()`.
   - Registrar la cancelación automática de la llamada si se cruza la meta sin haber entrado a boxes.
   - Enviar estado en el payload de telemetría para la interfaz.
3. **Mecánica de Parada y Duración (`src/game/pit/PitStopManager.ts`):**
   - Ajustar `totalDuration`: 4.0s si `isPitRequestedThisLap === true`, 6.0s (+2.0s) si no se avisó.
   - Implementar actualización exhaustiva del color de las 4 ruedas de recambio, cunas térmicas y LEDs al compuesto seleccionado.
   - Asegurar que las ruedas desmontadas muestren el compuesto anterior y las nuevas muestren el compuesto nuevo sin fallos.
4. **Controles Táctiles (`src/components/TouchControls.tsx`):**
   - Añadir el botón sutil "BOX" justo encima del botón de DRS del flanco izquierdo, sin alterar el botón de DRS.
   - Configurar estilo de cristal ámbar reactivo y feedback táctil inmediato.
5. **Selector de Neumáticos en HUD (`src/components/HUD.tsx`):**
   - Bloquear la interacción con el selector una vez que el monoplaza cruce a la línea de boxes (`isInPit`).
   - Mostrar indicador gráfico de estrategia bloqueada durante la estancia en boxes.
6. **Verificación y Compilación:**
   - Validar con `compile_applet` y comprobar fluidez a 60/120 FPS sin degradación de memoria.
