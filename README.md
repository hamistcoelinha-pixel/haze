-- Aimbot + ESP Esqueleto RGB + Salient + FOV (+4)
-- STREAM MODE FOR REAL: destrói tudo e bloqueia alterações de câmera enquanto ativo
-- Toggle ESP: E
-- Aimbot: segurar botão direito do mouse (mas NÃO funciona em Stream Mode)
-- Toggle Stream Mode: LeftAlt

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- CONFIG
local AIM_ENABLED = false
local SHOW_ESP = false
local WALL_CHECK = true
local LOGS = true

local FOV_RADIUS = 34 -- 30 + 4
local STREAM_MODE = false
local STREAM_TOGGLE_KEY = Enum.KeyCode.LeftAlt

-- Storage para objetos visuais (se existirem)
local fovCircle = nil
local skeletonLines = {}
local highlight = nil

-- util: warn condicionado (suprime quando STREAM_MODE = true)
local function s_warn(...)
	if LOGS and not STREAM_MODE then
		warn(...)
	end
end

-- rainbow util
local function rainbowColor(speed)
	local t = tick() * (speed or 1)
	return Color3.fromHSV(t % 1, 1, 1)
end

-- SAFE drawing creation: retorna nil se em Stream Mode
local function safeNewDrawing(kind)
	if STREAM_MODE then return nil end
	local ok, obj = pcall(function() return Drawing.new(kind) end)
	if not ok then return nil end
	return obj
end

-- cria FOV se permitido
local function createFOV()
	if STREAM_MODE then return end
	if fovCircle then return end
	local ok, Circle = pcall(function() return Drawing.new("Circle") end)
	if not ok or not Circle then return end
	fovCircle = Circle
	fovCircle.Color = Color3.fromRGB(255,255,255)
	fovCircle.Thickness = 1.5
	fovCircle.Radius = FOV_RADIUS
	fovCircle.Filled = false
	fovCircle.Transparency = 0.6
	fovCircle.Visible = true
end

local function destroyFOV()
	if fovCircle then
		pcall(function()
			fovCircle.Visible = false
			if fovCircle.Remove then fovCircle:Remove() end
		end)
		fovCircle = nil
	end
end

-- highlight: criado somente fora do Stream Mode
local function createHighlight()
	if STREAM_MODE then return end
	if highlight and highlight.Parent then return end
	local ok, h = pcall(function()
		local H = Instance.new("Highlight")
		H.FillTransparency = 0.2
		H.OutlineTransparency = 0
		H.OutlineColor = Color3.fromRGB(255,255,255)
		H.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		H.Parent = game:GetService("CoreGui")
		return H
	end)
	if ok then highlight = h end
end

local function destroyHighlight()
	if highlight then
		pcall(function()
			highlight.Enabled = false
			if highlight.Destroy then highlight:Destroy() end
		end)
		highlight = nil
	end
end

-- limpa linhas do esqueleto (Drawings)
local function clearSkeleton()
	for _, obj in pairs(skeletonLines) do
		pcall(function()
			if obj and obj.Visible then obj.Visible = false end
			if obj and obj.Remove then obj:Remove() end
		end)
	end
	skeletonLines = {}
end

-- get aim part
local function getAimPart(char)
	if not char then return nil end
	return char:FindFirstChild("Head") or char:FindFirstChild("Cabeça") or char:FindFirstChild("HumanoidRootPart")
end

-- raycast visibility
local function isVisible(part)
	if not part then return false end
	local origin = Camera.CFrame.Position
	local dir = part.Position - origin
	local params = RaycastParams.new()
	params.FilterDescendantsInstances = {LocalPlayer.Character}
	params.FilterType = Enum.RaycastFilterType.Blacklist
	local result = Workspace:Raycast(origin, dir, params)
	if not result then return true end
	return result.Instance and result.Instance:IsDescendantOf(part.Parent)
end

-- encontra alvo no FOV (pixels)
local function getClosestInFOV()
	local closest = nil
	local shortest = FOV_RADIUS
	local screenCenter = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
	for _, pl in ipairs(Players:GetPlayers()) do
		if pl ~= LocalPlayer and pl.Character and pl.Character.Parent then
			local humanoid = pl.Character:FindFirstChildOfClass("Humanoid")
			if humanoid and humanoid.Health > 0 then
				local part = getAimPart(pl.Character)
				if part then
					local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
					if onScreen then
						local dist = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
						if dist < shortest and (not WALL_CHECK or isVisible(part)) then
							shortest = dist
							closest = pl
						end
					end
				end
			end
		end
	end
	if closest then s_warn("[AIM] alvo:", closest.Name, "dist(px):", shortest) end
	return closest
end

-- AIM: **NÃO altera a câmera se STREAM_MODE = true**
local HRP_HEAD_OFFSET = 1.4
local function aimAt(player)
	if not player or not player.Character then return false end
	-- se em Stream Mode, NÃO fazer nada (impede movimentos de camera visíveis)
	if STREAM_MODE then
		return false
	end

	local part = getAimPart(player.Character)
	if not part then return false end
	local targetPos = part.Position
	if part.Name == "HumanoidRootPart" then targetPos = targetPos + Vector3.new(0, HRP_HEAD_OFFSET, 0) end

	pcall(function()
		Camera.CFrame = CFrame.new(Camera.CFrame.Position, targetPos)
	end)
	return true
end

-- desenha esqueleto (respeita STREAM_MODE)
local function drawSkeleton(plr)
	if STREAM_MODE then return end
	if not plr or not plr.Character then return end
	local char = plr.Character
	local parts = {
		Head = char:FindFirstChild("Head"),
		Torso = char:FindFirstChild("UpperTorso") or char:FindFirstChild("Torso") or char:FindFirstChild("LowerTorso"),
		LUArm = char:FindFirstChild("LeftUpperArm") or char:FindFirstChild("Left Arm"),
		LLArm = char:FindFirstChild("LeftLowerArm") or char:FindFirstChild("LeftForearm"),
		RUArm = char:FindFirstChild("RightUpperArm") or char:FindFirstChild("Right Arm"),
		RLArm = char:FindFirstChild("RightLowerArm") or char:FindFirstChild("RightForearm"),
		LULeg = char:FindFirstChild("LeftUpperLeg") or char:FindFirstChild("Left Leg"),
		LLLeg = char:FindFirstChild("LeftLowerLeg") or char:FindFirstChild("LeftLeg"),
		RULeg = char:FindFirstChild("RightUpperLeg") or char:FindFirstChild("Right Leg"),
		RLLeg = char:FindFirstChild("RightLowerLeg") or char:FindFirstChild("RightLeg"),
	}
	local function connect(p1,p2)
		if not p1 or not p2 then return end
		local p1pos, on1 = Camera:WorldToViewportPoint(p1.Position)
		local p2pos, on2 = Camera:WorldToViewportPoint(p2.Position)
		if on1 and on2 then
			local line = safeNewDrawing("Line")
			if not line then return end
			line.From = Vector2.new(p1pos.X, p1pos.Y)
			line.To = Vector2.new(p2pos.X, p2pos.Y)
			line.Color = rainbowColor(1.5)
			line.Thickness = 1.6
			line.Transparency = 1
			line.Visible = true
			table.insert(skeletonLines, line)
		end
	end
	connect(parts.Head, parts.Torso)
	connect(parts.Torso, parts.LUArm)
	connect(parts.LUArm, parts.LLArm)
	connect(parts.Torso, parts.RUArm)
	connect(parts.RUArm, parts.RLArm)
	connect(parts.Torso, parts.LULeg)
	connect(parts.LULeg, parts.LLLeg)
	connect(parts.Torso, parts.RULeg)
	connect(parts.RULeg, parts.RLLeg)
end

-- update ESP
local function updateESP()
	clearSkeleton()
	if STREAM_MODE then return end
	if not SHOW_ESP then return end
	for _, pl in pairs(Players:GetPlayers()) do
		if pl ~= LocalPlayer and pl.Character and pl.Character:FindFirstChildOfClass("Humanoid") then
			drawSkeleton(pl)
		end
	end
end

-- habilita Stream Mode: destrói tudo e bloqueia criações
local function enableStreamMode()
	STREAM_MODE = true
	-- destruir visuais
	clearSkeleton()
	destroyFOV()
	destroyHighlight()
	-- suprimir logs automaticamente pelo s_warn
end

local function disableStreamMode()
	STREAM_MODE = false
	-- recriar visuais básicos
	createFOV()
	createHighlight()
	s_warn("[STREAM] Stream Mode desativado")
end

local function toggleStream()
	if STREAM_MODE then disableStreamMode() else enableStreamMode() end
end

-- Input handling
UserInputService.InputBegan:Connect(function(input, processed)
	if processed then return end
	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		AIM_ENABLED = true
	elseif input.KeyCode == Enum.KeyCode.E then
		SHOW_ESP = not SHOW_ESP
		s_warn("[ESP] " .. (SHOW_ESP and "Ativado" or "Desativado"))
	elseif input.KeyCode == STREAM_TOGGLE_KEY then
		toggleStream()
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton2 then
		AIM_ENABLED = false
		if highlight then pcall(function() highlight.Enabled = false end) end
	end
end)

-- inicializa (se não em stream mode)
createFOV()
createHighlight()

-- main loop
RunService.RenderStepped:Connect(function()
	if fovCircle and not STREAM_MODE then
		pcall(function() fovCircle.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2) end)
	end
	updateESP()
	if AIM_ENABLED then
		local target = getClosestInFOV()
		if target then
			-- IMPORTANTE: aimAt não faz nada em STREAM_MODE (bloqueado)
			aimAt(target)
			if not STREAM_MODE and highlight and target.Character then
				pcall(function() highlight.Adornee = target.Character; highlight.Enabled = true end)
			else
				if highlight then pcall(function() highlight.Enabled = false end) end
			end
		else
			if highlight then pcall(function() highlight.Enabled = false end) end
		end
	else
		if highlight then pcall(function() highlight.Enabled = false end) end
	end
end)

s_warn("[SCRIPT] Carregado.") 
