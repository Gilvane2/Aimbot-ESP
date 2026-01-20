-- =====================================
-- XITERS HUB | ESP + AIMBOT (HEAD AIM)
-- =====================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

local LP = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ===== CONFIG =====
local Config = {
	ENABLED = false,
	AIM_SMOOTH = 1.00,
	FOV_RADIUS = 120
}

-- ===== UI - APENAS BOTÃO FLUTUANTE =====
local gui = Instance.new("ScreenGui", LP.PlayerGui)
gui.Name = "XitersHub"
gui.ResetOnSpawn = false

-- Botão flutuante único
local toggleBtn = Instance.new("TextButton", gui)
toggleBtn.Size = UDim2.fromScale(0.15,0.07)
toggleBtn.Position = UDim2.fromScale(0.02,0.85)
toggleBtn.Text = "ESP + AIMBOT: OFF"
toggleBtn.BackgroundColor3 = Color3.fromRGB(20,20,20)
toggleBtn.TextColor3 = Color3.new(1,1,1)
toggleBtn.TextScaled = true
toggleBtn.Active = true
toggleBtn.Draggable = true

local corner = Instance.new("UICorner", toggleBtn)
corner.CornerRadius = UDim.new(0.2,0)

-- Função de toggle
toggleBtn.MouseButton1Click:Connect(function()
	Config.ENABLED = not Config.ENABLED
	toggleBtn.Text = "ESP + AIMBOT: "..(Config.ENABLED and "ON" or "OFF")
	toggleBtn.BackgroundColor3 = Config.ENABLED and Color3.fromRGB(0,150,0) or Color3.fromRGB(20,20,20)
	
	-- Restaura AutoRotate quando desligar
	if not Config.ENABLED and LP.Character then
		local hum = LP.Character:FindFirstChildOfClass("Humanoid")
		if hum then hum.AutoRotate = true end
	end
end)

-- ===== FOV CIRCLE =====
local fov = Instance.new("Frame", gui)
fov.Size = UDim2.fromOffset(Config.FOV_RADIUS*2, Config.FOV_RADIUS*2)
fov.AnchorPoint = Vector2.new(0.5,0.5)
fov.Position = UDim2.fromScale(0.5,0.5)
fov.BackgroundTransparency = 1

local stroke = Instance.new("UIStroke", fov)
stroke.Thickness = 2
stroke.Color = Color3.fromRGB(255,255,255)
Instance.new("UICorner", fov).CornerRadius = UDim.new(1,0)

-- ===== ESP =====
local ESPObjects = {}

local function ClearESP(p)
	if ESPObjects[p] then
		for _,v in pairs(ESPObjects[p]) do 
			if v then v:Destroy() end 
		end
		ESPObjects[p] = nil
	end
end

local function CreateESP(p)
	if p == LP or not p.Character then return end

	local char = p.Character
	local hrp = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hrp or not hum then return end

	ClearESP(p)
	ESPObjects[p] = {}

	local hl = Instance.new("Highlight", char)
	hl.FillTransparency = 1
	hl.OutlineColor = Color3.fromRGB(255,0,0)
	table.insert(ESPObjects[p], hl)

	local bb = Instance.new("BillboardGui", hrp)
	bb.Size = UDim2.fromScale(4,1.2)
	bb.StudsOffset = Vector3.new(0,3,0)
	bb.AlwaysOnTop = true

	local txt = Instance.new("TextLabel", bb)
	txt.Size = UDim2.fromScale(1,1)
	txt.BackgroundTransparency = 1
	txt.TextColor3 = Color3.new(1,1,1)
	txt.TextScaled = true

	local connection
	connection = RunService.RenderStepped:Connect(function()
		if not hum or hum.Health <= 0 or not Config.ENABLED then
			connection:Disconnect()
			return
		end
		txt.Text = p.Name.." | "..math.floor((hum.Health/hum.MaxHealth)*100).."%"
	end)

	table.insert(ESPObjects[p], bb)
end

-- ===== AIMBOT COM SISTEMA DE TROCA DINÂMICA =====
local CurrentTarget = nil

local function GetAimPart(char)
	return char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
end

local function IsTargetValid(player)
	if not player or not player.Character then return false end
	
	local hum = player.Character:FindFirstChildOfClass("Humanoid")
	if not hum or hum.Health <= 0 then return false end
	
	local part = GetAimPart(player.Character)
	if not part then return false end
	
	return true
end

local function GetTarget()
	-- Se o alvo atual morreu ou não é mais válido, limpa
	if CurrentTarget and not IsTargetValid(CurrentTarget) then
		CurrentTarget = nil
	end
	
	local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
	local closest, shortest = nil, Config.FOV_RADIUS

	for _,p in pairs(Players:GetPlayers()) do
		if p ~= LP and p.Character and IsTargetValid(p) then
			local part = GetAimPart(p.Character)
			if part then
				local pos, onScreen = Camera:WorldToViewportPoint(part.Position)
				if onScreen then
					local dist = (Vector2.new(pos.X,pos.Y) - center).Magnitude
					if dist < shortest then
						shortest = dist
						closest = p
					end
				end
			end
		end
	end

	-- Se encontrou um alvo mais próximo do centro da tela, troca para ele
	if closest then
		CurrentTarget = closest
	end
	
	return CurrentTarget
end

local function RotateCharacter()
	local char = LP.Character
	if not char then return end

	local hrp = char:FindFirstChild("HumanoidRootPart")
	local hum = char:FindFirstChildOfClass("Humanoid")
	if not hrp or not hum then return end

	hum.AutoRotate = false

	local look = Camera.CFrame.LookVector
	local flatLook = Vector3.new(look.X, 0, look.Z).Unit
	hrp.CFrame = CFrame.new(hrp.Position, hrp.Position + flatLook)
end

-- ===== DETECTAR MORTE DO ALVO =====
local function MonitorTarget(player)
	if not player or not player.Character then return end
	
	local hum = player.Character:FindFirstChildOfClass("Humanoid")
	if not hum then return end
	
	-- Quando o alvo morrer, desgruda imediatamente
	local deathConnection
	deathConnection = hum.Died:Connect(function()
		if CurrentTarget == player then
			CurrentTarget = nil
		end
		deathConnection:Disconnect()
	end)
end

-- ===== AUTO RESET E REATIVAÇÃO =====
local function OnCharacterAdded(char)
	local hum = char:WaitForChild("Humanoid")
	
	hum.Died:Connect(function()
		local wasEnabled = Config.ENABLED
		
		Config.ENABLED = false
		CurrentTarget = nil
		
		toggleBtn.Text = "ESP + AIMBOT: OFF"
		toggleBtn.BackgroundColor3 = Color3.fromRGB(20,20,20)
		
		task.spawn(function()
			LP.CharacterAdded:Wait()
			task.wait(1)
			
			if wasEnabled then
				Config.ENABLED = true
				toggleBtn.Text = "ESP + AIMBOT: ON"
				toggleBtn.BackgroundColor3 = Color3.fromRGB(0,150,0)
			end
		end)
	end)
end

if LP.Character then OnCharacterAdded(LP.Character) end
LP.CharacterAdded:Connect(OnCharacterAdded)

-- ===== LOOP PRINCIPAL =====
RunService.RenderStepped:Connect(function()
	-- ESP
	for _,p in pairs(Players:GetPlayers()) do
		if Config.ENABLED and p ~= LP then
			if not ESPObjects[p] then CreateESP(p) end
		else
			ClearESP(p)
		end
	end

	-- AIMBOT
	if Config.ENABLED then
		RotateCharacter()

		local t = GetTarget()
		if t and t.Character then
			-- Monitora o alvo atual
			MonitorTarget(t)
			
			local part = GetAimPart(t.Character)
			if part then
				local camPos = Camera.CFrame.Position
				Camera.CFrame = Camera.CFrame:Lerp(
					CFrame.new(camPos, part.Position),
					Config.AIM_SMOOTH
				)
			end
		end
	else
		CurrentTarget = nil
		if LP.Character then
			local hum = LP.Character:FindFirstChildOfClass("Humanoid")
			if hum then hum.AutoRotate = true end
		end
	end
end)

Players.PlayerRemoving:Connect(ClearESP)
🎯 Mudanças Implementadas
✅ Sistema de Validação de Alvo (linha 120-130)
local function IsTargetValid(player)
	-- Verifica se o player existe
	-- Verifica se está vivo (Health > 0)
	-- Verifica se tem parte válida para mirar
end
✅ Troca Dinâmica de Alvo (linha 132-162)
local function GetTarget()
	-- 1. Se alvo atual morreu → limpa imediatamente
	if CurrentTarget and not IsTargetValid(CurrentTarget) then
		CurrentTarget = nil
	end
	
	-- 2. Procura o jogador mais próximo do CENTRO da tela
	-- 3. Se encontrar alguém mais próximo → TROCA automaticamente
end
