-- Tele Kill Hub, creating by Claude

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local VirtualUser = game:GetService("VirtualUser")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local HttpService = game:GetService("HttpService")

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
local teamCheckEnabled = false -- ON = never target players on your Roblox team
local teamNames = {} -- teammates you added by name (never targeted)
local currentTeleportTarget = nil -- player we are currently teleported onto (any teleport mode)
local lastTeleportTargetTime = 0
local lastTeleportFire = 0
local walkFlingActive = false
local fakeLagActive = false
local fakeLagTeleportOnClose = true
local fakeLagHideBody = false -- ON = anyone you walk into gets flung, no teleport needed
local flyHeadActive = false -- ON = your head stays locked onto the target's head
local lastFlingStep = 0 -- time of our last own fling step (anti fling pauses while we fling)
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

--// Anti-AFK: prevents the idle kick (can be switched in the Anti tab)
local antiAfkEnabled = true
if not env._TELEKILL_ANTIAFK then
	env._TELEKILL_ANTIAFK = true
	pcall(function()
		player.Idled:Connect(function()
			if not antiAfkEnabled then return end
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

--// Helper: is this player a teammate we should never target?
local function isProtected(target)
	if teamCheckEnabled and player.Team ~= nil and target.Team == player.Team then
		return true
	end
	local username = target.Name:lower()
	local displayName = target.DisplayName:lower()
	for _, n in ipairs(teamNames) do
		local name = n:lower()
		if username == name or displayName == name then
			return true
		end
	end
	return false
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

	local usingNames = #targetPlayerNames > 0
	for _, target in ipairs(candidates) do
		if target ~= player and (usingNames or not isProtected(target)) then
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

--// Anti Fling: other players can't push you, and crazy spin/velocity on you gets cancelled
local antiFlingActive, antiFlingConn = false, nil
local antiFlingBackup = {}

local function restoreAntiFlingCollisions()
	for part in pairs(antiFlingBackup) do
		antiFlingBackup[part] = nil
		pcall(function()
			if part.Parent then
				part.CanCollide = true
			end
		end)
	end
end

local function setAntiFling(state)
	antiFlingActive = state
	if antiFlingConn then
		antiFlingConn:Disconnect()
		antiFlingConn = nil
	end
	if not state then
		restoreAntiFlingCollisions()
		return
	end

	antiFlingConn = RunService.Stepped:Connect(function()
		-- pause while WE are flinging someone (needs collision)
		if os.clock() - lastFlingStep < 0.5 then
			restoreAntiFlingCollisions()
			return
		end

		-- 1) no collision with other players' bodies
		for _, other in ipairs(Players:GetPlayers()) do
			if other ~= player and other.Character then
				for _, part in ipairs(other.Character:GetDescendants()) do
					if part:IsA("BasePart") and part.CanCollide then
						antiFlingBackup[part] = true
						part.CanCollide = false
					end
				end
			end
		end

		-- 2) cancel crazy velocity / spin on yourself
		local root = getCharacterRoot(player.Character)
		if root and not flyActive then
			local lin = root.AssemblyLinearVelocity
			local ang = root.AssemblyAngularVelocity
			local limit = math.max(300, speedValue * 2)
			local horizontal = Vector3.new(lin.X, 0, lin.Z).Magnitude

			if ang.Magnitude > 100 or horizontal > limit or math.abs(lin.Y) > 500 then
				root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
				root.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
			end
		end
	end)
end

local function resetPlayerMods()
	setAntiFling(false)
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

		-- Don't fight the Fling code: it needs CanCollide = true for a moment to throw the target
		if os.clock() - lastFlingStep < 0.5 then
			return
		end

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

--// Called by every teleport loop: remembers who we are on, and fires the tool at them if enabled
local function teleportFireStep(target)
	currentTeleportTarget = target
	lastTeleportTargetTime = os.clock()
	if fireOnTeleport and os.clock() - lastTeleportFire >= autoClickDelay then
		lastTeleportFire = os.clock()
		fireToolOnce({ target })
	end
end

--// Fly the Head: keep your head locked onto the target's head (follows them around)
local flyHeadToken = 0

local function setFlyHead(state)
	flyHeadActive = state
	flyHeadToken += 1
	local myToken = flyHeadToken
	if not state then
		restoreCollision()
		return
	end

	spawnLoop(function()
		while flyHeadActive and myToken == flyHeadToken do
			local character = player.Character
			local root = getCharacterRoot(character)
			local targets = getTargetPlayers()
			local target = targets[1]
			local targetHead = target and target.Character and target.Character:FindFirstChild("Head")

			if root and targetHead and character then
				if flingOnTeleport then
					-- Fling mode: ram into the target's head instead of just following it
					pcall(function()
						root.CanCollide = true
						lastFlingStep = os.clock()
						root.CFrame = targetHead.CFrame
						root.AssemblyAngularVelocity = Vector3.new(0, 30000, 0)
						root.AssemblyLinearVelocity = Vector3.new(15000, 15000, 15000)
					end)
				else
					noclipCharacter(character) -- stops you getting stuck on walls/floor while following
					pcall(function()
						root.CFrame = targetHead.CFrame
						root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
					end)
				end
				task.wait(RETURN_TIME)
			else
				-- no target yet: don't spin a tight empty loop, check again shortly
				task.wait(0.3)
			end
		end
	end)
end

--// Walk Fling: fling anyone you bump into while just walking around (no teleport)
local function setWalkFling(state)
	walkFlingActive = state
	if not state then return end

	spawnLoop(function()
		local cooldowns = {}

		while walkFlingActive do
			local character = player.Character
			local root = getCharacterRoot(character)

			if root then
				for _, other in ipairs(Players:GetPlayers()) do
					if other ~= player and not isProtected(other) then
						local otherRoot = getCharacterRoot(other.Character)
						local now = os.clock()

						if otherRoot and (otherRoot.Position - root.Position).Magnitude <= 7
							and (not cooldowns[other] or now - cooldowns[other] >= 0.6) then
							cooldowns[other] = now

							-- Short burst: ram into them, then snap yourself back so YOU don't get flung too
							local savedCFrame = root.CFrame
							local burstEnd = os.clock() + 0.15

							while walkFlingActive and os.clock() < burstEnd do
								local r = getCharacterRoot(player.Character)
								local tr = getCharacterRoot(other.Character)
								if not r or not tr or not other.Parent then break end

								pcall(function()
									r.CanCollide = true
									lastFlingStep = os.clock()
									r.CFrame = tr.CFrame
									r.AssemblyAngularVelocity = Vector3.new(0, 30000, 0)
									r.AssemblyLinearVelocity = Vector3.new(15000, 15000, 15000)
								end)
								task.wait(RETURN_TIME)
							end

							-- Snap back to where you were standing and stop your own velocity
							pcall(function()
								local r = getCharacterRoot(player.Character)
								if r then
									r.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
									r.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
									r.CFrame = savedCFrame
								end
							end)
						end
					end
				end
			end
			task.wait(RETURN_TIME)
		end
	end)
end

--// Fake Lag (Ghost Clone): freeze your real body in place while you control a
--// fake clone that actually moves, so other players see you standing still/lagging.
local FakeLagControls = nil
pcall(function()
	local PlayerModule = require(player:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule"))
	FakeLagControls = PlayerModule:GetControls()
end)

local fakeLagRealChar, fakeLagClone, fakeLagOriginalCF = nil, nil, nil
local fakeLagConns = {}
local fakeLagAnims = {}

local FAKELAG_ANIM_IDS = {
	Idle = "rbxassetid://507766388",
	Run  = "rbxassetid://507767714",
	Jump = "rbxassetid://507765000",
	Fall = "rbxassetid://507767968",
}

local function fakeLagLoadAnim(hum, id)
	local a = Instance.new("Animation")
	a.AnimationId = id
	local ok, track = pcall(function() return hum:LoadAnimation(a) end)
	return ok and track or nil
end

local function fakeLagStopAllAnims()
	for _, track in pairs(fakeLagAnims) do
		if track and track.IsPlaying then
			pcall(function() track:Stop(0.1) end)
		end
	end
end

local function setFakeLag(state)
	if state == fakeLagActive then return end

	if not state then
		fakeLagActive = false
		for _, conn in ipairs(fakeLagConns) do
			pcall(function() conn:Disconnect() end)
		end
		fakeLagConns = {}
		fakeLagStopAllAnims()
		fakeLagAnims = {}

		if fakeLagClone then
			local lastCF
			pcall(function() lastCF = fakeLagClone:GetPrimaryPartCFrame() end)
			pcall(function() fakeLagClone:Destroy() end)
			fakeLagClone = nil

			local rc = fakeLagRealChar
			local root = rc and rc:FindFirstChild("HumanoidRootPart")
			if root then
				root.Anchored = false
				if fakeLagTeleportOnClose and lastCF then
					root.CFrame = lastCF
				elseif fakeLagOriginalCF then
					root.CFrame = fakeLagOriginalCF
				end
			end
			pcall(function()
				workspace.CurrentCamera.CameraSubject = rc and rc:FindFirstChildOfClass("Humanoid")
			end)
		end
		return
	end

	-- Turning ON
	local realChar = player.Character
	local root = realChar and realChar:FindFirstChild("HumanoidRootPart")
	if not root then
		notify("Fake Lag", "Character not ready")
		return
	end

	fakeLagActive = true
	fakeLagRealChar = realChar
	fakeLagOriginalCF = root.CFrame

	local ok, clone = pcall(function()
		realChar.Archivable = true
		return realChar:Clone()
	end)
	if not ok or not clone then
		fakeLagActive = false
		notify("Fake Lag", "Could not create clone")
		return
	end
	clone.Name = "TeleKillFakeLagClone"
	clone.Parent = workspace
	fakeLagClone = clone

	if fakeLagHideBody then
		root.CFrame = fakeLagOriginalCF + Vector3.new(0, 5000, 0)
	end
	root.Anchored = true

	for _, v in ipairs(clone:GetDescendants()) do
		if v:IsA("LocalScript") or v:IsA("Script") then
			pcall(function() v:Destroy() end)
		elseif v:IsA("BasePart") then
			v.Anchored = false
			v.CanCollide = (v.Name == "HumanoidRootPart")
			if v.Name == "HumanoidRootPart" then
				v.Transparency = 1
			end
		end
	end

	local cloneHum = clone:FindFirstChildOfClass("Humanoid")
	local cloneRoot = clone:FindFirstChild("HumanoidRootPart")
	if not cloneHum or not cloneRoot then
		pcall(function() clone:Destroy() end)
		fakeLagClone = nil
		fakeLagActive = false
		root.Anchored = false
		notify("Fake Lag", "Clone is missing parts")
		return
	end

	fakeLagAnims.Idle = fakeLagLoadAnim(cloneHum, FAKELAG_ANIM_IDS.Idle)
	fakeLagAnims.Run  = fakeLagLoadAnim(cloneHum, FAKELAG_ANIM_IDS.Run)
	fakeLagAnims.Jump = fakeLagLoadAnim(cloneHum, FAKELAG_ANIM_IDS.Jump)
	fakeLagAnims.Fall = fakeLagLoadAnim(cloneHum, FAKELAG_ANIM_IDS.Fall)

	pcall(function() workspace.CurrentCamera.CameraSubject = cloneHum end)
	if fakeLagAnims.Idle then fakeLagAnims.Idle:Play() end

	table.insert(fakeLagConns, cloneHum.StateChanged:Connect(function(_, newState)
		if newState == Enum.HumanoidStateType.Jumping then
			fakeLagStopAllAnims()
			if fakeLagAnims.Jump then fakeLagAnims.Jump:Play() end
		elseif newState == Enum.HumanoidStateType.Freefall then
			fakeLagStopAllAnims()
			if fakeLagAnims.Fall then fakeLagAnims.Fall:Play() end
		elseif newState == Enum.HumanoidStateType.Landed then
			fakeLagStopAllAnims()
			if fakeLagAnims.Idle then fakeLagAnims.Idle:Play() end
		end
	end))

	table.insert(fakeLagConns, RunService.RenderStepped:Connect(function()
		if not fakeLagActive or not cloneHum or not cloneRoot or not cloneRoot.Parent then return end

		local cam = workspace.CurrentCamera
		if not cam then return end

		cloneHum.WalkSpeed = speedEnabled and speedValue or 16

		local inputVector = Vector3.new(0, 0, 0)
		if FakeLagControls then
			local ok2, mv = pcall(function() return FakeLagControls:GetMoveVector() end)
			if ok2 and mv then inputVector = mv end
		end
		if inputVector.Magnitude == 0 then
			if UserInputService:IsKeyDown(Enum.KeyCode.W) then inputVector += Vector3.new(0, 0, -1) end
			if UserInputService:IsKeyDown(Enum.KeyCode.S) then inputVector += Vector3.new(0, 0, 1) end
			if UserInputService:IsKeyDown(Enum.KeyCode.A) then inputVector += Vector3.new(-1, 0, 0) end
			if UserInputService:IsKeyDown(Enum.KeyCode.D) then inputVector += Vector3.new(1, 0, 0) end
		end

		local finalDir = (cam.CFrame.RightVector * inputVector.X) + (cam.CFrame.LookVector * -inputVector.Z)
		finalDir = Vector3.new(finalDir.X, 0, finalDir.Z)

		local isGrounded = cloneHum:GetState() == Enum.HumanoidStateType.Running
			or cloneHum:GetState() == Enum.HumanoidStateType.RunningNoPhysics

		if finalDir.Magnitude > 0 then
			cloneHum:Move(finalDir.Unit, false)
			if isGrounded and fakeLagAnims.Run and not fakeLagAnims.Run.IsPlaying then
				fakeLagStopAllAnims()
				fakeLagAnims.Run:Play(0.1)
			end
		else
			cloneHum:Move(Vector3.new(0, 0, 0), false)
			if isGrounded and fakeLagAnims.Idle and not fakeLagAnims.Idle.IsPlaying then
				fakeLagStopAllAnims()
				fakeLagAnims.Idle:Play(0.1)
			end
		end
	end))

	table.insert(fakeLagConns, UserInputService.JumpRequest:Connect(function()
		if fakeLagActive and cloneHum and cloneHum:GetState() ~= Enum.HumanoidStateType.Freefall then
			cloneHum.UseJumpPower = true
			cloneHum.JumpPower = jumpEnabled and jumpValue or 50
			cloneHum.Jump = true
		end
	end))
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
							lastFlingStep = os.clock()
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
							lastFlingStep = os.clock()
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
	setFlyHead(false)
	walkFlingActive = false
	setFakeLag(false)
	setAntiFling(false)
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
					teleportFireStep(locked)
					spinAngle += math.rad(180)

					if flingOnTeleport then
						pcall(function()
							root.CanCollide = true
							lastFlingStep = os.clock()
							root.CFrame = targetRoot.CFrame
							root.AssemblyAngularVelocity = Vector3.new(0, 30000, 0)
							root.AssemblyLinearVelocity = Vector3.new(15000, 15000, 15000)
						end)
					else
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
				end

				task.wait(RETURN_TIME)
			end
		end
	end)
end

--// Config save / load (needs executor file functions: writefile, readfile, listfiles...)
local CONFIG_ROOT = "TeleKillHub"
local CONFIG_FOLDER = "TeleKillHub/configs"

local function filesAvailable()
	return typeof(writefile) == "function" and typeof(readfile) == "function"
end

local function ensureConfigFolder()
	pcall(function()
		if typeof(makefolder) == "function" then
			if typeof(isfolder) ~= "function" or not isfolder(CONFIG_ROOT) then
				makefolder(CONFIG_ROOT)
			end
			if typeof(isfolder) ~= "function" or not isfolder(CONFIG_FOLDER) then
				makefolder(CONFIG_FOLDER)
			end
		end
	end)
end

local function cleanConfigName(name)
	name = tostring(name or ""):gsub("[^%w _%-]", ""):match("^%s*(.-)%s*$")
	return name
end

local function listConfigs()
	local names = {}
	pcall(function()
		if typeof(listfiles) == "function" then
			for _, path in ipairs(listfiles(CONFIG_FOLDER)) do
				local n = tostring(path):match("([^/\\]+)%.json$")
				if n then
					table.insert(names, n)
				end
			end
		end
	end)
	table.sort(names)
	return names
end

local AUTOLOAD_FILE = "TeleKillHub/autoload.txt"

local function getAutoLoad()
	local name
	pcall(function()
		if typeof(readfile) == "function" and typeof(isfile) == "function" and isfile(AUTOLOAD_FILE) then
			name = cleanConfigName(readfile(AUTOLOAD_FILE))
		end
	end)
	if name == "" then name = nil end
	return name
end

local function setAutoLoad(name)
	ensureConfigFolder()
	return pcall(function()
		writefile(AUTOLOAD_FILE, name or "")
	end)
end

local function collectSettings()
	return {
		targetPlayerNames = targetPlayerNames,
		teleportAllSeconds = teleportAllSeconds,
		autoClickDelay = autoClickDelay,
		toolNameInput = toolNameInput,
		teleportKey = teleportKey.Name,
		uiToggleKey = uiToggleKey.Name,
		selectedTheme = selectedTheme,
		selectedHexColor = selectedHexColor,
		teleportDownUnderActive = teleportDownUnderActive,
		fireOnTeleport = fireOnTeleport,
		flingOnTeleport = flingOnTeleport,
		notificationsEnabled = notificationsEnabled,
		espNames = espNames,
		espDistance = espDistance,
		espRainbow = espRainbow,
		espColor = espColor:ToHex(),
		espTextSize = espTextSize,
		speedValue = speedValue,
		jumpValue = jumpValue,
		flySpeed = flySpeed,
		antiAfkEnabled = antiAfkEnabled,
		teamCheckEnabled = teamCheckEnabled,
		teamNames = teamNames,
	}
end

local function applySettings(t)
	if type(t) ~= "table" then return end

	if type(t.targetPlayerNames) == "table" then targetPlayerNames = t.targetPlayerNames end
	if tonumber(t.teleportAllSeconds) then teleportAllSeconds = clampSeconds(t.teleportAllSeconds) end
	if tonumber(t.autoClickDelay) then autoClickDelay = math.clamp(t.autoClickDelay, 0.05, 5) end
	if type(t.toolNameInput) == "string" then toolNameInput = t.toolNameInput end
	if type(t.teleportKey) == "string" and Enum.KeyCode[t.teleportKey] then teleportKey = Enum.KeyCode[t.teleportKey] end
	if type(t.uiToggleKey) == "string" and Enum.KeyCode[t.uiToggleKey] then uiToggleKey = Enum.KeyCode[t.uiToggleKey] end
	if type(t.selectedTheme) == "string" then selectedTheme = t.selectedTheme end
	if type(t.selectedHexColor) == "string" then selectedHexColor = t.selectedHexColor end
	if type(t.teleportDownUnderActive) == "boolean" then teleportDownUnderActive = t.teleportDownUnderActive end
	if type(t.fireOnTeleport) == "boolean" then fireOnTeleport = t.fireOnTeleport end
	if type(t.flingOnTeleport) == "boolean" then flingOnTeleport = t.flingOnTeleport end
	if type(t.notificationsEnabled) == "boolean" then notificationsEnabled = t.notificationsEnabled end
	if type(t.espNames) == "boolean" then espNames = t.espNames end
	if type(t.espDistance) == "boolean" then espDistance = t.espDistance end
	if type(t.espRainbow) == "boolean" then espRainbow = t.espRainbow end
	if type(t.espColor) == "string" then
		local ok, col = pcall(function() return Color3.fromHex(t.espColor) end)
		if ok and col then espColor = col end
	end
	if tonumber(t.espTextSize) then espTextSize = math.clamp(t.espTextSize, 8, 40) end
	if tonumber(t.speedValue) then speedValue = math.clamp(t.speedValue, 16, 300) end
	if tonumber(t.jumpValue) then jumpValue = math.clamp(t.jumpValue, 50, 500) end
	if tonumber(t.flySpeed) then flySpeed = math.clamp(t.flySpeed, 10, 300) end
	if type(t.antiAfkEnabled) == "boolean" then antiAfkEnabled = t.antiAfkEnabled end
	if type(t.teamCheckEnabled) == "boolean" then teamCheckEnabled = t.teamCheckEnabled end
	if type(t.teamNames) == "table" then teamNames = t.teamNames end
end

--// Chat command: shows/hides the UI
--// NOTE: "/e" is Roblox's built-in emote command and swallows the message before
--// it ever reaches Chatted/SendingMessage, so that prefix can never work here.
--// Using "/tk" (and ";tk" in case your game also claims "/") instead.
if not env._TELEKILL_CHATCMD then
	env._TELEKILL_CHATCMD = true

	local function handleToggleCommand(msg)
		msg = tostring(msg or ""):match("^%s*(.-)%s*$"):lower()
		if msg == "/c" or msg == "/c telekill" or msg == "/c tkl"
			or msg == "/tk" or msg == ";tk" or msg == ";telekill" or msg == ";tkl" then
			pcall(function()
				if env._TELEKILL_WINDOW then
					env._TELEKILL_WINDOW:Toggle()
				end
			end)
		end
	end

	pcall(function()
		player.Chatted:Connect(handleToggleCommand)
	end)
	pcall(function()
		local TextChatService = game:GetService("TextChatService")
		if TextChatService.ChatInputBarConfiguration then
			TextChatService.SendingMessage:Connect(function(message)
				handleToggleCommand(message.Text)
			end)
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
			Enabled = false, -- hidden on PC: reopen with the UI Toggle Key (RightShift) or "/e telekill"
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

	local FlyHeadTab = PositioningSection:Tab({
		Title = "Fly the Head",
		Icon = "solar:scooter-bold",
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

	local TeamTab = TelekillSection:Tab({
		Title = "Team",
		Icon = "solar:users-group-rounded-bold",
		Border = true,
	})

	local AntiTab = TelekillSection:Tab({
		Title = "Anti",
		Icon = "solar:shield-bold",
		Border = true,
	})

	local ConfigTab = TelekillSection:Tab({
		Title = "Configs",
		Icon = "solar:diskette-bold",
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
		Value = table.concat(targetPlayerNames, ", "),
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

									teleportFireStep(target)

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
							if p ~= player and not isProtected(p) then
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

									teleportFireStep(target)

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
							if p ~= player and not isProtected(p) then
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

									teleportFireStep(target)

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

									teleportFireStep(target)

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
						local fireTargets
						if currentTeleportTarget and os.clock() - lastTeleportTargetTime < 0.5 then
							fireTargets = { currentTeleportTarget } -- hit whoever Teleport All / Loop / Under is on
						else
							fireTargets = getTargetPlayers()
						end
						fireToolOnce(fireTargets)
						task.wait(autoClickDelay)
					end
				end)
			end
		end
	})

	AutoClickTab:Space()

	AutoClickTab:Toggle({
		Title = "Fire on Teleport Once",
		Desc = "ON = tool fires at the target during Teleport Once / All / Loop / Under / Closer Player, stops when teleported back | OFF = no firing",
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

	--// Fly the Head Tab Elements
	FlyHeadTab:Toggle({
		Title = "Fly the Head",
		Desc = "Your head stays locked onto the target's head; with Fling on Teleport ON it flings them instead",
		Value = flyHeadActive,
		Callback = function(state)
			setFlyHead(state)
		end
	})

	FlyHeadTab:Space()

	FlyHeadTab:Button({
		Title = "Teleport the Head",
		Desc = "Teleports your head onto the target's (or closest player's) head once",
		Justify = "Center",
		Callback = function()
			local root = getCharacterRoot(player.Character)
			local targets = getTargetPlayers()
			local target = targets[1]
			local targetHead = target and target.Character and target.Character:FindFirstChild("Head")
			if not root or not targetHead then
				notify("Fly the Head", "No target found")
				return
			end
			if flingOnTeleport then
				pcall(function()
					root.CanCollide = true
					lastFlingStep = os.clock()
					root.CFrame = targetHead.CFrame
					root.AssemblyAngularVelocity = Vector3.new(0, 30000, 0)
					root.AssemblyLinearVelocity = Vector3.new(15000, 15000, 15000)
				end)
			else
				pcall(function()
					root.CFrame = targetHead.CFrame
					root.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
				end)
			end
			notify("Fly the Head", "Teleported to " .. target.Name .. "'s head")
		end
	})

	--// Team Tab Elements (teammates are never targeted / teleported to / flung)
	TeamTab:Toggle({
		Title = "Team Check",
		Desc = "ON = never target players on your Roblox team",
		Value = teamCheckEnabled,
		Callback = function(state)
			teamCheckEnabled = state
		end
	})

	TeamTab:Space()

	TeamTab:Input({
		Title = "Teammates",
		Desc = "Names of friends/teammates to never target (comma separated)",
		Placeholder = "e.g. mika, lua",
		Value = table.concat(teamNames, ", "),
		Callback = function(text)
			teamNames = {}
			if text then
				for name in string.gmatch(text, "[^,]+") do
					local cleaned = name:match("^%s*(.-)%s*$")
					if cleaned and cleaned ~= "" then
						table.insert(teamNames, cleaned)
					end
				end
			end
			notify("Team", #teamNames .. " teammate(s) saved")
		end
	})

	TeamTab:Space()

	TeamTab:Button({
		Title = "Add Closest Player",
		Desc = "Adds the nearest player to your teammates list",
		Justify = "Center",
		Callback = function()
			local myRoot = getCharacterRoot(player.Character)
			local nearest, best = nil, math.huge
			for _, other in ipairs(Players:GetPlayers()) do
				local r = other ~= player and getCharacterRoot(other.Character)
				if myRoot and r then
					local d = (r.Position - myRoot.Position).Magnitude
					if d < best then
						best, nearest = d, other
					end
				end
			end
			if nearest then
				table.insert(teamNames, nearest.Name)
				notify("Team", "Added teammate: " .. nearest.Name)
			else
				notify("Team", "No player nearby")
			end
		end
	})

	TeamTab:Space()

	TeamTab:Button({
		Title = "Clear Teammates",
		Desc = "Empties the teammates list",
		Justify = "Center",
		Callback = function()
			teamNames = {}
			notify("Team", "Teammates cleared")
		end
	})

	--// Anti Tab Elements
	AntiTab:Toggle({
		Title = "Anti Fling",
		Desc = "Other players can't fling you: no collision with them + cancels crazy spin/velocity",
		Value = antiFlingActive,
		Callback = function(state)
			setAntiFling(state)
		end
	})

	AntiTab:Space()

	AntiTab:Toggle({
		Title = "Anti AFK",
		Desc = "Stops the idle kick so you are not disconnected for standing still",
		Value = antiAfkEnabled,
		Callback = function(state)
			antiAfkEnabled = state
		end
	})

	--// Configs Tab Elements (save / load by name)
	local configName = ""
	local selectedConfig = nil
	local configDropdown = nil

	local function configPath(name)
		return CONFIG_FOLDER .. "/" .. name .. ".json"
	end

	local function refreshConfigList()
		local list = listConfigs()
		if #list == 0 then list = { "(no saved configs)" } end
		pcall(function()
			if configDropdown and configDropdown.Refresh then
				configDropdown:Refresh(list)
			end
		end)
		return list
	end

	ConfigTab:Input({
		Title = "Config Name",
		Desc = "Type a name for your settings (letters, numbers, spaces)",
		Placeholder = "e.g. my setup",
		Callback = function(text)
			configName = cleanConfigName(text)
		end
	})

	ConfigTab:Space()

	ConfigTab:Button({
		Title = "Save Config",
		Desc = "Saves all your current settings under the name above",
		Justify = "Center",
		Callback = function()
			if not filesAvailable() then
				notify("Configs", "Your executor has no file support (writefile)")
				return
			end
			if configName == "" then
				notify("Configs", "Type a config name first")
				return
			end
			ensureConfigFolder()
			local ok, err = pcall(function()
				writefile(configPath(configName), HttpService:JSONEncode(collectSettings()))
			end)
			if ok then
				notify("Configs", "Saved: " .. configName)
				refreshConfigList()
			else
				notify("Configs", "Save failed")
				warn("[Configs] " .. tostring(err))
			end
		end
	})

	ConfigTab:Space()

	local initialList = listConfigs()
	if #initialList == 0 then initialList = { "(no saved configs)" } end

	configDropdown = ConfigTab:Dropdown({
		Title = "Saved Configs",
		Desc = "Pick a saved config, then press Load Config",
		Values = initialList,
		Callback = function(option)
			selectedConfig = option
		end
	})

	ConfigTab:Space()

	ConfigTab:Button({
		Title = "Load Config",
		Desc = "Loads the selected config and rebuilds the UI with those settings",
		Justify = "Center",
		Callback = function()
			if not filesAvailable() then
				notify("Configs", "Your executor has no file support (readfile)")
				return
			end
			local name = selectedConfig
			if not name or name == "(no saved configs)" then
				name = configName ~= "" and configName or nil
			end
			if not name then
				notify("Configs", "Select a config first")
				return
			end

			local ok, data = pcall(function()
				return HttpService:JSONDecode(readfile(configPath(name)))
			end)
			if not ok or type(data) ~= "table" then
				notify("Configs", "Could not load: " .. tostring(name))
				return
			end

			applySettings(data)
			stopAllLoops()
			notify("Configs", "Loaded: " .. name)

			task.spawn(function()
				task.wait(0.4)
				local okBuild, errBuild = pcall(buildUI)
				if not okBuild then
					warn("[Configs] rebuild failed: " .. tostring(errBuild))
				end
			end)
		end
	})

	ConfigTab:Space()

	ConfigTab:Button({
		Title = "Set Auto Load",
		Desc = "The selected config loads by itself every time you run the script",
		Justify = "Center",
		Callback = function()
			if not filesAvailable() then
				notify("Configs", "Your executor has no file support")
				return
			end
			local name = selectedConfig
			if not name or name == "(no saved configs)" then
				name = configName ~= "" and configName or nil
			end
			if not name then
				notify("Configs", "Select a config first")
				return
			end
			if setAutoLoad(name) then
				notify("Configs", "Auto Load: " .. name)
			else
				notify("Configs", "Could not set Auto Load")
			end
		end
	})

	ConfigTab:Space()

	ConfigTab:Button({
		Title = "Clear Auto Load",
		Desc = "Stops loading a config automatically",
		Justify = "Center",
		Callback = function()
			if setAutoLoad(nil) then
				notify("Configs", "Auto Load cleared")
			end
		end
	})

	ConfigTab:Space()

	ConfigTab:Button({
		Title = "Delete Config",
		Desc = "Deletes the selected config",
		Justify = "Center",
		Callback = function()
			local name = selectedConfig
			if not name or name == "(no saved configs)" then
				notify("Configs", "Select a config first")
				return
			end
			local ok = pcall(function()
				if typeof(delfile) == "function" then
					delfile(configPath(name))
				else
					error("no delfile")
				end
			end)
			if ok then
				notify("Configs", "Deleted: " .. name)
				selectedConfig = nil
				refreshConfigList()
			else
				notify("Configs", "Delete failed (executor has no delfile)")
			end
		end
	})

	ConfigTab:Space()

	ConfigTab:Button({
		Title = "Refresh List",
		Desc = "Reloads the saved configs list",
		Justify = "Center",
		Callback = function()
			refreshConfigList()
			notify("Configs", "List refreshed")
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
		Title = "Fake Lag (Ghost Clone)",
		Desc = "Your real body freezes in place (looks laggy to others) while you control a hidden clone",
		Value = fakeLagActive,
		Callback = function(state)
			setFakeLag(state)
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Fake Lag: Teleport Back",
		Desc = "ON = when you turn Fake Lag off, you appear where the clone was | OFF = back to where you started",
		Value = fakeLagTeleportOnClose,
		Callback = function(state)
			fakeLagTeleportOnClose = state
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Fake Lag: Hide Real Body",
		Desc = "Moves your frozen real body far away so it can't be seen while Fake Lag is on",
		Value = fakeLagHideBody,
		Callback = function(state)
			fakeLagHideBody = state
		end
	})

	PlayerTab:Space()

	PlayerTab:Toggle({
		Title = "Walk Fling",
		Desc = "ON = anyone you walk/bump into gets flung automatically, no teleport needed",
		Value = walkFlingActive,
		Callback = function(state)
			setWalkFling(state)
		end
	})

	PlayerTab:Space()

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

--// Auto Load: apply the chosen config before the UI is built
do
	local autoName = getAutoLoad()
	if autoName then
		pcall(function()
			applySettings(HttpService:JSONDecode(readfile(CONFIG_FOLDER .. "/" .. autoName .. ".json")))
		end)
	end
end

buildUI()
