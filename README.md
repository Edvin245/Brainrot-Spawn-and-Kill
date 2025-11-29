--// BrainrotClickHandler.server.lua — v42
-- BASE: v41 (gems + crit fixed, pooled brainrots)
-- Changes in v42:
--  - Each brainrot reads Configuration.RespawnTime (NumberValue/IntValue) on spawn.
--  - That RespawnTime is stored in BrainrotState.respawnTime.
--  - Death respawn delay uses state.respawnTime (per-brainrot), falling back to global RespawnTime.
--  - Initial spawn no longer does any weird 3600s clamp; everything respawns strictly by config.

-----------------------------
-- SERVICES
-----------------------------
local ServerStorage     = game:GetService("ServerStorage")
local Workspace         = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService      = game:GetService("TweenService")
local Players           = game:GetService("Players")
local RunService        = game:GetService("RunService")
local DataStoreService  = game:GetService("DataStoreService")
local Debris            = game:GetService("Debris")

-- [PERSISTENCE] Upgrades DataStore (UNCHANGED, VERSIONED)
local UpgradeStore = DataStoreService:GetDataStore("PlayerUpgradesV3")

-----------------------------
-- REMOTE HELPERS
-----------------------------
local function getOrCreateRemote(name)
	local r = ReplicatedStorage:FindFirstChild(name)
	if not r then
		r = Instance.new("RemoteEvent")
		r.Name = name
		r.Parent = ReplicatedStorage
	end
	return r
end

local shakeEvent         = getOrCreateRemote("BrainrotClickShake")
local lootPopupEvent     = getOrCreateRemote("BrainrotCollected")
local displayHealthEvent = getOrCreateRemote("DisplayBrainrotHealth")
local critEvent          = getOrCreateRemote("BrainrotCriticalHit")

local screenShakeEvent   = getOrCreateRemote("BrainrotScreenShake")
local focusCollectEvent  = getOrCreateRemote("BrainrotCollectFocus")

-----------------------------
-- CONFIGURATION
-----------------------------
local RespawnTime       = 5      -- fallback if Configuration.RespawnTime is missing/invalid
local IdleResetTime     = 10     -- HP fully resets after 10s of no hits
local CollectTime       = 0.50
local CollectionSoundId = "rbxassetid://9085320874"

local ClickSoundId      = "rbxassetid://95457013391750"
local ClickSoundIds = {
	"rbxassetid://71536184154470",
	"rbxassetid://95457013391750",
	"rbxassetid://77268396695038",
	"rbxassetid://86582791348874",
}

local CritSoundId       = "rbxassetid://132185946768399"

local MinDistance       = 10
local ClickCooldown     = 0.10

-- Gem drop config
local GEM_DROP_CHANCE_PERCENT = 10
local GEM_WEIGHTS = { [1]=30,[2]=25,[3]=15,[4]=10,[5]=7,[6]=5,[7]=4,[8]=2,[9]=1,[10]=1 }
local BONUS_GEM_CHANCE = 0

-- Crit config
local CRIT_CHANCE_PERCENT = 10
local CRIT_MULTIPLIER     = 2

-- Network throttling
local HEALTH_BROADCAST_INTERVAL = 0.25   -- max 4 HP updates per second per brainrot per player
local MIN_SHAKE_INTERVAL        = 0.12   -- max ~8 shake events per second per player

-----------------------------
-- AREAS & ACTIVE TRACKING
-----------------------------
local areasFolder      = Workspace:WaitForChild("Areas")
local activeBrainrots  = {}  -- { {Template=Model, Area=Instance, Position=Vector3}, ... }

-----------------------------
-- BRAINROT POOL
-----------------------------
local brainrotPoolFolder = ServerStorage:FindFirstChild("BrainrotPool")
if not brainrotPoolFolder then
	brainrotPoolFolder = Instance.new("Folder")
	brainrotPoolFolder.Name = "BrainrotPool"
	brainrotPoolFolder.Parent = ServerStorage
end

-- template -> { pooled clones }
local brainrotPools: {[Instance]: {Model}} = {}

local function getPooledBrainrot(template: Model): Model
	local pool = brainrotPools[template]
	if pool and #pool > 0 then
		local inst = table.remove(pool)
		if inst then
			return inst
		end
	end

	local clone = template:Clone()
	pcall(function()
		if clone:IsA("Model") and clone.ModelStreamingMode then
			clone.ModelStreamingMode = Enum.ModelStreamingMode.Atomic
		end
	end)
	return clone
end

local function releaseBrainrot(template: Model, inst: Model)
	if not inst then return end
	inst.Parent = brainrotPoolFolder

	local pool = brainrotPools[template]
	if not pool then
		pool = {}
		brainrotPools[template] = pool
	end
	table.insert(pool, inst)
end

-----------------------------
-- PLAYER UPGRADES
-----------------------------
local DEFAULT_UPGRADES = {
	Damage = 0,
	CoinMultiplier = 0,
	CritChance = 0,
	GemDropChance = 0,

	DamageMultiplier = 1,
	DamageMultShop = 1,

	CritDmgMultShop = 1,
	Pass_CritDamageFactor = 1,
	Pass_CPSFactor = 1,

	GemYieldShop = 1,
	Pass_GemYieldFactor = 1,
}

local function ensureUpgradeFolder(player)
	local folder = player:FindFirstChild("Upgrades")
	if not folder then
		folder = Instance.new("Folder")
		folder.Name = "Upgrades"
		folder.Parent = player
	end
	for statName, defaultValue in pairs(DEFAULT_UPGRADES) do
		if not folder:FindFirstChild(statName) then
			local nv = Instance.new("NumberValue")
			nv.Name = statName
			nv.Value = defaultValue
			nv.Parent = folder
		end
	end
	return folder
end

local function loadUpgrades(player)
	local upgradesFolder = ensureUpgradeFolder(player)
	local key = "U_" .. player.UserId
	local ok, data = pcall(function() return UpgradeStore:GetAsync(key) end)
	if ok and type(data) == "table" then
		for statName, val in pairs(data) do
			local nv = upgradesFolder:FindFirstChild(statName)
			if not nv then
				nv = Instance.new("NumberValue")
				nv.Name = statName
				nv.Parent = upgradesFolder
			end
			nv.Value = tonumber(val) or 0
		end

		local dmLegacy = upgradesFolder:FindFirstChild("DamageMultiplier")
		if dmLegacy and (dmLegacy.Value <= 0) then dmLegacy.Value = 1 end

		local dmUnified = upgradesFolder:FindFirstChild("DamageMultShop")
		if dmUnified and (dmUnified.Value <= 0) then dmUnified.Value = 1 end
	else
		if not ok then
			warn(("[Upgrades] Load failed for %s: %s"):format(player.Name, tostring(data)))
		end
	end
end

local function saveUpgrades(player)
	local upgradesFolder = player:FindFirstChild("Upgrades")
	if not upgradesFolder then return end
	local payload = {}
	for _, child in ipairs(upgradesFolder:GetChildren()) do
		if child:IsA("NumberValue") then
			payload[child.Name] = child.Value
		end
	end
	local key = "U_" .. player.UserId
	local ok, err = pcall(function() UpgradeStore:SetAsync(key, payload) end)
	if not ok then
		warn(("[Upgrades] Save failed for %s: %s"):format(player.Name, tostring(err)))
	end
end

Players.PlayerAdded:Connect(loadUpgrades)
Players.PlayerRemoving:Connect(saveUpgrades)
game:BindToClose(function()
	for _, plr in ipairs(Players:GetPlayers()) do
		saveUpgrades(plr)
	end
end)

-----------------------------
-- STATS READING
-----------------------------
local function readNum(folder, name, default)
	local n = folder and folder:FindFirstChild(name)
	if n and n:IsA("NumberValue") then
		return tonumber(n.Value) or default
	end
	return default
end

-- Returns: damageFinal, critChancePct, gemChancePct, critMultEff
local function computeEffectiveCombat(upgradesFolder)
	local dmgShop    = readNum(upgradesFolder, "Damage", 0)
	local dmgPassAdd = readNum(upgradesFolder, "Pass_DamageAdd", 0)

	local dmgMultUnified = math.max(
		1,
		readNum(upgradesFolder, "DamageMultShop", readNum(upgradesFolder, "DamageMultiplier", 1))
	)

	-- Clamp so VIP + 2x etc. don't blow up beyond ~2.5x
	if dmgMultUnified > 2.5 then
		dmgMultUnified = 2.5
	end

	local damageFinal = (1 + math.max(0, dmgShop) + math.max(0, dmgPassAdd)) * dmgMultUnified

	local critChancePct = math.max(
		0,
		CRIT_CHANCE_PERCENT
			+ (readNum(upgradesFolder, "CritChance", 0) + readNum(upgradesFolder, "Pass_CritChanceAdd", 0)) * 100
	)

	local gemChancePct  = math.clamp(
		GEM_DROP_CHANCE_PERCENT
			+ (readNum(upgradesFolder, "GemDropChance", 0) + readNum(upgradesFolder, "Pass_GemDropAdd", 0)) * 100,
		0, 100
	)

	local extraCritMult = readNum(upgradesFolder, "CritDmgMultShop", 1) * readNum(upgradesFolder, "Pass_CritDamageFactor", 1)
	if extraCritMult < 1 then extraCritMult = 1 end
	local critMultEff = CRIT_MULTIPLIER * extraCritMult

	return damageFinal, critChancePct, gemChancePct, critMultEff
end

local function computeEffectiveCPSFactor(upgradesFolder)
	local cmAdd   = readNum(upgradesFolder, "CoinMultiplier", 0.0)
	local passCPS = readNum(upgradesFolder, "Pass_CPSFactor", 1.0)
	return (1 + math.max(0, cmAdd)), math.max(1, passCPS)
end

local function computeGemYieldEff(upgradesFolder)
	local eff = readNum(upgradesFolder, "GemYieldEff", 0)
	if eff > 0 then return eff end
	local shop = readNum(upgradesFolder, "GemYieldShop", 1)
	local pass = readNum(upgradesFolder, "Pass_GemYieldFactor", 1)
	return math.max(1, shop * pass)
end

-----------------------------
-- SPAWN GEOMETRY
-----------------------------
local function getSpawnBounds(area)
	if area:IsA("BasePart") then
		return area.CFrame, area.Size
	elseif area:IsA("Model") then
		return area:GetBoundingBox()
	end
	return nil, nil
end

local function getTemplateHorizontalRadius(template)
	if not template then return 0 end
	local part = template.PrimaryPart or template:FindFirstChildWhichIsA("BasePart")
	if part then
		local s = part.Size
		return math.max(s.X, s.Z) * 0.5
	end
	local ok, _, size = pcall(function() return template:GetBoundingBox() end)
	if ok and size then
		return math.max(size.X, size.Z) * 0.5
	end
	return 0
end

local function isFarEnoughFromOthers(newPos, template, area, minDist)
	local newXz = Vector3.new(newPos.X, 0, newPos.Z)
	local newRadius = getTemplateHorizontalRadius(template)
	for _, posEntry in ipairs(activeBrainrots) do
		if posEntry.Area == area then
			local existingPos = posEntry.Position
			if existingPos then
				local existingXz = Vector3.new(existingPos.X, 0, existingPos.Z)
				local dist = (existingXz - newXz).Magnitude
				local existingRadius = getTemplateHorizontalRadius(posEntry.Template)
				local required = minDist + newRadius + existingRadius
				if dist < required then
					return false
				end
			end
		end
	end
	return true
end

local function getRandomSpawnPosition(template, area)
	local cf, size = getSpawnBounds(area)
	if not cf or not size then
		return Vector3.new(0,5,0)
	end

	local margin = 0.08
	local halfX = size.X * 0.5 * (1 - margin * 2)
	local halfZ = size.Z * 0.5 * (1 - margin * 2)
	local maxAttempts = 250

	for _ = 1, maxAttempts do
		local rx = (math.random() - 0.5) * 2 * halfX
		local rz = (math.random() - 0.5) * 2 * halfZ
		local candidate = (cf * CFrame.new(rx, -size.Y * 0.5 + 2, rz)).p
		if isFarEnoughFromOthers(candidate, template, area, MinDistance) then
			return candidate
		end
	end

	local fallback = (cf * CFrame.new(
		math.clamp(math.random(-halfX, halfX), -halfX, halfX),
		-size.Y * 0.5 + 2,
		math.clamp(math.random(-halfZ, halfZ), -halfZ, halfZ)
		)).p
	return fallback
end

-----------------------------
-- STABILITY HELPERS
-----------------------------
local function ensurePrimaryPart(model: Model)
	if model.PrimaryPart then return model.PrimaryPart end
	local main = model:FindFirstChild("MainPart") or model:FindFirstChildWhichIsA("BasePart")
	if main then
		model.PrimaryPart = main
		return main
	end
	return nil
end

local function hardFreezeModel(model: Model)
	for _, d in ipairs(model:GetDescendants()) do
		if d:IsA("BasePart") then
			d.Anchored = true
			d.CanCollide = false
			pcall(function() d:SetNetworkOwner(nil) end)
		end
	end
end

local function safePivot(model: Model, cf: CFrame)
	local ok = pcall(function() model:PivotTo(cf) end)
	if ok then return end

	local pp = ensurePrimaryPart(model)
	if pp then
		pcall(function() model:SetPrimaryPartCFrame(cf) end)
		return
	end

	local current = CFrame.new()
	pcall(function() current = model:GetPivot() end)
	local delta = cf.Position - current.Position
	for _, d in ipairs(model:GetDescendants()) do
		if d:IsA("BasePart") then
			d.CFrame = d.CFrame + delta
		end
	end
end

-----------------------------
-- GEM HELPERS
-----------------------------
local function weightedGemRoll(weightsDict)
	local total = 0
	for _, w in pairs(weightsDict) do total += math.max(0, w) end
	if total <= 0 then return 1 end
	local roll = math.random(1, total)
	local cumulative = 0
	for amount = 1, 10 do
		local w = weightsDict[amount] or 0
		if w > 0 then
			cumulative += w
			if roll <= cumulative then
				return amount
			end
		end
	end
	return 1
end

-----------------------------
-- REWARDS
-----------------------------
local function getOrCreateGemsStat(player)
	local leaderstats = player:FindFirstChild("leaderstats")
	if not leaderstats then return nil end

	local candidates = { "Gems", "GEMs", "G", "Gems 💎", "💎Gems" }
	for _, name in ipairs(candidates) do
		local stat = leaderstats:FindFirstChild(name)
		if stat and (stat:IsA("IntValue") or stat:IsA("NumberValue")) then
			return stat
		end
	end

	local gems = Instance.new("IntValue")
	gems.Name = "Gems"
	gems.Value = 0
	gems.Parent = leaderstats
	return gems
end

local function giveBrainrot(player, brainrotModel)
	local inventory   = player:FindFirstChild("Inventory")
	local leaderstats = player:FindFirstChild("leaderstats")
	if not inventory or not leaderstats then return end

	local config = brainrotModel:FindFirstChild("Configuration")
	local cps    = (config and config:FindFirstChild("CoinsPerSecond") and config.CoinsPerSecond.Value) or 1
	local name   = brainrotModel.Name
	local rarity = (config and config:FindFirstChild("Rarity") and config.Rarity.Value) or "Common"

	local upgrades = player:FindFirstChild("Upgrades")
	if upgrades then
		local cmLocalFactor, _ = computeEffectiveCPSFactor(upgrades)
		cps = cps * (cmLocalFactor or 1)
	end

	local existing = inventory:FindFirstChild(name)
	if existing then
		local count = existing:FindFirstChild("Count")
		if not count then
			count = Instance.new("IntValue")
			count.Name = "Count"
			count.Value = 1
			count.Parent = existing
		else
			count.Value += 1
		end
	else
		local newBrainrot = Instance.new("IntValue")
		newBrainrot.Name = name
		newBrainrot.Value = cps
		newBrainrot.Parent = inventory

		local count = Instance.new("IntValue")
		count.Name = "Count"
		count.Value = 1
		count.Parent = newBrainrot
	end

	local best = leaderstats:FindFirstChild("Best Brainrot")
	if best and cps > best.Value then best.Value = cps end

	lootPopupEvent:FireClient(player, { Name = name, Rarity = rarity, CPS = cps, Model = brainrotModel })
end

local function addGemsSilent(player, amount)
	amount = tonumber(amount) or 0
	if amount <= 0 then return end

	local gemsStat = getOrCreateGemsStat(player)
	if not gemsStat then return end

	gemsStat.Value = gemsStat.Value + amount
end

-----------------------------
-- LIGHTWEIGHT COLLECT ORB
-----------------------------
local function playCollectOrb(startPos: Vector3, player)
	local anchor = Instance.new("Part")
	anchor.Name = "__PickupSoundAnchor"
	anchor.Anchored = true
	anchor.CanCollide = false
	anchor.CanTouch = false
	anchor.Transparency = 1
	anchor.CFrame = CFrame.new(startPos)
	anchor.Parent = Workspace

	local sound = Instance.new("Sound")
	sound.Name    = "RespawnSound"
	sound.SoundId = CollectionSoundId
	sound.Volume  = 0.5
	sound.Parent  = anchor
	sound:Play()
	Debris:AddItem(anchor, 2)

	local targetPos = startPos
	if player and player.Character then
		local char = player.Character
		local hrp =
			char:FindFirstChild("HumanoidRootPart")
			or char.PrimaryPart
			or char:FindFirstChild("Torso")
			or char:FindFirstChild("UpperTorso")

		if hrp and hrp:IsA("BasePart") then
			targetPos = hrp.Position + Vector3.new(0, 2, 0)
		end
	end

	local orb = Instance.new("Part")
	orb.Name = "__BrainrotCollectOrb"
	orb.Shape = Enum.PartType.Ball
	orb.Material = Enum.Material.Neon
	orb.Color = Color3.new(1, 1, 1)
	orb.Size = Vector3.new(1, 1, 1)
	orb.Anchored = true
	orb.CanCollide = false
	orb.CanTouch = false
	orb.CFrame = CFrame.new(startPos)
	orb.Parent = Workspace

	local moveTime = 0.25
	local moveInfo = TweenInfo.new(
		moveTime,
		Enum.EasingStyle.Quad,
		Enum.EasingDirection.In
	)

	local goal = {
		CFrame       = CFrame.new(targetPos),
		Size         = Vector3.new(0.5, 0.5, 0.5),
		Transparency = 1,
	}
	TweenService:Create(orb, moveInfo, goal):Play()
	Debris:AddItem(orb, moveTime + 0.1)
end

-----------------------------
-- SOUND HELPERS
-----------------------------
local function ensureClickAndCritSounds(mainPart: BasePart)
	if not mainPart:FindFirstChild("ClickSound") then
		local clickSound = Instance.new("Sound")
		clickSound.Name    = "ClickSound"
		clickSound.SoundId = ClickSoundId
		clickSound.Volume  = 0.5
		clickSound.Parent  = mainPart
	end

	for i, id in ipairs(ClickSoundIds) do
		local name = "ClickSound" .. i
		if not mainPart:FindFirstChild(name) then
			local s = Instance.new("Sound")
			s.Name = name
			s.SoundId = id
			s.Volume = 0.5
			s.Parent = mainPart
		end
	end

	if not mainPart:FindFirstChild("CritSound") then
		local cs = Instance.new("Sound")
		cs.Name = "CritSound"
		cs.SoundId = CritSoundId
		cs.Volume = 0.65
		cs.Parent = mainPart
		cs.RollOffMaxDistance = 80
	end
end

local function playRandomClick(mainPart: BasePart)
	local variants = {}
	for i = 1, #ClickSoundIds do
		local s = mainPart:FindFirstChild("ClickSound" .. i)
		if s and s:IsA("Sound") then
			table.insert(variants, s)
		end
	end
	if #variants > 0 then
		local pick = variants[math.random(1, #variants)]
		pcall(function() pick:Play() end)
		return
	end
	local legacy = mainPart:FindFirstChild("ClickSound")
	if legacy and legacy:IsA("Sound") then
		pcall(function() legacy:Play() end)
	end
end

local function playCrit(mainPart: BasePart)
	local cs = mainPart:FindFirstChild("CritSound")
	if cs and cs:IsA("Sound") then
		pcall(function() cs:Play() end)
	end
end

-----------------------------
-- PER-BRAINROT STATE
-----------------------------
-- BrainrotState[clone] = {
--   template, area, position,
--   mainPart,
--   clicksRequired,
--   clicks,
--   dead,
--   lastClickTime,
--   lastHealthBroadcastTime,
--   playersTracking = { [player] = true },
--   uiShown = { [player] = true },
--   respawnTime = number (seconds, from Configuration.RespawnTime or fallback)
-- }
local BrainrotState: {[Model]: {
	template: Model,
	area: Instance,
	position: Vector3,
	mainPart: BasePart?,
	clicksRequired: number,
	clicks: number,
	dead: boolean,
	lastClickTime: number,
	lastHealthBroadcastTime: number,
	playersTracking: {[Player]: boolean},
	uiShown: {[Player]: boolean},
	respawnTime: number,
}} = {}

-- Throttling for shake
-- LastShakeTimes[player.UserId] = lastShakeTime
local LastShakeTimes: {[number]: number} = {}

-----------------------------
-- DEATH HANDLER
-----------------------------
local function spawnBrainrotInArea(template: Model, area: Instance) end -- forward declaration

local function handleBrainrotDeath(clone: Model, killer: Player, upgradesFolder, gemChancePct: number)
	local state = BrainrotState[clone]
	if not state or state.dead then return end
	state.dead = true

	local template    = state.template
	local area        = state.area
	local position    = state.position
	local mainPart    = state.mainPart
	local deathPos    = mainPart and mainPart.Position or (position or Vector3.new())
	local respawnTime = state.respawnTime or RespawnTime

	-- Copy & clear tracking table
	local trackedPlayers = state.playersTracking
	state.playersTracking = {}

	-- Destroy HP UI + highlight for everyone that had it
	local uiShown = state.uiShown or {}
	state.uiShown = {}

	for plr,_ in pairs(trackedPlayers) do
		displayHealthEvent:FireClient(plr, "Destroy", clone)
		displayHealthEvent:FireClient(plr, "HighlightOff", clone)
	end

	-- Remove from activeBrainrots for spacing logic
	for i = #activeBrainrots, 1, -1 do
		local entry = activeBrainrots[i]
		if entry.Template == template
			and entry.Area == area
			and (entry.Position - position).Magnitude < 0.001
		then
			table.remove(activeBrainrots, i)
			break
		end
	end

	-- inventory reward
	giveBrainrot(killer, template)

	-- GEM DROP
	local gemRollChance = math.clamp(gemChancePct or GEM_DROP_CHANCE_PERCENT, 0, 100)
	if math.random(1, 100) <= gemRollChance then
		local amount = weightedGemRoll(GEM_WEIGHTS)

		if upgradesFolder then
			local gy = computeGemYieldEff(upgradesFolder)
			amount = math.max(1, math.floor(amount * gy + 0.0001))
		end

		addGemsSilent(killer, amount)

		if BONUS_GEM_CHANCE > 0 and math.random(1, BONUS_GEM_CHANCE) == 1 then
			addGemsSilent(killer, amount)
		end
	end

	-- Camera shake + focus on death
	if screenShakeEvent then
		screenShakeEvent:FireClient(killer, "Death", clone, deathPos)
	end
	if focusCollectEvent then
		focusCollectEvent:FireClient(killer, clone, deathPos, 1.0)
	end

	-- Move clone out of Workspace (pooled) and play orb
	releaseBrainrot(template, clone)
	if deathPos then
		playCollectOrb(deathPos, killer)
	end

	-- Respawn later using per-brainrot respawn time
	if respawnTime and respawnTime > 0 then
		task.delay(respawnTime, function()
			if area and area.Parent and template and template.Parent then
				spawnBrainrotInArea(template, area)
			end
		end)
	end
end

-----------------------------
-- CLICK HANDLER
-----------------------------
local function onBrainrotClicked(clone: Model, player: Player)
	local state = BrainrotState[clone]
	if not state or state.dead then return end

	local now = time()
	if (now - state.lastClickTime) < ClickCooldown then return end
	state.lastClickTime = now

	local mainPart = state.mainPart
	if not mainPart or not mainPart.Parent then return end

	local upgrades = player:FindFirstChild("Upgrades")

	local totalDamage      = 1
	local critChancePct    = CRIT_CHANCE_PERCENT
	local gemChancePct     = GEM_DROP_CHANCE_PERCENT
	local critMultEff      = CRIT_MULTIPLIER

	if upgrades then
		totalDamage, critChancePct, gemChancePct, critMultEff = computeEffectiveCombat(upgrades)
	end

	local critRoll = math.random(1, 100)
	local isCrit   = (critRoll <= math.clamp(critChancePct, 0, 100))
	local delta    = totalDamage * (isCrit and critMultEff or 1)

	state.clicks += delta

	-- CLICK SFX
	playRandomClick(mainPart)
	if isCrit then
		playCrit(mainPart)
	end

	-- NETWORK-THROTTLED SHAKE
	local uid = player.UserId
	local lastShake = LastShakeTimes[uid] or 0
	if (now - lastShake) >= MIN_SHAKE_INTERVAL then
		LastShakeTimes[uid] = now

		if screenShakeEvent then
			screenShakeEvent:FireClient(player, "Click", clone, mainPart.Position)
		end
		shakeEvent:FireClient(player, isCrit and 1 or 0, 0)
	end

	if isCrit then
		critEvent:FireClient(player, clone, delta, mainPart.Position)
	end

	-- Track this player for HP UI
	local playersTracking = state.playersTracking
	if not playersTracking[player] then
		playersTracking[player] = true
	end

	-- HP UI + WHITE HIGHLIGHT ONLY FOR THIS PLAYER
	local uiShown = state.uiShown
	if not uiShown then
		uiShown = {}
		state.uiShown = uiShown
	end

	if not uiShown[player] then
		uiShown[player] = true
		displayHealthEvent:FireClient(player, "Create", clone, state.clicksRequired, state.clicks)
	else
		displayHealthEvent:FireClient(player, "Update", clone, state.clicksRequired, state.clicks)
	end

	-- Per-player highlight while attacking
	displayHealthEvent:FireClient(player, "HighlightOn", clone)

	if (not state.dead) and state.clicks >= state.clicksRequired then
		handleBrainrotDeath(clone, player, upgrades, gemChancePct)
	end
end

-----------------------------
-- SPAWN LOGIC (USING POOL + STATE)
-----------------------------
function spawnBrainrotInArea(template: Model, area: Instance)
	local position = getRandomSpawnPosition(template, area)
	table.insert(activeBrainrots, {Template = template, Area = area, Position = position})

	local clone = getPooledBrainrot(template)
	clone.Parent = Workspace

	local mainPart = ensurePrimaryPart(clone) or clone:FindFirstChildWhichIsA("BasePart")
	if not mainPart then
		warn(("Template %s has no BasePart to anchor"):format(template.Name))
		releaseBrainrot(template, clone)
		return
	end

	safePivot(clone, CFrame.new(position))
	hardFreezeModel(clone)

	mainPart.Transparency = 1
	mainPart.CanCollide   = false

	if not mainPart:FindFirstChild("RespawnSound") then
		local sound = Instance.new("Sound")
		sound.Name   = "RespawnSound"
		sound.SoundId= CollectionSoundId
		sound.Volume = 0.5
		sound.Parent = mainPart
	end

	ensureClickAndCritSounds(mainPart)

	local config              = template:FindFirstChild("Configuration")
	local clicksRequiredValue = config and config:FindFirstChild("ClicksRequired")
	local clicksRequired      = (clicksRequiredValue and clicksRequiredValue.Value) or 10

	-- NEW: per-brainrot RespawnTime read + clamped
	local respawnTime = RespawnTime
	if config then
		local rt = config:FindFirstChild("RespawnTime")
		if rt and (rt:IsA("NumberValue") or rt:IsA("IntValue")) then
			local val = tonumber(rt.Value)
			if val and val > 0 then
				respawnTime = val
			end
		end
	end

	-- STATE INIT / RESET
	local state = BrainrotState[clone]
	if not state then
		state = {
			template                = template,
			area                    = area,
			position                = position,
			mainPart                = mainPart,
			clicksRequired          = clicksRequired,
			clicks                  = 0,
			dead                    = false,
			lastClickTime           = 0,
			lastHealthBroadcastTime = 0,
			playersTracking         = {},
			uiShown                 = {},
			respawnTime             = respawnTime,
		}
		BrainrotState[clone] = state
	else
		state.template                = template
		state.area                    = area
		state.position                = position
		state.mainPart                = mainPart
		state.clicksRequired          = clicksRequired
		state.clicks                  = 0
		state.dead                    = false
		state.lastClickTime           = 0
		state.lastHealthBroadcastTime = 0
		state.playersTracking         = {}
		state.uiShown                 = {}
		state.respawnTime             = respawnTime
	end

	local clickDetector = mainPart:FindFirstChildWhichIsA("ClickDetector")
	if not clickDetector then
		clickDetector = Instance.new("ClickDetector")
		clickDetector.Parent = mainPart
	end
	clickDetector.MaxActivationDistance = 32

	-- Attach MouseClick ONCE per clone.
	if not clone:GetAttribute("ClickBound") then
		clone:SetAttribute("ClickBound", true)
		clickDetector.MouseClick:Connect(function(player)
			onBrainrotClicked(clone, player)
		end)
	end
end

-----------------------------
-- GLOBAL IDLE RESET + HP UI LOOP
-----------------------------
task.spawn(function()
	while true do
		local now = time()

		for clone, state in pairs(BrainrotState) do
			if clone.Parent == Workspace and not state.dead then
				-- IDLE RESET
				if state.clicks > 0 and (now - state.lastClickTime) >= IdleResetTime then
					state.clicks = 0

					local trackedPlayers = state.playersTracking
					state.playersTracking = {}

					local uiShown = state.uiShown or {}
					state.uiShown = {}

					for plr,_ in pairs(trackedPlayers) do
						-- Remove HP UI + highlight when idle timeout hits
						displayHealthEvent:FireClient(plr, "Destroy", clone)
						displayHealthEvent:FireClient(plr, "HighlightOff", clone)
					end
				else
					-- HP UI BROADCAST (THROTTLED) – keep all attackers in sync
					if state.clicks > 0 and (now - (state.lastHealthBroadcastTime or 0)) >= HEALTH_BROADCAST_INTERVAL then
						state.lastHealthBroadcastTime = now

						local uiShown = state.uiShown
						if not uiShown then
							uiShown = {}
							state.uiShown = uiShown
						end

						for plr,_ in pairs(state.playersTracking) do
							if not uiShown[plr] then
								uiShown[plr] = true
								displayHealthEvent:FireClient(plr, "Create", clone, state.clicksRequired, state.clicks)
							else
								displayHealthEvent:FireClient(plr, "Update", clone, state.clicksRequired, state.clicks)
							end
						end
					end
				end
			end
		end

		task.wait(0.1)
	end
end)

-----------------------------
-- INITIALIZE
-----------------------------
local spawnTemplates = ServerStorage:FindFirstChild("SpawnTemplates")
if not spawnTemplates then
	warn("SpawnTemplates folder not found in ServerStorage")
else
	for _, brainrot in ipairs(spawnTemplates:GetChildren()) do
		if brainrot:IsA("Model") then
			local areaNameValue = brainrot:FindFirstChild("AreaName")
			if areaNameValue then
				for _, area in ipairs(areasFolder:GetChildren()) do
					if area.Name == areaNameValue.Value then
						-- Initial spawn is immediate; respawn delay is fully handled by Configuration.RespawnTime.
						spawnBrainrotInArea(brainrot, area)
					end
				end
			end
		end
	end
end

-----------------------------
-- OPTIONAL WATCHDOG (low frequency)
-----------------------------
local debrisAccum = 0
RunService.Stepped:Connect(function(_, step)
	debrisAccum += step
	if debrisAccum < 1 then return end
	debrisAccum = 0

	for _, p in ipairs(Workspace:GetChildren()) do
		if p:IsA("BasePart") and p.Name == "__LooseBrainrotDebris" then
			Debris:AddItem(p, 5)
		end
	end
end)
