# Direct-Rendered UI (DUI) y Render Targets en FiveM

Guía técnica completa y patrones de producción para renderizar navegadores web Chromium (CEF) directamente sobre texturas runtime, objetos 3D del mundo, carteles, pantallas de TV, tablets y Scaleforms en FiveM.

---

## 1. ¿Qué es DUI y cómo se diferencia de NUI?

- **NUI (Normal User Interface)**: Crea un `<iframe>` transparente que se superpone a pantalla completa en la vista 2D del jugador. Tiene foco directo de ratón y teclado mediante `SetNuiFocus`.
- **DUI (Direct-Rendered UI)**: Crea una instancia invisible de Chromium y vuelca su salida visual en un **Texture Handle de GPU** (`DuiHandle`). Esa textura puede proyectarse en:
  1. Objetos del mundo 3D (pantallas de cine, televisores, carteles publicitarios) mediante `AddReplaceTexture`.
  2. Props en mano (tablets policiales, smartphones) mediante **Render Targets**.
  3. Sprites 2D en pantalla con `DrawSprite`.

---

## 2. Nativas Esenciales de DUI

| Nativa | Propósito |
|---|---|
| `CreateDui(url, width, height)` | Instancia un navegador web en memoria con resolución dada |
| `IsDuiAvailable(dui)` | Retorna `true` cuando la textura ya está disponible para el render |
| `GetDuiHandle(dui)` | Obtiene el identificador de textura en memoria (string) |
| `DestroyDui(dui)` | **Crítico:** Libera memoria y destruye el proceso CEF |
| `SetDuiUrl(dui, url)` | Cambia la URL sin necesidad de destruir y recrear la textura |
| `SendDuiMessage(dui, jsonString)` | Envía un evento `message` a la página web |
| `SendDuiMouseMove(dui, x, y)` | Simula el cursor en coordenadas de píxeles locales `(0 a width, 0 a height)` |
| `SendDuiMouseDown(dui, 'left'/'right')` | Simula click presionado |
| `SendDuiMouseUp(dui, 'left'/'right')` | Simula liberación de click |
| `SendDuiMouseWheel(dui, deltaY, deltaX)`| Simula scroll de rueda de ratón |

---

## 3. Patrón 1: Reemplazo de Textura en Objetos del Mundo (`AddReplaceTexture`)

Este método es el más eficiente y limpio para televisores fijos, monitores de oficina y pantallas públicas.

### Código de Implementación (`client/screen.lua`):
```lua
local currentDui = nil
local textureDictionaty = 'custom_screen_txd'
local textureName = 'screen_texture'
local isReplaced = false

-- Modelo del prop y texturas originales a reemplazar
-- (Identificables mediante CodeWalker o OpenIV)
local TARGET_MODEL = `prop_tv_flat_01`
local TARGET_TXD = 'prop_tv_flat_01'
local TARGET_TXN = 'script_rt_tvscreen'

local function StartDuiScreen(url, width, height)
    if currentDui then return end

    width = width or 1280
    height = height or 720

    -- 1. Crear el navegador DUI
    currentDui = CreateDui(url, width, height)
    
    -- Esperar a que el framebuffer del navegador esté listo
    while not IsDuiAvailable(currentDui) do
        Wait(10)
    end

    -- 2. Crear el diccionario y textura en memoria
    local txd = CreateRuntimeTxd(textureDictionaty)
    local duiHandle = GetDuiHandle(currentDui)
    CreateRuntimeTextureFromDuiHandle(txd, textureName, duiHandle)

    -- 3. Reemplazar la textura del modelo de GTA con nuestra textura web
    AddReplaceTexture(TARGET_TXD, TARGET_TXN, textureDictionaty, textureName)
    isReplaced = true
end

local function StopDuiScreen()
    if isReplaced then
        RemoveReplaceTexture(TARGET_TXD, TARGET_TXN)
        isReplaced = false
    end

    if currentDui then
        DestroyDui(currentDui)
        currentDui = nil
    end
end

-- Limpieza obligatoria si el recurso se detiene o reinicia
AddEventHandler('onResourceStop', function(resName)
    if resName == GetCurrentResourceName() then
        StopDuiScreen()
    end
end)
```

---

## 4. Patrón 2: Render Targets Nombrados (Tablets y Dispositivos)

Los render targets permiten dibujar directamente sobre la pantalla de un prop específico que el jugador sostiene o que está en el interior de un vehículo.

```lua
local renderTargetId = -1
local modelHash = `prop_cs_tablet`
local renderTargetName = 'tablet_screen' -- Nombre asignado en el modelo 3D

-- Inicializar Render Target
CreateThread(function()
    if not IsNamedRenderTargetRegistered(renderTargetName) then
        RegisterNamedRenderTarget(renderTargetName, false)
    end

    LinkNamedRenderTarget(modelHash)

    while renderTargetId == -1 do
        renderTargetId = GetNamedRenderTargetRenderId(renderTargetName)
        Wait(100)
    end
end)

-- Bucle de Render por Frame (solo mientras la tablet esté activa)
local function DrawRenderTargetLoop(txd, txn)
    CreateThread(function()
        while isTabletOpen do
            Wait(0)
            -- Asignar el destino de dibujo a la pantalla de la tablet
            SetTextRenderId(renderTargetId)
            Set_2dLayer(4)
            SetScriptGfxDrawBehindPausemenu(true)

            -- Dibujar la textura generada por el DUI cubriendo toda la pantalla
            DrawSprite(txd, txn, 0.5, 0.5, 1.0, 1.0, 0.0, 255, 255, 255, 255)

            -- Restaurar el render target predeterminado del juego
            SetTextRenderId(GetDefaultScriptRendertargetRenderId())
        end
    end)
end
```

---

## 5. Interactividad: Enviar Clicks y Ratón a la Textura DUI

A diferencia de NUI, la textura DUI no intercepta el ratón del sistema automáticamente. Si deseas interacción:

### Mapeo de Coordenadas de Raycast a Píxeles de la Web
```lua
function SendMouseToDui(duiInstance, hitNormX, hitNormY, screenW, screenH)
    -- hitNormX y hitNormY van de 0.0 a 1.0
    local localX = math.floor(hitNormX * screenW)
    local localY = math.floor(hitNormY * screenH)

    SendDuiMouseMove(duiInstance, localX, localY)

    if IsControlJustPressed(0, 24) then -- Attack / Left Click
        SendDuiMouseDown(duiInstance, 'left')
    elseif IsControlJustReleased(0, 24) then
        SendDuiMouseUp(duiInstance, 'left')
    end
end
```

---

## 6. Comunicación: DUI ➔ Servidor / Cliente de FiveM

Dado que la página web dentro de un DUI se ejecuta en el mismo motor CEF que NUI, puede enviar callbacks HTTP al cliente de FiveM:

### En el JavaScript de la página DUI (`screen.html`):
```js
fetch(`https://${GetParentResourceName()}/onDuiButtonPressed`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json; charset=UTF-8' },
    body: JSON.stringify({ button: 'play_video', trackId: 42 })
}).then(resp => resp.json());
```

### En el script cliente de FiveM (`client.lua`):
```lua
RegisterNuiCallback('onDuiButtonPressed', function(data, cb)
    print(('Botón presionado en la pantalla DUI: %s'):format(data.button))
    cb({ success = true })
end)
```

---

## 7. Optimización y Reglas de Oro

1. **Gestión de Distancia (LOD)**: Un DUI activo consume memoria RAM y CPU de Chromium. Nunca dejes un DUI activo en segundo plano si el jugador no está viendo la pantalla. Utiliza un chequeo de distancia (`#(coords - targetCoords) < 15.0`) o un `lib.points` para crear el DUI al acercarse y destruirlo con `DestroyDui` al alejarse.
2. **URLs Locales seguras**: Utiliza siempre el protocolo seguro `https://cfx-nui-<recurso>/web/dist/...` y añade los archivos al array `files { ... }` de tu `fxmanifest.lua`.
3. **Liberación de Memoria**: Siempre llama a `RemoveReplaceTexture` y `DestroyDui` al cerrar el script o en `onResourceStop`. Si no se destruye, la textura permanece en la memoria VRAM del cliente.
