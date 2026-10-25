-- RC2 HUD: Auto Nests + Hop, Spin, Equip Best, Rebirth, Fuse

local ALLOWED_IDS = { 103076400317371 }
local SCRIPT_URL = "https://raw.githubusercontent.com/cxrloosz/asasassasas/refs/heads/main/README4.md"
local CONFIG_FILE = "rc2_config.json"
local LOCAL_FILE = "rc2_hud.lua"

local FUSE_NAMES = { "PombagiraHolograma", "ExuNeon", "ExuDallas" }
local FUSE_LEVELS = { 1, 2, 3, 4 }
local FUSE_DELAY = 0.15
local EQUIP_BEST_DELAY = 1
local MAX_RUNTIME = 120
local MAX_ATTEMPTS = 3 -- buscas de servidor antes de deixar o Roblox escolher
local TERREIRO_NAMES = { "Terreiro9" } -- {} = todos os terreiros
local RETURN_CFRAME = CFrame.new(20, 5.83, -28, 0, 0, -1, 1, 0, 0, 0, -1, 0)

local function notify(text)
    task.spawn(function()
        for _ = 1, 10 do
            local ok = pcall(function()
                game:GetService("StarterGui"):SetCore("SendNotification", {
                    Title = "RC2", Text = text, Duration = 8,
                })
            end)
            if ok then break end
            task.wait(0.5)
        end
    end)
end

local function isAllowedGame()
    for _, id in ipairs(ALLOWED_IDS) do
        if game.PlaceId == id or game.GameId == id then return true end
    end
    return false
end

if not isAllowedGame() then
    notify("Jogo não permitido.")
    return
end

------------------------------------------------------------------
-- Instância única (a mais nova encerra a anterior)
------------------------------------------------------------------

local INSTANCE_ID = tostring(math.random(1, 1e9)) .. tostring(os.clock())
local cleaned, gui, dragConn = false, nil, nil
local connections, tokens = {}, {}

local function bump(id)
    tokens[id] = (tokens[id] or 0) + 1
    return tokens[id]
end

local function cleanup()
    cleaned = true
    for id in pairs(tokens) do bump(id) end
    for _, c in ipairs(connections) do pcall(function() c:Disconnect() end) end
    pcall(function() if dragConn then dragConn:Disconnect() end end)
    pcall(function() if gui then gui:Destroy() end end)
end

if getgenv then
    local env = getgenv()
    if env.RC2_CLEANUP then pcall(env.RC2_CLEANUP) end
    env.RC2_CLEANUP, env.RC2_ID = cleanup, INSTANCE_ID
end

local function dead()
    if not cleaned and getgenv and getgenv().RC2_ID ~= INSTANCE_ID then cleanup() end
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
-- Config (salva em arquivo, sobrevive ao server hop)
------------------------------------------------------------------

local BOOL_KEYS = { "farm", "spin", "equipBest", "rebirth" }
local config = { fuse = {} }
for _, k in ipairs(BOOL_KEYS) do config[k] = false end

pcall(function()
    if isfile and isfile(CONFIG_FILE) then
        local data = HttpService:JSONDecode(readfile(CONFIG_FILE))
        if type(data) == "table" then
            for _, k in ipairs(BOOL_KEYS) do config[k] = data[k] == true end
            if type(data.fuse) == "table" then config.fuse = data.fuse end
        end
    end
end)

local function saveConfig()
    if dead() then return end
    pcall(function()
        if writefile then writefile(CONFIG_FILE, HttpService:JSONEncode(config)) end
    end)
end

------------------------------------------------------------------
-- Recarrega o script depois de cada teleporte
------------------------------------------------------------------

local qf = queue_on_teleport or queueonteleport
    or (syn and syn.queue_on_teleport) or (fluxus and fluxus.queue_on_teleport)

local function queueScript()
    if not qf then return end

    local ids = {}
    for _, id in ipairs(ALLOWED_IDS) do table.insert(ids, string.format("%d", id)) end

    pcall(qf, ([==[
        local allowed = false
        for _, id in ipairs({%s}) do
            if game.PlaceId == id or game.GameId == id then allowed = true end
        end
        if not allowed then return end
        repeat task.wait() until game:IsLoaded()
        task.wait(2)
        local src
        pcall(function()
            if isfile and isfile("%s") then src = readfile("%s") end
        end)
        if not src or #src < 200 then
            local ok, res = pcall(function() return game:HttpGet("%s") end)
            if ok then src = res end
        end
        local fn, err = loadstring(src or "")
        if fn then fn() else warn("[RC2] Erro: " .. tostring(err)) end
    ]==]):format(table.concat(ids, ","), LOCAL_FILE, LOCAL_FILE, SCRIPT_URL))
end

if qf then queueScript() else warn("[RC2] queue_on_teleport indisponível.") end

------------------------------------------------------------------
-- Status
------------------------------------------------------------------

local statusLabel

local function setStatus(text)
    if statusLabel and statusLabel.Parent then
        statusLabel.Text = "Status: " .. tostring(text)
    end
end

------------------------------------------------------------------
-- Server hop
------------------------------------------------------------------

local hopping = false

-- Servidores públicos com vaga (menos o atual). Retorna nil se a API falhar.
local function fetchServers()
    local ok, result = pcall(function()
        local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=%s&excludeFullGames=true&limit=100")
            :format(game.PlaceId, math.random(2) == 1 and "Asc" or "Desc")
        return HttpService:JSONDecode(game:HttpGet(url))
    end)
    if not ok or type(result) ~= "table" or type(result.data) ~= "table" then return nil end

    local ids = {}
    for _, s in ipairs(result.data) do
        if s.id ~= game.JobId and s.playing < s.maxPlayers - 1 then
            table.insert(ids, s.id)
        end
    end
    return ids
end

local function serverHop()
    if hopping or dead() or not config.farm then return end
    hopping = true

    local chosen
    for attempt = 1, MAX_ATTEMPTS do
        setStatus(("Procurando servidor... (%d)"):format(attempt))
        local ids = fetchServers()

        if ids and #ids > 0 then
            chosen = ids[math.random(#ids)]
            break
        end

        if attempt < MAX_ATTEMPTS then
            local delay = ids and 3 or 5 * attempt
            if not ids then
                setStatus(("API limitada, aguardando %ds... (%d)"):format(delay, attempt))
            end
            task.wait(delay)
        end
    end

    if dead() or not config.farm then
        hopping = false
        return
    end

    queueScript()
    setStatus("Trocando de servidor...")

    pcall(function()
        if chosen then
            TeleportService:TeleportToPlaceInstance(game.PlaceId, chosen, LocalPlayer)
        else
            TeleportService:Teleport(game.PlaceId, LocalPlayer) -- deixa o Roblox escolher
        end
    end)

    task.delay(25, function() hopping = false end)
end

table.insert(connections, TeleportService.TeleportInitFailed:Connect(function()
    hopping = false
    task.wait(3)
    serverHop()
end))

------------------------------------------------------------------
-- Auto Nests
------------------------------------------------------------------

local promptsConnected, terreiroDiagShown = false, false

local function fixPrompt(obj)
    if obj:IsA("ProximityPrompt") then
        obj.HoldDuration = 0
        obj.MaxActivationDistance = 9999
    end
end

local function enablePrompts()
    for _, obj in ipairs(Workspace:GetDescendants()) do fixPrompt(obj) end
    if not promptsConnected then
        promptsConnected = true
        table.insert(connections, Workspace.DescendantAdded:Connect(fixPrompt))
    end
end

local function getRoot()
    local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    return char:WaitForChild("HumanoidRootPart", 10)
end

local function getObjectCFrame(obj)
    if obj:IsA("BasePart") then return obj.CFrame end
    if obj:IsA("Model") then
        return obj.PrimaryPart and obj.PrimaryPart.CFrame or obj:GetPivot()
    end
end

local function forceInteract(target)
    local prompt = target:FindFirstChildWhichIsA("ProximityPrompt", true)
    if prompt then
        fixPrompt(prompt)
        pcall(function() if fireproximityprompt then fireproximityprompt(prompt) end end)
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

-- Terreiros (TerreiroN) que têm a pasta Nests; {} em TERREIRO_NAMES = todos
local function findTerreiros()
    local wanted, useAll = {}, #TERREIRO_NAMES == 0
    for _, n in ipairs(TERREIRO_NAMES) do wanted[n] = true end

    local found = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj.Name:match("^Terreiro%d+$") and (useAll or wanted[obj.Name]) then
            local nests = obj:FindFirstChild("Nests")
            if nests then table.insert(found, nests) end
        end
    end
    return found
end

local function listTerreiros()
    local seen, names = {}, {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if obj.Name:match("^Terreiro%d+$") and not seen[obj.Name] then
            seen[obj.Name] = true
            table.insert(names, obj.Name)
        end
    end
    table.sort(names)
    return #names > 0 and table.concat(names, ", ") or "nenhum"
end

local function runFarm()
    local my = bump("farm")
    local function alive() return config.farm and tokens.farm == my and not dead() end

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

        local terreiros, start = {}, tick()
        while alive() and #terreiros == 0 and tick() - start < 30 do
            terreiros = findTerreiros()
            if #terreiros == 0 then task.wait(1) end
        end
        if not alive() then return end

        if #terreiros == 0 then
            local names = listTerreiros()
            warn("[RC2] Terreiro/Nests não encontrado. Existem: " .. names)
            if not terreiroDiagShown then
                terreiroDiagShown = true
                notify(("Terreiro não achado (%s). Existem: %s")
                    :format(table.concat(TERREIRO_NAMES, ", "), names):sub(1, 180))
            end
            return serverHop()
        end

        local nests = {}
        for _, folder in ipairs(terreiros) do
            for _, nest in ipairs(folder:GetChildren()) do table.insert(nests, nest) end
        end

        for i, nest in ipairs(nests) do
            if not alive() then return end

            local cf = getObjectCFrame(nest)
            if cf then
                setStatus(("Nest %d/%d"):format(i, #nests))

                local root = getRoot()
                if root then root.CFrame = cf + Vector3.new(0, 1.5, 0) end
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
        bump("farm")
        setStatus("Parado")
    end
end

------------------------------------------------------------------
-- Loops de remotes (Spin, Equip Best, Rebirth, Fuse)
------------------------------------------------------------------

local function getRemote(name)
    local remotes = ReplicatedStorage:WaitForChild("Remotes", 15)
    return remotes and remotes:WaitForChild(name, 15)
end

-- Chama remote:InvokeServer(unpack(args)) em loop enquanto isOn() for verdadeiro
local function startLoop(id, isOn, delay, remoteName, args)
    local my = bump(id)
    task.spawn(function()
        local remote = getRemote(remoteName)
        if not remote then
            notify("Remote " .. remoteName .. " não encontrado!")
            return
        end
        while not dead() and tokens[id] == my and isOn() do
            pcall(function() remote:InvokeServer(unpack(args)) end)
            task.wait(delay)
        end
    end)
end

local function makeSetter(key, remoteName, delay, args)
    return function(on)
        config[key] = on
        saveConfig()
        bump(key)
        if on then
            startLoop(key, function() return config[key] end, delay, remoteName, args)
        end
    end
end

local setSpin = makeSetter("spin", "Spin", 0, {})
local setEquipBest = makeSetter("equipBest", "Shop", EQUIP_BEST_DELAY, { "equipBest" })
local setRebirth = makeSetter("rebirth", "Rewards", 0, { "rebirth" })

local function setFuse(name, level, on)
    local key = name .. ":" .. level
    config.fuse[key] = on or nil
    saveConfig()
    bump(key)
    if on then
        startLoop(key, function() return config.fuse[key] end, FUSE_DELAY, "Fuse", { name, level })
    end
end

------------------------------------------------------------------
-- HUD
------------------------------------------------------------------

local ON_COLOR, OFF_COLOR = Color3.fromRGB(46, 160, 90), Color3.fromRGB(58, 58, 68)
local toggles, hudHidden = {}, false

local function new(class, props, parent)
    local inst = Instance.new(class)
    for k, v in pairs(props) do inst[k] = v end
    inst.Parent = parent
    return inst
end

local function corner(inst, radius)
    new("UICorner", { CornerRadius = UDim.new(0, radius) }, inst)
end

local function guiParents()
    local list = {}
    pcall(function() if gethui then table.insert(list, gethui()) end end)
    pcall(function() table.insert(list, game:GetService("CoreGui")) end)
    pcall(function() table.insert(list, LocalPlayer:WaitForChild("PlayerGui", 5)) end)
    return list
end

local function destroyOldGuis()
    for _, parent in ipairs(guiParents()) do
        pcall(function()
            for _, child in ipairs(parent:GetChildren()) do
                if child.Name == "RC2_HUD" then child:Destroy() end
            end
        end)
    end
end

local function attachGui(g)
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(g) end end)
    for _, parent in ipairs(guiParents()) do
        if pcall(function() g.Parent = parent end) and g.Parent then return end
    end
end

local function buildGui()
    destroyOldGuis()
    toggles = {}
    if dragConn then pcall(function() dragConn:Disconnect() end) end

    gui = new("ScreenGui", {
        Name = "RC2_HUD", ResetOnSpawn = false, IgnoreGuiInset = true,
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling, DisplayOrder = 2147483647,
        Enabled = not hudHidden,
    })

    local frame = new("Frame", {
        Size = UDim2.new(0, 240, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
        Position = UDim2.new(0, 20, 0.3, 0), BackgroundColor3 = Color3.fromRGB(24, 24, 30),
        BorderSizePixel = 0,
    }, gui)
    corner(frame, 8)
    new("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder }, frame)

    local title = new("Frame", {
        Size = UDim2.new(1, 0, 0, 30), BackgroundColor3 = Color3.fromRGB(36, 36, 46),
        BorderSizePixel = 0, LayoutOrder = 1,
    }, frame)
    corner(title, 8)

    new("TextLabel", {
        Size = UDim2.new(1, -40, 1, 0), Position = UDim2.new(0, 10, 0, 0),
        BackgroundTransparency = 1, Text = "Carlinhos Exuzete",
        TextXAlignment = Enum.TextXAlignment.Left, Font = Enum.Font.GothamBold,
        TextSize = 14, TextColor3 = Color3.new(1, 1, 1),
    }, title)

    local minBtn = new("TextButton", {
        Size = UDim2.new(0, 28, 0, 22), Position = UDim2.new(1, -34, 0, 4),
        BackgroundColor3 = OFF_COLOR, Text = "-", Font = Enum.Font.GothamBold,
        TextSize = 16, TextColor3 = Color3.new(1, 1, 1),
    }, title)
    corner(minBtn, 5)

    local content = new("ScrollingFrame", {
        Size = UDim2.new(1, 0, 0, 340), CanvasSize = UDim2.new(0, 0, 0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y, ScrollBarThickness = 4,
        BorderSizePixel = 0, BackgroundTransparency = 1, LayoutOrder = 2,
    }, frame)
    new("UIPadding", {
        PaddingTop = UDim.new(0, 8), PaddingBottom = UDim.new(0, 10),
        PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10),
    }, content)
    new("UIListLayout", { Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder }, content)

    minBtn.MouseButton1Click:Connect(function()
        content.Visible = not content.Visible
        minBtn.Text = content.Visible and "-" or "+"
    end)

    -- arrastar pela barra de título
    local dragging, dragStart, startPos
    title.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging, dragStart, startPos = true, input.Position, frame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    dragConn = UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch) then
            local d = input.Position - dragStart
            frame.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + d.X,
                startPos.Y.Scale, startPos.Y.Offset + d.Y
            )
        end
    end)

    local order = 0
    local function nextOrder()
        order += 1
        return order
    end

    local function label(text)
        return new("TextLabel", {
            Size = UDim2.new(1, 0, 0, 18), BackgroundTransparency = 1, Text = text,
            TextXAlignment = Enum.TextXAlignment.Left, Font = Enum.Font.GothamMedium,
            TextSize = 12, TextColor3 = Color3.fromRGB(180, 180, 195), LayoutOrder = nextOrder(),
        }, content)
    end

    local function toggle(parent, text, initial, onChange, compact)
        local btn = new("TextButton", {
            AutoButtonColor = false, Font = Enum.Font.GothamMedium, TextSize = 13,
            TextColor3 = Color3.new(1, 1, 1),
            Size = compact and UDim2.new(0.25, -3, 1, 0) or UDim2.new(1, 0, 0, 28),
            LayoutOrder = compact and 0 or nextOrder(),
        }, parent)
        corner(btn, 6)
        if not compact then
            btn.TextScaled = true
            new("UITextSizeConstraint", { MaxTextSize = 13 }, btn)
        end

        local state = initial and true or false
        local t = {}

        local function render()
            btn.BackgroundColor3 = state and ON_COLOR or OFF_COLOR
            btn.Text = compact and text or (text .. (state and "  [ON]" or "  [OFF]"))
        end

        function t.Set(v)
            state = v and true or false
            render()
            onChange(state)
        end

        btn.MouseButton1Click:Connect(function() t.Set(not state) end)
        render()
        table.insert(toggles, t)
    end

    label("AUTO FARM")
    toggle(content, "Auto Nests + Hop", config.farm, setFarm)
    statusLabel = label(config.farm and "Status: Ligado" or "Status: Parado")

    label("EXTRAS")
    toggle(content, "Auto Spin", config.spin, setSpin)
    toggle(content, "Equip Best", config.equipBest, setEquipBest)
    toggle(content, "Auto Rebirth", config.rebirth, setRebirth)

    label("FUSE (níveis 1 a 4)")
    for _, name in ipairs(FUSE_NAMES) do
        label(name)
        local row = new("Frame", {
            Size = UDim2.new(1, 0, 0, 26), BackgroundTransparency = 1, LayoutOrder = nextOrder(),
        }, content)
        new("UIListLayout", { FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 4) }, row)

        for _, level in ipairs(FUSE_LEVELS) do
            toggle(row, tostring(level), config.fuse[name .. ":" .. level] == true,
                function(on) setFuse(name, level, on) end, true)
        end
    end

    label("END = desligar tudo | INSERT = esconder HUD")
    attachGui(gui)
end

-- END desliga tudo, INSERT esconde/mostra o HUD
table.insert(connections, UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.End then
        for _, t in ipairs(toggles) do t.Set(false) end
    elseif input.KeyCode == Enum.KeyCode.Insert then
        hudHidden = not hudHidden
        if gui then gui.Enabled = not hudHidden end
    end
end))

------------------------------------------------------------------
-- Início
------------------------------------------------------------------

buildGui()

-- recria o HUD se o jogo ou o executor apagar
task.spawn(function()
    while task.wait(1) and not dead() do
        if not gui or not gui.Parent then buildGui() end
    end
end)

-- retoma o que estava ligado (importante depois do server hop)
if config.farm then runFarm() end
if config.spin then setSpin(true) end
if config.equipBest then setEquipBest(true) end
if config.rebirth then setRebirth(true) end

for _, name in ipairs(FUSE_NAMES) do
    for _, level in ipairs(FUSE_LEVELS) do
        if config.fuse[name .. ":" .. level] then setFuse(name, level, true) end
    end
end
