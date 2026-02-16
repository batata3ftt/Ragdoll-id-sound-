-- ===== RAGDOLL CUSTOM SOUND =====
-- Cole seu ID de audio abaixo:
local ragdollSoundId = "SEU_ID_AQUI"

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local player = Players.LocalPlayer

local soundCooldown = {}
local SOUND_COOLDOWN = 2.5
local lastTackleHit = 0
local TACKLE_WINDOW = 1.5

local function playSound(soundId)
    if not soundId or soundId == "" then return end
    local id = tostring(soundId):match("%d+")
    if not id then return end
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://" .. id
    sound.Volume = 2
    sound.Parent = Workspace
    sound:Play()
    sound.Ended:Connect(function() sound:Destroy() end)
end

pcall(function()
    local TackleHitRE = ReplicatedStorage:WaitForChild("TackleHit_RE", 10)
    if TackleHitRE then
        TackleHitRE.OnClientEvent:Connect(function()
            if not player.Character then return end
            if not player.Character:FindFirstChild("HumanoidRootPart") then return end
            lastTackleHit = tick()
        end)
    end
end)

pcall(function()
    local TackleRE = ReplicatedStorage:WaitForChild("Tackle_RE", 10)
    if TackleRE then
        TackleRE.OnClientEvent:Connect(function()
            lastTackleHit = tick()
        end)
    end
end)

local function isRecentTackle()
    return (tick() - lastTackleHit) <= TACKLE_WINDOW
end

local function canPlay(p)
    if not p then return false end
    local t = soundCooldown[p.UserId]
    if not t then return true end
    return (tick() - t) >= SOUND_COOLDOWN
end

local function monitorChar(p, character)
    if p == player then return end
    local humanoid = character:WaitForChild("Humanoid")

    local function tryPlay()
        if not isRecentTackle() then return end
        if not canPlay(p) then return end
        soundCooldown[p.UserId] = tick()
        playSound(ragdollSoundId)
        task.delay(SOUND_COOLDOWN, function()
            soundCooldown[p.UserId] = nil
        end)
    end

    humanoid.StateChanged:Connect(function(oldState, newState)
        if (newState == Enum.HumanoidStateType.Ragdoll
            or newState == Enum.HumanoidStateType.FallingDown)
        and oldState ~= Enum.HumanoidStateType.Ragdoll
        and oldState ~= Enum.HumanoidStateType.FallingDown then
            tryPlay()
        end
    end)

    local function checkBool(obj)
        if obj:IsA("BoolValue") and obj.Name:lower():find("ragdoll") then
            local lastVal = obj.Value
            obj:GetPropertyChangedSignal("Value"):Connect(function()
                if obj.Value == true and lastVal == false then tryPlay() end
                lastVal = obj.Value
            end)
        end
    end

    for _, obj in pairs(character:GetDescendants()) do checkBool(obj) end
    character.DescendantAdded:Connect(function(obj)
        checkBool(obj)
        if obj:IsA("BallSocketConstraint") then tryPlay() end
    end)

    pcall(function()
        local lastAttr = false
        character:GetAttributeChangedSignal("Ragdoll"):Connect(function()
            local val = character:GetAttribute("Ragdoll")
            if val == true and lastAttr == false then tryPlay() end
            lastAttr = val
        end)
    end)

    local wasRagdoll = false
    local hbConn
    hbConn = RunService.Heartbeat:Connect(function()
        if not character.Parent then
            hbConn:Disconnect()
            return
        end
        local state = humanoid:GetState()
        local isNow = state == Enum.HumanoidStateType.Ragdoll
            or state == Enum.HumanoidStateType.FallingDown
        if isNow and not wasRagdoll then
            wasRagdoll = true
            tryPlay()
        elseif not isNow then
            wasRagdoll = false
        end
    end)
end

for _, p in pairs(Players:GetPlayers()) do
    if p ~= player then
        if p.Character then monitorChar(p, p.Character) end
        p.CharacterAdded:Connect(function(char)
            task.wait(0.5)
            monitorChar(p, char)
        end)
    end
end

Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function(char)
        task.wait(0.5)
        monitorChar(p, char)
    end)
end)

print("Ragdoll Custom Sound carregado! ID: " .. ragdollSoundId)
