# Guía de Ox Inventory Personalizado (Custom Ox Inventory)

Referencia técnica completa para extender, personalizar y crear sistemas avanzados sobre **ox_inventory** (v2.48+).

---

## 1. Definición de Ítems Personalizados

Los ítems se configuran en `ox_inventory/data/items.lua` o se cargan con metadatos dinámicos.

### Estructura de Ítem Avanzado con Animación y Props
```lua
['medikit_pro'] = {
    label = 'Botiquín Avanzado',
    weight = 500,
    stack = true,
    close = true,
    description = 'Restaura el 100% de la salud con un tratamiento completo.',
    client = {
        anim = { dict = 'anim@amb@clubhouse@tutorial@bkr_tut_ig3@', clip = 'machinic_loop_mechandplayer', flag = 49 },
        prop = { model = `prop_ld_health_pack`, pos = vec3(0.1, 0.0, 0.0), rot = vec3(0.0, 0.0, 0.0) },
        disable = { move = true, car = true, combat = true },
        usetime = 5000,
        cancel = true,
        export = 'my_resource.useMedikitPro'
    }
},
['id_card'] = {
    label = 'Documento de Identidad',
    weight = 10,
    stack = false,
    close = false,
    description = 'Documento personal intransferible.',
    buttons = {
        {
            label = 'Mostrar Identificación',
            action = function(slot)
                TriggerServerEvent('my_resource:showID', slot)
            end
        }
    }
}
```

---

## 2. Consumo de Ítems Personalizados (Client & Server Exports)

### Lado Cliente: Validación y UX (`client.lua`)
```lua
-- Export llamado automáticamente por client.export en items.lua
exports('useMedikitPro', function(data, slot)
    local ped = cache.ped
    local maxHealth = GetEntityMaxHealth(ped)
    local health = GetEntityHealth(ped)

    if health >= maxHealth then
        return lib.notify({
            title = 'Salud completa',
            description = 'No necesitas usar este botiquín en este momento.',
            type = 'error'
        })
    end

    -- Proceder con el consumo oficial en ox_inventory
    exports.ox_inventory:useItem(data, function(data)
        if not data then return end -- El servidor canceló la acción (ej. no tiene el ítem)
        
        SetEntityHealth(ped, maxHealth)
        lib.notify({
            title = 'Tratamiento completado',
            description = 'Has recuperado toda tu vitalidad.',
            type = 'success'
        })
    end)
end)
```

### Lado Servidor: Control de Seguridad y Mutaciones (`server.lua`)
```lua
-- Si defines server.export = 'my_resource.useItemServer' en items.lua
exports('useItemServer', function(event, item, inventory, slot, data)
    local src = inventory.id -- Cuando es jugador, id = source

    if event == 'usingItem' then
        -- Validar si el jugador está esposado, muerto o en cooldown
        if Player(src).state.isCuffed or Player(src).state.isDead then
            return false -- Cancela el uso del ítem inmediatamente
        end
        return true
    elseif event == 'usedItem' then
        -- El ítem ya fue descontado del inventario
        -- Aquí puedes registrar logs, actualizar bases de datos o dar recompensas
    end
end)
```

---

## 3. Metadatos Dinámicos (Metadata) Personalizados

`ox_inventory` permite almacenar cualquier tabla de metadatos por cada ítem individual en un slot.

### Campos especiales reconocidos por la UI:
- `label`: Sobrescribe el nombre visual del ítem.
- `description`: Sobrescribe o añade texto markdown al tooltip.
- `imageurl`: URL externa para la imagen del ítem (requiere configurar `inventory:validhosts`).
- `durability`: Número (0 a 100 o timestamp) que muestra barra de desgaste.
- `serial`: Número de serie mostrado automáticamente en armas.

### Ejemplo: Crear arma numerada con registro
```lua
local ox_inventory = exports.ox_inventory

local metadata = {
    serial = 'POL-' .. math.random(10000, 99999),
    registered = 'Oficial John Doe',
    label = 'Pistola de Servicio Calibre 9mm',
    description = '**Asignada a**: John Doe\n**Placa**: 402',
    durability = 100
}

ox_inventory:AddItem(source, 'WEAPON_COMBATPISTOL', 1, metadata)
```

### Ejemplo: Tarjeta de crédito o documento con foto externa (Fivemanage CDN)
```lua
local metadata = {
    citizenid = 'ABC12345',
    firstname = 'Carlos',
    lastname = 'Santana',
    imageurl = 'https://r2.fivemanage.com/identities/photo_123.png',
    description = 'ID: ABC12345 | Nacimiento: 1995-04-12'
}
ox_inventory:AddItem(source, 'id_card', 1, metadata)
```

---

## 4. Hooks Avanzados (`registerHook`)

Los hooks interceptan acciones del inventario en el servidor antes o después de ejecutarse.

### A. Asignar Metadatos Automáticos al Crear Ítem (`createItem`)
```lua
exports.ox_inventory:registerHook('createItem', function(payload)
    -- payload: { item, count, metadata, inventory }
    if payload.item.name == 'phone' and not payload.metadata.phoneNumber then
        payload.metadata.phoneNumber = '555-' .. math.random(1000, 9999)
        payload.metadata.description = ('Número: %s'):format(payload.metadata.phoneNumber)
        return payload.metadata
    end
end, {
    print = false,
    itemFilter = {
        phone = true
    }
})
```

### B. Restringir Mover Ítems a Maleteros / Alijos (`swapItems`)
```lua
exports.ox_inventory:registerHook('swapItems', function(payload)
    -- payload: { source, fromInventory, toInventory, fromSlot, toSlot, count }
    local toInv = payload.toInventory

    -- Bloquear meter armas pesadas en maleteros de motocicletas
    if type(toInv) == 'string' and toInv:find('trunk') then
        local item = payload.fromSlot
        if item.name:find('WEAPON_SNIPER') or item.name:find('WEAPON_RPG') then
            TriggerClientEvent('ox_lib:notify', payload.source, {
                type = 'error',
                description = 'Esta arma no cabe en este vehículo.'
            })
            return false -- Cancela el movimiento
        end
    end
    return true
end)
```

### C. Hook para Compras en Tiendas (`buyItem`)
```lua
exports.ox_inventory:registerHook('buyItem', function(payload)
    -- payload: { source, shopType, shopId, toInventory, toSlot, itemName, count, price, currency }
    if payload.shopType == 'blackmarket' then
        -- Validar reputación de facción criminal
        local rep = Player(payload.source).state.gangReputation or 0
        if rep < 50 then
            TriggerClientEvent('ox_lib:notify', payload.source, {
                type = 'error',
                description = 'No tienes suficiente reputación en el bajo mundo.'
            })
            return false
        end
    end
    return true
end)
```

---

## 5. Alijos (Stashes) Personalizados Dinámicos

### Registrar un Alijo en Servidor
```lua
local stashId = 'safe_house_gang_1'
local stashLabel = 'Caja Fuerte - Banda Central'
local slots = 50
local maxWeight = 100000 -- 100 kg
local owner = nil -- nil = compartido; string/identifier = privado
local groups = { ['ballas'] = 2 } -- Rol o trabajo con permiso

exports.ox_inventory:RegisterStash(stashId, stashLabel, slots, maxWeight, owner, groups)
```

### Abrir un Alijo desde el Cliente
```lua
exports.ox_inventory:openInventory('stash', 'safe_house_gang_1')
```

### Alijo Temporal (Instanciado)
```lua
-- Alijo en memoria que se destruye al cerrarse
exports.ox_inventory:RegisterStash('temp_inspect_' .. source, 'Inspección de Evidencia', 20, 20000, source)
exports.ox_inventory:forceOpenInventory(source, 'stash', 'temp_inspect_' .. source)
```

---

## 6. Personalización Visual e Imágenes

### Servir Imágenes Propias
Configura en tu `server.cfg`:
```cfg
setr inventory:imagepath "nui://my_custom_resource/web/images"
```
O dentro de un ítem particular:
```lua
['custom_watch'] = {
    label = 'Reloj de Lujo',
    weight = 100,
    client = {
        image = 'https://r2.fivemanage.com/images/rolex_gold.png'
    }
}
```

### Hosts permitidos para imágenes dinámicas
En `server.cfg`:
```cfg
set inventory:validhosts "{\"r2.fivemanage.com\":true,\"i.imgur.com\":true}"
```
