-- RC2 HUD: Auto Nests + Server Hop + Fuse + Auto Spin + Equip Best + Exu Lua Cheia/Sete Chaves + EclipseExu

local function notify(text)
    task.spawn(function()
        for _ = 1, 10 do
            local ok = pcall(function()
                game:GetService("StarterGui"):SetCore("SendNotification", {
                    Title = "RC2",
                    Text = text,
                    Duration = 8,
                })
            end)
            if ok then break end
            task.wait(0.5)
        end
    end)
end

local ALLOWED_IDS = { 103076400317371 }

local function isAllowedGame()
    for _, id in ipairs(ALLOWED_IDS) do
        if game.PlaceId == id or game.GameId == id then
            return true
        end
    end
    return false
end

if not isAllowedGame() then
    notify("Jogo não permitido.")
    return
end

local SCRIPT_URL = "https://raw.githubusercontent.com/cxrloosz/asasassasas/refs/heads/main/README2.md"
local CONFIG_FILE = "rc2_config.json"
local LOCAL_FILE = "rc2_hud.lua"

local FUSE_DELAY = 0.15
local MAX_RUNTIME = 120
local FUSE_NAMES = { "ExuSeteChaves", "PombagiraRainha", "ExuVeludo", "ExuLuaNegra", "ExuEclipse", "ExuLuaCheia" }
local FUSE_LEVELS = { 1, 2, 3, 4 }
local EQUIP_BEST_DELAY = 1

local INSTANCE_ID = tostring(math.random(1, 1e9)) .. "-" .. tostring(os.clock())
local cleaned = false
local gui, dragConn
local connections = {}
local farmToken = 0
local spinToken = 0
local equipBestToken = 0
local luaCheiaToken = 0
local eclipseToken = 0
local fuseTokens = {}

local function cleanup()
    cleaned = true
    farmToken += 1
    spinToken += 1
    equipBestToken += 1
    luaCheiaToken += 1
    eclipseToken += 1
    for k in pairs(fuseTokens) do
        fuseTokens[k] += 1
    end
    for _, c in ipairs(connections) do
        pcall(function() c:Disconnect() end)
    end
    pcall(function()
        if dragConn then dragConn:Disconnect() end
    end)
    pcall(function()
        if gui then gui:Destroy() end
    end)
end

if getgenv then
    if getgenv().RC2_CLEANUP then
        pcall(getgenv().RC2_CLEANUP)
    end
    getgenv().RC2_CLEANUP = cleanup
    getgenv().RC2_ID = INSTANCE_ID
end

local function dead()
    if not cleaned and getgenv and getgenv().RC2_ID ~= INSTANCE_ID then
        cleanup()
    end
    return cleaned
end

repeat task.wait() until game:IsLoaded()

local Players = game:GetService("Players")
repeat task.wait() until Players.LocalPlayer

local LocalPlayer = Players.LocalPlayer
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local UserInputService = game:GetService("UserInputService")

if dead() then return end

------------------------------------------------------------------
-- CONFIG
------------------------------------------------------------------

local config = {
    farm = false,
    spin = false,
    fuse = {},
    equipBest = false,
    exuLuaCheia = false,
    eclipse = false
}

pcall(function()
    if isfile and isfile(CONFIG_FILE) then
        local data = HttpService:JSONDecode(readfile(CONFIG_FILE))
        if type(data) == "table" then
            config.farm = data.farm == true
            config.spin = data.spin == true
            config.equipBest = data.equipBest == true
            config.exuLuaCheia = data.exuLuaCheia == true
            config.eclipse = data.eclipse == true
            config.fuse = type(data.fuse) == "table" and data.fuse or {}
        end
    end
end)

local function saveConfig()
    if dead() then return end
    pcall(function()
        if writefile then
            writefile(CONFIG_FILE, HttpService:JSONEncode(config))
        end
    end)
end

------------------------------------------------------------------
-- QUEUE ON TELEPORT
------------------------------------------------------------------

local qf = queue_on_teleport or queueonteleport
    or (syn and syn.queue_on_teleport)
    or (fluxus and fluxus.queue_on_teleport)

local function queueScript()
    if not qf then return end

    local ids = {}
    for _, id in ipairs(ALLOWED_IDS) do
        table.insert(ids, tostring(id))
    end

    local code = ([==[
        local allowed = false
        for _, id in ipairs({%s}) do
            if game.PlaceId == id or game.GameId == id then
                allowed = true
            end
        end
        if not allowed then return end
        repeat task.wait() until game:IsLoaded()
        task.wait(2)
        local src
        pcall(function()
            if isfile and isfile("%s") then
                src = readfile("%s")
            end
        end)
        if not src or #src < 200 then
            local ok, res = pcall(function()
                return game:HttpGet("%s")
            end)
            if ok then src = res end
        end
        local fn, err = loadstring(src or "")
        if not fn then
            warn("[RC2] Erro: " .. tostring(err))
            return
        end
        fn()
    ]==]):format(table.concat(ids, ","), LOCAL_FILE, LOCAL_FILE, SCRIPT_URL)

    pcall(qf, code)
end

if qf then
    queueScript()
else
    warn("[RC2] queue_on_teleport indisponível.")
end

------------------------------------------------------------------
-- STATUS
------------------------------------------------------------------

local statusLabel

local function setStatus(text)
    if statusLabel and statusLabel.Parent then
        statusLabel.Text = "Status: " .. tostring(text)
    end
end

------------------------------------------------------------------
-- ECLIPSEEXU (PRIORIDADE)
------------------------------------------------------------------

-- Acha todos os EclipseExu que existem agora no mapa
local function getEclipseTargets()
    local targets = {}
    local realm = Workspace:FindFirstChild("EclipseRealm", true)
    if not realm then return targets end

    if realm.Name == "EclipseExu" and (realm:IsA("Model") or realm:IsA("BasePart")) then
        table.insert(targets, realm)
    end

    for _, obj in ipairs(realm:GetDescendants()) do
        if obj.Name == "EclipseExu" and (obj:IsA("Model") or obj:IsA("BasePart")) then
            table.insert(targets, obj)
        end
    end

    return targets
end

-- Enquanto o toggle do EclipseExu estiver ligado e existir algum no mapa,
-- as outras rotinas (Nests, Lua Cheia, troca de servidor) esperam.
local function waitEclipseClear(aliveFn, onWait)
    while aliveFn() and config.eclipse and #getEclipseTargets() > 0 do
        setStatus("Aguardando EclipseExu sumirem...")
        if onWait then onWait() end
        task.wait(1)
    end
end

------------------------------------------------------------------
-- SERVIDORES VISITADOS (não reentrar no mesmo servidor)
------------------------------------------------------------------

local VISITED_FILE = "rc2_visited.json"
local VISITED_TTL = 600    -- segundos que um servidor visitado fica bloqueado (10 min)
local VISITED_MAX = 300    -- limite de servidores guardados

local visited = {}

local function pruneVisited()
    local now = os.time()
    local count = 0

    for id, t in pairs(visited) do
        if type(t) ~= "number" or now - t > VISITED_TTL then
            visited[id] = nil
        else
            count += 1
        end
    end

    if count > VISITED_MAX then
        local arr = {}
        for id, t in pairs(visited) do
            table.insert(arr, { id = id, t = t })
        end
        table.sort(arr, function(x, y) return x.t > y.t end)
        visited = {}
        for i = 1, VISITED_MAX do
            visited[arr[i].id] = arr[i].t
        end
    end
end

local function saveVisited()
    pcall(function()
        if writefile then
            writefile(VISITED_FILE, HttpService:JSONEncode(visited))
        end
    end)
    pcall(function()
        if getgenv then getgenv().RC2_VisitedServers = visited end
    end)
end

local function loadVisited()
    visited = {}

    -- arquivo (sobrevive ao teleporte em qualquer executor)
    pcall(function()
        if isfile and isfile(VISITED_FILE) then
            local data = HttpService:JSONDecode(readfile(VISITED_FILE))
            if type(data) == "table" then
                for id, t in pairs(data) do
                    if type(id) == "string" and type(t) == "number" then
                        visited[id] = t
                    end
                end
            end
        end
    end)

    -- memória do executor (se existir)
    pcall(function()
        if getgenv and type(getgenv().RC2_VisitedServers) == "table" then
            for id, t in pairs(getgenv().RC2_VisitedServers) do
                if type(id) == "string" then
                    local n = type(t) == "number" and t or os.time()
                    if not visited[id] or n > visited[id] then
                        visited[id] = n
                    end
                end
            end
        end
    end)

    pruneVisited()
end

local function markVisited(id)
    if not id or id == "" then return end
    visited[id] = os.time()
    pruneVisited()
    saveVisited()
end

local function isVisited(id)
    local t = visited[id]
    return t ~= nil and os.time() - t <= VISITED_TTL
end

-- Limpa o histórico (o servidor atual continua marcado)
local function resetVisited()
    visited = {}
    markVisited(game.JobId)
end

loadVisited()
markVisited(game.JobId) -- o servidor atual já conta como visitado

------------------------------------------------------------------
-- SERVER HOP
------------------------------------------------------------------

local hopping = false

local function hopEnabled()
    return config.farm or config.exuLuaCheia
end

local VISITED_RESET_AFTER = 2  -- buscas sem servidor novo até resetar o histórico
local FALLBACK_AFTER = 8        -- buscas vazias até deixar o Roblox escolher (math.huge = nunca)

local function fetchCandidates(allowNearFull)
    local candidates, failed = {}, false

    for _, sortOrder in ipairs({ "Asc", "Desc" }) do
        local cursor = ""

        for _ = 1, 3 do
            local ok, result = pcall(function()
                local url = "https://games.roblox.com/v1/games/" .. game.PlaceId
                    .. "/servers/Public?sortOrder=" .. sortOrder
                    .. "&excludeFullGames=true&limit=100"
                    .. (cursor ~= "" and ("&cursor=" .. cursor) or "")
                return HttpService:JSONDecode(game:HttpGet(url))
            end)

            if not ok or type(result) ~= "table" or type(result.data) ~= "table" then
                failed = true
                break
            end

            for _, s in ipairs(result.data) do
                local limit = allowNearFull and s.maxPlayers or (s.maxPlayers - 1)
                if s.id ~= game.JobId
                    and not isVisited(s.id)
                    and s.playing < limit then
                    table.insert(candidates, s.id)
                end
            end

            cursor = result.nextPageCursor or ""
            if cursor == "" or #candidates >= 15 then break end
        end

        -- já tem opções (ou a API falhou): não faz mais requisições
        if #candidates >= 15 or failed then break end
    end

    return candidates, failed
end

local function serverHop()
    if hopping or dead() or not hopEnabled() then return end
    hopping = true

    -- EclipseExu tem prioridade: não sai enquanto existir algum
    waitEclipseClear(function() return not dead() and hopEnabled() end)

    local chosen
    local useFallback = false
    local attempts, emptyStreak, failStreak = 0, 0, 0

    while not dead() and hopEnabled() do
        attempts += 1
        setStatus(("Procurando servidor novo... (%d)"):format(attempts))

        -- depois do primeiro reset, aceita também servidores quase cheios
        local candidates, failed = fetchCandidates(emptyStreak >= VISITED_RESET_AFTER)

        if #candidates > 0 then
            chosen = candidates[math.random(#candidates)]
            break
        end

        if failed then
            -- API limitando as requisições: espera mais a cada falha seguida
            failStreak += 1
            local waitTime = math.min(10 * failStreak, 40)
            setStatus(("API limitada, aguardando %ds... (%d)"):format(waitTime, attempts))
            task.wait(waitTime)
        else
            failStreak = 0
            emptyStreak += 1

            -- nada de novo: reseta o histórico de visitados
            if emptyStreak % VISITED_RESET_AFTER == 0 then
                warn("[RC2] Sem servidores novos, resetando servidores visitados.")
                resetVisited()
            end

            -- último recurso: deixa o Roblox escolher o servidor
            if emptyStreak >= FALLBACK_AFTER then
                useFallback = true
                break
            end

            task.wait(3)
        end
    end

    if dead() or not hopEnabled() or (not chosen and not useFallback) then
        hopping = false
        return
    end

    queueScript()
    setStatus("Trocando de servidor...")

    if chosen then
        markVisited(chosen)
        pcall(function()
            TeleportService:TeleportToPlaceInstance(game.PlaceId, chosen, LocalPlayer)
        end)
    else
        pcall(function()
            TeleportService:Teleport(game.PlaceId, LocalPlayer)
        end)
    end

    -- se o teleporte não acontecer, libera para tentar de novo
    task.delay(25, function()
        hopping = false
    end)
end

table.insert(connections, TeleportService.TeleportInitFailed:Connect(function()
    hopping = false
    task.wait(3)
    serverHop()
end))

------------------------------------------------------------------
-- AUTO NESTS
------------------------------------------------------------------

local RETURN_CFRAME = CFrame.new(
    20, 5.83, -28,
    0, 0, -1,
    1, 0, 0,
    0, -1, 0
)

local promptConnected = false

local function fixPrompt(obj)
    if obj:IsA("ProximityPrompt") then
        obj.HoldDuration = 0
        obj.MaxActivationDistance = 9999
    end
end

local function enablePrompts()
    for _, obj in ipairs(Workspace:GetDescendants()) do
        fixPrompt(obj)
    end

    if not promptConnected then
        promptConnected = true
        table.insert(connections,
            Workspace.DescendantAdded:Connect(fixPrompt)
        )
    end
end

local function getObjectCFrame(obj)
    if obj:IsA("BasePart") then
        return obj.CFrame
    end
    if obj:IsA("Model") then
        return obj.PrimaryPart and obj.PrimaryPart.CFrame or obj:GetPivot()
    end
end

local function forceInteract(target)
    local prompt = target:FindFirstChildWhichIsA("ProximityPrompt", true)

    if prompt then
        fixPrompt(prompt)
        pcall(function()
            if fireproximityprompt then
                fireproximityprompt(prompt)
            end
        end)
        pcall(function()
            prompt:InputHoldBegin()
            task.wait(0.05)
            prompt:InputHoldEnd()
        end)
    end

    task.wait(0.1)
    VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.E, false, game)
    task.wait(0.2)
    VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.E, false, game)
end

local function getRoot()
    local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    return char:WaitForChild("HumanoidRootPart", 10)
end

local function runFarm()
    farmToken += 1
    local my = farmToken

    local function alive()
        return config.farm and my == farmToken and not dead()
    end

    task.delay(MAX_RUNTIME, function()
        if alive() then
            setStatus("Tempo limite, trocando...")
            serverHop()
        end
    end)

    task.spawn(function()
        setStatus("Carregando mapa...")
        enablePrompts()
        getRoot()

        local terreiro8
        local start = tick()

        while alive() and not terreiro8 and tick() - start < 30 do
            terreiro8 = Workspace:FindFirstChild("Terreiro8", true)
            if not terreiro8 then task.wait(0.5) end
        end

        if not alive() then return end

        local nests = terreiro8 and terreiro8:FindFirstChild("Nests")
        if not nests then
            warn("[RC2] Terreiro8/Nests não encontrado.")
            return serverHop()
        end

        local list = nests:GetChildren()

        for i, nest in ipairs(list) do
            if not alive() then return end

            -- EclipseExu tem prioridade
            waitEclipseClear(alive)
            if not alive() then return end

            local cf = getObjectCFrame(nest)
            if cf then
                setStatus(("Nest %d/%d"):format(i, #list))

                local root = getRoot()
                if root then
                    root.CFrame = cf + Vector3.new(0, 1.5, 0)
                end

                task.wait(0.3)
                forceInteract(nest)
                task.wait(0.2)

                root = getRoot()
                if root then root.CFrame = RETURN_CFRAME end

                task.wait(0.6)
            end
        end

        if not alive() then return end

        setStatus("Finalizado, trocando...")
        task.wait(1)
        serverHop()
    end)
end

local function setFarm(on)
    config.farm = on
    saveConfig()

    if on then
        runFarm()
    else
        farmToken += 1
        setStatus("Parado")
    end
end

------------------------------------------------------------------
-- AUTO EXU LUA CHEIA + SETE CHAVES (mesmo toggle)
------------------------------------------------------------------

local EXU_TARGET_NAMES = {
    Exu_ExuLuaCheia = true,
    Exu_ExuSeteChaves = true,
}

local function findExuTargets()
    local found = {}

    local function isTarget(obj)
        return EXU_TARGET_NAMES[obj.Name] and (obj:IsA("Model") or obj:IsA("BasePart"))
    end

    local runtime = Workspace:FindFirstChild("Runtime")
    if runtime then
        for _, obj in ipairs(runtime:GetChildren()) do
            if isTarget(obj) then table.insert(found, obj) end
        end
    end

    if #found == 0 then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if isTarget(obj) then table.insert(found, obj) end
        end
    end

    return found
end

local function runExuLuaCheia()
    luaCheiaToken += 1
    local my = luaCheiaToken

    local function alive()
        return config.exuLuaCheia and my == luaCheiaToken and not dead()
    end

    -- Failsafe: o prazo é renovado enquanto espera os EclipseExu
    local deadline = tick() + MAX_RUNTIME
    local function extendDeadline()
        deadline = tick() + MAX_RUNTIME
    end

    task.spawn(function()
        while alive() do
            task.wait(1)
            if tick() > deadline then
                setStatus("Tempo limite, trocando...")
                serverHop()
                return
            end
        end
    end)

    task.spawn(function()
        enablePrompts()
        getRoot()

        -- EclipseExu tem prioridade: só continua quando todos sumirem
        waitEclipseClear(alive, extendDeadline)
        if not alive() then return end

        setStatus("Procurando Lua Cheia / Sete Chaves (4s)...")

        local targets = {}
        local start = tick()

        while alive() and #targets == 0 and (tick() - start < 4) do
            targets = findExuTargets()
            if #targets == 0 then task.wait(0.2) end
        end

        if not alive() then return end

        if #targets == 0 then
            setStatus("Não encontrado (4s), trocando servidor...")
            task.wait(0.5)
            return serverHop()
        end

        for i, obj in ipairs(targets) do
            if not alive() then return end

            -- se um EclipseExu aparecer no meio do caminho, espera ele sumir
            waitEclipseClear(alive, extendDeadline)
            if not alive() then return end

            if obj:IsDescendantOf(Workspace) then
                local cf = getObjectCFrame(obj)
                if cf then
                    setStatus(("Capturando %s (%d/%d)..."):format(obj.Name, i, #targets))

                    local root = getRoot()
                    if root then
                        root.CFrame = cf + Vector3.new(0, 1.5, 0)
                    end

                    task.wait(0.3)
                    forceInteract(obj)
                    task.wait(0.2)

                    root = getRoot()
                    if root then root.CFrame = RETURN_CFRAME end

                    task.wait(0.6)
                end
            end
        end

        if not alive() then return end

        setStatus("Exus processados, trocando...")
        task.wait(0.5)
        serverHop()
    end)
end

local function setExuLuaCheia(on)
    config.exuLuaCheia = on
    saveConfig()

    if on then
        runExuLuaCheia()
    else
        luaCheiaToken += 1
        setStatus("Parado")
    end
end

------------------------------------------------------------------
-- AUTO ECLIPSEEXU
------------------------------------------------------------------

local SAFE_CFRAME = CFrame.new(
    2, 900.280029, -5182,
    0, -1, 0,
    1, 0, 0,
    0, 0, 1
)

local function runEclipse()
    eclipseToken += 1
    local my = eclipseToken

    local function alive()
        return config.eclipse and my == eclipseToken and not dead()
    end

    task.spawn(function()
        enablePrompts()

        while alive() do
            local targets = getEclipseTargets()

            if #targets == 0 then
                task.wait(1)
            else
                for i, target in ipairs(targets) do
                    if not alive() then return end

                    if target:IsDescendantOf(Workspace) then
                        local cf = getObjectCFrame(target)
                        if cf then
                            setStatus(("EclipseExu %d/%d"):format(i, #targets))

                            local root = getRoot()
                            if root then
                                root.CFrame = cf + Vector3.new(0, 1.5, 0)
                            end

                            task.wait(0.4)
                            forceInteract(target)
                            task.wait(0.2)

                            root = getRoot()
                            if root then root.CFrame = SAFE_CFRAME end

                            task.wait(0.5)
                        end
                    end
                end

                task.wait(0.3)
            end
        end
    end)
end

local function setEclipse(on)
    config.eclipse = on
    saveConfig()

    if on then
        runEclipse()
    else
        eclipseToken += 1
        setStatus("EclipseExu desligado")
    end
end

------------------------------------------------------------------
-- FUSE
------------------------------------------------------------------

local function setFuse(name, level, on)
    local key = name .. ":" .. level
    config.fuse[key] = on or nil
    fuseTokens[key] = (fuseTokens[key] or 0) + 1
    saveConfig()

    if not on then return end

    local my = fuseTokens[key]

    task.spawn(function()
        local remotes = ReplicatedStorage:WaitForChild("Remotes", 15)
        local fuse = remotes and remotes:WaitForChild("Fuse", 15)

        if not fuse then
            warn("[RC2] Remote Fuse não encontrado!")
            return
        end

        while not dead()
            and fuseTokens[key] == my
            and config.fuse[key] do

            pcall(function()
                fuse:InvokeServer(name, level)
            end)

            task.wait(FUSE_DELAY)
        end
    end)
end

------------------------------------------------------------------
-- AUTO SPIN
------------------------------------------------------------------

local SPIN_DELAY = 0

local function setSpin(on)
    config.spin = on and true or false
    spinToken += 1
    saveConfig()

    if not on then return end

    local my = spinToken

    task.spawn(function()
        local remotes = ReplicatedStorage:WaitForChild("Remotes", 15)
        local spin = remotes and remotes:WaitForChild("Spin", 15)

        if not spin then
            warn("[RC2] Remote Spin não encontrado!")
            notify("Remote Spin não encontrado!")
            return
        end

        while not dead()
            and spinToken == my
            and config.spin do

            pcall(function()
                spin:InvokeServer()
            end)

            task.wait(SPIN_DELAY)
        end
    end)
end

------------------------------------------------------------------
-- AUTO EQUIP BEST
------------------------------------------------------------------

local function setEquipBest(on)
    config.equipBest = on and true or false
    equipBestToken += 1
    saveConfig()

    if not on then return end

    local my = equipBestToken

    task.spawn(function()
        local remotes = ReplicatedStorage:WaitForChild("Remotes", 15)
        local shop = remotes and remotes:WaitForChild("Shop", 15)

        if not shop then
            warn("[RC2] Remote Shop não encontrado!")
            notify("Remote Shop não encontrado!")
            return
        end

        while not dead()
            and equipBestToken == my
            and config.equipBest do

            pcall(function()
                shop:InvokeServer("equipBest")
            end)

            task.wait(EQUIP_BEST_DELAY)
        end
    end)
end

------------------------------------------------------------------
-- HUD
------------------------------------------------------------------

local ON_COLOR = Color3.fromRGB(46, 160, 90)
local OFF_COLOR = Color3.fromRGB(58, 58, 68)
local BG_COLOR = Color3.fromRGB(24, 24, 30)
local TOP_COLOR = Color3.fromRGB(36, 36, 46)

local toggles = {}
local hudHidden = false

local function destroyOldGuis()
    local parents = {}

    pcall(function()
        if gethui then table.insert(parents, gethui()) end
    end)

    pcall(function()
        table.insert(parents, game:GetService("CoreGui"))
    end)

    pcall(function()
        table.insert(parents, LocalPlayer:FindFirstChild("PlayerGui"))
    end)

    for _, p in ipairs(parents) do
        pcall(function()
            for _, child in ipairs(p:GetChildren()) do
                if child.Name == "RC2_HUD" then
                    child:Destroy()
                end
            end
        end)
    end
end

local function attachGui(g)
    pcall(function()
        if syn and syn.protect_gui then syn.protect_gui(g) end
    end)

    local ok = pcall(function()
        g.Parent = gethui()
    end)
    if ok and g.Parent then return "gethui" end

    ok = pcall(function()
        g.Parent = game:GetService("CoreGui")
    end)
    if ok and g.Parent then return "CoreGui" end

    g.Parent = LocalPlayer:WaitForChild("PlayerGui")
    return "PlayerGui"
end

local function buildGui()
    destroyOldGuis()
    toggles = {}

    local order = 0
    local function nextOrder()
        order += 1
        return order
    end

    if dragConn then
        pcall(function() dragConn:Disconnect() end)
    end

    gui = Instance.new("ScreenGui")
    gui.Name = "RC2_HUD"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.DisplayOrder = 2147483647
    gui.Enabled = not hudHidden

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 240, 0, 0)
    frame.AutomaticSize = Enum.AutomaticSize.Y
    frame.Position = UDim2.new(0, 20, 0.3, 0)
    frame.BackgroundColor3 = BG_COLOR
    frame.BorderSizePixel = 0
    frame.Parent = gui
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)

    local frameLayout = Instance.new("UIListLayout", frame)
    frameLayout.SortOrder = Enum.SortOrder.LayoutOrder

    local title = Instance.new("Frame")
    title.Size = UDim2.new(1, 0, 0, 30)
    title.BackgroundColor3 = TOP_COLOR
    title.BorderSizePixel = 0
    title.LayoutOrder = 1
    title.Parent = frame
    Instance.new("UICorner", title).CornerRadius = UDim.new(0, 8)

    local titleText = Instance.new("TextLabel")
    titleText.Size = UDim2.new(1, -40, 1, 0)
    titleText.Position = UDim2.new(0, 10, 0, 0)
    titleText.BackgroundTransparency = 1
    titleText.Text = "Carlinhos Exuzete"
    titleText.TextXAlignment = Enum.TextXAlignment.Left
    titleText.Font = Enum.Font.GothamBold
    titleText.TextSize = 14
    titleText.TextColor3 = Color3.new(1, 1, 1)
    titleText.Parent = title

    local minBtn = Instance.new("TextButton")
    minBtn.Size = UDim2.new(0, 28, 0, 22)
    minBtn.Position = UDim2.new(1, -34, 0, 4)
    minBtn.BackgroundColor3 = OFF_COLOR
    minBtn.Text = "-"
    minBtn.Font = Enum.Font.GothamBold
    minBtn.TextSize = 16
    minBtn.TextColor3 = Color3.new(1, 1, 1)
    minBtn.Parent = title
    Instance.new("UICorner", minBtn).CornerRadius = UDim.new(0, 5)

    local content = Instance.new("ScrollingFrame")
    content.Size = UDim2.new(1, 0, 0, 340)
    content.CanvasSize = UDim2.new(0, 0, 0, 0)
    content.AutomaticCanvasSize = Enum.AutomaticSize.Y
    content.ScrollBarThickness = 4
    content.ScrollingDirection = Enum.ScrollingDirection.Y
    content.BorderSizePixel = 0
    content.BackgroundTransparency = 1
    content.LayoutOrder = 2
    content.Parent = frame

    local pad = Instance.new("UIPadding", content)
    pad.PaddingTop = UDim.new(0, 8)
    pad.PaddingBottom = UDim.new(0, 10)
    pad.PaddingLeft = UDim.new(0, 10)
    pad.PaddingRight = UDim.new(0, 10)

    local list = Instance.new("UIListLayout", content)
    list.Padding = UDim.new(0, 6)
    list.SortOrder = Enum.SortOrder.LayoutOrder

    minBtn.MouseButton1Click:Connect(function()
        content.Visible = not content.Visible
        minBtn.Text = content.Visible and "-" or "+"
    end)

    local dragging, dragStart, startPos

    title.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = true
            dragStart = input.Position
            startPos = frame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)

    dragConn = UserInputService.InputChanged:Connect(function(input)
        if dragging and (
            input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch
        ) then
            local d = input.Position - dragStart
            frame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + d.X,
                startPos.Y.Scale, startPos.Y.Offset + d.Y
            )
        end
    end)

    local function makeLabel(text, size)
        local l = Instance.new("TextLabel")
        l.Size = UDim2.new(1, 0, 0, size or 18)
        l.BackgroundTransparency = 1
        l.Text = text
        l.TextXAlignment = Enum.TextXAlignment.Left
        l.Font = Enum.Font.GothamMedium
        l.TextSize = 12
        l.TextColor3 = Color3.fromRGB(180, 180, 195)
        l.LayoutOrder = nextOrder()
        l.Parent = content
        return l
    end

    local function makeToggle(parent, text, initial, onChange, compact)
        local btn = Instance.new("TextButton")
        btn.AutoButtonColor = false
        btn.Font = Enum.Font.GothamMedium
        btn.TextSize = 13
        btn.TextColor3 = Color3.new(1, 1, 1)

        if compact then
            btn.Size = UDim2.new(0.25, -3, 1, 0)
        else
            btn.Size = UDim2.new(1, 0, 0, 28)
            btn.LayoutOrder = nextOrder()
            btn.TextScaled = true
            local sizeLimit = Instance.new("UITextSizeConstraint")
            sizeLimit.MaxTextSize = 13
            sizeLimit.Parent = btn
        end

        btn.Parent = parent
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

        local state = initial and true or false

        local function render()
            btn.BackgroundColor3 = state and ON_COLOR or OFF_COLOR
            btn.Text = compact and text
                or (text .. (state and "  [ON]" or "  [OFF]"))
        end

        local t = {}

        function t.Set(v, silent)
            state = v and true or false
            render()
            if not silent then onChange(state) end
        end

        btn.MouseButton1Click:Connect(function()
            t.Set(not state)
        end)

        render()
        table.insert(toggles, t)
        return t
    end

    -- Auto Farm
    makeLabel("AUTO FARM")
    makeToggle(content, "Auto Nests + Hop", config.farm, setFarm)
    makeToggle(content, "Auto Lua Cheia + Sete Chaves", config.exuLuaCheia, setExuLuaCheia)
    makeToggle(content, "Auto EclipseExu (prioridade)", config.eclipse, setEclipse)
    statusLabel = makeLabel(
        (config.farm or config.exuLuaCheia or config.eclipse) and "Status: Ligado" or "Status: Parado"
    )

    local resetBtn = Instance.new("TextButton")
    resetBtn.Size = UDim2.new(1, 0, 0, 24)
    resetBtn.LayoutOrder = nextOrder()
    resetBtn.BackgroundColor3 = OFF_COLOR
    resetBtn.Text = "Resetar servidores visitados"
    resetBtn.Font = Enum.Font.GothamMedium
    resetBtn.TextSize = 12
    resetBtn.TextColor3 = Color3.new(1, 1, 1)
    resetBtn.Parent = content
    Instance.new("UICorner", resetBtn).CornerRadius = UDim.new(0, 6)
    resetBtn.MouseButton1Click:Connect(function()
        resetVisited()
        notify("Servidores visitados resetados")
    end)

    -- Auto Spin
    makeLabel("SPIN")
    makeToggle(content, "Auto Spin", config.spin, setSpin)

    -- Equip Best
    makeLabel("SHOP")
    makeToggle(content, "Equip Best", config.equipBest, setEquipBest)

    -- Fuse
    makeLabel("FUSE (níveis 1 a 4)")

    for _, name in ipairs(FUSE_NAMES) do
        makeLabel(name)

        local row = Instance.new("Frame")
        row.Size = UDim2.new(1, 0, 0, 26)
        row.BackgroundTransparency = 1
        row.LayoutOrder = nextOrder()
        row.Parent = content

        local rowLayout = Instance.new("UIListLayout", row)
        rowLayout.FillDirection = Enum.FillDirection.Horizontal
        rowLayout.Padding = UDim.new(0, 4)

        for _, level in ipairs(FUSE_LEVELS) do
            local key = name .. ":" .. level

            makeToggle(
                row,
                tostring(level),
                config.fuse[key] == true,
                function(on)
                    setFuse(name, level, on)
                end,
                true
            )
        end
    end

    makeLabel("END = desligar tudo | INSERT = esconder HUD")

    local where = attachGui(gui)
    print("[RC2] HUD criado em: " .. where)
    notify("HUD criado em: " .. where .. " (INSERT esconde/mostra)")
end

------------------------------------------------------------------
-- ATALHOS
------------------------------------------------------------------

local function stopAll()
    for _, t in ipairs(toggles) do
        t.Set(false)
    end
    warn("[RC2] Tudo desligado.")
end

table.insert(connections, UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end

    if input.KeyCode == Enum.KeyCode.End then
        stopAll()
    elseif input.KeyCode == Enum.KeyCode.Insert then
        hudHidden = not hudHidden
        if gui then gui.Enabled = not hudHidden end
    end
end))

------------------------------------------------------------------
-- INICIALIZAÇÃO
------------------------------------------------------------------

buildGui()

task.spawn(function()
    while true do
        task.wait(1)
        if dead() then return end

        if not gui or not gui.Parent then
            warn("[RC2] HUD sumiu, recriando...")
            buildGui()
        end
    end
end)

if config.eclipse then
    runEclipse()
end

if config.farm then
    runFarm()
end

if config.exuLuaCheia then
    runExuLuaCheia()
end

for _, name in ipairs(FUSE_NAMES) do
    for _, level in ipairs(FUSE_LEVELS) do
        if config.fuse[name .. ":" .. level] then
            setFuse(name, level, true)
        end
    end
end

if config.spin then
    setSpin(true)
end

if config.equipBest then
    setEquipBest(true)
end
