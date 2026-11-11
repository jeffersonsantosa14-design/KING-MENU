-- KING MENU | LocalScript
-- ProtÃ³tipo para uso no seu prÃ³prio jogo Roblox

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- CONFIGURAÃ‡Ã•ES
local MENU_KEY = Enum.KeyCode.K
local DEMO_KEY = "KING-2026-VIP"

local ESP_ENABLED = false
local AIM_ENABLED = false
local menuUnlocked = false

-- LOCAIS DE TELEPORTE (ALTERE AS COORDENADAS)
local LOCATIONS = {
    BASE = Vector3.new(0, 5, 0),
    AEROPORTO = Vector3.new(100, 5, 100),
    GARAGEM = Vector3.new(200, 5, 100),
    PRAIA = Vector3.new(300, 5, 100)
}

-- GUI
local gui = Instance.new("ScreenGui")
gui.Name = "KingMenu"
gui.ResetOnSpawn = false
gui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(300, 470)
frame.Position = UDim2.new(0.5, -150, 0.5, -235)
frame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
frame.BorderSizePixel = 0
frame.Visible = true
frame.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 12)
corner.Parent = frame

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(220, 175, 40)
stroke.Thickness = 2
stroke.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 45)
title.BackgroundTransparency = 1
title.Text = "ðŸ‘‘ KING MENU"
title.TextColor3 = Color3.fromRGB(255, 205, 60)
title.Font = Enum.Font.GothamBold
title.TextSize = 23
title.Parent = frame

local function createButton(parent, text, position, size)
    local button = Instance.new("TextButton")
    button.Size = size or UDim2.new(1, -20, 0, 34)
    button.Position = position
    button.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
    button.TextColor3 = Color3.fromRGB(255, 220, 120)
    button.Text = text
    button.Font = Enum.Font.GothamBold
    button.TextSize = 13
    button.AutoButtonColor = true
    button.Parent = parent

    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 7)
    c.Parent = button

    return button
end

-- TELA DE CHAVE (DEMONSTRAÃ‡ÃƒO)
local keyBox = Instance.new("TextBox")
keyBox.Size = UDim2.new(1, -30, 0, 38)
keyBox.Position = UDim2.fromOffset(15, 55)
keyBox.PlaceholderText = "Digite sua chave..."
keyBox.Text = ""
keyBox.TextColor3 = Color3.new(1, 1, 1)
keyBox.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
keyBox.Font = Enum.Font.Gotham
keyBox.TextSize = 14
keyBox.Parent = frame

local keyCorner = Instance.new("UICorner")
keyCorner.CornerRadius = UDim.new(0, 7)
keyCorner.Parent = keyBox

local status = Instance.new("TextLabel")
status.Size = UDim2.new(1, -20, 0, 25)
status.Position = UDim2.fromOffset(10, 96)
status.BackgroundTransparency = 1
status.Text = "Aguardando chave..."
status.TextColor3 = Color3.fromRGB(255, 200, 60)
status.Font = Enum.Font.Gotham
status.TextSize = 12
status.Parent = frame

local content = Instance.new("Frame")
content.Size = UDim2.new(1, -20, 1, -135)
content.Position = UDim2.fromOffset(10, 130)
content.BackgroundTransparency = 1
content.Visible = false
content.Parent = frame

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 5)
layout.SortOrder = Enum.SortOrder.LayoutOrder
layout.Parent = content

local function addMenuButton(text, callback)
    local button = createButton(
        content,
        text,
        UDim2.new(),
        UDim2.new(1, 0, 0, 32)
    )
    button.LayoutOrder = #content:GetChildren()
    button.MouseButton1Click:Connect(callback)
    return button
end

local function getCharacter()
    return player.Character or player.CharacterAdded:Wait()
end

local function teleportTo(position)
    local character = getCharacter()
    local root = character:FindFirstChild("HumanoidRootPart")

    if root then
        character:PivotTo(CFrame.new(position + Vector3.new(0, 3, 0)))
    end
end

-- ESP
local espObjects = {}

local function clearESP()
    for _, highlight in pairs(espObjects) do
        if highlight then
            highlight:Destroy()
        end
    end
    table.clear(espObjects)
end

local function updateESP()
    clearESP()

    if not ESP_ENABLED then
        return
    end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            local highlight = Instance.new("Highlight")
            highlight.FillColor = Color3.fromRGB(255, 190, 40)
            highlight.OutlineColor = Color3.new(1, 1, 1)
            highlight.Adornee = otherPlayer.Character
            highlight.Parent = gui
            table.insert(espObjects, highlight)
        end
    end
end

-- AIMBOT DE TREINO: SOMENTE NPCs COM A TAG TrainingTarget
local function getNearestTrainingTarget()
    local myCharacter = player.Character
    local myRoot = myCharacter and myCharacter:FindFirstChild("HumanoidRootPart")

    if not myRoot then
        return nil
    end

    local nearest = nil
    local shortestDistance = math.huge

    for _, object in ipairs(workspace:GetDescendants()) do
        if object:IsA("Model")
            and object:FindFirstChild("Humanoid")
            and object:FindFirstChild("HumanoidRootPart")
            and object:GetAttribute("TrainingTarget") == true then

            local distance =
                (object.HumanoidRootPart.Position - myRoot.Position).Magnitude

            if distance < shortestDistance then
                shortestDistance = distance
                nearest = object
            end
        end
    end

    return nearest
end

-- BOTÃ•ES
local keyButton = createButton(
    frame,
    "VALIDAR CHAVE",
    UDim2.fromOffset(15, 125),
    UDim2.new(1, -30, 0, 34)
)

keyButton.MouseButton1Click:Connect(function()
    if keyBox.Text == DEMO_KEY then
        menuUnlocked = true
        keyBox.Visible = false
        keyButton.Visible = false
        status.Text = "ACESSO LIBERADO!"
        status.TextColor3 = Color3.fromRGB(70, 255, 100)
        content.Visible = true
    else
        status.Text = "CHAVE INVÃLIDA!"
        status.TextColor3 = Color3.fromRGB(255, 70, 70)
    end
end)

addMenuButton("AIMBOT TREINO: DESLIGADO", function(button)
    AIM_ENABLED = not AIM_ENABLED
    button.Text = AIM_ENABLED and "AIMBOT TREINO: LIGADO"
        or "AIMBOT TREINO: DESLIGADO"
end)

addMenuButton("ESP: DESLIGADO", function(button)
    ESP_ENABLED = not ESP_ENABLED
    button.Text = ESP_ENABLED and "ESP: LIGADO" or "ESP: DESLIGADO"
    updateESP()
end)

addMenuButton("PUXAR ARMAS", function()
    status.Text = "Configure os pontos de armas no seu jogo."
end)

addMenuButton("PEGAR CARROS", function()
    status.Text = "Configure os veÃ­culos no seu jogo."
end)

-- TP PARA JOGADORES: CICLA ENTRE OS JOGADORES DISPONÃVEIS
local targetIndex = 0

addMenuButton("TP JOGADORES: PRÃ“XIMO", function()
    local targets = {}

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player
            and otherPlayer.Character
            and otherPlayer.Character:FindFirstChild("HumanoidRootPart") then
            table.insert(targets, otherPlayer)
        end
    end

    if #targets == 0 then
        status.Text = "NENHUM JOGADOR DISPONÃVEL."
        return
    end

    targetIndex = (targetIndex % #targets) + 1
    local target = targets[targetIndex]
    local root = target.Character.HumanoidRootPart

    teleportTo(root.Position + Vector3.new(4, 0, 0))
    status.Text = "ALVO: " .. target.Name
end)

-- TP PARA LUGARES
for placeName, position in pairs(LOCATIONS) do
    addMenuButton("TP " .. placeName, function()
        teleportTo(position)
        status.Text = "TELETRANSPORTE: " .. placeName
    end)
end

-- AIMBOT DE TREINO: APENAS LOCALIZA O NPC MAIS PRÃ“XIMO
RunService.RenderStepped:Connect(function()
    if not AIM_ENABLED or not menuUnlocked then
        return
    end

    local target = getNearestTrainingTarget()
    if target then
        -- Ponto de integraÃ§Ã£o para cÃ¢mera de treino no seu prÃ³prio jogo.
    end
end)

-- TECLA K ABRE/FECHA
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then
        return
    end

    if input.KeyCode == MENU_KEY and menuUnlocked then
        frame.Visible = not frame.Visible
    end
end)

print("KING MENU carregado!")
