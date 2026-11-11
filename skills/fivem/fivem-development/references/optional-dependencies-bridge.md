# Desarrollo Flexible: Soporte sin dependencias obligatorias de ox_lib ni ox_inventory

Esta guía establece el estándar para crear recursos y scripts FiveM modulares y portables que **no dependan obligatoriamente de `ox_lib` ni de `ox_inventory`**, adaptándose dinámicamente al stack presente (QBCore, ESX Legacy, Standalone o Overextended).

---

## 1. Principio Fundamental: Zero-Hard-Dependency

Un script modular nunca debe poner `ox_lib` u `ox_inventory` en `dependencies { ... }` de su `fxmanifest.lua` a menos que sea un recurso exclusivo de ese ecosistema.

### Configuración en `fxmanifest.lua`:
```lua
fx_version 'cerulean'
game 'gta5'
lua54 'yes'

-- NO incluir dependencies { 'ox_lib', 'ox_inventory' } si es modular
-- NO incluir shared_script '@ox_lib/init.lua' de forma incondicional
```

---

## 2. Patrón de Adaptador UI y Utilidades (`ox_lib` Fallback)

Crea un módulo ligero de abstracción (por ejemplo `bridge/client_ui.lua` o `utils/ui.lua`):

```lua
local hasOxLib = GetResourceState('ox_lib') == 'started'

-- Actualizar si ox_lib inicia en tiempo de ejecución
AddEventHandler('onResourceStart', function(resName)
    if resName == 'ox_lib' then hasOxLib = true end
end)
AddEventHandler('onResourceStop', function(resName)
    if resName == 'ox_lib' then hasOxLib = false end
end)

UI = {}

--- Notificaciones
function UI.Notify(title, description, notifType)
    if hasOxLib then
        return exports.ox_lib:notify({
            title = title,
            description = description,
            type = notifType or 'inform'
        })
    end

    -- Fallback QBCore
    if GetResourceState('qb-core') == 'started' then
        local qbType = notifType == 'inform' and 'primary' or notifType
        return exports['qb-core']:GetCoreObject().Functions.Notify(description or title, qbType)
    end

    -- Fallback ESX
    if GetResourceState('es_extended') == 'started' then
        return exports['es_extended']:getSharedObject().ShowNotification(description or title)
    end

    -- Fallback Nativo GTA V
    BeginTextCommandThefeedPost('STRING')
    AddTextComponentSubstringPlayerName((title and ('~b~' .. title .. '~s~\n') or '') .. (description or ''))
    EndTextCommandThefeedPostTicker(false, true)
end

--- Progress Bar / Tiempos de acción
function UI.ProgressBar(data)
    -- data: { duration, label, canCancel, anim, disable }
    if hasOxLib then
        return exports.ox_lib:progressBar({
            duration = data.duration,
            label = data.label,
            useWhileDead = false,
            canCancel = data.canCancel,
            disable = data.disable,
            anim = data.anim
        })
    end

    -- Fallback QBCore progressbar
    if GetResourceState('progressbar') == 'started' then
        local p = promise.new()
        exports['progressbar']:Progress({
            name = 'custom_action',
            duration = data.duration,
            label = data.label,
            useWhileDead = false,
            canCancel = data.canCancel,
            controlDisables = data.disable,
            animation = data.anim
        }, function(cancelled)
            p:resolve(not cancelled)
        end)
        return Citizen.Await(p)
    end

    -- Fallback Standalone con animación nativa
    if data.anim and data.anim.dict and data.anim.clip then
        RequestAnimDict(data.anim.dict)
        while not HasAnimDictLoaded(data.anim.dict) do Wait(10) end
        TaskPlayAnim(PlayerPedId(), data.anim.dict, data.anim.clip, 8.0, -8.0, data.duration, 49, 0, false, false, false)
    end
    Wait(data.duration)
    ClearPedTasks(PlayerPedId())
    return true
end

--- Puntos / Zonas de interacción sin lib.points ni lib.zones
function UI.DrawText3D(coords, text)
    local onScreen, x, y = World3dToScreen2d(coords.x, coords.y, coords.z)
    if onScreen then
        SetTextScale(0.35, 0.35)
        SetTextFont(4)
        SetTextProportional(1)
        SetTextColour(255, 255, 255, 215)
        SetTextEntry('STRING')
        SetTextCentre(1)
        AddTextComponentString(text)
        DrawText(x, y)
    end
end
```

---

## 3. Callbacks Cliente-Servidor sin `ox_lib`

Cuando `ox_lib` no está disponible (`lib.callback.await` / `lib.callback.register`), usa la alternativa según el framework o eventos nativos con promesas:

### Servidor (`server.lua`):
```lua
local hasOxLib = GetResourceState('ox_lib') == 'started'

function RegisterServerCallback(name, handler)
    if hasOxLib then
        return exports.ox_lib:callbackRegister(name, handler)
    end

    if GetResourceState('qb-core') == 'started' then
        return exports['qb-core']:GetCoreObject().Functions.CreateCallback(name, handler)
    end

    if GetResourceState('es_extended') == 'started' then
        return exports['es_extended']:getSharedObject().RegisterServerCallback(name, handler)
    end

    -- Standalone nativo con evento de respuesta
    RegisterNetEvent('callback:server:' .. name, function(ticket, ...)
        local src = source
        local response = table.pack(handler(src, ...))
        TriggerClientEvent('callback:client:' .. name .. ':' .. ticket, src, table.unpack(response))
    end)
end
```

---

## 4. Adaptador de Inventario Dinámico (`ox_inventory` Fallback)

El inventario se consulta en tiempo de ejecución. Si `ox_inventory` no está iniciado, se interactúa con el inventario nativo del framework (`qb-inventory` o `es_extended`).

### Servidor (`bridge/inventory.lua`):
```lua
local Inventory = {}

local function getInventoryType()
    if GetResourceState('ox_inventory') == 'started' then return 'ox' end
    if GetResourceState('qb-inventory') == 'started' then return 'qb' end
    if GetResourceState('es_extended') == 'started' then return 'esx' end
    return 'none'
end

function Inventory.AddItem(source, item, count, metadata)
    local invType = getInventoryType()

    if invType == 'ox' then
        return exports.ox_inventory:AddItem(source, item, count, metadata)
    elseif invType == 'qb' then
        local Player = exports['qb-core']:GetCoreObject().Functions.GetPlayer(source)
        if not Player then return false end
        return Player.Functions.AddItem(item, count, nil, metadata)
    elseif invType == 'esx' then
        local xPlayer = exports['es_extended']:getSharedObject().GetPlayerFromId(source)
        if not xPlayer then return false end
        if xPlayer.canCarryItem and not xPlayer.canCarryItem(item, count) then return false end
        xPlayer.addInventoryItem(item, count, metadata)
        return true
    end

    return false
end

function Inventory.RemoveItem(source, item, count, metadata)
    local invType = getInventoryType()

    if invType == 'ox' then
        return exports.ox_inventory:RemoveItem(source, item, count, metadata)
    elseif invType == 'qb' then
        local Player = exports['qb-core']:GetCoreObject().Functions.GetPlayer(source)
        if not Player then return false end
        return Player.Functions.RemoveItem(item, count)
    elseif invType == 'esx' then
        local xPlayer = exports['es_extended']:getSharedObject().GetPlayerFromId(source)
        if not xPlayer then return false end
        xPlayer.removeInventoryItem(item, count)
        return true
    end

    return false
end

function Inventory.GetItemCount(source, item)
    local invType = getInventoryType()

    if invType == 'ox' then
        return exports.ox_inventory:GetItemCount(source, item)
    elseif invType == 'qb' then
        local Player = exports['qb-core']:GetCoreObject().Functions.GetPlayer(source)
        if not Player then return 0 end
        local itemData = Player.Functions.GetItemByName(item)
        return itemData and itemData.amount or 0
    elseif invType == 'esx' then
        local xPlayer = exports['es_extended']:getSharedObject().GetPlayerFromId(source)
        if not xPlayer then return 0 end
        local itemData = xPlayer.getInventoryItem(item)
        return itemData and itemData.count or 0
    end

    return 0
end
```

---

## 5. Resumen de Reglas para el Asistente de IA

1. **Inspeccionar antes de asumir**: Comprueba el `fxmanifest.lua` del recurso sobre el que estás trabajando. Si no lista `@ox_lib/init.lua`, **no utilices `lib.*` sin verificar `GetResourceState('ox_lib')`**.
2. **Compatibilidad Standalone/QBCore/ESX**: Ofrece siempre implementaciones nativas o puentes adaptadores cuando el entorno no disponga de Overextended.
3. **No generar errores de runtime**: Si un export o librería no existe en el servidor del usuario, el código debe registrar un log informativo y ofrecer la alternativa nativa, nunca crashear con `attempt to index a nil value (global 'lib')`.
