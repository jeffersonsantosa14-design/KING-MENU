--[[
=====================================================================
 MENU DE ADMIN PARA ROBLOX (para o SEU jogo)
=====================================================================
 InstalaÃ§Ã£o no Roblox Studio:

 1) ServerScriptService  -> Script       chamado  "AdminMenuServer"
    (cole a PARTE 1)

 2) StarterPlayer > StarterPlayerScripts -> LocalScript chamado "AdminMenuClient"
    (cole a PARTE 2)

 Coloque seu UserId na tabela ADMINS (parte 1).
 Tecla para abrir/fechar o menu: M
=====================================================================
]]


--====================================================================
-- PARTE 1: SCRIPT (ServerScriptService/AdminMenuServer)
--====================================================================

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Debris = game:GetService("Debris")

-- Coloque aqui os UserIds de quem pode usar o menu
local ADMINS = {
	[1234567890] = true, -- troque pelo seu UserId
}

local CAR_SEARCH_RADIUS = 60   -- distÃ¢ncia mÃ¡xima para achar um carro perto
local LIFT_SPEED = 80          -- velocidade ao "levantar" o carro
local MOON_SPEED = 500         -- velocidade ao "jogar para a lua"
local DELETE_CAR_AFTER_MOON = true
local MOON_LIFETIME = 8        -- segundos atÃ© o carro sumir

local remote = ReplicatedStorage:FindFirstChild("AdminAction")
if not remote then
	remote = Instance.new("RemoteEvent")
	remote.Name = "AdminAction"
	remote.Parent = ReplicatedStorage
end

local function isAdmin(player)
	return ADMINS[player.UserId] == true or RunService:IsStudio()
end

Players.PlayerAdded:Connect(function(player)
	player:SetAttribute("IsAdmin", isAdmin(player))
end)
for _, p in ipairs(Players:GetPlayers()) do
	p:SetAttribute("IsAdmin", isAdmin(p))
end

-- Acha o carro em que o jogador estÃ¡ sentado ou o mais prÃ³ximo
local function findCarSeat(player)
	local char = player.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	local hrp = char and char:FindFirstChild("HumanoidRootPart")
	if not (hum and hrp) then return nil end

	if hum.SeatPart and hum.SeatPart:IsA("VehicleSeat") then
		return hum.SeatPart
	end

	local best, bestDist = nil, CAR_SEARCH_RADIUS
	for _, inst in ipairs(workspace:GetDescendants()) do
		if inst:IsA("VehicleSeat") then
			local d = (inst.Position - hrp.Position).Magnitude
			if d < bestDist then
				best, bestDist = inst, d
			end
		end
	end
	return best
end

local function launchCar(player, speed, antiGravity)
	local seat = findCarSeat(player)
	if not seat then return end

	local root = seat.AssemblyRootPart or seat
	pcall(function() root:SetNetworkOwner(nil) end)

	root.AssemblyLinearVelocity = Vector3.new(0, speed, 0)

	if antiGravity then
		-- Cancela a gravidade para o carro continuar subindo
		local att = Instance.new("Attachment")
		att.Parent = root
		local force = Instance.new("VectorForce")
		force.Attachment0 = att
		force.RelativeTo = Enum.ActuatorRelativeTo.World
		force.ApplyAtCenterOfMass = true
		force.Force = Vector3.new(0, root.AssemblyMass * workspace.Gravity, 0)
		force.Parent = root

		if DELETE_CAR_AFTER_MOON then
			local model = seat:FindFirstAncestorOfClass("Model")
			if model and model ~= workspace then
				Debris:AddItem(model, MOON_LIFETIME)
			end
		else
			Debris:AddItem(force, MOON_LIFETIME)
			Debris:AddItem(att, MOON_LIFETIME)
		end
	end
end

local function teleportToPlayer(admin, targetName)
	if typeof(targetName) ~= "string" or #targetName == 0 then return end
	targetName = targetName:lower()

	local target
	for _, p in ipairs(Players:GetPlayers()) do
		if p ~= admin and (p.Name:lower():sub(1, #targetName) == targetName
			or p.DisplayName:lower():sub(1, #targetName) == targetName) then
			target = p
			break
		end
	end
	if not target then return end

	local myHRP = admin.Character and admin.Character:FindFirstChild("HumanoidRootPart")
	local theirHRP = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
	if myHRP and theirHRP then
		myHRP.CFrame = theirHRP.CFrame * CFrame.new(0, 0, 5)
	end
end

remote.OnServerEvent:Connect(function(player, action, arg)
	if not isAdmin(player) then return end -- seguranÃ§a: sÃ³ admins

	if action == "LiftCar" then
		launchCar(player, LIFT_SPEED, false)
	elseif action == "MoonCar" then
		launchCar(player, MOON_SPEED, true)
	elseif action == "TeleportToPlayer" then
		teleportToPlayer(player, arg)
	end
end)


--====================================================================
-- PARTE 2: LOCALSCRIPT (StarterPlayerScripts/AdminMenuClient)
--====================================================================

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local remote = ReplicatedStorage:WaitForChild("AdminAction")

-- Espera o servidor confirmar que vocÃª Ã© admin
local t = 0
while player:GetAttribute("IsAdmin") == nil and t < 5 do
	t += task.wait(0.1)
end
if not player:GetAttribute("IsAdmin") then return end

local TOGGLE_KEY = Enum.KeyCode.M
local AIM_FOV_PIXELS = 250
local AIM_SMOOTHNESS = 0.35 -- 0 a 1 (maior = mais rÃ¡pido)

local espEnabled = false
local aimEnabled = false

-- ============ GUI ============
local gui = Instance.new("ScreenGui")
gui.Name = "AdminMenu"
gui.ResetOnSpawn = false
gui.Enabled = false
gui.Parent = player:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.fromOffset(240, 330)
frame.Position = UDim2.fromOffset(20, 120)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
frame.BorderSizePixel = 0
frame.Active = true
frame.Draggable = true
frame.Parent = gui
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 10)

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 36)
title.BackgroundTransparency = 1
title.Text = "Menu Admin  [M]"
title.TextColor3 = Color3.new(1, 1, 1)
title.Font = Enum.Font.GothamBold
title.TextSize = 16
title.Parent = frame

local layout = Instance.new("UIListLayout")
layout.Padding = UDim.new(0, 8)
layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
layout.Parent = frame

local pad = Instance.new("UIPadding")
pad.PaddingTop = UDim.new(0, 40)
pad.Parent = frame

local function makeButton(text, callback)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(1, -24, 0, 36)
	b.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
	b.TextColor3 = Color3.new(1, 1, 1)
	b.Font = Enum.Font.Gotham
	b.TextSize = 14
	b.Text = text
	b.AutoButtonColor = true
	b.Parent = frame
	Instance.new("UICorner", b).CornerRadius = UDim.new(0, 8)
	b.MouseButton1Click:Connect(function() callback(b) end)
	return b
end

local function setToggleVisual(button, label, on)
	button.Text = label .. (on and ": ON" or ": OFF")
	button.BackgroundColor3 = on and Color3.fromRGB(40, 130, 70) or Color3.fromRGB(50, 50, 60)
end

-- ============ Carro ============
makeButton("Levantar carro", function()
	remote:FireServer("LiftCar")
end)

makeButton("Jogar carro para a Lua", function()
	remote:FireServer("MoonCar")
end)

-- ============ ESP (Highlight) ============
local highlights = {}

local function addESP(plr)
	if plr == player then return end
	local char = plr.Character
	if not char then return end
	if highlights[plr] then highlights[plr]:Destroy() end

	local h = Instance.new("Highlight")
	h.FillColor = Color3.fromRGB(255, 60, 60)
	h.OutlineColor = Color3.new(1, 1, 1)
	h.FillTransparency = 0.6
	h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
	h.Adornee = char
	h.Parent = char
	highlights[plr] = h
end

local function removeESP(plr)
	if highlights[plr] then
		highlights[plr]:Destroy()
		highlights[plr] = nil
	end
end

local function refreshESP()
	for _, plr in ipairs(Players:GetPlayers()) do
		if espEnabled then addESP(plr) else removeESP(plr) end
	end
end

local function hookPlayer(plr)
	plr.CharacterAdded:Connect(function()
		task.wait(0.5)
		if espEnabled then addESP(plr) end
	end)
end
for _, plr in ipairs(Players:GetPlayers()) do hookPlayer(plr) end
Players.PlayerAdded:Connect(hookPlayer)
Players.PlayerRemoving:Connect(removeESP)

local espBtn
espBtn = makeButton("ESP: OFF", function()
	espEnabled = not espEnabled
	setToggleVisual(espBtn, "ESP", espEnabled)
	refreshESP()
end)

-- ============ Aimbot (segure o botÃ£o direito do mouse) ============
local aimBtn
aimBtn = makeButton("Aimbot: OFF", function()
	aimEnabled = not aimEnabled
	setToggleVisual(aimBtn, "Aimbot", aimEnabled)
end)

local function getClosestTarget()
	local mousePos = UserInputService:GetMouseLocation()
	local best, bestDist = nil, AIM_FOV_PIXELS

	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { player.Character }

	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= player and plr.Character then
			local head = plr.Character:FindFirstChild("Head")
			local hum = plr.Character:FindFirstChildOfClass("Humanoid")
			if head and hum and hum.Health > 0 then
				local screenPos, onScreen = camera:WorldToViewportPoint(head.Position)
				if onScreen then
					local d = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
					if d < bestDist then
						-- checa se nÃ£o tem parede no meio
						local dir = head.Position - camera.CFrame.Position
						local hit = workspace:Raycast(camera.CFrame.Position, dir, params)
						if not hit or hit.Instance:IsDescendantOf(plr.Character) then
							best, bestDist = head, d
						end
					end
				end
			end
		end
	end
	return best
end

RunService.RenderStepped:Connect(function()
	if not aimEnabled then return end
	if not UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then return end

	local target = getClosestTarget()
	if target then
		local goal = CFrame.lookAt(camera.CFrame.Position, target.Position)
		camera.CFrame = camera.CFrame:Lerp(goal, AIM_SMOOTHNESS)
	end
end)

-- ============ TP atÃ© jogador ============
local box = Instance.new("TextBox")
box.Size = UDim2.new(1, -24, 0, 32)
box.BackgroundColor3 = Color3.fromRGB(40, 40, 48)
box.TextColor3 = Color3.new(1, 1, 1)
box.PlaceholderText = "Nome do jogador"
box.PlaceholderColor3 = Color3.fromRGB(150, 150, 160)
box.Font = Enum.Font.Gotham
box.TextSize = 14
box.Text = ""
box.ClearTextOnFocus = false
box.Parent = frame
Instance.new("UICorner", box).CornerRadius = UDim.new(0, 8)

makeButton("TP atÃ© jogador", function()
	remote:FireServer("TeleportToPlayer", box.Text)
end)

-- ============ Abrir/fechar ============
UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.KeyCode == TOGGLE_KEY then
		gui.Enabled = not gui.Enabled
	end
end)
