local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local VirtualUser = game:GetService("VirtualUser")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer

--// Load WindUI Library (Official Loader)
local cloneref = (cloneref or clonereference or function(instance) return instance end)
local ReplicatedStorage = cloneref(game:GetService("ReplicatedStorage"))
local HttpService = cloneref(game:GetService("HttpService"))

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

--// Settings & State Variables
local SPIN_DURATION = 0.5
local RETURN_TIME = 0.01 -- Fast spin update rate
local teleportKey = Enum.KeyCode.Tab

local targetPlayerNames = {}
local loopTeleporting = false
local teleportAllActive = false
local autoClickActive = false
local spinAngle = 0
local toolNameInput = "Sword"

--// Helper: Find Single Player Matching a Query
local function findSinglePlayer(text)
	if not text then return nil end
	text = text:lower():match("^%s*(.-)%s*$")
	if text == "" then return nil end

	for _, target in Players:GetPlayers() do
		if target ~= player then
			local username = target.Name:lower()
			local displayName = target.DisplayName:lower()

			if username:sub(1, #text) == text or displayName:sub(1, #text) == text then
				return target
			end
		end
	end
	return nil
end

--// Helper: Find Multiple Players Matching current input list
local function getTargetPlayers()
	local targets = {}
	if #targetPlayerNames == 0 then return targets end

	for _, text in ipairs(targetPlayerNames) do
		local matchedPlayer = findSinglePlayer(text)
		if matchedPlayer and not table.find(targets, matchedPlayer) then
			table.insert(targets, matchedPlayer)
		end
	end
	return targets
end

--// Core Action: Perform Teleport Once
local function executeTeleportOnce()
	local character = player.Character
	local root = character and character:FindFirstChild("HumanoidRootPart")
	if not root then 
		WindUI:Notify({ Title = "TeleKill", Content = "Character not ready", Duration = 2 })
		return 
	end

	local targets = getTargetPlayers()
	if #targets == 0 then
		WindUI:Notify({ Title = "TeleKill", Content = "No targets found/matched", Duration = 2 })
		return
	end

	-- Save position before teleporting
	local originalCFrame = root.CFrame

	task.spawn(function()
		for _, target in ipairs(targets) do
			local targetCharacter = target.Character
			local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")
			if targetRoot then
				local startTime = os.clock()
				while os.clock() - startTime < SPIN_DURATION do
					local activeChar = player.Character
					local activeRoot = activeChar and activeChar:FindFirstChild("HumanoidRootPart")
					if not activeRoot then break end

					spinAngle += math.rad(45)
					activeRoot.Velocity = Vector3.new(0, 0, 0)
					activeRoot.CFrame = targetRoot.CFrame * CFrame.new(3, 0, 0) * CFrame.Angles(0, spinAngle, 0)
					task.wait(0.01)
				end
			end
		end

		-- Teleport back to starting position safely
		if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
			local finalRoot = player.Character.HumanoidRootPart
			finalRoot.Velocity = Vector3.new(0, 0, 0)
			finalRoot.CFrame = originalCFrame
		end
	end)
end

--// Create Window
local Window = WindUI:CreateWindow({
	Title = "TeleKill Hub | WindUI",
	Folder = "TeleKillHub",
	Icon = "solar:folder-2-bold-duotone",
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
			Color3.fromHex("#30FF6A"),
			Color3.fromHex("#e7ff2f")
		),
	},
	Topbar = {
		Height = 44,
		ButtonsType = "Mac",
	},
})

--// Window Tags
do
	Window:Tag({
		Title = "v" .. (WindUI.Version or "1.0"),
		Icon = "github",
		Color = Color3.fromHex("#1c1c1c"),
		Border = true,
	})
end

--// Create Sections & Tabs
local TelekillSection = Window:Section({
	Title = "TeleKill Controls",
})

local MainTab = TelekillSection:Tab({
	Title = "Main Operations",
	Icon = "solar:home-2-bold",
	Border = true,
})

local PositionTab = TelekillSection:Tab({
	Title = "Positioning",
	Icon = "solar:map-point-bold",
	Border = true,
})

local SettingsTab = TelekillSection:Tab({
	Title = "Configuration",
	Icon = "solar:settings-bold",
	Border = true,
})

--// Main Tab Elements
MainTab:Input({
	Title = "Target Players",
	Desc = "Separate multiple names/tags with commas (e.g. Roblox, John)",
	Placeholder = "Enter Player Names...",
	Callback = function(text)
		targetPlayerNames = {}
		for name in string.gmatch(text, "[^,]+") do
			local cleaned = name:match("^%s*(.-)%s*$")
			if cleaned ~= "" then
				table.insert(targetPlayerNames, cleaned)
			end
		end
	end
})

MainTab:Space()

MainTab:Button({
	Title = "Teleport Once",
	Desc = "Teleports & spins around target(s), then teleports back to starting position",
	Justify = "Center",
	Callback = function()
		executeTeleportOnce()
	end
})

MainTab:Space()

MainTab:Toggle({
	Title = "Loop Teleport Targets",
	Desc = "Continuously teleports & spins around target(s) without returning",
	Value = false,
	Callback = function(state)
		loopTeleporting = state
		if loopTeleporting then
			task.spawn(function()
				while loopTeleporting do
					local targets = getTargetPlayers()
					if #targets == 0 then
						task.wait(0.5)
					else
						for _, target in ipairs(targets) do
							if not loopTeleporting then break end
							local tickStart = os.clock()
							while loopTeleporting and (os.clock() - tickStart < 0.5) do
								local character = player.Character
								local root = character and character:FindFirstChild("HumanoidRootPart")
								local targetCharacter = target.Character
								local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")

								if not root or not targetRoot or not target.Parent then break end

								spinAngle += math.rad(45)
								root.Velocity = Vector3.new(0, 0, 0)
								root.CFrame = targetRoot.CFrame * CFrame.new(3, 0, 0) * CFrame.Angles(0, spinAngle, 0)
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
	Desc = "Cycles through and attacks/teleports to ALL server players",
	Value = false,
	Callback = function(state)
		teleportAllActive = state
		if teleportAllActive then
			task.spawn(function()
				while teleportAllActive do
					local otherPlayers = {}
					for _, p in Players:GetPlayers() do
						if p ~= player then
							table.insert(otherPlayers, p)
						end
					end

					if #otherPlayers == 0 then
						task.wait(1)
					else
						for _, target in otherPlayers do
							if not teleportAllActive then break end
							local tickStart = os.clock()
							while teleportAllActive and (os.clock() - tickStart < 0.5) do
								local activeChar = player.Character
								local activeRoot = activeChar and activeChar:FindFirstChild("HumanoidRootPart")
								local targetChar = target.Character
								local targetRoot = targetChar and targetChar:FindFirstChild("HumanoidRootPart")

								if not activeRoot or not targetRoot or not target.Parent then break end

								spinAngle += math.rad(45)
								activeRoot.Velocity = Vector3.new(0, 0, 0)
								activeRoot.CFrame = targetRoot.CFrame * CFrame.new(3, 0, 0) * CFrame.Angles(0, spinAngle, 0)
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

--// Position Tab Elements (One-Click Teleport Underneath)
PositionTab:Button({
	Title = "Teleport Under Once",
	Desc = "Teleports directly underneath target(s) facing up, then returns back",
	Justify = "Center",
	Callback = function()
		local character = player.Character
		local root = character and character:FindFirstChild("HumanoidRootPart")
		if not root then 
			WindUI:Notify({ Title = "TeleKill", Content = "Character not ready", Duration = 2 })
			return 
		end

		local targets = getTargetPlayers()
		if #targets == 0 then
			WindUI:Notify({ Title = "TeleKill", Content = "No targets found/matched", Duration = 2 })
			return
		end

		local originalCFrame = root.CFrame

		task.spawn(function()
			for _, target in ipairs(targets) do
				local targetCharacter = target.Character
				local targetRoot = targetCharacter and targetCharacter:FindFirstChild("HumanoidRootPart")
				if targetRoot then
					local startTime = os.clock()
					while os.clock() - startTime < SPIN_DURATION do
						local activeChar = player.Character
						local activeRoot = activeChar and activeChar:FindFirstChild("HumanoidRootPart")
						if not activeRoot then break end

						-- Stop velocity to avoid speed/falling glitch
						activeRoot.Velocity = Vector3.new(0, 0, 0)
						activeRoot.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
						-- Position 4 studs underneath facing upwards
						activeRoot.CFrame = targetRoot.CFrame * CFrame.new(0, -4, 0) * CFrame.Angles(math.rad(90), 0, 0)
						task.wait(0.01)
					end
				end
			end

			-- Safely return to starting position
			if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
				local finalRoot = player.Character.HumanoidRootPart
				finalRoot.Velocity = Vector3.new(0, 0, 0)
				finalRoot.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
				finalRoot.CFrame = originalCFrame
			end
		end)
	end
})

--// Settings Tab Elements
SettingsTab:Input({
	Title = "Tool / Auto Name",
	Desc = "Tool name to equip & auto-click",
	Value = "Sword",
	Callback = function(text)
		toolNameInput = text
	end
})

SettingsTab:Space()

SettingsTab:Input({
	Title = "Quick-Bind Key",
	Desc = "Enter key code name (e.g. Tab, T, E, Q)",
	Value = "Tab",
	Callback = function(text)
		if text and #text > 0 then
			local cleanedText = text:match("^%s*(.-)%s*$")
			local success, keyCode = pcall(function()
				return Enum.KeyCode[cleanedText]
			end)
			if success and keyCode then
				teleportKey = keyCode
				WindUI:Notify({ Title = "TeleKill", Content = "Quick-bind set to: " .. cleanedText, Duration = 2 })
			end
		end
	end
})

SettingsTab:Space()

SettingsTab:Toggle({
	Title = "Auto Clicker",
	Desc = "Automatically equips and activates the tool",
	Value = false,
	Callback = function(state)
		autoClickActive = state
		if autoClickActive then
			task.spawn(function()
				while autoClickActive do
					local inputName = toolNameInput:lower():match("^%s*(.-)%s*$")
					if inputName == "" then break end

					local character = player.Character
					local backpack = player:FindFirstChildOfClass("Backpack")

					if character then
						local equippedTool = nil
						for _, item in character:GetChildren() do
							if item:IsA("Tool") and item.Name:lower() == inputName then
								equippedTool = item
								break
							end
						end

						if not equippedTool and backpack then
							for _, item in backpack:GetChildren() do
								if item:IsA("Tool") and item.Name:lower() == inputName then
									item.Parent = character
									equippedTool = item
									break
								end
							end
						end

						if equippedTool then
							pcall(function()
								equippedTool:Activate()
								VirtualUser:Button1Down(Vector2.new(0,0))
								task.wait(0.01)
								VirtualUser:Button1Up(Vector2.new(0,0))
							end)
						end
					end
					task.wait(0.05)
				end
			end)
		end
	end
})

--// Keyboard Quick-Bind Support (Runs "Teleport Once" logic)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if not gameProcessed and input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == teleportKey then
		executeTeleportOnce()
	end
end)
