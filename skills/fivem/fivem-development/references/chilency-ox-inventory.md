# Documentación Técnica: ox_inventory Personalizado (ChilencyRP / Origen / QBCore)

Documentación detallada del fork personalizado de `ox_inventory` (v2.45.0) ubicado en:
`chilencyRP\resources\resources\[qb]\[origen_inventory]\ox_inventory`

Este inventario actúa como un núcleo unificado proporcionando compatibilidad simultánea para **`qb-inventory`**, **`qb_inventory`** y **`origen_inventory`**, e incorporando submódulos nativos para llaves de vehículos en inventario, hornos de fundición (furnace), perfiles de armas avanzados y props de streaming dedicados (`seba_props`).

---

## 1. Identificación y Dependencias

- **Nombre en manifest**: `ox_inventory`
- **Aliases provistos (`provide`)**:
  - `qb-inventory`
  - `qb_inventory`
  - `origen_inventory`
- **Dependencias obligatorias**:
  - `oxmysql`
  - `seba_theme_core` (interfaz y estilos personalizados)
  - `ox_lib`

---

## 2. API de Llaves de Vehículos Físicas (`modules/keys/server.lua`)

El inventario gestiona las llaves de los vehículos como ítems físicos con metadatos de patente (`plate`), integrándose con la guantera del vehículo y verificando posesión en tiempo real.

### Exports del Servidor:

#### `GiveVehicleKeys(targetSource, plate, vehicleLabel?, vehicleId?)` / Alias: `SetOwner`
Entrega una llave física al inventario del jugador y sincroniza su estado de propietario runtime.
- **Parámetros**:
  - `targetSource` *(number)*: ID del jugador en el servidor.
  - `plate` *(string)*: Patente del vehículo (se normaliza automáticamente).
  - `vehicleLabel` *(string, opcional)*: Nombre legible del modelo del vehículo.
  - `vehicleId` *(number/string, opcional)*: ID de la base de datos del vehículo.
- **Retorno**: `boolean` (`true` si se creó la llave y se asignó el acceso).

#### `giveinsidevehicle(targetSource, plate, vehicleLabel?, vehicleId?)`
Asigna llaves directamente cuando el jugador se encuentra dentro del vehículo.

#### `RemoveVehicleKeys(targetSource, plate, vehicleId?)` / Alias: `GarageRemoveKeys`
Elimina la llave física correspondiente a esa patente del inventario del jugador y revoca el acceso en runtime.
- **Retorno**: `boolean`.

#### `HasKeys(source, plate, vehicleId?)`
Verifica si el jugador posee la llave del vehículo (ya sea en su inventario o acceso runtime autorizado).
- **Retorno**: `boolean`.

#### `GetVehicleKeys(source)`
Devuelve una lista/tabla con todas las llaves y patentes válidas que posee el jugador.
- **Retorno**: `table` (ej. `{ ['ABC12345'] = true, ... }`).

#### `GloveboxHasKeyForPlate(plate)`
Comprueba si la guantera del vehículo con dicha patente contiene una llave física para arrancarlo.
- **Retorno**: `boolean`.

#### `GetInventoryKeyCount(source, plate, vehicleId?)`
Retorna la cantidad exacta de copias de llaves que el jugador lleva en sus slots.
- **Retorno**: `number`.

#### `InvalidateVehicleIdCache(plate)`
Limpia la caché interna de mapeo de patente a ID de entidad de vehículo.

---

## 3. API de Fundición / Hornos (`modules/furnace/`)

Sistema nativo de procesamiento y crafteo térmico integrado en la UI del inventario.

### Servidor (`server.lua`):
```lua
exports['ox_inventory']:RegisterFurnaceRecipes(furnaceType, recipes)
```
- `furnaceType` *(string)*: Identificador del tipo de horno (ej. `'iron_furnace'`, `'smelter'`).
- `recipes` *(table)*: Definición de recetas con ítems requeridos, resultado, tiempo de fundición y temperatura.

### Cliente (`client.lua`):
```lua
-- Abrir la interfaz de fundición
exports['ox_inventory']:OpenFurnace(furnaceId, inputSlots, recipes)

-- Cerrar el horno activo
exports['ox_inventory']:CloseFurnace()

-- Obtener la API interna de manipulación de hornos en el cliente
local furnaceAPI = exports['ox_inventory']:GetFurnaceAPI()
```

---

## 4. API de Estadísticas Balísticas de Armas (`modules/weapon/server.lua`)

Proporciona telemetría y datos detallados de daño, alcance, accesorios y perfiles balísticos para la UI y sistemas de daño personalizados:

- `RefreshWeaponStats()`: Recarga el catálogo de configuraciones balísticas.
- `GetWeaponStatsCatalog()`: Retorna el catálogo completo con todas las armas registradas.
- `GetWeaponStatsByCode(weaponCode)`: Estadísticas por código (ej. `'WEAPON_PISTOL'`).
- `GetWeaponStatsByHash(weaponHash)`: Estadísticas por hash nativo de GTA.
- `GetWeaponClipVariants(weaponCode)`: Variantes de cargador (normal, extendido, tambor).
- `GetWeaponDamageProfile(weaponCode)`: Perfil de caída de daño por distancia.
- `GetWeaponTooltipStats(weaponCode, weaponHash, installedComponents)`: Datos formateados para mostrar en el tooltip de la UI.

---

## 5. Capa de Compatibilidad con `origen_inventory` (snake_case)

Permite que scripts diseñados para `origen_inventory` funcionen sin modificaciones:

| Export | Descripción |
|---|---|
| `createShop(shopId, shopData)` | Registra una tienda en formato Origen |
| `registerStash(stashId, label, slots, maxWeight, owner, groups, coords)` | Registra un alijo Origen |
| `addItem(inv, item, count, metadata, slot)` | Añade ítems (compatible con IDs Origen) |
| `removeItem(inv, item, count, metadata, slot, ignoreTotal)` | Remueve ítems |
| `getItem(inv, item, metadata, returnsCount)` | Consulta un ítem |
| `getSlot(inv, slot)` | Consulta un slot específico |
| `getSlotIdWithItem(inv, item, metadata, strict)` | Primer slot con el ítem |
| `getSlotsIdWithItem(inv, item, metadata, strict)` | Todos los slots con el ítem |
| `getInventoryItems(inv)` | Lista de todos los ítems del inventario |
| `getInventory(inv)` | Objeto completo de inventario |
| `getItemCount(inv, item, metadata, strict)` | Cantidad total |
| `setMetadata(inv, slot, metadata)` | Actualiza metadatos de un slot |
| `canCarryItem(inv, item, count, metadata)` | Comprobación de peso y espacio |
| `setMaxWeight(inv, maxWeight)` | Modifica el peso máximo de un inventario |
| `setMaxSlots(inv, slots)` | Modifica los slots máximos |

---

## 6. Capa de Compatibilidad con `qb-inventory` (PascalCase)

Permite ejecutar cualquier script de la comunidad QBCore sin cambiar llamadas:

| Export | Descripción |
|---|---|
| `CreateShop(shopData)` | Crear tienda QBCore |
| `OpenShop(source, name)` | Abrir tienda por nombre |
| `OpenInventory(source, identifier, data)` | Abrir inventario secundario |
| `OpenInventoryById(source, targetPlayerId)` | Abrir inventario de otro jugador (cacheo/registro) |
| `CreateInventory(identifier, data)` | Crear inventario personalizado en memoria |
| `LoadInventory(source, citizenid)` | Carga inicial del inventario del jugador |
| `SaveInventory(source, offline)` | Forzar guardado en MySQL |
| `ClearStash(identifier)` | Vaciar completamente un alijo |
| `CloseInventory(source, identifier)` | Forzar cierre de inventario |
| `HasItem(source, items, amount)` | Comprueba si el jugador tiene los ítems (en cliente `HasItem(items, amount)`) |
| `AddItemMetadata(source, slot, metadataPatch)` | Aplica un parche/merge de metadatos a un slot |
| `RemoveItemMetadata(source, slot, keys)` | Elimina claves específicas de metadatos |
| `GetFreeWeight(source)` | Peso libre restante en gramos |
| `GetSlots(identifier)` | Obtiene tabla de slots |
| `GetItemBySlot(source, slot)` | Ítem en slot específico |
| `GetItemByName(source, itemName)` | Primer ítem que coincida con el nombre |

---

## 7. Props y Streamings Propios (`stream/seba_props`)

El recurso cuenta con props 3D propios definidos en archivos `.ydr` y `.ytyp` (`DLC_ITYP_REQUEST`):
- Bebidas energéticas Prime (`brum_prime_blueraspberry`, `brum_prime_grape`, `brum_prime_icepop`, etc.)
- Cafés y termos Starbucks (`brum_starbuckscan_*`, `brum_slothbucks`)
- Refrescos de lata (`brum_coke`, `brum_pepsi_diet`, `brum_dppepper`, `brum_fanta`, `brum_bang`, `brum_sprite`)
- Imágenes web de producción en formato de alta compresión: `web/images/items/production/**/*.avif`.
