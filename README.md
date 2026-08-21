-- 👤 HoleStalker
-- só sala atual com mesa | spawn aleatório RARO
-- /holestalker

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local IMAGE = "rbxassetid://71101552694513"
local HIT_SOUND = "rbxassetid://134262192520482"
local DAMAGE_PER_TICK = 4
local TICK_TIME = 0.35
local TOUCH_DIST = 7

local current = nil
local lookConn = nil
local hitting = false

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function getCurrentRoom()
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil end
	local val = ReplicatedStorage.GameData.LatestRoom.Value
	return rooms:FindFirstChild(tostring(val)) or rooms:FindFirstChild(val)
end

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.0, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.6, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function findTableInCurrentRoom(hrp)
	local room = getCurrentRoom()
	if not room then return nil end

	local candidates = {}
	for _, v in pairs(room:GetDescendants()) do
		if v:IsA("BasePart") then
			local n = v.Name:lower()
			local parentN = v.Parent and v.Parent.Name:lower() or ""

			local isTable =
				n:find("table")
				or n:find("desk")
				or parentN:find("table")
				or parentN:find("desk")

			if isTable and v.Size.Magnitude > 2 then
				table.insert(candidates, v)
			end
		end
	end

	if #candidates == 0 then return nil end

	local pick = candidates[math.random(1, #candidates)]
	local pos = pick.Position - Vector3.new(0, pick.Size.Y * 0.5 + 1.0, 0)
	return pos, pick
end

local function cleanup()
	if lookConn then
		lookConn:Disconnect()
		lookConn = nil
	end
	if current and current.Parent then
		current:Destroy()
	end
	current = nil
end

local function onTouch(part)
	if hitting then return end
	hitting = true

	local sound = Instance.new("Sound", Workspace)
	sound.SoundId = HIT_SOUND
	sound.Volume = 4
	sound:Play()

	startDarkFog()

	if lookConn then
		lookConn:Disconnect()
		lookConn = nil
	end
	if part and part.Parent then part:Destroy() end
	if current == part then current = nil end

	task.spawn(function()
		while sound and sound.Parent and sound.IsPlaying do
			local h = getHumanoid()
			if h and h.Health > 0 then
				h:TakeDamage(DAMAGE_PER_TICK)
			end
			task.wait(TICK_TIME)
		end

		if sound and sound.Parent and sound.IsPlaying then
			sound.Ended:Wait()
		end

		task.wait(0.15)
		endDarkFog()
		if sound and sound.Parent then sound:Destroy() end
		hitting = false
	end)

	task.spawn(function()
		task.wait(0.25)
		local len = 8
		pcall(function()
			if sound.TimeLength and sound.TimeLength > 0.5 then
				len = sound.TimeLength + 0.4
			end
		end)
		task.wait(len)
		if hitting then
			pcall(function()
				if sound and sound.Parent then
					sound:Stop()
					sound:Destroy()
				end
			end)
			endDarkFog()
			hitting = false
		end
	end)
end

local function spawnHoleStalker()
	if hitting or current then return end
	cleanup()

	local hrp = getHRP()
	if not hrp then return end

	local pos, tablePart = findTableInCurrentRoom(hrp)
	if not pos then
		return -- sala sem mesa
	end

	print("HoleStalker spawnou na mesa!")

	local part = Instance.new("Part")
	part.Name = "HoleStalker"
	part.Size = Vector3.new(5.5, 6.5, 0.22)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = CFrame.new(pos, hrp.Position)
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGE
		d.Face = face
		d.Parent = part
	end

	lookConn = RunService.RenderStepped:Connect(function()
		if not part or not part.Parent then
			if lookConn then lookConn:Disconnect() lookConn = nil end
			return
		end
		local r = getHRP()
		if r then
			local p = part.Position
			part.CFrame = CFrame.new(p, Vector3.new(r.Position.X, p.Y, r.Position.Z))

			if not hitting and (p - r.Position).Magnitude < TOUCH_DIST then
				onTouch(part)
			end
		end
	end)

	task.delay(22, function()
		if current == part and not hitting then
			cleanup()
		end
	end)
end

-- aleatório BEM RARO: ao abrir porta, chance baixa + delay
task.spawn(function()
	while true do
		ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
		task.wait(math.random(8, 18))
		-- chance bem baixa
		if not current and not hitting and math.random(1, 12) == 1 then
			spawnHoleStalker()
		end
	end
end)

player.Chatted:Connect(function(msg)
	local m = msg:lower():gsub("%s+", "")
	if m == "/holestalker" or m == "holestalker" then
		spawnHoleStalker()
	end
end)

print("✅ HoleStalker | aleatório RARO (1/12 por porta) | só com mesa")
print("/holestalker")


-- 🔍 Zoom Button
-- FOV mais baixo = vê mais longe (zoom pra frente)
-- clica = zoom e FICA | clica de novo = normal

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local camera = Workspace.CurrentCamera

local ZOOM_FOV = 38      -- mais longe / mais alcance (antes 48)
local NORMAL_FOV = 70
local TWEEN_TIME = 0.35

local zoomed = false
local animating = false
local holdConn = nil
local activeTween = nil

pcall(function()
	playerGui:FindFirstChild("ZoomButtonGui"):Destroy()
end)

local gui = Instance.new("ScreenGui")
gui.Name = "ZoomButtonGui"
gui.IgnoreGuiInset = true
gui.ResetOnSpawn = false
gui.DisplayOrder = 60
gui.Parent = playerGui

local btn = Instance.new("TextButton")
btn.Name = "ZoomBtn"
btn.Size = UDim2.new(0, 42, 0, 42)
btn.Position = UDim2.new(1, -58, 0.38, 0)
btn.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
btn.BackgroundTransparency = 0.15
btn.BorderSizePixel = 0
btn.Text = ""
btn.AutoButtonColor = false
btn.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 6)
corner.Parent = btn

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(255, 255, 255)
stroke.Thickness = 1.5
stroke.Transparency = 0.55
stroke.Parent = btn

local icon = Instance.new("TextLabel")
icon.Name = "Icon"
icon.Size = UDim2.new(1, 0, 1, 0)
icon.BackgroundTransparency = 1
icon.Text = "🔍"
icon.TextSize = 20
icon.Font = Enum.Font.GothamBold
icon.TextColor3 = Color3.fromRGB(230, 230, 230)
icon.Parent = btn

local shadow = Instance.new("Frame")
shadow.Size = UDim2.new(1, 4, 1, 4)
shadow.Position = UDim2.new(0, 2, 0, 2)
shadow.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
shadow.BackgroundTransparency = 0.7
shadow.ZIndex = btn.ZIndex - 1
shadow.Parent = btn
local sc = Instance.new("UICorner")
sc.CornerRadius = UDim.new(0, 6)
sc.Parent = shadow

local function stopHold()
	if holdConn then
		holdConn:Disconnect()
		holdConn = nil
	end
end

local function cancelTween()
	if activeTween then
		pcall(function() activeTween:Cancel() end)
		activeTween = nil
	end
end

local function startHoldZoom()
	stopHold()
	holdConn = RunService.RenderStepped:Connect(function()
		if zoomed and not animating and camera then
			camera.FieldOfView = ZOOM_FOV
		end
	end)
end

local function setZoom(on)
	if animating then return end
	animating = true
	zoomed = on

	stopHold()
	cancelTween()

	local target = on and ZOOM_FOV or NORMAL_FOV

	activeTween = TweenService:Create(camera, TweenInfo.new(TWEEN_TIME, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		FieldOfView = target
	})
	activeTween:Play()

	if on then
		btn.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
		stroke.Transparency = 0.25
		icon.Text = "⬤"
		icon.TextSize = 14
	else
		btn.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
		stroke.Transparency = 0.55
		icon.Text = "🔍"
		icon.TextSize = 20
	end

	activeTween.Completed:Wait()
	activeTween = nil
	animating = false

	if on then
		if camera then camera.FieldOfView = ZOOM_FOV end
		startHoldZoom()
	else
		if camera then camera.FieldOfView = NORMAL_FOV end
	end
end

btn.MouseButton1Click:Connect(function()
	setZoom(not zoomed)
end)

btn.MouseEnter:Connect(function()
	if not zoomed and not animating then
		TweenService:Create(btn, TweenInfo.new(0.12), {
			BackgroundColor3 = Color3.fromRGB(35, 35, 35)
		}):Play()
	end
end)

btn.MouseLeave:Connect(function()
	if not zoomed and not animating then
		TweenService:Create(btn, TweenInfo.new(0.12), {
			BackgroundColor3 = Color3.fromRGB(20, 20, 20)
		}):Play()
	end
end)

player.CharacterAdded:Connect(function()
	task.wait(0.6)
	zoomed = false
	animating = false
	stopHold()
	cancelTween()
	icon.Text = "🔍"
	icon.TextSize = 20
	btn.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
	stroke.Transparency = 0.55
	pcall(function()
		camera.FieldOfView = NORMAL_FOV
	end)
end)

print("✅ Zoom FOV 38 (mais longe) | animado | fixo até clicar de novo")

-- 👤 Stalker Window + Stalker Jumpscare (script único)
-- só spawna depois de 15s NA MESMA SALA
-- /stalkerwindow | /stalkerjumspecare

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local WINDOW_IMAGE = "rbxassetid://87802065154223"
local JUMP_IMAGES = {
	"rbxassetid://111565947739320",
	"rbxassetid://73509060375391",
	"rbxassetid://135807608825423"
}

local windowCurrent = nil
local jumpCurrent = nil
local jumpImageIndex = 1
local jumpTouching = false

local roomEnterTime = tick()
local lastRoom = nil

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

-- tempo na sala atual
local function timeInRoom()
	return tick() - roomEnterTime
end

local function canSpawn()
	return timeInRoom() >= 15
end

-- track sala
ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function(val)
	lastRoom = val
	roomEnterTime = tick()
	print("Nova sala — timer 15s resetado")
end)

-- inicia timer na sala atual
pcall(function()
	lastRoom = ReplicatedStorage.GameData.LatestRoom.Value
	roomEnterTime = tick()
end)

local function isLookingAt(targetPos)
	local camPos = camera.CFrame.Position
	local camLook = camera.CFrame.LookVector
	local dir = (targetPos - camPos)
	if dir.Magnitude > 50 then return false end
	dir = dir.Unit
	return camLook:Dot(dir) > 0.90
end

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

-- ================== WINDOW ==================
local function findWindow()
	local hrp = getHRP()
	if not hrp then return nil end

	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil end

	local candidates = {}
	for _, v in pairs(rooms:GetDescendants()) do
		local name = v.Name:lower()
		if v:IsA("BasePart") and (name:find("window") or name:find("glass") or name:find("pane")) then
			if (v.Position - hrp.Position).Magnitude < 80 then
				table.insert(candidates, v)
			end
		end
	end

	if #candidates > 0 then
		return candidates[math.random(1, #candidates)]
	end
	return nil
end

local function spawnWindow()
	if windowCurrent then return end
	if not canSpawn() then
		print("StalkerWindow: espera 15s na sala (" .. math.floor(timeInRoom()) .. "s)")
		return
	end

	local hrp = getHRP()
	if not hrp then return end

	local window = findWindow()
	if not window then
		print("StalkerWindow: sem janela")
		return
	end

	local spawnCF = CFrame.new(window.Position + Vector3.new(0, 0, 1.2), hrp.Position)

	local part = Instance.new("Part")
	part.Name = "StalkerWindow"
	part.Size = Vector3.new(6, 8, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace
	windowCurrent = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = WINDOW_IMAGE
		d.Face = face
		d.Parent = part
	end

	print("StalkerWindow na janela!")

	local fixedPos = part.Position
	local looked = false

	task.spawn(function()
		while part.Parent and not looked do
			part.CFrame = CFrame.new(fixedPos, camera.CFrame.Position)

			if isLookingAt(fixedPos) then
				looked = true
				local downPos = fixedPos + Vector3.new(0, -14, 0)
				local tween = TweenService:Create(part, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
					Position = downPos
				})
				tween:Play()
				tween.Completed:Wait()
				if part and part.Parent then part:Destroy() end
				windowCurrent = nil
				print("StalkerWindow sumiu pra baixo!")
				break
			end
			task.wait(0.05)
		end
	end)
end

-- ================== JUMPSCARE PORTA ==================
local function findDoorFrontOrBack()
	local hrp = getHRP()
	if not hrp then return nil, nil end

	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil, nil end

	local frontDoors, backDoors = {}, {}
	local look = hrp.CFrame.LookVector

	for _, v in pairs(rooms:GetDescendants()) do
		if not v:IsA("BasePart") then continue end

		local name = v.Name:lower()
		local parentName = v.Parent and v.Parent.Name:lower() or ""
		local full = name .. " " .. parentName

		if full:find("wardrobe") or full:find("locker") or full:find("closet") or full:find("cabinet") then
			continue
		end

		local isDoor = name == "door" or name:find("door") or parentName:find("door")
		if not isDoor then continue end
		if v.Size.Magnitude < 4 then continue end

		local dist = (v.Position - hrp.Position).Magnitude
		if dist < 5 or dist > 55 then continue end

		local toDoor = (v.Position - hrp.Position).Unit
		local dot = look:Dot(toDoor)

		if dot > 0.1 then
			table.insert(frontDoors, {part = v, dist = dist})
		elseif dot < -0.1 then
			table.insert(backDoors, {part = v, dist = dist})
		else
			table.insert(frontDoors, {part = v, dist = dist})
		end
	end

	local function pickClosest(list)
		table.sort(list, function(a, b) return a.dist < b.dist end)
		return list[1] and list[1].part or nil
	end

	if math.random(1, 2) == 1 then
		local d = pickClosest(frontDoors)
		if d then return d, "frente" end
		d = pickClosest(backDoors)
		if d then return d, "trás" end
	else
		local d = pickClosest(backDoors)
		if d then return d, "trás" end
		d = pickClosest(frontDoors)
		if d then return d, "frente" end
	end

	return nil, nil
end

local function spawnJumpscare()
	if jumpCurrent then return end
	if not canSpawn() then
		print("StalkerJumpscare: espera 15s na sala (" .. math.floor(timeInRoom()) .. "s)")
		return
	end

	local hrp = getHRP()
	if not hrp then return end

	jumpTouching = false

	local door, side = findDoorFrontOrBack()
	local spawnCF

	if door then
		local mid = door.Position + Vector3.new(0, 1.3, 0)
		local sideOff = door.CFrame.RightVector * 1.1
		local towardPlayer = (hrp.Position - door.Position)
		if towardPlayer.Magnitude > 0.1 then
			towardPlayer = towardPlayer.Unit * 0.8
		else
			towardPlayer = Vector3.zero
		end
		spawnCF = CFrame.new(mid + sideOff + towardPlayer, hrp.Position)
		print("Jumpscare na PORTA (" .. (side or "?") .. ")")
	else
		if math.random(1, 2) == 1 then
			spawnCF = hrp.CFrame * CFrame.new(1.2, 3, -8)
		else
			spawnCF = hrp.CFrame * CFrame.new(1.2, 3, 11)
		end
		print("Jumpscare fallback")
	end

	local img = JUMP_IMAGES[jumpImageIndex]
	jumpImageIndex = jumpImageIndex + 1
	if jumpImageIndex > #JUMP_IMAGES then jumpImageIndex = 1 end

	local part = Instance.new("Part")
	part.Name = "StalkerJumpscareDoor"
	part.Size = Vector3.new(6, 9, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace
	jumpCurrent = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = img
		d.Face = face
		d.Parent = part
	end

	task.spawn(function()
		while part.Parent do
			local root = getHRP()
			if root then
				part.CFrame = CFrame.new(part.Position, root.Position)

				if not jumpTouching and (part.Position - root.Position).Magnitude < 9 then
					jumpTouching = true
					jumpCurrent = nil
					startDarkFog()
					part:Destroy()
					task.wait(2.5)
					endDarkFog()
					jumpTouching = false
					break
				end
			end
			task.wait(0.04)
		end
	end)
end

-- spawn aleatório: só se já está 15s+ na sala
task.spawn(function()
	while true do
		task.wait(math.random(20, 40))
		if canSpawn() then
			if not windowCurrent and math.random(1, 3) == 1 then
				spawnWindow()
			elseif not jumpCurrent and math.random(1, 3) == 1 then
				spawnJumpscare()
			end
		end
	end
end)

player.Chatted:Connect(function(msg)
	local m = msg:lower()
	if m == "/stalkerwindow" then
		spawnWindow()
	elseif m == "/stalkerjumspecare" then
		spawnJumpscare()
	end
end)

print("✅ Window + Jumpscare juntos | só após 15s na mesma sala")
print("/stalkerwindow | /stalkerjumspecare")

-- 📜👤 Stalker Mod INTRO + STALKER + SEEK MUSIC (tudo junto)

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera
local humanoid = char:WaitForChild("Humanoid")

player.CharacterAdded:Connect(function(c)
	char = c
	hrp = c:WaitForChild("HumanoidRootPart")
	humanoid = c:WaitForChild("Humanoid")
end)

-- ================== INTRO ==================
local INTRO_SOUND = "rbxassetid://138466523093279"
local introShowed = false

local function showIntro()
	if introShowed then return end
	introShowed = true

	pcall(function()
		playerGui:FindFirstChild("StalkerModIntro"):Destroy()
	end)

	local sound = Instance.new("Sound", Workspace)
	sound.Name = "StalkerIntroSound"
	sound.SoundId = INTRO_SOUND
	sound.Volume = 3
	sound.Looped = false
	sound:Play()
	Debris:AddItem(sound, 20)

	local gui = Instance.new("ScreenGui")
	gui.Name = "StalkerModIntro"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 150
	gui.ResetOnSpawn = false
	gui.Parent = playerGui

	local credits = Instance.new("TextLabel")
	credits.AnchorPoint = Vector2.new(1, 0)
	credits.Position = UDim2.new(0.98, 0, 0.22, 0)
	credits.Size = UDim2.new(0.42, 0, 0.6, 0)
	credits.BackgroundTransparency = 1
	credits.TextColor3 = Color3.fromRGB(255, 255, 255)
	credits.TextStrokeTransparency = 0.5
	credits.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	credits.Font = Enum.Font.Gotham
	credits.TextSize = 15
	credits.TextXAlignment = Enum.TextXAlignment.Right
	credits.TextYAlignment = Enum.TextYAlignment.Top
	credits.TextWrapped = true
	credits.RichText = true
	credits.TextTransparency = 1
	credits.Text = [[<b>Stalker mod made by: LEO -LDX- (guide)</b>

Credits to:

Twixxel's stalkers (Inspiration for all this)
omayoba (new entities and much more — all by him)
pyxlfunkin (ideas for this mod and much more!)
lohan0389_97689 (entity ideas and new mechanics)

[Thanks for playing]
This mod]]
	credits.Parent = gui

	TweenService:Create(credits, TweenInfo.new(1.0, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		TextTransparency = 0
	}):Play()

	task.wait(4)

	TweenService:Create(credits, TweenInfo.new(1.0, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		TextTransparency = 1
	}):Play()
	task.wait(1.1)

	if sound and sound.Parent then
		local fade = TweenService:Create(sound, TweenInfo.new(0.5), {Volume = 0})
		fade:Play()
		fade.Completed:Wait()
		sound:Stop()
		sound:Destroy()
	end

	gui:Destroy()
end

task.spawn(function()
	ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
	task.wait(0.6)
	showIntro()
end)

-- ================== STALKER ==================
local stalkerImages = {
	"rbxassetid://71725176156204",
	"rbxassetid://122094429760163",
	"rbxassetid://100917760105588",
	"rbxassetid://127842346062233"
}

local currentStalker = nil
local activated = false
local achievementGiven = false
local touching = false
local lookTime = 0
local attacking = false

local spawnSound = 136833080474934
local randomScarySounds = {136350971091939, 113917217579668}
local JUMPSCARE_IMG = "rbxassetid://88898243238361"
local JUMPSCARE_SOUND = "rbxassetid://9125472062"

local function caption(text)
	pcall(function()
		local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
		local func = require(MainUI.Initiator.Main_Game)
		func.caption(text, true)
	end)
end

local function playSound(id, volume)
	local sound = Instance.new("Sound", Workspace)
	sound.SoundId = "rbxassetid://" .. tostring(id):gsub("rbxassetid://", "")
	sound.Volume = volume or 3.5
	sound:Play()
	Debris:AddItem(sound, 6)
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true
	pcall(function()
		local achievementGiver = loadstring(game:HttpGet(
			"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
		))()
		achievementGiver({
			Title = "Find a Stalker",
			Desc = "Let him watch you.",
			Reason = "But don't get close...",
			Image = "rbxassetid://99486859440104"
		})
	end)
end

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function makeSmoke(part)
	local emitter = Instance.new("ParticleEmitter")
	emitter.Texture = "rbxassetid://243660364"
	emitter.Color = ColorSequence.new(Color3.fromRGB(40, 40, 40), Color3.fromRGB(90, 90, 90))
	emitter.Size = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 2.5),
		NumberSequenceKeypoint.new(1, 6)
	})
	emitter.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.25),
		NumberSequenceKeypoint.new(1, 1)
	})
	emitter.Lifetime = NumberRange.new(1.4, 2.2)
	emitter.Rate = 50
	emitter.Speed = NumberRange.new(1, 3)
	emitter.SpreadAngle = Vector2.new(50, 50)
	emitter.Parent = part
	return emitter
end

local function vanishWithSmoke(part)
	if not part or not part.Parent then return end
	local smoke = makeSmoke(part)
	local groundPos = Vector3.new(part.Position.X, part.Position.Y - 16, part.Position.Z)
	local downTween = TweenService:Create(part, TweenInfo.new(1.6, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = groundPos
	})
	downTween:Play()
	downTween.Completed:Wait()
	smoke.Enabled = false
	task.wait(0.4)
	if part and part.Parent then part:Destroy() end
end

local function screenJumpscare()
	local gui = Instance.new("ScreenGui")
	gui.Name = "StalkerJS"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 100
	gui.Parent = playerGui

	local img = Instance.new("ImageLabel")
	img.Size = UDim2.new(1.1, 0, 1.1, 0)
	img.Position = UDim2.new(-0.05, 0, -0.05, 0)
	img.BackgroundTransparency = 1
	img.Image = JUMPSCARE_IMG
	img.ScaleType = Enum.ScaleType.Crop
	img.Parent = gui

	local sound = Instance.new("Sound", Workspace)
	sound.SoundId = JUMPSCARE_SOUND
	sound.Volume = 4
	sound:Play()
	Debris:AddItem(sound, 8)

	local shaking = true
	task.spawn(function()
		while shaking and img and img.Parent do
			img.Position = UDim2.new(
				-0.05 + math.random(-8, 8) / 200,
				0,
				-0.05 + math.random(-8, 8) / 200,
				0
			)
			task.wait(0.03)
		end
	end)

	task.wait(1.2)
	shaking = false

	local fade = TweenService:Create(img, TweenInfo.new(1.4, Enum.EasingStyle.Quad), {
		ImageTransparency = 1
	})
	fade:Play()
	fade.Completed:Wait()
	gui:Destroy()
end

local function isLookingAt(targetPos)
	local camPos = camera.CFrame.Position
	local camLook = camera.CFrame.LookVector
	local dir = (targetPos - camPos)
	if dir.Magnitude > 60 then return false end
	dir = dir.Unit
	return camLook:Dot(dir) > 0.85
end

local function hitPlayer(stalker)
	if touching or not stalker or not stalker.Parent then return end
	touching = true
	attacking = false
	currentStalker = nil

	startDarkFog()
	caption("...")
	local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
	if hum then hum:TakeDamage(20) end

	task.spawn(screenJumpscare)
	vanishWithSmoke(stalker)

	task.wait(0.5)
	endDarkFog()

	giveAchievement()
	touching = false
	lookTime = 0
end

local function spawnStalker()
	local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
	if not root then return end

	if currentStalker then currentStalker:Destroy() end
	touching = false
	attacking = false
	lookTime = 0

	local stalkerPart = Instance.new("Part")
	stalkerPart.Name = "Stalker"
	stalkerPart.Size = Vector3.new(9, 12, 2.5)
	stalkerPart.Transparency = 1
	stalkerPart.Anchored = true
	stalkerPart.CanCollide = false
	stalkerPart.Parent = Workspace

	local randomImage = stalkerImages[math.random(1, #stalkerImages)]
	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local decal = Instance.new("Decal")
		decal.Texture = randomImage
		decal.Face = face
		decal.Parent = stalkerPart
	end

	stalkerPart.CFrame = root.CFrame * CFrame.new(0, 3, 24)
	currentStalker = stalkerPart

	task.spawn(function()
		while stalkerPart.Parent do
			if not attacking then
				stalkerPart.CFrame = CFrame.new(stalkerPart.Position, camera.CFrame.Position)
			end
			task.wait(0.03)
		end
	end)

	playSound(spawnSound, 4)
	if math.random(1, 3) == 1 then
		playSound(randomScarySounds[math.random(1, #randomScarySounds)], 3.5)
	end

	task.delay(math.random(45, 90), function()
		if currentStalker == stalkerPart and stalkerPart.Parent and not attacking and not touching then
			currentStalker = nil
			vanishWithSmoke(stalkerPart)
		end
	end)
end

local function jumpscare()
	local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
	if not root then return end

	local js = Instance.new("Part")
	js.Name = "StalkerJumpscare"
	js.Size = Vector3.new(7, 10, 1.5)
	js.Transparency = 1
	js.Anchored = true
	js.CanCollide = false
	js.Parent = Workspace

	local img = "rbxassetid://73629850429939"
	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local decal = Instance.new("Decal")
		decal.Texture = img
		decal.Face = face
		decal.Parent = js
	end

	js.CFrame = root.CFrame * CFrame.new(0, 3, -6)

	task.spawn(function()
		while js.Parent do
			js.CFrame = CFrame.new(js.Position, camera.CFrame.Position)
			task.wait(0.03)
		end
	end)

	task.wait(1.4)
	js:Destroy()
end

task.spawn(function()
	while true do
		if currentStalker and not touching then
			local stalker = currentStalker
			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			if root and stalker.Parent then
				local distance = (stalker.Position - root.Position).Magnitude
				if distance < 9 then
					hitPlayer(stalker)
				else
					if isLookingAt(stalker.Position) then
						lookTime = lookTime + 0.1
						if lookTime >= 5 and not attacking then
							attacking = true
							task.spawn(function()
								while stalker.Parent and attacking do
									local r = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
									if not r then break end
									local dir = (r.Position - stalker.Position)
									if dir.Magnitude < 9 then
										hitPlayer(stalker)
										break
									end
									if dir.Magnitude > 0.5 then
										dir = dir.Unit
										stalker.CFrame = CFrame.new(stalker.Position + dir * 22 * 0.05, r.Position)
									end
									task.wait(0.03)
								end
							end)
						end
					else
						lookTime = math.max(0, lookTime - 0.15)
					end
				end
			end
		else
			lookTime = 0
		end
		task.wait(0.1)
	end
end)

-- ================== SEEK MUSIC ==================
local NEW_SEEK_ID = "rbxassetid://125959136412325"
local changedSounds = {}

local function changeSeekMusic()
	for _, obj in pairs(Workspace:GetDescendants()) do
		if obj:IsA("Sound") and not changedSounds[obj] then
			local nameLower = obj.Name:lower()
			local parentName = obj.Parent and obj.Parent.Name:lower() or ""
			if nameLower:find("seek") or nameLower:find("chase")
				or parentName:find("seek") or parentName:find("chase") then
				changedSounds[obj] = true
				local wasPlaying = obj.IsPlaying
				local oldTime = obj.TimePosition
				obj.SoundId = NEW_SEEK_ID
				obj.Volume = 3.8
				obj.PlaybackSpeed = 1
				obj.Looped = true
				if wasPlaying then
					obj:Stop()
					task.wait(0.05)
					obj.TimePosition = oldTime
					obj:Play()
				end
			end
		end
	end
end

task.spawn(function()
	while true do
		changeSeekMusic()
		task.wait(2)
	end
end)

ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function()
	task.wait(1)
	changeSeekMusic()
	if not activated then
		activated = true
	end
end)

task.wait(3)
changeSeekMusic()

-- spawns aleatórios
task.spawn(function()
	while true do
		task.wait(math.random(35, 70))
		if activated and math.random(1, 4) == 1 then
			spawnStalker()
		end
	end
end)

task.spawn(function()
	while true do
		task.wait(math.random(40, 85))
		if activated and math.random(1, 5) == 1 then
			jumpscare()
		end
	end
end)

player.Chatted:Connect(function(msg)
	local m = msg:lower()
	if m == "/stalker" then
		spawnStalker()
	elseif m == "/stalker3" then
		jumpscare()
	end
end)

print("✅ Intro + Stalker + Seek Music (script único)")
print("/stalker | /stalker3")

-- 👤 Window Entity (Rework)
-- sala com janela = mais chance | 2 versões (sobe / desce)
-- abrir cedo = chase + névoa estilo Stalker
-- /window

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local WINDOW_IMAGE_UP = "rbxassetid://136542774742776"
local WINDOW_IMAGE_DOWN = "rbxassetid://98800851128806"
local CHASE_IMAGE = "rbxassetid://107999364222287"
local LOOK_SOUND = "rbxassetid://78494358244371"
local CHASE_MUSIC = "rbxassetid://91203761863073"
local ACHIEVEMENT_IMAGE = "rbxassetid://133276883616111"

local currentWindowEntity = nil
local cooldownActive = false
local cooldownEnd = 0
local chaseEntity = nil
local chaseMusic = nil
local achievementGiven = false

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function caption(text)
	pcall(function()
		local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
		local func = require(MainUI.Initiator.Main_Game)
		func.caption(text, true)
	end)
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true
	pcall(function()
		local achievementGiver = loadstring(game:HttpGet(
			"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
		))()
		achievementGiver({
			Title = "Don't Go Outside",
			Desc = "You waited long enough.",
			Reason = "The window was watching...",
			Image = ACHIEVEMENT_IMAGE
		})
	end)
end

local function playSound(id, volume, speed)
	local s = Instance.new("Sound", Workspace)
	s.SoundId = id
	s.Volume = volume or 3
	s.PlaybackSpeed = speed or 1
	s:Play()
	Debris:AddItem(s, 10)
	return s
end

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function findWindowsNear()
	local hrp = getHRP()
	if not hrp then return {} end

	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return {} end

	local candidates = {}
	for _, v in pairs(rooms:GetDescendants()) do
		local name = v.Name:lower()
		if v:IsA("BasePart") and (name:find("window") or name:find("glass") or name:find("pane")) then
			if (v.Position - hrp.Position).Magnitude < 90 then
				table.insert(candidates, v)
			end
		end
	end
	return candidates
end

local function hasWindowRoom()
	return #findWindowsNear() > 0
end

local function isLookingAt(targetPos)
	local camPos = camera.CFrame.Position
	local camLook = camera.CFrame.LookVector
	local dir = (targetPos - camPos)
	if dir.Magnitude > 45 then return false end
	dir = dir.Unit
	return camLook:Dot(dir) > 0.92
end

local function spawnChase()
	if chaseEntity then return end

	local hrp = getHRP()
	if not hrp then return end

	startDarkFog() -- névoa estilo Stalker

	chaseMusic = Instance.new("Sound", Workspace)
	chaseMusic.SoundId = CHASE_MUSIC
	chaseMusic.Volume = 4
	chaseMusic.Looped = true
	chaseMusic:Play()

	local part = Instance.new("Part")
	part.Name = "WindowChase"
	part.Size = Vector3.new(8, 11, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = CHASE_IMAGE
		d.Face = face
		d.Parent = part
	end

	part.CFrame = hrp.CFrame * CFrame.new(0, 3, 16)
	chaseEntity = part

	print("Window Entity PERSEGUINDO + névoa!")

	task.spawn(function()
		while part.Parent do
			local root = getHRP()
			local hum = getHumanoid()
			if root then
				local dir = (root.Position - part.Position)
				if dir.Magnitude > 0.5 then
					dir = dir.Unit
					part.CFrame = CFrame.new(part.Position + dir * 10 * 0.05, root.Position)
				end
				if (part.Position - root.Position).Magnitude < 8 then
					if hum then hum.Health = 0 end
					part:Destroy()
					chaseEntity = nil
					if chaseMusic then
						chaseMusic:Stop()
						chaseMusic:Destroy()
						chaseMusic = nil
					end
					endDarkFog()
					break
				end
			end
			task.wait(0.03)
		end
	end)
end

local function startCountdown()
	cooldownActive = true
	cooldownEnd = tick() + 10

	caption("dont go outside")

	task.spawn(function()
		for i = 10, 0, -1 do
			if not cooldownActive then return end
			caption(tostring(i))
			task.wait(1)
		end

		if cooldownActive and tick() >= cooldownEnd then
			cooldownActive = false
			caption("you can go")
			giveAchievement()
			print("Esperou → conquista!")
		end
	end)
end

local function spawnOnWindow()
	if currentWindowEntity or chaseEntity or cooldownActive then return end

	local hrp = getHRP()
	if not hrp then return end

	local windows = findWindowsNear()
	local window = (#windows > 0) and windows[math.random(1, #windows)] or nil

	local spawnCF
	if window then
		spawnCF = window.CFrame * CFrame.new(0, 0, 1.2)
	else
		-- sem janela: ainda pode, mas mais atrás
		spawnCF = hrp.CFrame * CFrame.new(0, 3, -14)
	end

	-- 2 versões: sobe ou desce
	local goesUp = math.random(1, 2) == 1
	local faceImage = goesUp and WINDOW_IMAGE_UP or WINDOW_IMAGE_DOWN

	local part = Instance.new("Part")
	part.Name = "WindowEntity"
	part.Size = Vector3.new(6, 8, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = faceImage
		d.Face = face
		d.Parent = part
	end

	currentWindowEntity = part
	print("Window Entity | versão: " .. (goesUp and "SOBE" or "DESCE"))

	local looked = false
	task.spawn(function()
		while part.Parent and not looked do
			part.CFrame = CFrame.new(part.Position, camera.CFrame.Position)

			if isLookingAt(part.Position) then
				looked = true
				playSound(LOOK_SOUND, 3.5, 0.20)

				local offset = goesUp and Vector3.new(0, 12, 0) or Vector3.new(0, -14, 0)
				local endPos = part.Position + offset

				local tween = TweenService:Create(part, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
					Position = endPos
				})
				tween:Play()
				tween.Completed:Wait()

				if part and part.Parent then part:Destroy() end
				currentWindowEntity = nil
				startCountdown()
				break
			end
			task.wait(0.05)
		end
	end)
end

-- abriu porta cedo demais → chase + névoa
ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function()
	if cooldownActive and tick() < cooldownEnd then
		cooldownActive = false
		print("Abriu cedo demais!")
		task.wait(0.4)
		spawnChase()
	end
end)

-- Spawn: mais chance se tem janela perto
task.spawn(function()
	while true do
		ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
		task.wait(math.random(4, 10))

		if currentWindowEntity or chaseEntity or cooldownActive then
			continue
		end

		local hasWin = hasWindowRoom()
		-- com janela: 1/4 | sem janela: 1/14 (bem menos)
		local chance = hasWin and 4 or 14
		if math.random(1, chance) == 1 then
			print("Window spawn | janela=" .. tostring(hasWin))
			spawnOnWindow()
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/window" then
		spawnOnWindow()
	end
end)

print("✅ Window Rework!")
print("Janela = mais chance | 2 versões (sobe/desce) | chase com névoa")
print("/window")

-- 👤 Stalker 2 - Só uma vez

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera
local humanoid = char:WaitForChild("Humanoid")

local CHASE_IMAGE = "rbxassetid://99869789201897"
local CHASE_MUSIC = "rbxassetid://129085629036594"
local ACHIEVEMENT_IMAGE = "rbxassetid://80917066864552"

local chaseEntity = nil
local chaseMusic = nil
local redGui = nil
local redLabel = nil
local runShaking = false
local cameraShake = false
local shakeConn = nil
local achievementGiven = false
local alreadyAppeared = false -- só 1 vez

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	local achievementGiver = loadstring(game:HttpGet(
		"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
	))()

	achievementGiver({
		Title = "Too Far Away",
		Desc = "You outran the distance.",
		Reason = "499 blocks was not enough...",
		Image = ACHIEVEMENT_IMAGE
	})
end

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function startCameraShake()
	if cameraShake then return end
	cameraShake = true
	shakeConn = RunService.RenderStepped:Connect(function()
		if not cameraShake then return end
		camera.CFrame = camera.CFrame * CFrame.new(
			math.random(-6, 6) / 18,
			math.random(-6, 6) / 18,
			0
		)
	end)
end

local function stopCameraShake()
	cameraShake = false
	if shakeConn then
		shakeConn:Disconnect()
		shakeConn = nil
	end
end

local function clearRedText()
	runShaking = false
	if redGui then
		redGui:Destroy()
		redGui = nil
		redLabel = nil
	end
end

local function redText(text, isRun)
	if not redGui or not redGui.Parent then
		redGui = Instance.new("ScreenGui")
		redGui.Name = "Stalker2Text"
		redGui.Parent = player:WaitForChild("PlayerGui")

		redLabel = Instance.new("TextLabel")
		redLabel.Size = UDim2.new(0.7, 0, 0, 32)
		redLabel.Position = UDim2.new(0.15, 0, 0.74, 0)
		redLabel.BackgroundTransparency = 1
		redLabel.TextColor3 = Color3.fromRGB(255, 45, 45)
		redLabel.TextStrokeTransparency = 0.35
		redLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
		redLabel.TextScaled = true
		redLabel.Font = Enum.Font.Gotham
		redLabel.Parent = redGui
	end

	redLabel.Text = text

	if isRun and not runShaking then
		runShaking = true
		task.spawn(function()
			while runShaking and redLabel and redLabel.Parent do
				redLabel.Position = UDim2.new(
					0.15 + math.random(-6, 6) / 300,
					0,
					0.74 + math.random(-6, 6) / 300,
					0
				)
				task.wait(0.04)
			end
			if redLabel and redLabel.Parent then
				redLabel.Position = UDim2.new(0.15, 0, 0.74, 0)
			end
		end)
	elseif not isRun then
		runShaking = false
	end
end

local function vanishDown(part)
	if not part or not part.Parent then return end

	local smoke = Instance.new("ParticleEmitter")
	smoke.Texture = "rbxassetid://243660364"
	smoke.Color = ColorSequence.new(Color3.fromRGB(40, 40, 40), Color3.fromRGB(90, 90, 90))
	smoke.Size = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 2.5),
		NumberSequenceKeypoint.new(1, 6)
	})
	smoke.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.25),
		NumberSequenceKeypoint.new(1, 1)
	})
	smoke.Lifetime = NumberRange.new(1.4, 2.2)
	smoke.Rate = 50
	smoke.Speed = NumberRange.new(1, 3)
	smoke.SpreadAngle = Vector2.new(50, 50)
	smoke.Parent = part

	local groundPos = Vector3.new(part.Position.X, part.Position.Y - 16, part.Position.Z)
	local downTween = TweenService:Create(part, TweenInfo.new(1.6, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = groundPos
	})
	downTween:Play()
	downTween.Completed:Wait()

	smoke.Enabled = false
	task.wait(0.35)

	if part and part.Parent then
		part:Destroy()
	end
end

local function startChaseMode()
	if chaseEntity then return end
	if alreadyAppeared then
		print("Stalker2 já apareceu nessa partida.")
		return
	end
	alreadyAppeared = true

	redText("It is 499 blocks away from you", false)
	startDarkFog()
	startCameraShake()

	chaseMusic = Instance.new("Sound", Workspace)
	chaseMusic.SoundId = CHASE_MUSIC
	chaseMusic.Volume = 3.2
	chaseMusic.Looped = true
	chaseMusic:Play()

	local chaser = Instance.new("Part")
	chaser.Name = "Stalker2"
	chaser.Size = Vector3.new(8, 12, 0.2)
	chaser.Transparency = 1
	chaser.Anchored = true
	chaser.CanCollide = false
	chaser.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = CHASE_IMAGE
		d.Face = face
		d.Parent = chaser
	end

	chaser.CFrame = hrp.CFrame * CFrame.new(0, 4, 50)
	chaseEntity = chaser

	local speed = 7.5
	local startTime = tick()
	local duration = 20

	task.spawn(function()
		while chaseEntity and chaseEntity.Parent do
			if (tick() - startTime) >= duration then
				local entity = chaseEntity
				chaseEntity = nil

				stopCameraShake()
				if chaseMusic then
					chaseMusic:Stop()
					chaseMusic:Destroy()
				end
				clearRedText()

				vanishDown(entity)
				endDarkFog()

				redText("You survived", false)
				giveAchievement()
				task.delay(3, clearRedText)
				break
			end

			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			local hum = player.Character and player.Character:FindFirstChild("Humanoid")
			if not root then
				task.wait(0.2)
				continue
			end

			local realDist = (chaser.Position - root.Position).Magnitude
			local distance = math.max(0, math.floor(realDist * 2.2))

			if realDist <= 20 then
				redText("RUN", true)
				speed = 11
			else
				redText("It is " .. distance .. " blocks away from you", false)
			end

			Lighting.Ambient = Color3.fromRGB(140, 0, 0)
			Lighting.Brightness = 0.35

			local dir = (root.Position - chaser.Position)
			if dir.Magnitude > 0.5 then
				dir = dir.Unit
				local shakeOffset = Vector3.new(
					math.random(-4, 4) / 12,
					math.random(-3, 3) / 12,
					0
				)
				chaser.CFrame = CFrame.new(chaser.Position + dir * speed * 0.1 + shakeOffset, root.Position)
			end

			if realDist < 9 then
				if hum then
					hum:TakeDamage(40)
				end

				local entity = chaseEntity
				chaseEntity = nil

				stopCameraShake()
				if chaseMusic then
					chaseMusic:Stop()
					chaseMusic:Destroy()
				end
				clearRedText()

				vanishDown(entity)
				endDarkFog()

				Lighting.Ambient = Color3.new(1, 1, 1)
				Lighting.Brightness = 1
				break
			end

			task.wait(0.1)
		end
	end)
end

-- Spawn aleatório (só se ainda não apareceu)
task.spawn(function()
	while not alreadyAppeared do
		task.wait(math.random(150, 280))
		if not alreadyAppeared and math.random(1, 8) == 1 then
			startChaseMode()
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/stalker2" then
		startChaseMode()
	end
end)

print("✅ Stalker 2 - Só 1 vez por partida!")
print("Use /stalker2")
