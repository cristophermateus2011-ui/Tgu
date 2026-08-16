--[[
    PAINEL DE AUTOMAÇÃO - ROBLOX STUDIO
    ====================================
    Coloque este LocalScript em:
    StarterPlayer > StarterPlayerScripts

    IMPORTANTE:
    Este código foi feito para um jogo PRÓPRIO de teste no Roblox Studio.
    Não utiliza exploits, executores externos ou bypasses.

    ADAPTAÇÕES:
    - NPCs: por padrão procura Models com Humanoid dentro de Workspace.
    - Frutas: por padrão procura objetos/Models cujo nome contenha "Fruit".
    - Combate: a função performAttack() é apenas um ponto de integração.
      Adapte-a ao sistema de combate legítimo do seu jogo.
    - Sorte: o script NÃO falsifica Luck. Ele apenas procura atributos
      legítimos chamados "Luck", "Lucky" ou "LuckMultiplier".
]]

--// Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

--// Configuração
local Config = {
    AutoFarm = false,
    AutoAttack = false,
    AutoFruit = false,
    FruitESP = false,
    TeleportFruit = false,
    PlayerESP = false,
    Lucky = false,

    WalkSpeed = 16,

    NPCDistance = 50,
    PlayerDistance = 100,
    FruitDistance = 500,

    AttackInterval = 0.25,
}

--// Controle interno
local Connections = {}
local ESPObjects = {}
local LastAttack = 0

--------------------------------------------------
--// FUNÇÕES BÁSICAS
--------------------------------------------------

local function getCharacter()
    local character = LocalPlayer.Character

    if not character then
        return nil
    end

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local root = character:FindFirstChild("HumanoidRootPart")

    if not humanoid or not root then
        return nil
    end

    if humanoid.Health <= 0 then
        return nil
    end

    return character, humanoid, root
end

local function getRoot(model)
    if not model then
        return nil
    end

    return model:FindFirstChild("HumanoidRootPart")
        or model.PrimaryPart
        or model:FindFirstChildWhichIsA("BasePart")
end

local function getDistance(part1, part2)
    if not part1 or not part2 then
        return math.huge
    end

    return (part1.Position - part2.Position).Magnitude
end

--------------------------------------------------
--// NPC
--------------------------------------------------

local function isNPC(model)
    if not model or not model:IsA("Model") then
        return false
    end

    local humanoid = model:FindFirstChildOfClass("Humanoid")

    if not humanoid then
        return false
    end

    -- Evita considerar o próprio jogador como NPC.
    if Players:GetPlayerFromCharacter(model) then
        return false
    end

    return humanoid.Health > 0
end

local function findNearestNPC()
    local character, humanoid, root = getCharacter()

    if not root then
        return nil
    end

    local nearest = nil
    local nearestDistance = Config.NPCDistance

    for _, object in ipairs(workspace:GetDescendants()) do
        if isNPC(object) then
            local npcRoot = getRoot(object)

            if npcRoot then
                local distance = getDistance(root, npcRoot)

                if distance <= nearestDistance then
                    nearest = object
                    nearestDistance = distance
                end
            end
        end
    end

    return nearest
end

--------------------------------------------------
--// ATAQUE
--------------------------------------------------

local function performAttack(npc)
    --[[
        ADAPTE ESTA FUNÇÃO AO SEU SISTEMA DE COMBATE.

        Exemplo de estrutura para um jogo próprio:

        local ReplicatedStorage = game:GetService("ReplicatedStorage")
        local AttackEvent = ReplicatedStorage:WaitForChild("AttackEvent")

        AttackEvent:FireServer(npc)

        NÃO utilize RemoteEvents de jogos de terceiros.
    ]]

    if not npc or not npc.Parent then
        return
    end

    local humanoid = npc:FindFirstChildOfClass("Humanoid")

    if not humanoid or humanoid.Health <= 0 then
        return
    end

    -- Apenas demonstração.
    -- O dano real deve ser controlado pelo servidor do seu jogo.
end

--------------------------------------------------
--// AUTO ATTACK
--------------------------------------------------

local function autoAttackStep()
    if not Config.AutoAttack then
        return
    end

    local now = os.clock()

    if now - LastAttack < Config.AttackInterval then
        return
    end

    LastAttack = now

    local npc = findNearestNPC()

    if npc then
        performAttack(npc)
    end
end

--------------------------------------------------
--// AUTO FARM
--------------------------------------------------

local function autoFarmStep()
    if not Config.AutoFarm then
        return
    end

    local character, humanoid, root = getCharacter()

    if not root then
        return
    end

    local npc = findNearestNPC()

    if not npc then
        return
    end

    local npcRoot = getRoot(npc)

    if not npcRoot then
        return
    end

    -- Mantém o personagem próximo do NPC.
    -- Adapte esta lógica ao sistema de movimentação
    -- do seu jogo próprio.

    local targetPosition =
        npcRoot.Position
        + Vector3.new(0, 0, 4)

    character:PivotTo(
        CFrame.new(
            targetPosition,
            npcRoot.Position
        )
    )
end

--------------------------------------------------
--// FRUTAS
--------------------------------------------------

local function isFruit(object)
    if not object then
        return false
    end

    local name = string.lower(object.Name)

    -- ADAPTE caso as frutas do seu jogo tenham outro padrão de nomes.
    return string.find(name, "fruit", 1, true) ~= nil
        or string.find(name, "fruta", 1, true) ~= nil
end

local function getFruitPosition(object)
    if not object then
        return nil
    end

    if object:IsA("BasePart") then
        return object.Position
    end

    if object:IsA("Model") then
        local part = getRoot(object)

        if part then
            return part.Position
        end
    end

    return nil
end

local function findFruits()
    local fruits = {}

    for _, object in ipairs(workspace:GetDescendants()) do
        if isFruit(object) then
            local position = getFruitPosition(object)

            if position then
                table.insert(fruits, {
                    Object = object,
                    Position = position,
                })
            end
        end
    end

    return fruits
end

--------------------------------------------------
--// TELEPORT PARA FRUTA
--------------------------------------------------

local function teleportToFruit()
    if not Config.TeleportFruit then
        return
    end

    local character, humanoid, root = getCharacter()

    if not root then
        return
    end

    local closestFruit = nil
    local closestDistance = Config.FruitDistance

    for _, fruitData in ipairs(findFruits()) do
        local object = fruitData.Object

        if object and object.Parent then
            local distance =
                (root.Position - fruitData.Position).Magnitude

            if distance <= closestDistance then
                closestDistance = distance
                closestFruit = fruitData
            end
        end
    end

    if closestFruit
        and closestFruit.Object
        and closestFruit.Object.Parent then

        character:PivotTo(
            CFrame.new(closestFruit.Position + Vector3.new(0, 3, 0))
        )
    end
end

--------------------------------------------------
--// LUCKY
--------------------------------------------------

local function checkLegitimateLuck()
    if not Config.Lucky then
        return
    end

    local character = LocalPlayer.Character

    if character then
        local luck =
            character:GetAttribute("Luck")
            or character:GetAttribute("Lucky")
            or character:GetAttribute("LuckMultiplier")

        if luck ~= nil then
            print("Luck legítimo encontrado:", luck)
        else
            print("Lucky: nenhum sistema de Luck acessível ao cliente.")
        end
    end
end

--------------------------------------------------
--// ESP
--------------------------------------------------

local function removeESP(object)
    local gui = ESPObjects[object]

    if gui then
        gui:Destroy()
        ESPObjects[object] = nil
    end
end

local function createESP(object, text)
    if not object or not object.Parent then
        return
    end

    if ESPObjects[object] then
        ESPObjects[object].Text = text
        return
    end

    local part = getRoot(object)

    if not part then
        return
    end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "AutomationESP"
    billboard.Adornee = part
    billboard.Size = UDim2.fromOffset(180, 40)
    billboard.StudsOffset = Vector3.new(0, 3, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = part

    local label = Instance.new("TextLabel")
    label.Size = UDim2.fromScale(1, 1)
    label.BackgroundTransparency = 1
    label.TextScaled = true
    label.Text = text
    label.Parent = billboard

    ESPObjects[object] = label
end

local function updateFruitESP()
    if not Config.FruitESP then
        return
    end

    local character, humanoid, root = getCharacter()

    if not root then
        return
    end

    local currentObjects = {}

    for _, fruitData in ipairs(findFruits()) do
        local object = fruitData.Object

        if object
            and object.Parent
            and (fruitData.Position - root.Position).Magnitude
                <= Config.FruitDistance then

            currentObjects[object] = true

            createESP(
                object,
                object.Name
                .. "\n"
                .. math.floor(
                    (fruitData.Position - root.Position).Magnitude
                )
                .. " studs"
            )
        end
    end

    for object in pairs(ESPObjects) do
        if not currentObjects[object] then
            removeESP(object)
        end
    end
end

local function updatePlayerESP()
    if not Config.PlayerESP then
        return
    end

    local character, humanoid, root = getCharacter()

    if not root then
        return
    end

    local currentObjects = {}

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local targetCharacter = player.Character

            if targetCharacter then
                local targetRoot = getRoot(targetCharacter)

                if targetRoot then
                    local distance =
                        getDistance(root, targetRoot)

                    if distance <= Config.PlayerDistance then
                        currentObjects[targetCharacter] = true

                        createESP(
                            targetCharacter,
                            player.Name
                            .. "\n"
                            .. math.floor(distance)
                            .. " studs"
                        )
                    end
                end
            end
        end
    end

    for object in pairs(ESPObjects) do
        if not currentObjects[object]
            and object ~= LocalPlayer.Character then

            removeESP(object)
        end
    end
end

local function clearAllESP()
    for object, gui in pairs(ESPObjects) do
        if gui then
            gui:Destroy()
        end

        ESPObjects[object] = nil
    end
end

--------------------------------------------------
--// WALKSPEED
--------------------------------------------------

local function updateWalkSpeed()
    local character, humanoid = getCharacter()

    if humanoid then
        humanoid.WalkSpeed = Config.WalkSpeed
    end
end

--------------------------------------------------
--// STOP ALL
--------------------------------------------------

local function stopAll()
    Config.AutoFarm = false
    Config.AutoAttack = false
    Config.AutoFruit = false
    Config.FruitESP = false
    Config.TeleportFruit = false
    Config.PlayerESP = false
    Config.Lucky = false

    clearAllESP()

    print("STOP ALL: todas as funções foram desligadas.")
end

--------------------------------------------------
--// INTERFACE
--------------------------------------------------

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AutomationPanel"
screenGui.ResetOnSpawn = false
screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local main = Instance.new("Frame")
main.Size = UDim2.fromOffset(330, 520)
main.Position = UDim2.new(0.5, -165, 0.5, -260)
main.BackgroundTransparency = 0.05
main.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 12)
corner.Parent = main

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 45)
title.BackgroundTransparency = 1
title.Text = "AUTOMATION PANEL"
title.TextScaled = true
title.Parent = main

local scroll = Instance.new("ScrollingFrame")
scroll.Position = UDim2.fromOffset(10, 55)
scroll.Size = UDim2.new(1, -20, 1, -65)
scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
scroll.ScrollBarThickness = 6
scroll.BackgroundTransparency = 1
scroll.Parent = main

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 7)
layout.Parent = scroll

--------------------------------------------------
--// BOTÃO ON/OFF
--------------------------------------------------

local Buttons = {}

local function createToggle(name, callback)
    local button = Instance.new("TextButton")

    button.Size = UDim2.new(1, -5, 0, 42)
    button.TextScaled = true
    button.Text = name .. ": OFF"
    button.Parent = scroll

    Buttons[name] = button

    local enabled = false

    button.Activated:Connect(function()
        enabled = not enabled

        if enabled then
            button.Text = name .. ": ON"
        else
            button.Text = name .. ": OFF"
        end

        callback(enabled)
    end)

    return button
end

createToggle("Auto Farm", function(value)
    Config.AutoFarm = value
end)

createToggle("Auto Attack", function(value)
    Config.AutoAttack = value
end)

createToggle("Auto Fruit", function(value)
    Config.AutoFruit = value
end)

createToggle("Fruit ESP", function(value)
    Config.FruitESP = value

    if not value then
        clearAllESP()
    end
end)

createToggle("Teleport Fruit", function(value)
    Config.TeleportFruit = value

    if value then
        teleportToFruit()
    end
end)

createToggle("Player ESP", function(value)
    Config.PlayerESP = value

    if not value then
        clearAllESP()
    end
end)

createToggle("Lucky", function(value)
    Config.Lucky = value
    checkLegitimateLuck()
end)

--------------------------------------------------
--// CAMPOS NUMÉRICOS
--------------------------------------------------

local function createNumberBox(name, defaultValue, callback)
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -5, 0, 30)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextScaled = true
    label.Parent = scroll

    local box = Instance.new("TextBox")
    box.Size = UDim2.new(1, -5, 0, 38)
    box.Text = tostring(defaultValue)
    box.PlaceholderText = tostring(defaultValue)
    box.TextScaled = true
    box.Parent = scroll

    box.FocusLost:Connect(function()
        local value = tonumber(box.Text)

        if value then
            callback(value)
        else
            box.Text = tostring(defaultValue)
        end
    end)

    return box
end

createNumberBox(
    "WalkSpeed",
    Config.WalkSpeed,
    function(value)
        Config.WalkSpeed = math.clamp(value, 0, 200)
        updateWalkSpeed()
    end
)

createNumberBox(
    "NPC Distance",
    Config.NPCDistance,
    function(value)
        Config.NPCDistance = math.clamp(value, 5, 500)
    end
)

createNumberBox(
    "Player Distance",
    Config.PlayerDistance,
    function(value)
        Config.PlayerDistance = math.clamp(value, 5, 1000)
    end
)

createNumberBox(
    "Fruit Distance",
    Config.FruitDistance,
    function(value)
        Config.FruitDistance = math.clamp(value, 5, 2000)
    end
)

createNumberBox(
    "Attack Interval",
    Config.AttackInterval,
    function(value)
        Config.AttackInterval = math.clamp(value, 0.05, 5)
    end
)

--------------------------------------------------
--// STOP ALL
--------------------------------------------------

local stopButton = Instance.new("TextButton")
stopButton.Size = UDim2.new(1, -5, 0, 50)
stopButton.Text = "STOP ALL"
stopButton.TextScaled = true
stopButton.Parent = scroll

stopButton.Activated:Connect(function()
    stopAll()

    for name, button in pairs(Buttons) do
        button.Text = name .. ": OFF"
    end
end)

--------------------------------------------------
--// JANELA ARRASTÁVEL
--// Funciona com mouse e toque.
--------------------------------------------------

local dragging = false
local dragStart
local startPosition

local function updateDrag(input)
    local delta = input.Position - dragStart

    main.Position = UDim2.new(
        startPosition.X.Scale,
        startPosition.X.Offset + delta.X,
        startPosition.Y.Scale,
        startPosition.Y.Offset + delta.Y
    )
end

title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = true
        dragStart = input.Position
        startPosition = main.Position
    end
end)

title.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        dragging = false
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging
        and (
            input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch
        ) then

        updateDrag(input)
    end
end)

--------------------------------------------------
--// LOOPS
--------------------------------------------------

-- Loop principal com intervalo moderado.
task.spawn(function()
    while screenGui.Parent do

        if Config.AutoAttack then
            autoAttackStep()
        end

        if Config.AutoFarm then
            autoFarmStep()
        end

        if Config.AutoFruit then
            -- Atualiza a detecção periodicamente.
            -- A função findFruits() sempre verifica se o objeto ainda existe.
            local fruits = findFruits()

            for _, fruitData in ipairs(fruits) do
                if fruitData.Object and fruitData.Object.Parent then
                    -- Fruta encontrada.
                    -- Você pode conectar aqui ao seu sistema de coleta.
                end
            end
        end

        if Config.FruitESP then
            updateFruitESP()
        end

        if Config.PlayerESP then
            updatePlayerESP()
        end

        task.wait(0.25)
    end
end)

--------------------------------------------------
--// MANTER WALKSPEED APÓS RESPAWN
--------------------------------------------------

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1)

    if Config.WalkSpeed then
        updateWalkSpeed()
    end
end)

print("Automation Panel carregado com sucesso.")
