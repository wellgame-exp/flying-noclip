local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local VirtualUser = game:GetService("VirtualUser")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")

local player = Players.LocalPlayer

--// Optional Remote Source for Re-Execute (set if hosting/loading via URL)
local SCRIPT_URL = "" -- e.g. "https://raw.githubusercontent.com/.../script.lua"

local function getEnvTable()
	if typeof(getgenv) == "function" then
		return getgenv()
	end
	return _G
end

local env = getEnvTable()

--// Load WindUI Library (Official Loader)
local cloneref = (cloneref or clonereference or function(instance) return instance end)
local ReplicatedStorage = cloneref(game:GetService("ReplicatedStorage"))

local WindUI

do
	local ok, result = pcall(function()
		return require("./src/Init")
	end)

	if ok then
		WindUI = result
	else
		if cloneref(RunService):IsStudio() then
			WindUI = require(cloneref(ReplicatedStorage:WaitForChild("WindUI"):WaitForChild("Init")))
		else
			WindUI = loadstring(game:HttpGet("https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"))()
		end
	end
end

--// Helper Notification Wrapper (off by default - toggle in Setting tab)
local notificationsEnabled = true

local function notify(title, content)
	if not notificationsEnabled then return end
	pcall(function()
		if WindUI and WindUI.Notify then
			WindUI:Notify({ Title = title, Content = content, Duration = 2 })
		end
	end)
end

--// Settings & State Variables
local SPIN_DURATION = 0.5
local RETURN_TIME = 0.01
local CLOSER_DISTANCE = 0 -- studs away from the closer player (0 = right on them)
local teleportAllSeconds = 1 -- 1 to 100 seconds per player in Teleport All
local teleportKey = Enum.KeyCode.Tab
local uiToggleKey = Enum.KeyCode.RightShift

local targetPlayerNames = {}
local loopTeleporting = false
local teleportAllActive = false
local teleportUnderAllActive = false
local teleportUnderToggleActive = false
local teleportDownUnderActive = false
local closerPlayerActive = false
local closestPlayer = nil
local autoClickActive = false
local spinAngle = 0
local toolNameInput = "Sword"
local flingOnTeleport = false -- ON = fling the target during Teleport Once, stops when teleported back
local fireOnTeleport = false -- ON = tool fires during Teleport Once, stops when teleported back
local autoClickDelay = 0.05 -- seconds between clicks (small = fast, big = slow)

local selectedTheme = "Dark"
local selectedHexColor = "#30FF6A"
local inputConnection = nil
local closerLoopRunning = false
local closerToken = 0
local rainbowActive = false
local noclipActive = false
local noclipConn = nil
local hasBuiltUI = false -- true after first UI build (Re-Execute must not wipe WindUI's own ScreenGui)

--// Helper: Clamp Seconds (1 - 100)
local function clampSeconds(val)
	local num = tonumber(val) or 1
	return math.clamp(num, 1, 100)
end

--// Helper: Supervised loop - restarts itself if an error ever kills it (keeps running through death/respawn)
local function spawnLoop(fn)
	task.spawn(function()
		while true do
			local ok, err = pcall(fn)
			if ok then break end
			warn("[TeleKill] loop error (restarting): " .. tostring(err))
			task.wait(0.2)
		end
	end)
end

--// Anti-AFK: prevents the idle kick so the script keeps running
if not env._TELEKILL_ANTIAFK then
	env._TELEKILL_ANTIAFK = true
	pcall(function()
		player.Idled:Connect(function()
			VirtualUser:CaptureController()
			VirtualUser:ClickButton2(Vector2.new())
		end)
	end)
end

--// Helper: Parse Hex String to Color3
local function hexToColor3(hex)
	if type(hex) ~= "string" then return nil end
	local cleaned = hex:gsub("#", ""):match("^%s*(.-)%s*$")
	if #cleaned == 6 and cleaned:match("^%x+$") then
		local r = tonumber(cleaned:sub(1, 2), 16) / 255
		local g = tonumber(cleaned:sub(3, 4), 16) / 255
		local b = tonumber(cleaned:sub(5, 6), 16) / 255
		return Color3.new(r, g, b)
	end
	return nil
end

--// Helper: Get a valid live root part for a player's character
local function getCharacterRoot(character)
	if not character or not character.Parent then
		return nil
	end

	local humanoid = character:FindFirstChildOfClass("Humanoid")
	if humanoid and humanoid.Health <= 0 then
		return nil
	end

	return character:FindFirstChild("HumanoidRootPart")
		or character.PrimaryPart
		or character:FindFirstChild("Head")
end

--// Helper: Find Single Player Matching a Query (exact > starts-with > contains)
local function findSinglePlayer(text)
	if not text or text == "" then return nil end
	text = text:lower():match("^%s*(.-)%s*$")
	if text == "" then return nil end

	local prefixMatch, containsMatch = nil, nil

	for _, target in ipairs(Players:GetPlayers()) do
		if target ~= player then
			local username = target.Name:lower()
			local displayName = target.DisplayName:lower()

			if username == text or displayName == text then
				return target
			end
			if not prefixMatch and (username:sub(1, #text) == text or displayName:sub(1, #text) == text) then
				prefixMatch = target
			end
			if not containsMatch and (username:find(text, 1, true) or displayName:find(text, 1, true)) then
				containsMatch = target
			end
		end
	end
	return prefixMatch or containsMatch
end

--// Helper: Resolve typed Target Players names into actual players
local function resolveNamedPlayers()
	local list = {}
	for _, text in ipairs(targetPlayerNames) do
		local matched = findSinglePlayer(text)
		if matched and not table.find(list, matched) then
			table.insert(list, matched)
		end
	end
	return list
end

--// Helper: Find Closer Player (other than LocalPlayer)
local function getClosestPlayer()
	local myRoot = getCharacterRoot(player.Character)
	if not myRoot then
		return nil, nil
	end

	local nearestPlayer = nil
	local shortestDistance = math.huge

	-- If names are typed in Target Players, only those players are candidates
	local candidates = Players:GetPlayers()
	if #targetPlayerNames > 0 then
		candidates = resolveNamedPlayers()
	end

	for _, target in ipairs(candidates) do
		if target ~= player then
			local targetRoot = getCharacterRoot(target.Character)
			if targetRoot then
				local dist = (targetRoot.Position - myRoot.Position).Magnitude
				if dist < shortestDistance then
					shortestDistance = dist
					nearestPlayer = target
				end
			end
		end
	end

	return nearestPlayer, shortestDistance
end

--// Helper: Find Multiple Players Matching current input list (or Closer Player)
local function getTargetPlayers(forceCloser)
	if closerPlayerActive or forceCloser then
		local nearest = getClosestPlayer()
		closestPlayer = nearest
		if nearest then
			return { nearest }
		end
		return {}
	end

	if #targetPlayerNames == 0 then
		-- No name typed: automatically use the closer player
		local nearest = getClosestPlayer()
		closestPlayer = nearest
		return nearest and { nearest } or {}
	end

	return resolveNamedPlayers()
end

--// Fire the tool once at the given targets (equip, activate, send the hit to the server)
local function fireToolOnce(targets)
	local inputName = toolNameInput:lower():match("^%s*(.-)%s*$")
	if not inputName or inputName == "" then return end

	local character = player.Character
	local backpack = player:FindFirstChildOfClass("Backpack")
	if not character then return end

	local equippedTool = nil
	for _, item in ipairs(character:GetChildren()) do
		if item:IsA("Tool") and item.Name:lower() == inputName then
			equippedTool = item
			break
		end
	end

	if not equippedTool and backpack then
		for _, item in ipairs(backpack:GetChildren()) do
			if item:IsA("Tool") and item.Name:lower() == inputName then
				local hum = character:FindFirstChildOfClass("Humanoid")
				if hum then
					pcall(function() hum:EquipTool(item) end)
				else
					item.Parent = character
				end
				equippedTool = item
				break
			end
		end
	end

	if not equippedTool then return end

	-- 1) Activate the tool (no fake mouse input, so your real mouse never sticks)
	pcall(function()
		equippedTool:Activate()
	end)

	-- 2) Fire the hit to the server: touch the target with the tool's Handle
	if typeof(firetouchinterest) == "function" then
		local handle = equippedTool:FindFirstChild("Handle")
		if handle then
			for _, target in ipairs(targets or {}) do
				local tChar = target.Character
				if tChar and getCharacterRoot(tChar) then
					for _, partName in ipairs({ "HumanoidRootPart", "Head" }) do
						local part = tChar:FindFirstChild(partName)
						if part then
							pcall(function()
								firetouchinterest(handle, part, 0)
								firetouchinterest(handle, part, 1)
							end)
						end
					end
				end
			end
		end
	end
end

--// Noclip helpers: remember each part's ORIGINAL CanCollide and put it back exactly
--// (forcing CanCollide = true on every part makes arms/legs collide with the body and get stuck)
local collisionBackup = {}

local function noclipCharacter(character)
	pcall(function()
		for _, part in ipairs(character:GetDescendants()) do
			if part:IsA("BasePart") then
				if collisionBackup[part] == nil then
					collisionBackup[part] = part.CanCollide
				end
				part.CanCollide = false
			end
		end
	end)
end

local function restoreCollision()
	for part, original in pairs(collisionBackup) do
		collisionBackup[part] = nil
		pcall(function()
			if part.Parent then
				part.CanCollide = original
			end
		end)
	end
end

--// Noclip toggle: walk through walls all the time (not only while teleporting)
local function setNoclip(state)
	noclipActive = state
	if noclipConn then
		noclipConn:Disconnect()
		noclipConn = nil
	end
	if state then
		noclipConn = RunService.Stepped:Connect(function()
			local c = player.Character
			if c then
				noclipCharacter(c)
			end
		end)
	else
		restoreCollision()
	end
end

--// Player mods: Speed, Jump Power, Infinite Jump, Fly
local speedEnabled, speedValue, defaultWalkSpeed = false, 50, 16
local jumpEnabled, jumpValue, defaultJumpPower = false, 100, 50
local infJumpActive, infJumpConn = false, nil
local flyActive, flySpeed = false, 50
local flyConn, flyBV, flyBG, flyRoot = nil, nil, nil, nil
local playerModConn = nil

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function updatePlayerModLoop()
	if (speedEnabled or jumpEnabled) and not playerModConn then
		playerModConn = RunService.Heartbeat:Connect(function()
			local hum = getHumanoid()
			if not hum then return end
			if speedEnabled then
				hum.WalkSpeed = speedValue
			end
			if jumpEnabled then
				hum.UseJumpPower = true
				hum.JumpPower = jumpValue
			end
		end)
	elseif not speedEnabled and not jumpEnabled and playerModConn then
		playerModConn:Disconnect()
		playerModConn = nil
	end
end

local function setSpeed(state)
	local hum = getHumanoid()
	if state and not speedEnabled and hum then
		defaultWalkSpeed = hum.WalkSpeed
	end
	speedEnabled = state
	if not state and hum then
		hum.WalkSpeed = defaultWalkSpeed
	end
	updatePlayerModLoop()
end

local function setJump(state)
	local hum = getHumanoid()
	if state and not jumpEnabled and hum then
		defaultJumpPower = hum.JumpPower
	end
	jumpEnabled = state
	if not state and hum then
		hum.JumpPower = defaultJumpPower
	end
	updatePlayerModLoop()
end

local function setInfJump(state)
	infJumpActive = state
	if infJumpConn then
		infJumpConn:Disconnect()
		infJumpConn = nil
	end
	if state then
		infJumpConn = UserInputService.JumpRequest:Connect(function()
			local hum = getHumanoid()
			if hum then
				hum:ChangeState(Enum.HumanoidStateType.Jumping)
			end
		end)
	end
end

local function setFly(state)
	flyActive = state

	if flyConn then flyConn:Disconnect() flyConn = nil end
	if flyBV then pcall(function() flyBV:Destroy() end) flyBV = nil end
	if flyBG then pcall(function() flyBG:Destroy() end) flyBG = nil end
	flyRoot = nil

	local hum = getHumanoid()
	if not state then
		if hum then hum.PlatformStand = false end
		return
	end

	flyConn = RunService.Heartbeat:Connect(function()
		local r = getCharacterRoot(player.Character)
		local h = getHumanoid()
		local cam = workspace.CurrentCamera
		if not r or not h or not cam then return end

		-- (Re)attach the fly forces whenever we have a new character
		if r ~= flyRoot or not flyBV or not flyBV.Parent then
			if flyBV then pcall(function() flyBV:Destroy() end) end
			if flyBG then pcall(function() flyBG:Destroy() end) end

			flyBV = Instance.new("BodyVelocity")
			flyBV.MaxForce = Vector3.new(1e9, 1e9, 1e9)
			flyBV.Velocity = Vector3.new(0, 0, 0)
			flyBV.Parent = r

			flyBG = Instance.new("BodyGyro")
			flyBG.MaxTorque = Vector3.new(1e9, 1e9, 1e9)
			flyBG.P = 9e4
			flyBG.CFrame = r.CFrame
			flyBG.Parent = r

			flyRoot = r
		end

		h.PlatformStand = true

		-- Move relative to where the camera looks (works with keyboard and mobile joystick)
		local move = cam.CFrame:VectorToObjectSpace(h.MoveDirection)
		local dir = cam.CFrame.LookVector * (-move.Z) + cam.CFrame.RightVector * move.X

		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
			dir += Vector3.new(0, 1, 0)
		end
		if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
			dir -= Vector3.new(0, 1, 0)
		end

		flyBV.Velocity = (dir.Magnitude > 0) and (dir.Unit * flySpeed) or Vector3.new(0, 0, 0)
		flyBG.CFrame = cam.CFrame
	end)
end

local function resetPlayerMods()
	setSpeed(false)
	setJump(false)
	setInfJump(false)
	setFly(false)
end

--// Auto noclip for ALL teleport loops: on while any loop runs, restored when they all stop
do
	if env._TELEKILL_NOCLIPWATCH then
		pcall(function() env._TELEKILL_NOCLIPWATCH:Disconnect() end)
	end

	local wasActive = false
	env._TELEKILL_NOCLIPWATCH = RunService.Stepped:Connect(function()
		local anyLoop = loopTeleporting or teleportAllActive or teleportUnderAllActive
			or teleportUnderToggleActive or closerPlayerActive

		if anyLoop then
			wasActive = true
			local c = player.Character
			if c then
				noclipCharacter(c)
			end
		elseif wasActive then
			wasActive = false
			if not noclipActive then
				restoreCollision()
			end
		end
	end)
end

--// Core Action: Perform Teleport Down Once (With 1-Second Noclip)
local function executeTeleportDownOnce(forceCloser, forceFling)
	local character = player.Character
	local root = getCharacterRoot(character)
	if not root then
		notify("TeleKill", "Character not ready")
		return
	end

	local targets = getTargetPlayers(forceCloser)
	if #targets == 0 then
		notify("TeleKill", "No targets found/matched")
		return
	end

	local originalCFrame = root.CFrame

	task.spawn(function()
		noclipCharacter(character)

		for _, target in ipairs(targets) do
			local targetRoot = getCharacterRoot(target.Character)
			if targetRoot then
				local startTime = os.clock()
				local lastFire = 0
				while os.clock() - startTime < clampSeconds(teleportAllSeconds) do
					local activeRoot = getCharacterRoot(player.Character)
					if not activeRoot then break end

					-- Fire toggle: attack while teleported; stops automatically once we teleport back
					if fireOnTeleport and os.clock() - lastFire >= autoClickDelay then
						lastFire = os.clock()
						fireToolOnce({ target })
					end

					spinAngle = spinAngle + math.rad(180)
					pcall(function()
						activeRoot.Velocity = Vector3.new(0, 0, 0)
						activeRoot.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
						activeRoot.CFrame = targetRoot.CFrame * CFrame.new(0, -4, 0) * CFrame.Angles(math.rad(90), 0, spinAngle)
					end)
					-- Fling: spin hard inside the target so they get thrown (stops once we teleport back)
					if flingOnTeleport or forceFling then
						pcall(function()
							activeRoot.CanCollide = true
							activeRoot.CFrame = targetRoot.CFrame
							activeRoot.AssemblyAngularVelocity = Vector3.new(0, 30000, 0)
							activeRoot.AssemblyLinearVelocity = Vector3.new(15000, 15000, 15000)
						end)
					end
					task.wait(0.01)
				end
			end
		end

		local finalRoot = getCharacterRoot(player.Character)
		if finalRoot then
			pcall(function()
				finalRoot.Velocity = Vector3.new(0, 0, 0)
				finalRoot.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
				finalRoot.AssemblyAngularVelocity = Vector3.new(0, 0, 0) -- fling OFF
				finalRoot.CFrame = originalCFrame
			end)
		end

		task.wait(0.5)
		restoreCollision()
	end)
end

--// Core Action: Perform Teleport Once (Super Fast Spin)
local function executeTeleportOnce(forceCloser, forceFling)
	if teleportDownUnderActive then
		executeTeleportDownOnce(forceCloser, forceFling)
		return
	end

	local character = player.Character
	local root = getCharacterRoot(character)
	if not root then
		notify("TeleKill", "Character not ready")
		return
	end

	local targets = getTargetPlayers(forceCloser)
	if #targets == 0 then
		notify("TeleKill", "No targets found/matched")
		return
	end

	local originalCFrame = root.CFrame

	task.spawn(function()
		noclipCharacter(character)

		for _, target in ipairs(targets) do
			local targetRoot = getCharacterRoot(target.Character)
			if targetRoot then
				local startTime = os.clock()
				local lastFire = 0
				while os.clock() - startTime < clampSeconds(teleportAllSeconds) do
					local activeRoot = getCharacterRoot(player.Character)
					if not activeRoot then break end

					-- Fire toggle: attack while teleported; stops automatically once we teleport back
					if fireOnTeleport and os.clock() - lastFire >= autoClickDelay then
						lastFire = os.clock()
						fireToolOnce({ target })
					end

					spinAngle = spinAngle + math.rad(180)
					pcall(function()
						activeRoot.Velocity = Vector3.new(0, 0, 0)
						activeRoot.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
						activeRoot.CFrame = targetRoot.CFrame * CFrame.new((closerPlayerActive or forceCloser) and CLOSER_DISTANCE or 3, 0, 0) * CFrame.Angles(0, spinAngle, 0)
					end)
					-- Fling: spin hard inside the target so they get thrown (stops once we teleport back)
					if flingOnTeleport or forceFling then
						pcall(function()
							activeRoot.CanCollide = true
							activeRoot.CFrame = targetRoot.CFrame
							activeRoot.AssemblyAngularVelocity = Vector3.new(0, 30000, 0)
							activeRoot.AssemblyLinearVelocity = Vector3.new(15000, 15000, 15000)
						end)
					end
					task.wait(0.01)
				end
			end
		end

		local finalRoot = getCharacterRoot(player.Character)
		if finalRoot then
			pcall(function()
				finalRoot.Velocity = Vector3.new(0, 0, 0)
				finalRoot.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
				finalRoot.AssemblyAngularVelocity = Vector3.new(0, 0, 0) -- fling OFF
				finalRoot.CFrame = originalCFrame
			end)
		end

		task.wait(0.25)
		restoreCollision()
	end)
end

local function cleanupExistingUI()
	setNoclip(false)
	resetPlayerMods()
	rainbowActive = false
	espEnabled = false
	if inputConnection then
		pcall(function() inputConnection:Disconnect() end)
		inputConnection = nil
	end

	if env._TELEKILL_WINDOW then
		pcall(function()
			if env._TELEKILL_WINDOW.Destroy then
				env._TELEKILL_WINDOW:Destroy()
			elseif env._TELEKILL_WINDOW.Close then
				env._TELEKILL_WINDOW:Close()
			end
		end)
		env._TELEKILL_WINDOW = nil
	end

	pcall(function()
		if env._TELEKILL_EXEC2 then
			env._TELEKILL_EXEC2:Destroy()
			env._TELEKILL_EXEC2 = nil
		end
	end)

	-- Only sweep leftover GUIs from an OLD run of the script. On Re-Execute we keep
	-- WindUI's own ScreenGui, otherwise the new window has nowhere to appear.
	if not hasBuiltUI then
		pcall(function()
			for _, container in ipairs({ CoreGui, player:FindFirstChildOfClass("PlayerGui") }) do
				if container then
					for _, gui in ipairs(container:GetChildren()) do
						if gui:IsA("ScreenGui") and (gui.Name == "TeleKillHub" or gui.Name:find("WindUI")) then
							gui:Destroy()
						end
					end
				end
			end
		end)
	end
end

--// Sound: says "Closer" when the Closer GUI button is clicked
local CLOSER_SOUND_ID = "" -- optional: a Roblox sound asset id (e.g. "rbxassetid://123456") to use instead of the voice

local function playCloserSound()
	local SoundService = game:GetService("SoundService")

	-- Option 1: your own sound asset
	if CLOSER_SOUND_ID ~= "" then
		pcall(function()
			local snd = Instance.new("Sound")
			snd.SoundId = CLOSER_SOUND_ID
			snd.Volume = 1
			snd.Parent = SoundService
			snd:Play()
			task.delay(5, function() snd:Destroy() end)
		end)
		return
	end

	-- Option 2: Roblox text-to-speech voice saying "Closer"
	pcall(function()
		local tts = Instance.new("AudioTextToSpeech")
		tts.Text = "Closer"
		tts.Volume = 1
		tts.Parent = SoundService

		local out = Instance.new("AudioDeviceOutput")
		out.Parent = SoundService

		local wire = Instance.new("Wire")
		wire.SourceInstance = tts
		wire.TargetInstance = out
		wire.Parent = SoundService

		tts:Play()
		task.delay(5, function()
			tts:Destroy()
			out:Destroy()
			wire:Destroy()
		end)
	end)
end

--// ESP (see players through walls)
local espEnabled = false
local espNames = true
local espDistance = true
local espColor = Color3.fromRGB(255, 60, 60)
local espRainbow = false -- true = every player gets an animated rainbow color
local espTextSize = 14
local espObjects = {}

local function getESPFolder()
	if env._TELEKILL_ESP and env._TELEKILL_ESP.Parent then
		return env._TELEKILL_ESP
	end
	local folder = Instance.new("Folder")
	folder.Name = "TeleKillESP"
	local ok = false
	if typeof(gethui) == "function" then
		ok = pcall(function() folder.Parent = gethui() end) and folder.Parent ~= nil
	end
	if not ok then
		ok = pcall(function() folder.Parent = CoreGui end) and folder.Parent ~= nil
	end
	if not ok then
		folder.Parent = player:WaitForChild("PlayerGui")
	end
	env._TELEKILL_ESP = folder
	return folder
end

local function removeESP(target)
	local o = espObjects[target]
	if o then
		pcall(function() o.highlight:Destroy() end)
		pcall(function() o.billboard:Destroy() end)
		espObjects[target] = nil
	end
end

local function clearESP()
	for target in pairs(espObjects) do
		removeESP(target)
	end
	pcall(function()
		if env._TELEKILL_ESP then
			env._TELEKILL_ESP:Destroy()
			env._TELEKILL_ESP = nil
		end
	end)
end

local function updateESP()
	local folder = getESPFolder()
	local myRoot = getCharacterRoot(player.Character)
	local espIndex = 0

	for _, target in ipairs(Players:GetPlayers()) do
		if target ~= player then
			local char = target.Character
			local root = getCharacterRoot(char)

			if root then
				espIndex += 1
				local o = espObjects[target]
				if not o or o.char ~= char or not o.highlight.Parent then
					removeESP(target)

					local h = Instance.new("Highlight")
					h.Adornee = char
					h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
					h.FillTransparency = 0.6
					h.OutlineTransparency = 0
					h.Parent = folder

					local bb = Instance.new("BillboardGui")
					bb.Adornee = char:FindFirstChild("Head") or root
					bb.AlwaysOnTop = true
					bb.Size = UDim2.fromOffset(200, 40)
					bb.StudsOffset = Vector3.new(0, 3, 0)
					bb.Parent = folder

					local lbl = Instance.new("TextLabel")
					lbl.Size = UDim2.fromScale(1, 1)
					lbl.BackgroundTransparency = 1
					lbl.Font = Enum.Font.GothamBold
					lbl.TextSize = 14
					lbl.TextStrokeTransparency = 0
					lbl.Parent = bb

					o = { char = char, highlight = h, billboard = bb, label = lbl }
					espObjects[target] = o
				end

				local color = espColor
				if espRainbow then
					color = Color3.fromHSV((os.clock() * 0.3 + espIndex * 0.13) % 1, 1, 1)
				end
				o.highlight.FillColor = color
				o.highlight.OutlineColor = color
				o.label.TextColor3 = color
				o.label.TextSize = espTextSize
				o.billboard.Size = UDim2.fromOffset(240, espTextSize * 2.5)

				local text = ""
				if espNames then
					text = target.DisplayName
				end
				if espDistance and myRoot then
					local d = math.floor((root.Position - myRoot.Position).Magnitude)
					text = (text ~= "") and (text .. " [" .. d .. "]") or ("[" .. d .. "]")
				end
				o.billboard.Enabled = text ~= ""
				o.label.Text = text
			else
				removeESP(target)
			end
		end
	end

	for target in pairs(espObjects) do
		if not target.Parent then
			removeESP(target)
		end
	end
end

local function stopAllLoops()
	loopTeleporting = false
	teleportAllActive = false
	teleportUnderAllActive = false
	teleportUnderToggleActive = false
	autoClickActive = false
	setNoclip(false)
	resetPlayerMods()
	rainbowActive = false
	espEnabled = false
	closerPlayerActive = false
	closerToken += 1
end

--// Closer Player: locks onto nearest player and keeps spinning around them
local function startCloserLoop()
	closerToken += 1
	local myToken = closerToken

	spawnLoop(function()
		local locked = nil
		local lockedAt = 0

		while closerPlayerActive and myToken == closerToken do
			local root = getCharacterRoot(player.Character)

			if not root then
				task.wait(0.2)
			else
				-- Re-pick the nearest player if there's no target, it died/left, or the Teleport Seconds timer ran out
				if not locked or not locked.Parent or not getCharacterRoot(locked.Character)
					or (os.clock() - lockedAt >= clampSeconds(teleportAllSeconds)) then
					locked = getClosestPlayer() or locked
					lockedAt = os.clock()
					closestPlayer = locked
				end

				local targetRoot = locked and getCharacterRoot(locked.Character)
				if targetRoot then
					spinAngle += math.rad(180)
					pcall(function()
						root.Velocity = Vector3.new(0, 0, 0)
						root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)

						if teleportDownUnderActive then
							root.CFrame = targetRoot.CFrame * CFrame.new(0, -4, 0) * CFrame.Angles(math.rad(90), 0, spinAngle)
						else
							root.CFrame = targetRoot.CFrame * CFrame.new(closerPlayerActive and CLOSER_DISTANCE or 3, 0, 0) * CFrame.Angles(0, spinAngle, 0)
						end
					end)
				end

				task.wait(RETURN_TIME)
			end
		end
	end)
end

local function buildUI()
	cleanupExistingUI()

	local Window = WindUI:CreateWindow({
		Title = "TeleKill Hub | WindUI",
		Folder = "TeleKillHub",
		Icon = "solar:folder-2-bold-duotone",
		Theme = selectedTheme,
		NewElements = true,
		HideSearchBar = false,
		OpenButton = {
			Title = "Open TeleKill UI",
			CornerRadius = UDim.new(1, 0),
			StrokeThickness = 3,
			Enabled = true,
			Draggable = true,
			OnlyMobile = false,
			Scale = 0.5,
			Color = ColorSequence.new(
				hexToColor3(selectedHexColor) or Color3.fromHex("#30FF6A"),
				Color3.fromHex("#e7ff2f")
			),
		},
		Topbar = {
			Height = 44,
			ButtonsType = "Mac",
		},
	})

	env._TELEKILL_WINDOW = Window
	hasBuiltUI = true

	local function applyThemeAndColor(themeName, hexStr)
		local col = hexToColor3(hexStr)
		pcall(function()
			if themeName and Window.SetTheme then
				Window:SetTheme(themeName)
			elseif themeName and WindUI.SetTheme then
				WindUI:SetTheme(themeName)
			end
		end)

		if col then
			pcall(function()
				if Window.OpenButton and Window.OpenButton.Color then
					Window.OpenButton.Color = ColorSequence.new(col, Color3.fromHex("#e7ff2f"))
				end
			end)
			pcall(function()
				if WindUI.SetAccentColor then
					WindUI:SetAccentColor(col)
				end
			end)
		end
	end

	if Window.Tag then
		Window:Tag({
			Title = "v" .. (WindUI.Version or "1.0"),
			Icon = "github",
			Color = Color3.fromHex("#1c1c1c"),
			Border = true,
		})
	end

	local TelekillSection = Window:Section({
		Title = "TeleKill Controls",
	})

	local PositioningSection = Window:Section({
		Title = "Positioning",
	})

	local MainTab = PositioningSection:Tab({
		Title = "Main Operations",
		Icon = "solar:home-2-bold",
		Border = true,
	})

	local PositionTab = PositioningSection:Tab({
		Title = "Positioning",
		Icon = "solar:map-point-bold",
		Border = true,
	})

	local CloserPlayerTab = PositioningSection:Tab({
		Title = "Closer Player",
		Icon = "solar:radar-2-bold",
		Border = true,
	})

	local SettingsTab = TelekillSection:Tab({
		Title = "Configuration",
		Icon = "solar:settings-bold",
		Border = true,
	})

	local AutoClickTab = TelekillSection:Tab({
		Title = "Auto Clicker",
		Icon = "solar:cursor-bold",
		Border = true,
	})

	local SettingTab = TelekillSection:Tab({
		Title = "Setting",
		Icon = "solar:palette-bold",
		Border = true,
	})

	local FlingTab = TelekillSection:Tab({
		Title = "Fling",
		Icon = "solar:bolt-bold",
		Border = true,
	})

	local PlayerTab = TelekillSection:Tab({
		Title = "Player",
		Icon = "solar:user-bold",
		Border = true,
	})

	local ESPTab = TelekillSection:Tab({
		Title = "ESP",
		Icon = "solar:eye-bold",
		Border = true,
	})

	local ExecutorTab = TelekillSection:Tab({
		Title = "Executor",
		Icon = "solar:code-bold",
		Border = true,
	})

	--// Main Operations Tab Elements (Under Positioning)
	MainTab:Input({
		Title = "Target Players",
		Desc = "Optional: leave empty to auto-target the closer player. Separate multiple names with commas",
		Placeholder = "Enter Player Names...",
		Callback = function(text)
			targetPlayerNames = {}
			if text then
				for name in string.gmatch(text, "[^,]+") do
					local cleaned = name:match("^%s*(.-)%s*$")
					if cleaned and cleaned ~= "" then
						table.insert(targetPlayerNames, cleaned)
					end
				end
			end

			if #targetPlayerNames == 0 then
				notify("Target Players", "Cleared (will use closer player)")
				return
			end

			local found, missing = {}, {}
			for _, n in ipairs(targetPlayerNames) do
				local m = findSinglePlayer(n)
				if m then table.insert(found, m.Name) else table.insert(missing, n) end
			end
			if #found > 0 then
				notify("Target Players", "Targeting: " .. table.concat(found, ", "))
			end
			if #missing > 0 then
				notify("Target Players", "Not found: " .. table.concat(missing, ", "))
			end
		end
	})

	MainTab:Space()

	MainTab:Button({
		Title = "Teleport Once",
		Desc = "Teleports & spins fast around target(s), then teleports back",
		Justify = "Center",
		Callback = function()
			executeTeleportOnce()
		end
	})

	MainTab:Space()

	MainTab:Toggle({
		Title = "Loop Teleport Targets",
		Desc = "Continuously teleports & spins fast around target(s)",
		Value = loopTeleporting,
		Callback = function(state)
			loopTeleporting = state
			if loopTeleporting then
				spawnLoop(function()
					while loopTeleporting do
						local targets = getTargetPlayers()
						if #targets == 0 then
							task.wait(0.5)
						else
							for _, target in ipairs(targets) do
								if not loopTeleporting then break end
								local tickStart = os.clock()
								while loopTeleporting and (os.clock() - tickStart < clampSeconds(teleportAllSeconds)) do
									local root = getCharacterRoot(player.Character)
									local targetRoot = getCharacterRoot(target.Character)

									if not root or not targetRoot or not target.Parent then break end

									spinAngle = spinAngle + math.rad(180)
									pcall(function()
										root.Velocity = Vector3.new(0, 0, 0)
										root.CFrame = targetRoot.CFrame * CFrame.new(3, 0, 0) * CFrame.Angles(0, spinAngle, 0)
									end)
									task.wait(RETURN_TIME)
								end
							end
						end
						task.wait(RETURN_TIME)
					end
				end)
			end
		end
	})

	MainTab:Space()

	MainTab:Toggle({
		Title = "Teleport All",
		Desc = "Cycles through and fast-spins around ALL server players",
		Value = teleportAllActive,
		Callback = function(state)
			teleportAllActive = state
			if teleportAllActive then
				spawnLoop(function()
					while teleportAllActive do
						local otherPlayers = {}
						for _, p in ipairs(Players:GetPlayers()) do
							if p ~= player then
								table.insert(otherPlayers, p)
							end
						end

						if #otherPlayers == 0 then
							task.wait(1)
						else
							for _, target in ipairs(otherPlayers) do
								if not teleportAllActive then break end
								local stayDuration = clampSeconds(teleportAllSeconds)
								local tickStart = os.clock()
								while teleportAllActive and (os.clock() - tickStart < stayDuration) do
									local activeRoot = getCharacterRoot(player.Character)
									local targetRoot = getCharacterRoot(target.Character)

									if not activeRoot or not targetRoot or not target.Parent then break end

									spinAngle = spinAngle + math.rad(180)
									pcall(function()
										activeRoot.Velocity = Vector3.new(0, 0, 0)
										activeRoot.CFrame = targetRoot.CFrame * CFrame.new(3, 0, 0) * CFrame.Angles(0, spinAngle, 0)
									end)
									task.wait(RETURN_TIME)
								end
							end
						end
						task.wait(RETURN_TIME)
					end
				end)
			end
		end
	})

	--// Positioning Tab Elements
	PositionTab:Toggle({
		Title = "Teleport Down Under",
		Desc = "ON = Teleport Once uses Underground mode | OFF = Uses Main Operations",
		Value = teleportDownUnderActive,
		Callback = function(state)
			teleportDownUnderActive = state
		end
	})

	PositionTab:Space()

	PositionTab:Toggle({
		Title = "Teleport Under All",
		Desc = "Continuously cycles through and fast-spins underneath ALL server players",
		Value = teleportUnderAllActive,
		Callback = function(state)
			teleportUnderAllActive = state
			if teleportUnderAllActive then
				spawnLoop(function()
					while teleportUnderAllActive do
						local otherPlayers = {}
						for _, p in ipairs(Players:GetPlayers()) do
							if p ~= player then
								table.insert(otherPlayers, p)
							end
						end

						if #otherPlayers == 0 then
							task.wait(1)
						else
							for _, target in ipairs(otherPlayers) do
								if not teleportUnderAllActive then break end
								local stayDuration = clampSeconds(teleportAllSeconds)
								local tickStart = os.clock()
								while teleportUnderAllActive and (os.clock() - tickStart < stayDuration) do
									local activeRoot = getCharacterRoot(player.Character)
									local targetRoot = getCharacterRoot(target.Character)

									if not activeRoot or not targetRoot or not target.Parent then break end

									spinAngle = spinAngle + math.rad(180)
									pcall(function()
										activeRoot.Velocity = Vector3.new(0, 0, 0)
										activeRoot.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
										activeRoot.CFrame = targetRoot.CFrame * CFrame.new(0, -4, 0) * CFrame.Angles(math.rad(90), 0, spinAngle)
									end)
									task.wait(RETURN_TIME)
								end
							end
						end
						task.wait(RETURN_TIME)
					end
				end)
			end
		end
	})

	PositionTab:Space()

	PositionTab:Toggle({
		Title = "Teleport Under Toggle",
		Desc = "Keeps you underneath your target(s) spinning fast (stays Teleport Seconds per target)",
		Value = teleportUnderToggleActive,
		Callback = function(state)
			teleportUnderToggleActive = state
			if teleportUnderToggleActive then
				spawnLoop(function()
					while teleportUnderToggleActive do
						local targets = getTargetPlayers()
						if #targets == 0 then
							task.wait(0.5)
						else
							for _, target in ipairs(targets) do
								if not teleportUnderToggleActive then break end
								local stayDuration = clampSeconds(teleportAllSeconds)
								local tickStart = os.clock()
								while teleportUnderToggleActive and (os.clock() - tickStart < stayDuration) do
									local root = getCharacterRoot(player.Character)
									local targetRoot = getCharacterRoot(target.Character)

									if not root or not targetRoot or not target.Parent then break end

									spinAngle = spinAngle + math.rad(180)
									pcall(function()
										root.Velocity = Vector3.new(0, 0, 0)
										root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
										root.CFrame = targetRoot.CFrame * CFrame.new(0, -4, 0) * CFrame.Angles(math.rad(90), 0, spinAngle)
									end)
									task.wait(RETURN_TIME)
								end
							end
						end
						task.wait(RETURN_TIME)
					end
				end)
			end
		end
	})

	PositionTab:Space()

	PositionTab:Button({
		Title = "Teleport Under Once",
		Desc = "Noclips for 1s, teleports underneath target(s) spinning fast up, then returns back",
		Justify = "Center",
		Callback = function()
			executeTeleportDownOnce()
		end
	})

	--// Closer Player Tab Elements
	CloserPlayerTab:Toggle({
		Title = "Closer Player",
		Desc = "Locks onto the nearest player and keeps spinning around them",
		Value = closerPlayerActive,
		Callback = function(state)
			closerPlayerActive = state
			if closerPlayerActive then
				local nearest, dist = getClosestPlayer()
				closestPlayer = nearest
				if nearest and dist then
					notify("Closer Player", string.format("Nearest: %s (%.1f studs)", nearest.Name, dist))
				else
					notify("Closer Player", "Enabled (no nearby player found)")
				end
				startCloserLoop()
			else
				closerToken += 1 -- stops the running loop
				closestPlayer = nil
				notify("Closer Player", "Disabled")
			end
		end
	})

	CloserPlayerTab:Space()

	CloserPlayerTab:Input({
		Title = "Target Players",
		Desc = "Optional: Closer Player only picks the nearest among these names (comma separated). Empty = anyone",
		Placeholder = "e.g. mika, John",
		Callback = function(text)
			targetPlayerNames = {}
			if text then
				for name in string.gmatch(text, "[^,]+") do
					local cleaned = name:match("^%s*(.-)%s*$")
					if cleaned and cleaned ~= "" then
						table.insert(targetPlayerNames, cleaned)
					end
				end
			end
			if #targetPlayerNames == 0 then
				notify("Target Players", "Cleared (anyone can be targeted)")
			else
				local names = {}
				for _, pl in ipairs(resolveNamedPlayers()) do table.insert(names, pl.Name) end
				notify("Target Players", #names > 0 and ("Targeting: " .. table.concat(names, ", ")) or "No player found with that name")
			end
		end
	})

	CloserPlayerTab:Space()

	CloserPlayerTab:Button({
		Title = "Teleport Closer Player",
		Desc = "Teleports once onto the closer player (right on them), then returns you back",
		Justify = "Center",
		Callback = function()
			local nearest, dist = getClosestPlayer()
			closestPlayer = nearest
			if not nearest then
				notify("Closer Player", "No closer player found")
				return
			end
			notify("Closer Player", string.format("Teleporting to %s (%.1f studs)", nearest.Name, dist or 0))
			executeTeleportOnce(true)
		end
	})

	--// Settings Tab Elements
	AutoClickTab:Input({
		Title = "Tool / Auto Name",
		Desc = "Tool name to equip & auto-click",
		Value = toolNameInput,
		Callback = function(text)
			if text then toolNameInput = text end
		end
	})

	SettingsTab:Space()

	SettingsTab:Input({
		Title = "Quick-Bind Key",
		Desc = "Enter key code name (e.g. Tab, T, E, Q)",
		Value = teleportKey.Name,
		Callback = function(text)
			if text and #text > 0 then
				local cleanedText = text:match("^%s*(.-)%s*$")
				local success, keyCode = pcall(function()
					return Enum.KeyCode[cleanedText]
				end)
				if success and keyCode then
					teleportKey = keyCode
					notify("TeleKill", "Quick-bind set to: " .. cleanedText)
				end
			end
		end
	})

	SettingsTab:Space()

	SettingsTab:Input({
		Title = "UI Toggle Key",
		Desc = "Enter key to hide/minimize UI (e.g. RightShift, LeftShift, Insert, K)",
		Value = uiToggleKey.Name,
		Callback = function(text)
			if text and #text > 0 then
				local cleanedText = text:match("^%s*(.-)%s*$")
				local success, keyCode = pcall(function()
					return Enum.KeyCode[cleanedText]
				end)
				if success and keyCode then
					uiToggleKey = keyCode
					notify("TeleKill", "UI Toggle Key set to: " .. cleanedText)
				end
			end
		end
	})

	SettingsTab:Space()

	AutoClickTab:Space()

	AutoClickTab:Slider({
		Title = "Click Delay (seconds)",
		Desc = "Time between clicks: small = FAST, big = SLOW (0.05 - 5 sec)",
		Step = 0.05,
		Value = {
			Min = 0.05,
			Max = 5,
			Default = autoClickDelay,
		},
		Callback = function(value)
			autoClickDelay = math.clamp(tonumber(value) or 0.05, 0.05, 5)
		end
	})

	AutoClickTab:Space()

	AutoClickTab:Toggle({
		Title = "Auto Clicker",
		Desc = "Equips the tool, activates it and fires the hit to your target(s) / closer player",
		Value = autoClickActive,
		Callback = function(state)
			autoClickActive = state
			if autoClickActive then
				spawnLoop(function()
					while autoClickActive do
						fireToolOnce(getTargetPlayers())
						task.wait(autoClickDelay)
					end
				end)
			end
		end
	})

	AutoClickTab:Space()

	AutoClickTab:Toggle({
		Title = "Fire on Teleport Once",
		Desc = "ON = tool fires while Teleport Once is on the target, stops when teleported back | OFF = no firing",
		Value = fireOnTeleport,
		Callback = function(state)
			fireOnTeleport = state
		end
	})

	--// Setting Tab Elements (Re-Execute, Teleport All Seconds 1 - 100 & Color)
	SettingTab:Button({
		Title = "Closer GUI",
		Desc = "Stops loops, rebuilds UI, and re-runs script",
		Justify = "Center",
		Callback = function()
			stopAllLoops()
			notify("Closer GUI", "Closer")

			task.spawn(function()
				task.wait(0.4)
				if SCRIPT_URL and SCRIPT_URL ~= "" then
					local ok, err = pcall(function()
						local src = game:HttpGet(SCRIPT_URL)
						local fn = loadstring(src)
						assert(type(fn) == "function", "Invalid script chunk")
						fn()
					end)
					if not ok then
						warn("Re-execute URL failed: " .. tostring(err))
						buildUI()
					end
				else
					local ok, err = pcall(buildUI)
					if not ok then
						warn("[TeleKill] Re-Execute rebuild failed: " .. tostring(err))
					end
				end
			end)
		end
	})

	SettingTab:Space()

	SettingTab:Toggle({
		Title = "Notifications",
		Desc = "Show pop-up messages (OFF = no messages)",
		Value = notificationsEnabled,
		Callback = function(state)
			notificationsEnabled = state
		end
	})

	SettingTab:Space()

	SettingTab:Slider({
		Title = "Teleport Seconds",
		Desc = "Seconds spent per player for ALL teleports: Main Operations, Positioning, Closer Player (1 - 100 sec)",
		Step = 1,
		Value = {
			Min = 1,
			Max = 100,
			Default = teleportAllSeconds,
		},
		Callback = function(value)
			teleportAllSeconds = clampSeconds(value)
		end
	})

	SettingTab:Space()

	-- Collect ALL themes WindUI has (falls back to the known built-in list)
	local allThemeNames = {}
	pcall(function()
		for name in pairs(WindUI:GetThemes()) do
			table.insert(allThemeNames, name)
		end
	end)
	if #allThemeNames == 0 then
		allThemeNames = {
			"Dark", "Light", "Rose", "Plant", "Red", "Indigo", "Sky", "Violet",
			"Amber", "Emerald", "Midnight", "Crimson", "MonokaiPro", "CottonCandy",
		}
	end
	table.sort(allThemeNames)
	table.insert(allThemeNames, 1, "Rainbow")

	-- Find the big main window frame of WindUI
	local function findMainFrame()
		local main
		pcall(function() main = Window.UIElements and Window.UIElements.Main end)
		if typeof(main) == "Instance" and main:IsA("GuiObject") then
			return main
		end

		local roots = {}
		pcall(function() if typeof(gethui) == "function" then table.insert(roots, gethui()) end end)
		table.insert(roots, CoreGui)
		local pg = player:FindFirstChildOfClass("PlayerGui")
		if pg then table.insert(roots, pg) end

		local viewX = 1920
		pcall(function() viewX = workspace.CurrentCamera.ViewportSize.X end)

		local best, bestArea = nil, 0
		for _, root in ipairs(roots) do
			for _, g in ipairs(root:GetChildren()) do
				if g:IsA("ScreenGui") and g.Name:find("WindUI") then
					for _, d in ipairs(g:GetDescendants()) do
						if d:IsA("Frame") and d.Visible and d.AbsoluteSize.X < viewX * 0.95 then
							local area = d.AbsoluteSize.X * d.AbsoluteSize.Y
							if area > bestArea then
								best, bestArea = d, area
							end
						end
					end
				end
			end
		end
		return best
	end

	local function startRainbow()
		rainbowActive = true

		-- Animated rainbow border around the main window
		local stroke
		pcall(function()
			local main = findMainFrame()
			if main then
				local old = main:FindFirstChild("RainbowStroke")
				if old then old:Destroy() end

				stroke = Instance.new("UIStroke")
				stroke.Name = "RainbowStroke"
				stroke.Thickness = 3
				stroke.Color = Color3.new(1, 1, 1)
				stroke.Parent = main

				local grad = Instance.new("UIGradient")
				grad.Color = ColorSequence.new({
					ColorSequenceKeypoint.new(0.00, Color3.fromHSV(0.00, 1, 1)),
					ColorSequenceKeypoint.new(0.17, Color3.fromHSV(0.17, 1, 1)),
					ColorSequenceKeypoint.new(0.33, Color3.fromHSV(0.33, 1, 1)),
					ColorSequenceKeypoint.new(0.50, Color3.fromHSV(0.50, 1, 1)),
					ColorSequenceKeypoint.new(0.67, Color3.fromHSV(0.67, 1, 1)),
					ColorSequenceKeypoint.new(0.83, Color3.fromHSV(0.83, 1, 1)),
					ColorSequenceKeypoint.new(1.00, Color3.fromHSV(1.00, 1, 1)),
				})
				grad.Parent = stroke
			end
		end)

		spawnLoop(function()
			local hue = 0
			while rainbowActive do
				hue = (hue + 0.01) % 1
				local c1 = Color3.fromHSV(hue, 1, 1)
				local c2 = Color3.fromHSV((hue + 0.15) % 1, 1, 1)

				pcall(function()
					local grad = stroke and stroke.Parent and stroke:FindFirstChildOfClass("UIGradient")
					if grad then grad.Rotation = (grad.Rotation + 4) % 360 end
				end)
				pcall(function()
					if WindUI.SetAccentColor then WindUI:SetAccentColor(c1) end
				end)
				pcall(function()
					if Window.OpenButton then Window.OpenButton.Color = ColorSequence.new(c1, c2) end
				end)
				pcall(function()
					local st = env._TELEKILL_EXEC2_STROKE
					if st and st.Parent then st.Color = c1 end
				end)
				task.wait(0.03)
			end

			pcall(function()
				if stroke then stroke:Destroy() end
			end)
		end)
	end

	SettingTab:Dropdown({
		Title = "Theme",
		Desc = "Select any WindUI theme (all available themes listed)",
		Values = allThemeNames,
		Value = selectedTheme,
		Callback = function(option)
			if option == "Rainbow" then
				if not rainbowActive then startRainbow() end
				notify("Setting", "Theme set to: Rainbow (animated)")
				return
			end
			rainbowActive = false
			selectedTheme = option
			applyThemeAndColor(selectedTheme, selectedHexColor)
			notify("Setting", "Theme set to: " .. tostring(option))
		end
	})

	SettingTab:Space()

	SettingTab:Input({
		Title = "Color (Hex)",
		Desc = "Enter hex color (e.g. #30FF6A, #FF3B30, #FFD60A)",
		Value = selectedHexColor,
		Callback = function(text)
			if text then
				selectedHexColor = text
				local parsed = hexToColor3(text)
				if parsed then
					applyThemeAndColor(selectedTheme, selectedHexColor)
					notify("Setting", "Color updated: " .. text)
				end
			end
		end
	})

	--// Fling Tab Elements
	FlingTab:Toggle({
		Title = "Fling on Teleport",
		Desc = "ON = Teleport Once flings the target while you are on them, fling stops when you teleport back | OFF = no fling",
		Value = flingOnTeleport,
		Callback = function(state)
			flingOnTeleport = state
		end
	})

	FlingTab:Space()

	FlingTab:Button({
		Title = "Fling Once",
		Desc = "Teleports onto the target (or closest player), flings them, then teleports back",
		Justify = "Center",
		Callback = function()
			executeTeleportOnce(false, true)
		end
	})

	--// Player Tab Elements
	PlayerTab:Toggle({
		Title = "Noclip",
		Desc = "Walk through walls all the time. (Teleports already noclip by themselves)",
		Value = noclipActive,
		Callback = function(state)
			setNoclip(state)
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Speed",
		Desc = "Changes your walk speed (set the amount with the slider below)",
		Value = speedEnabled,
		Callback = function(state)
			setSpeed(state)
		end
	})

	PlayerTab:Slider({
		Title = "Walk Speed",
		Desc = "Default Roblox speed is 16",
		Step = 1,
		Value = { Min = 16, Max = 300, Default = speedValue },
		Callback = function(value)
			speedValue = math.clamp(tonumber(value) or 50, 16, 300)
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Jump Power",
		Desc = "Changes how high you jump (set the amount with the slider below)",
		Value = jumpEnabled,
		Callback = function(state)
			setJump(state)
		end
	})

	PlayerTab:Slider({
		Title = "Jump Power Amount",
		Desc = "Default Roblox jump power is 50",
		Step = 1,
		Value = { Min = 50, Max = 500, Default = jumpValue },
		Callback = function(value)
			jumpValue = math.clamp(tonumber(value) or 100, 50, 500)
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Infinite Jump",
		Desc = "Jump again in mid-air as many times as you want",
		Value = infJumpActive,
		Callback = function(state)
			setInfJump(state)
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Fly + Noclip",
		Desc = "Fly through walls (turns Fly and Noclip on together)",
		Value = false,
		Callback = function(state)
			setFly(state)
			setNoclip(state)
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Fly",
		Desc = "Fly where your camera looks. Move with WASD / joystick, Space = up, Left Ctrl = down",
		Value = flyActive,
		Callback = function(state)
			setFly(state)
		end
	})

	PlayerTab:Slider({
		Title = "Fly Speed",
		Desc = "How fast you fly (10 - 300)",
		Step = 5,
		Value = { Min = 10, Max = 300, Default = flySpeed },
		Callback = function(value)
			flySpeed = math.clamp(tonumber(value) or 50, 10, 300)
		end
	})

	--// ESP Tab Elements
	ESPTab:Toggle({
		Title = "ESP",
		Desc = "Highlights all other players through walls",
		Value = espEnabled,
		Callback = function(state)
			espEnabled = state
			if espEnabled then
				spawnLoop(function()
					while espEnabled do
						updateESP()
						task.wait(espRainbow and 0.05 or 0.1)
					end
					clearESP()
				end)
			end
		end
	})

	ESPTab:Space()

	ESPTab:Toggle({
		Title = "Show Names",
		Desc = "Shows player names above their head",
		Value = espNames,
		Callback = function(state)
			espNames = state
		end
	})

	ESPTab:Space()

	ESPTab:Toggle({
		Title = "Show Distance",
		Desc = "Shows distance in studs",
		Value = espDistance,
		Callback = function(state)
			espDistance = state
		end
	})

	ESPTab:Space()

	local espColorPresets = {
		["Red"] = Color3.fromRGB(255, 60, 60),
		["Green"] = Color3.fromRGB(48, 255, 106),
		["Blue"] = Color3.fromRGB(60, 140, 255),
		["Yellow"] = Color3.fromRGB(255, 214, 10),
		["Cyan"] = Color3.fromRGB(0, 255, 255),
		["Purple"] = Color3.fromRGB(180, 80, 255),
		["Orange"] = Color3.fromRGB(255, 150, 30),
		["White"] = Color3.fromRGB(255, 255, 255),
	}

	ESPTab:Dropdown({
		Title = "ESP Color",
		Desc = "Pick a color, or Rainbow (Animated) = every player gets a moving rainbow color",
		Values = { "Rainbow (Animated)", "Red", "Green", "Blue", "Yellow", "Cyan", "Purple", "Orange", "White" },
		Value = "Red",
		Callback = function(option)
			if option == "Rainbow (Animated)" then
				espRainbow = true
			else
				espRainbow = false
				espColor = espColorPresets[option] or espColor
			end
		end
	})

	ESPTab:Space()

	ESPTab:Input({
		Title = "Custom Color (Hex)",
		Desc = "Optional: enter hex (e.g. #FF3C3C). Overrides the color above",
		Value = "",
		Callback = function(text)
			local parsed = hexToColor3(text)
			if parsed then
				espRainbow = false
				espColor = parsed
			end
		end
	})

	ESPTab:Space()

	ESPTab:Slider({
		Title = "Name Size",
		Desc = "Size of the name / distance text (8 - 40)",
		Step = 1,
		Value = {
			Min = 8,
			Max = 40,
			Default = espTextSize,
		},
		Callback = function(value)
			espTextSize = math.clamp(tonumber(value) or 14, 8, 40)
		end
	})

	--// Executor Tab Elements
	local function runSource(src, label)
		if type(src) ~= "string" or src:match("^%s*$") then
			notify("Executor", "Nothing to run")
			return
		end
		if typeof(loadstring) ~= "function" then
			notify("Executor", "loadstring is not available")
			return
		end

		local fn, compileErr = loadstring(src)
		if not fn then
			notify("Executor", "Syntax error: " .. tostring(compileErr):sub(1, 120))
			warn("[Executor] Syntax error: " .. tostring(compileErr))
			return
		end

		task.spawn(function()
			local ok, runErr = pcall(fn)
			if ok then
				notify("Executor", (label or "Script") .. " executed")
			else
				notify("Executor", "Error: " .. tostring(runErr):sub(1, 120))
				warn("[Executor] Runtime error: " .. tostring(runErr))
			end
		end)
	end

	--// Executor 2: standalone floating screen GUI
	local function toggleExecutor2GuiInner()
		if env._TELEKILL_EXEC2 and env._TELEKILL_EXEC2.Parent then
			env._TELEKILL_EXEC2.Enabled = not env._TELEKILL_EXEC2.Enabled
			return
		end

		local gui = Instance.new("ScreenGui")
		gui.Name = "TeleKillExecutor2"
		gui.ResetOnSpawn = false
		gui.DisplayOrder = 999
		gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
		gui.Enabled = true

		local okParent = false
		if typeof(gethui) == "function" then
			okParent = pcall(function() gui.Parent = gethui() end) and gui.Parent ~= nil
		end
		if not okParent then
			okParent = pcall(function() gui.Parent = CoreGui end) and gui.Parent ~= nil
		end
		if not okParent then
			gui.Parent = player:WaitForChild("PlayerGui")
		end
		env._TELEKILL_EXEC2 = gui

		local frame = Instance.new("Frame")
		frame.Size = UDim2.fromOffset(420, 300)
		frame.Position = UDim2.new(0.5, -210, 0.5, -150)
		frame.BackgroundColor3 = Color3.fromRGB(24, 24, 28)
		frame.BorderSizePixel = 0
		frame.Parent = gui
		Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 10)

		local stroke = Instance.new("UIStroke")
		stroke.Thickness = 2
		stroke.Color = Color3.fromRGB(60, 60, 70)
		stroke.Parent = frame
		env._TELEKILL_EXEC2_STROKE = stroke

		local bar = Instance.new("Frame")
		bar.Size = UDim2.new(1, 0, 0, 34)
		bar.BackgroundColor3 = Color3.fromRGB(34, 34, 40)
		bar.BorderSizePixel = 0
		bar.Parent = frame
		Instance.new("UICorner", bar).CornerRadius = UDim.new(0, 10)

		local title = Instance.new("TextLabel")
		title.Size = UDim2.new(1, -44, 1, 0)
		title.Position = UDim2.fromOffset(12, 0)
		title.BackgroundTransparency = 1
		title.Text = "Executor 2"
		title.Font = Enum.Font.GothamBold
		title.TextSize = 14
		title.TextColor3 = Color3.fromRGB(235, 235, 235)
		title.TextXAlignment = Enum.TextXAlignment.Left
		title.Parent = bar

		local closeBtn = Instance.new("TextButton")
		closeBtn.Size = UDim2.fromOffset(28, 24)
		closeBtn.Position = UDim2.new(1, -34, 0, 5)
		closeBtn.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
		closeBtn.Text = "X"
		closeBtn.Font = Enum.Font.GothamBold
		closeBtn.TextSize = 13
		closeBtn.TextColor3 = Color3.new(1, 1, 1)
		closeBtn.BorderSizePixel = 0
		closeBtn.Parent = bar
		Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 6)
		closeBtn.MouseButton1Click:Connect(function()
			gui.Enabled = false
		end)

		local box = Instance.new("TextBox")
		box.Size = UDim2.new(1, -20, 1, -90)
		box.Position = UDim2.fromOffset(10, 44)
		box.BackgroundColor3 = Color3.fromRGB(16, 16, 20)
		box.BorderSizePixel = 0
		box.ClearTextOnFocus = false
		box.MultiLine = true
		box.TextWrapped = false
		box.TextXAlignment = Enum.TextXAlignment.Left
		box.TextYAlignment = Enum.TextYAlignment.Top
		box.Font = Enum.Font.Code
		box.TextSize = 14
		box.TextColor3 = Color3.fromRGB(220, 220, 220)
		box.PlaceholderText = "-- paste Lua code here"
		box.Text = ""
		box.Parent = frame
		Instance.new("UICorner", box).CornerRadius = UDim.new(0, 8)

		local function makeButton(text, xOffset, color, callback)
			local b = Instance.new("TextButton")
			b.Size = UDim2.new(0.5, -15, 0, 32)
			b.Position = UDim2.new(xOffset, xOffset == 0 and 10 or 5, 1, -42)
			b.BackgroundColor3 = color
			b.Text = text
			b.Font = Enum.Font.GothamBold
			b.TextSize = 14
			b.TextColor3 = Color3.new(1, 1, 1)
			b.BorderSizePixel = 0
			b.Parent = frame
			Instance.new("UICorner", b).CornerRadius = UDim.new(0, 8)
			b.MouseButton1Click:Connect(callback)
		end

		makeButton("Execute", 0, Color3.fromRGB(48, 160, 90), function()
			runSource(box.Text, "Executor 2")
		end)
		makeButton("Clear", 0.5, Color3.fromRGB(80, 80, 90), function()
			box.Text = ""
		end)

		-- Dragging (mouse + touch) by the title bar
		local dragging, dragStart, startPos = false, nil, nil
		bar.InputBegan:Connect(function(input)
			if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
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
		UserInputService.InputChanged:Connect(function(input)
			if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
				local delta = input.Position - dragStart
				frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
			end
		end)
	end

	local function toggleExecutor2Gui()
		local ok, err = pcall(toggleExecutor2GuiInner)
		if not ok then
			warn("[Executor 2] " .. tostring(err))
			notify("Executor 2", "Error: " .. tostring(err):sub(1, 100))
		end
	end

	local function buildExecutorElements(tab)
		local code, url = "", ""

		tab:Input({
			Title = "Script",
			Desc = "Paste Lua code here, press Enter, then tap Execute",
			Placeholder = "print('Hello')",
			Type = "Textarea",
			Callback = function(text)
				code = text or ""
			end
		})

		tab:Space()

		tab:Button({
			Title = "Execute",
			Desc = "Runs the script above",
			Justify = "Center",
			Callback = function()
				runSource(code, "Script")
			end
		})

		tab:Space()

		tab:Input({
			Title = "Script URL",
			Desc = "Link to a raw Lua script (e.g. raw.githubusercontent.com/...)",
			Placeholder = "https://...",
			Callback = function(text)
				url = (text or ""):match("^%s*(.-)%s*$")
			end
		})

		tab:Space()

		tab:Button({
			Title = "Execute URL",
			Desc = "Downloads and runs the script from the link above",
			Justify = "Center",
			Callback = function()
				if url == "" then
					notify("Executor", "Enter a URL first")
					return
				end
				local ok, src = pcall(function()
					return game:HttpGet(url)
				end)
				if ok then
					runSource(src, "URL script")
				else
					notify("Executor", "Failed to download script")
				end
			end
		})

		tab:Space()

		tab:Button({
			Title = "Clear",
			Desc = "Forgets the saved script and URL",
			Justify = "Center",
			Callback = function()
				code = ""
				url = ""
				notify("Executor", "Cleared (retype in the box to replace the text)")
			end
		})

		tab:Space()

		tab:Button({
			Title = "Show to GUI Executor 2",
			Desc = "Opens/closes the floating Executor 2 window on screen",
			Justify = "Center",
			Callback = function()
				toggleExecutor2Gui()
			end
		})
	end

	buildExecutorElements(ExecutorTab)

	SettingTab:Space()

	SettingTab:Button({
		Title = "Open Executor",
		Desc = "Jump to the Executor tab",
		Justify = "Center",
		Callback = function()
			pcall(function() ExecutorTab:Select() end)
		end
	})

	SettingTab:Space()

	SettingTab:Button({
		Title = "Show to GUI Executor 2",
		Desc = "Opens/closes the floating Executor 2 window on screen",
		Justify = "Center",
		Callback = function()
			toggleExecutor2Gui()
		end
	})

	inputConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
		if input.UserInputType == Enum.UserInputType.Keyboard then
			if not gameProcessed and input.KeyCode == teleportKey then
				executeTeleportOnce()
			end

			if input.KeyCode == uiToggleKey then
				pcall(function()
					Window:Toggle()
				end)
			end
		end
	end)
end

--// Background Updater for Closer Player (single loop across rebuilds)
if not closerLoopRunning then
	closerLoopRunning = true
	task.spawn(function()
		while true do
			if closerPlayerActive then
				closestPlayer = getClosestPlayer()
			else
				closestPlayer = nil
			end
			task.wait(0.1)
		end
	end)
end

buildUI()
