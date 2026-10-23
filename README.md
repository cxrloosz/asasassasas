-- RC2 HUD: Auto Nests + Server Hop + Fuse loops (com toggles)

-- Só roda no jogo específico (aceita PlaceId ou GameId/UniverseId)
local ALLOWED_ID = 87029994390208
if game.PlaceId ~= ALLOWED_ID and game.GameId ~= ALLOWED_ID then
    -- Jogo errado: remove qualquer HUD/loop que tenha sobrado e encerra
    warn("[RC2] Jogo diferente do permitido. PlaceId atual: " .. tostring(game.PlaceId)
        .. " | GameId atual: " .. tostring(game.GameId))
    pcall(function()
        if getgenv and getgenv().RC2_CLEANUP then getgenv().RC2_CLEANUP() end
    end)
    pcall(function()
        for _, c in ipairs(game:GetService("CoreGui"):GetChildren()) do
            if c.Name == "RC2_HUD" then c:Destroy() end
        end
    end)
    pcall(function()
        for _, c in ipairs(gethui():GetChildren()) do
            if c.Name == "RC2_HUD" then c:Destroy() end
        end
    end)
    return
end

local SCRIPT_URL = "https://raw.githubusercontent.com/cxrloosz/rc2.lua/refs/heads/main/README.lua"
local CONFIG_FILE = "rc2_config.json"
local LOCAL_FILE = "rc2_hud.lua" -- cópia deste script na pasta workspace do executor
print("[RC2] Script iniciado | PlaceId: " .. tostring(game.PlaceId))
local FUSE_DELAY = 0.15   -- segundos entre cada Fuse (aumente se der lag/kick)
local MAX_RUNTIME = 120   -- segundos: se o farm travar, força o hop
local FUSE_NAMES = { "ExuSeteChaves", "PombagiraRainha", "ExuVeludo" }
local FUSE_LEVELS = { 1, 2, 3, 4 }

------------------------------------------------------------------
-- Instância única (resolve execuções duplicadas após o hop)
------------------------------------------------------------------
local INSTANCE_ID = tostring(math.random(1, 1e9)) .. "-" .. tostring(os.clock())
local cleaned = false
local gui, dragConn
local connections = {}
local farmToken = 0
local fuseTokens = {}

local function cleanup()
    cleaned = true
    farmToken += 1
    for k in pairs(fuseTokens) do fuseTokens[k] += 1 end
    for _, c in ipairs(connections) do pcall(function() c:Disconnect() end) end
    pcall(function() if dragConn then dragConn:Disconnect() end end)
    pcall(function() if gui then gui:Destroy() end end)
end

if getgenv then
    if getgenv().RC2_CLEANUP then pcall(getgenv().RC2_CLEANUP) end
    getgenv().RC2_CLEANUP = cleanup
    getgenv().RC2_ID = INSTANCE_ID
end

-- true se esta instância foi encerrada ou substituída por outra mais nova
local function dead()
    if not cleaned and getgenv and getgenv().RC2_ID ~= INSTANCE_ID then
        cleanup()
    end
    return cleaned
end

------------------------------------------------------------------
-- Serviços
------------------------------------------------------------------
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
-- Config persistente (sobrevive ao server hop)
------------------------------------------------------------------
local config = { farm = false, fuse = {} }

pcall(function()
    if isfile and isfile(CONFIG_FILE) then
        local data = HttpService:JSONDecode(readfile(CONFIG_FILE))
        if type(data) == "table" then
            config.farm = data.farm == true
            config.fuse = type(data.fuse) == "table" and data.fuse or {}
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
-- Fila para recarregar o script a cada teleporte
------------------------------------------------------------------
local qf = queue_on_teleport or queueonteleport
    or (syn and syn.queue_on_teleport)
    or (fluxus and fluxus.queue_on_teleport)

local function queueScript()
    if not qf then return end
    local code = ([==[
        if game.PlaceId ~= %s and game.GameId ~= %s then return end
        repeat task.wait() until game:IsLoaded()
        task.wait(2)
        print("[RC2] Loader: executando script enfileirado...")
        local src
        pcall(function()
            if isfile and isfile("%s") then src = readfile("%s") end
        end)
        if not src or #src < 200 then
            local ok, res = pcall(function() return game:HttpGet("%s") end)
            if ok then
                src = res
            else
                warn("[RC2] Loader: falha ao baixar o script: " .. tostring(res))
            end
        end
        local fn, err = loadstring(src or "")
        if not fn then
            warn("[RC2] Loader: erro ao carregar o script: " .. tostring(err))
            return
        end
        fn()
    ]==]):format(tostring(ALLOWED_ID), tostring(ALLOWED_ID), LOCAL_FILE, LOCAL_FILE, SCRIPT_URL)
    pcall(qf, code)
end

if qf then
    queueScript()
else
    warn("[RC2] Executor sem queue_on_teleport. Coloque o script na pasta autoexec!")
end

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
local function serverHop()
    if hopping or dead() or not config.farm then return end
    hopping = true
    setStatus("Procurando servidor...")

    local candidates = {}
    local cursor = ""
    for _ = 1, 3 do
        local ok, result = pcall(function()
            local url = "https://games.roblox.com/v1/games/" .. game.PlaceId
                .. "/servers/Public?sortOrder=Asc&limit=100"
                .. (cursor ~= "" and ("&cursor=" .. cursor) or "")
            return HttpService:JSONDecode(game:HttpGet(url))
        end)
        if not ok or not result or not result.data then break end
        for _, s in ipairs(result.data) do
            if s.id ~= game.JobId and s.playing < s.maxPlayers - 1 then
                table.insert(candidates, s.id)
            end
        end
        cursor = result.nextPageCursor or ""
        if cursor == "" then break end
    end

    hopping = false
    if dead() or not config.farm then return end
    queueScript()
    if #candidates > 0 then
        local id = candidates[math.random(#candidates)]
        pcall(TeleportService.TeleportToPlaceInstance, TeleportService, game.PlaceId, id, LocalPlayer)
    else
        pcall(TeleportService.Teleport, TeleportService, game.PlaceId, LocalPlayer)
    end
end

table.insert(connections, TeleportService.TeleportInitFailed:Connect(function(_, _, msg)
    warn("[RC2] Teleporte falhou: " .. tostring(msg))
    task.wait(3)
    serverHop()
end))

------------------------------------------------------------------
-- Auto Nests
------------------------------------------------------------------
local RETURN_CFRAME = CFrame.new(20, 5.83, -28, 0, 0, -1, 1, 0, 0, 0, -1, 0)
local promptConnected = false

local function fixPrompt(obj)
    if obj:IsA("ProximityPrompt") then
        obj.HoldDuration = 0
        obj.MaxActivationDistance = 9999
    end
end

local function enablePrompts()
    for _, obj in ipairs(Workspace:GetDescendants()) do fixPrompt(obj) end
    if not promptConnected then
        promptConnected = true
        table.insert(connections, Workspace.DescendantAdded:Connect(fixPrompt))
    end
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

local function getRoot()
    local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    return char:WaitForChild("HumanoidRootPart", 10)
end

local function runFarm()
    farmToken += 1
    local my = farmToken
    local function alive() return config.farm and my == farmToken and not dead() end

    -- Failsafe: se travar, troca de servidor
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
            warn("[RC2] Terreiro8/Nests não encontrado, trocando de servidor.")
            return serverHop()
        end

        local list = nests:GetChildren()
        for i, nest in ipairs(list) do
            if not alive() then return end
            local cf = getObjectCFrame(nest)
            if cf then
                setStatus(("Nest %d/%d"):format(i, #list))
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
        farmToken += 1
        setStatus("Parado")
    end
end

------------------------------------------------------------------
-- Fuse loops (um por personagem/nível)
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
        while not dead() and fuseTokens[key] == my and config.fuse[key] do
            pcall(function() fuse:InvokeServer(name, level) end)
            task.wait(FUSE_DELAY)
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

-- Remove HUDs antigos (de instâncias anteriores) em todos os locais possíveis
local function destroyOldGuis()
    local parents = {}
    pcall(function() if gethui then table.insert(parents, gethui()) end end)
    pcall(function() table.insert(parents, game:GetService("CoreGui")) end)
    pcall(function() table.insert(parents, LocalPlayer:FindFirstChild("PlayerGui")) end)
    for _, p in ipairs(parents) do
        pcall(function()
            for _, child in ipairs(p:GetChildren()) do
                if child.Name == "RC2_HUD" then child:Destroy() end
            end
        end)
    end
end

local function attachGui(g)
    pcall(function() if syn and syn.protect_gui then syn.protect_gui(g) end end)

    local ok = pcall(function() g.Parent = gethui() end)
    if ok and g.Parent then return "gethui" end

    ok = pcall(function() g.Parent = game:GetService("CoreGui") end)
    if ok and g.Parent then return "CoreGui" end

    g.Parent = LocalPlayer:WaitForChild("PlayerGui")
    return "PlayerGui"
end

local function buildGui()
    destroyOldGuis()
    toggles = {}
    local order = 0
    local function nextOrder() order += 1 return order end
    if dragConn then pcall(function() dragConn:Disconnect() end) end

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

    -- Barra de título (arrastável)
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
    titleText.Text = "RC2 HUD"
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

    local content = Instance.new("Frame")
    content.Size = UDim2.new(1, 0, 0, 0)
    content.AutomaticSize = Enum.AutomaticSize.Y
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

    -- Arrastar
    local dragging, dragStart, startPos
    title.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = frame.Position
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

    -- Helpers
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
        end
        btn.Parent = parent
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

        local state = initial and true or false
        local function render()
            btn.BackgroundColor3 = state and ON_COLOR or OFF_COLOR
            btn.Text = compact and text or (text .. (state and "  [ON]" or "  [OFF]"))
        end
        local t = {}
        function t.Set(v, silent)
            state = v and true or false
            render()
            if not silent then onChange(state) end
        end
        btn.MouseButton1Click:Connect(function() t.Set(not state) end)
        render()
        table.insert(toggles, t)
        return t
    end

    -- Seção: Auto Farm
    makeLabel("AUTO FARM")
    makeToggle(content, "Auto Nests + Hop", config.farm, setFarm)
    statusLabel = makeLabel(config.farm and "Status: Ligado" or "Status: Parado")

    -- Seção: Fuse
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
            makeToggle(row, tostring(level), config.fuse[key] == true, function(on)
                setFuse(name, level, on)
            end, true)
        end
    end

    makeLabel("END = desligar tudo | INSERT = esconder HUD")

    local where = attachGui(gui)
    print("[RC2] HUD criado em: " .. where)
end

------------------------------------------------------------------
-- Atalhos
------------------------------------------------------------------
local function stopAll()
    for _, t in ipairs(toggles) do t.Set(false) end -- dispara os callbacks e salva
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
-- Inicialização
------------------------------------------------------------------
buildGui()

-- Vigia: se o HUD sumir (jogo apagou, executor limpou), recria sozinho
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

-- Retoma o que estava ligado (importante após o server hop)
if config.farm then
    runFarm()
end
for _, name in ipairs(FUSE_NAMES) do
    for _, level in ipairs(FUSE_LEVELS) do
        if config.fuse[name .. ":" .. level] then
            setFuse(name, level, true)
        end
    end
end
