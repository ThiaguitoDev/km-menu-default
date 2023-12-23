# km-menu-default

Menu default to QB-Core

## Instalation

Add to qb-core/client to wrapper.lua file
Add to qb-core to import.lua file

## Add in initial code qb-core/client/functions.lua

```
    QBCore.RequestId = 0
    QBCore.TimeoutCallbacks          = {}

    local isNuiFocused = false

    -- Player

    function QBCore.Functions.GetPlayerData(cb)
        if not cb then return QBCore.PlayerData end
        cb(QBCore.PlayerData)
    end

    QBCore.Table = {}

    QBCore.Math = {}

    QBCore.Math.Round = function(value, numDecimalPlaces)
        if numDecimalPlaces then
            local power = 10^numDecimalPlaces
            return math.floor((value * power) + 0.5) / (power)
        else
            return math.floor(value + 0.5)
        end
    end

    QBCore.Math.Trim = function(value)
        if value then
            return (string.gsub(value, "^%s*(.-)%s*$", "%1"))
        else
            return nil
        end
    end

    -- nil proof alternative to #table
    function QBCore.Table.SizeOf(t)
        local count = 0

        for _,_ in pairs(t) do
            count = count + 1
        end

        return count
    end

    Citizen.CreateThread(function()
        while true do
            local wait = 500
            local currTime = GetGameTimer()
            if #QBCore.TimeoutCallbacks > 0 then
                wait = 0
                for i=1, #QBCore.TimeoutCallbacks, 1 do
                    if QBCore.TimeoutCallbacks[i] then
                        if currTime >= QBCore.TimeoutCallbacks[i].time then
                            QBCore.TimeoutCallbacks[i].cb()
                            QBCore.TimeoutCallbacks[i] = nil
                        end
                    end
                end
            end
            Wait(wait)
        end
    end)


    QBCore.SetTimeout = function(msec, cb)
        table.insert(QBCore.TimeoutCallbacks, {
            time = GetGameTimer() + msec,
            cb   = cb
        })
        return #QBCore.TimeoutCallbacks
    end

    QBCore.ClearTimeout = function(i)
        QBCore.TimeoutCallbacks[i] = nil
    end

    local entityEnumerator = {
        __gc = function(enum)
            if enum.destructor and enum.handle then
                enum.destructor(enum.handle)
            end

            enum.destructor = nil
            enum.handle = nil
        end
    }

    local function EnumerateEntities(initFunc, moveFunc, disposeFunc)
        return coroutine.wrap(function()
            local iter, id = initFunc()
            if not id or id == 0 then
                disposeFunc(iter)
                return
            end

            local enum = {handle = iter, destructor = disposeFunc}
            setmetatable(enum, entityEnumerator)
            local next = true

            repeat
                coroutine.yield(id)
                next, id = moveFunc(iter)
            until not next

            enum.destructor, enum.handle = nil, nil
            disposeFunc(iter)
        end)
    end

    function EnumerateEntitiesWithinDistance(entities, isPlayerEntities, coords, maxDistance)
        local nearbyEntities = {}

        if coords then
            coords = vector3(coords.x, coords.y, coords.z)
        else
            local playerPed = PlayerPedId()
            coords = GetEntityCoords(playerPed)
        end

        for k,entity in pairs(entities) do
            local distance = #(coords - GetEntityCoords(entity))

            if distance <= maxDistance then
                table.insert(nearbyEntities, isPlayerEntities and k or entity)
            end
        end

        return nearbyEntities
    end

    function EnumerateObjects()
        return EnumerateEntities(FindFirstObject, FindNextObject, EndFindObject)
    end

    function EnumeratePeds()
        return EnumerateEntities(FindFirstPed, FindNextPed, EndFindPed)
    end

    function EnumerateVehicles()
        return EnumerateEntities(FindFirstVehicle, FindNextVehicle, EndFindVehicle)
    end

    function EnumeratePickups()
        return EnumerateEntities(FindFirstPickup, FindNextPickup, EndFindPickup)
    end

```

## Add in final code qb-core/client/functions.lua

```
    QBCore.UI                        = {}
    QBCore.UI.Menu                   = {}
    QBCore.UI.Menu.RegisteredTypes   = {}
    QBCore.UI.Menu.Opened            = {}


    QBCore.UI.Menu.RegisterType = function(type, open, close)
        QBCore.UI.Menu.RegisteredTypes[type] = {
            open   = open,
            close  = close
        }
    end

    QBCore.UI.Menu.Open = function(type, namespace, name, data, submit, cancel, change, close)
        local menu = {}

        menu.type      = type
        menu.namespace = namespace
        menu.name      = name
        menu.data      = data
        menu.submit    = submit
        menu.cancel    = cancel
        menu.change    = change

        menu.close = function()

            QBCore.UI.Menu.RegisteredTypes[type].close(namespace, name)

            for i=1, #QBCore.UI.Menu.Opened, 1 do
                if QBCore.UI.Menu.Opened[i] then
                    if QBCore.UI.Menu.Opened[i].type == type and QBCore.UI.Menu.Opened[i].namespace == namespace and QBCore.UI.Menu.Opened[i].name == name then
                        QBCore.UI.Menu.Opened[i] = nil
                    end
                end
            end

            if close then
                close()
            end

        end

        menu.update = function(query, newData)

            for i=1, #menu.data.elements, 1 do
                local match = true

                for k,v in pairs(query) do
                    if menu.data.elements[i][k] ~= v then
                        match = false
                    end
                end

                if match then
                    for k,v in pairs(newData) do
                        menu.data.elements[i][k] = v
                    end
                end
            end

        end

        menu.refresh = function()
            QBCore.UI.Menu.RegisteredTypes[type].open(namespace, name, menu.data)
        end

        menu.setElement = function(i, key, val)
            menu.data.elements[i][key] = val
        end

        menu.setElements = function(newElements)
            menu.data.elements = newElements
        end

        menu.setTitle = function(val)
            menu.data.title = val
        end

        menu.removeElement = function(query)
            for i=1, #menu.data.elements, 1 do
                for k,v in pairs(query) do
                    if menu.data.elements[i] then
                        if menu.data.elements[i][k] == v then
                            table.remove(menu.data.elements, i)
                            break
                        end
                    end

                end
            end
        end

        table.insert(QBCore.UI.Menu.Opened, menu)
        QBCore.UI.Menu.RegisteredTypes[type].open(namespace, name, data)

        return menu
    end

    QBCore.UI.Menu.Close = function(type, namespace, name)
        for i=1, #QBCore.UI.Menu.Opened, 1 do
            if QBCore.UI.Menu.Opened[i] then
                if QBCore.UI.Menu.Opened[i].type == type and QBCore.UI.Menu.Opened[i].namespace == namespace and QBCore.UI.Menu.Opened[i].name == name then
                    QBCore.UI.Menu.Opened[i].close()
                    QBCore.UI.Menu.Opened[i] = nil
                end
            end
        end
    end

    QBCore.UI.Menu.CloseAll = function()
        for i=1, #QBCore.UI.Menu.Opened, 1 do
            if QBCore.UI.Menu.Opened[i] then
                QBCore.UI.Menu.Opened[i].close()
                QBCore.UI.Menu.Opened[i] = nil
            end
        end
    end

    QBCore.UI.Menu.GetOpened = function(type, namespace, name)
        for i=1, #QBCore.UI.Menu.Opened, 1 do
            if QBCore.UI.Menu.Opened[i] then
                if QBCore.UI.Menu.Opened[i].type == type and QBCore.UI.Menu.Opened[i].namespace == namespace and QBCore.UI.Menu.Opened[i].name == name then
                    return QBCore.UI.Menu.Opened[i]
                end
            end
        end
    end

    QBCore.UI.Menu.GetOpenedMenus = function()
        return QBCore.UI.Menu.Opened
    end

    QBCore.UI.Menu.IsOpen = function(type, namespace, name)
        return QBCore.UI.Menu.GetOpened(type, namespace, name) ~= nil
    end
```

## ADD IN FXManifest QB-CORE

```
    shared_scripts {
    'config.lua',
    'import.lua', -- ADD
    }

    client_scripts {
    'client/drawtext.lua',
    'client/wrapper.lua' -- ADD
}
```
