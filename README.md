# Sora-Hub
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer
local playerGui = LocalPlayer:WaitForChild("PlayerGui")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local player = Players.LocalPlayer

local NORMAL_GRAV = 196.2
local REDUCED_GRAV = 165
local NORMAL_JUMP = 50
local BOOST_JUMP = 75

local spoofedGravity = NORMAL_GRAV
pcall(function()
    local mt = getrawmetatable(Workspace)
    if mt then
        setreadonly(mt, false)
        local oldIndex = mt.__index
        mt.__index = function(self, k)
            if k == 'Gravity' then
                return spoofedGravity
            end
            return oldIndex(self, k)
        end
        setreadonly(mt, true)
    end
end)

local gravityLow = false
local sourceActive = false

local function setJumpPower(jump)
    local h = player.Character and player.Character:FindFirstChildOfClass('Humanoid')
    if h then
        h.JumpPower = jump
        h.UseJumpPower = true
    end
end

local speedBoostConn
local function enableSpeedBoostAssembly(state)
    if speedBoostConn then
        speedBoostConn:Disconnect()
        speedBoostConn = nil
    end
    if state then
        speedBoostConn = RunService.Heartbeat:Connect(function()
            local char = player.Character
            if char then
                local root = char:FindFirstChild('HumanoidRootPart')
                local h = char:FindFirstChildOfClass('Humanoid')
                if root and h and h.MoveDirection.Magnitude > 0 then
                    root.Velocity = Vector3.new(
                        h.MoveDirection.X * BOOST_SPEED,
                        root.Velocity.Y,
                        h.MoveDirection.Z * BOOST_SPEED
                    )
                end
            end
        end)
    end
end

local infiniteJumpConn
local function enableInfiniteJump(state)
    if infiniteJumpConn then
        infiniteJumpConn:Disconnect()
        infiniteJumpConn = nil
    end
    if state then
        infiniteJumpConn = UserInputService.JumpRequest:Connect(function()
            local h = player.Character and player.Character:FindFirstChildOfClass('Humanoid')
            if h and gravityLow and h:GetState() ~= Enum.HumanoidStateType.Seated then
                local root = player.Character:FindFirstChild('HumanoidRootPart')
                if root then
                    root.Velocity = Vector3.new(
                        root.Velocity.X,
                        h.JumpPower,
                        root.Velocity.Z
                    )
                end
            end
        end)
    end
end

local function antiRagdoll()
    local char = player.Character
    if char then
        for _, v in pairs(char:GetDescendants()) do
            if v:IsA('BodyVelocity') or v:IsA('BodyAngularVelocity') then
                v:Destroy()
            end
        end
    end
end

local function toggleForceField()
    local char = player.Character
    if char then
        if gravityLow then
            if not char:FindFirstChildOfClass('ForceField') then
                local ff = Instance.new('ForceField', char)
                ff.Visible = false
            end
        else
            for _, ff in ipairs(char:GetChildren()) do
                if ff:IsA('ForceField') then
                    ff:Destroy()
                end
            end
        end
    end
end

local function switchGravityJump()
    gravityLow = not gravityLow
    sourceActive = gravityLow
    Workspace.Gravity = gravityLow and REDUCED_GRAV or NORMAL_GRAV
    setJumpPower(gravityLow and BOOST_JUMP or NORMAL_JUMP)
    enableSpeedBoostAssembly(gravityLow)
    enableInfiniteJump(gravityLow)
    antiRagdoll()
    toggleForceField()
    spoofedGravity = NORMAL_GRAV
end

local flyActive = false
local flyConn = nil
local flyCharRemovingConn = nil
local destTouchedConn = nil
local flyLV = nil
local flyAtt = nil
local destPartRef = nil

local FLY_GRAV = 20
local FLY_JUMP = 7
local FLY_STOPDIST = 7
local FLY_XZ_SPEED = 22
local FLY_Y_BASE = -1.0
local FLY_Y_MAX = -2.2
local FLY_TIME_STEP = 1.5

local function setFlyButtonActive(state)
end

local function clearFlyConnections()
    if flyConn then
        flyConn:Disconnect()
        flyConn = nil
    end
    if flyCharRemovingConn then
        flyCharRemovingConn:Disconnect()
        flyCharRemovingConn = nil
    end
    if destTouchedConn then
        destTouchedConn:Disconnect()
        destTouchedConn = nil
    end
end

local function destroyFlyBodies()
    if flyLV then
        flyLV:Destroy()
        flyLV = nil
    end
    if flyAtt then
        flyAtt:Destroy()
        flyAtt = nil
    end
    destPartRef = nil
end

local function findMyDeliveryPart()
    local plots = Workspace:FindFirstChild('Plots')
    if plots then
        for _, plot in ipairs(plots:GetChildren()) do
            local sign = plot:FindFirstChild('PlotSign')
            if sign and sign:FindFirstChild('YourBase') and sign.YourBase.Enabled then
                local delivery = plot:FindFirstChild('DeliveryHitbox')
                if delivery and delivery:IsA('BasePart') then
                    return delivery
                end
            end
        end
    end
    return nil
end

local function flyGetDescent(dist)
    local maxdist = 200
    dist = math.clamp(dist, 0, maxdist)
    local t = 1 - (dist / maxdist)
    return FLY_Y_BASE + (FLY_Y_MAX - FLY_Y_BASE) * t
end

local function restoreSourceAndPhysics()
    local char = player.Character
    local hum = char and char:FindFirstChildOfClass('Humanoid')
    if hum then
        hum.JumpPower = 50
    end
end

local function cleanupFly()
    clearFlyConnections()
    destroyFlyBodies()
    restoreSourceAndPhysics()
    setFlyButtonActive(false)
    flyActive = false
end

local function startFlyToBase()
    if flyActive then
        cleanupFly()
        return
    end

    local destPart = findMyDeliveryPart()
    if not destPart then
        return
    end

    local char = player.Character
    local hum = char and char:FindFirstChildOfClass('Humanoid')
    local hrp = char and char:FindFirstChild('HumanoidRootPart')
    if not (hum and hrp) then
        return
    end

    hum.UseJumpPower = true
    hum.JumpPower = FLY_JUMP

    flyActive = true
    setFlyButtonActive(true)

    flyAtt = Instance.new('Attachment')
    flyAtt.Name = 'FlyToBaseAttachment'
    flyAtt.Parent = hrp

    flyLV = Instance.new('LinearVelocity')
    flyLV.Attachment0 = flyAtt
    flyLV.RelativeTo = Enum.ActuatorRelativeTo.World
    flyLV.MaxForce = 99999
    flyLV.Parent = hrp

    destPartRef = destPart

    local reached = false
    local lastYUpdate = 0

    local pos = hrp.Position
    local destPos = destPart.Position
    local distXZ = (Vector3.new(destPos.X, pos.Y, destPos.Z) - pos).Magnitude
    local yVel = flyGetDescent(distXZ)
    local dirXZ = Vector3.new(destPos.X - pos.X, 0, destPos.Z - pos.Z)
    if dirXZ.Magnitude > 0 then
        dirXZ = dirXZ.Unit
    else
        dirXZ = Vector3.new()
    end
    flyLV.VectorVelocity = Vector3.new(dirXZ.X * FLY_XZ_SPEED, yVel, dirXZ.Z * FLY_XZ_SPEED)
    lastYUpdate = tick()

    destTouchedConn = destPart.Touched:Connect(function(hit)
        if not flyActive then return end
        local ch = player.Character
        if ch and hit and hit:IsDescendantOf(ch) then
            reached = true
            cleanupFly()
        end
    end)

    flyConn = RunService.Heartbeat:Connect(function()
        if not flyActive then
            cleanupFly()
            return
        end

        if not (hrp and hrp.Parent and hum and hum.Parent) then
            cleanupFly()
            return
        end

        local pos = hrp.Position
        local destPos = destPart.Position
        local distXZ = (Vector3.new(destPos.X, pos.Y, destPos.Z) - pos).Magnitude

        if distXZ < FLY_STOPDIST and not reached then
            reached = true
            cleanupFly()
            return
        end

        local currentTime = tick()
        if currentTime - lastYUpdate >= FLY_TIME_STEP then
            local yVel = flyGetDescent(distXZ)
            local dirXZ = Vector3.new(destPos.X - pos.X, 0, destPos.Z - pos.Z)
            if dirXZ.Magnitude > 0 then
                dirXZ = dirXZ.Unit
            else
                dirXZ = Vector3.new()
            end
            flyLV.VectorVelocity = Vector3.new(
                dirXZ.X * FLY_XZ_SPEED,
                yVel,
                dirXZ.Z * FLY_XZ_SPEED
            )
            lastYUpdate = currentTime
        end
    end)

    flyCharRemovingConn = player.CharacterRemoving:Connect(function()
        cleanupFly()
    end)
end

local function DesyncV3()
    local flags = {
        {"GameNetPVHeaderRotationalVelocityZeroCutoffExponent", "-8000"},
        {"LargeReplicatorWrite5", "true"},
        {"LargeReplicatorEnabled9", "true"},
        {"AngularVelociryLimit", "360"},
        {"TimestepArbiterVelocityCriteriaThresholdTwoDt", "2147483646"},
        {"S2PhysicsSenderRate", "15000"},
        {"DisableDPIScale", "true"},
        {"MaxDataPacketPerSend", "2147483647"},
        {"ServerMaxBandwith", "52"},
        {"PhysicsSenderMaxBandwidthBps", "20000"},
        {"MaxTimestepMultiplierBuoyancy", "2147483647"},
        {"SimOwnedNOUCountThresholdMillionth", "2147483647"},
        {"MaxMissedWorldStepsRemembered", "-2147483648"},
        {"CheckPVDifferencesForInterpolationMinVelThresholdStudsPerSecHundredth", "1"},
        {"StreamJobNOUVolumeLengthCap", "2147483647"},
        {"DebugSendDistInSteps", "-2147483648"},
        {"MaxTimestepMultiplierAcceleration", "2147483647"},
        {"LargeReplicatorRead5", "true"},
        {"SimExplicitlyCappedTimestepMultiplier", "2147483646"},
        {"GameNetDontSendRedundantNumTimes", "1"},
        {"CheckPVLinearVelocityIntegrateVsDeltaPositionThresholdPercent", "1"},
        {"CheckPVCachedRotVelThresholdPercent", "10"},
        {"LargeReplicatorSerializeRead3", "true"},
        {"ReplicationFocusNouExtentsSizeCutoffForPauseStuds", "2147483647"},
        {"NextGenReplicatorEnabledWrite4", "true"},
        {"CheckPVDifferencesForInterpolationMinRotVelThresholdRadsPerSecHundredth", "1"},
        {"GameNetDontSendRedundantDeltaPositionMillionth", "1"},
        {"InterpolationFrameVelocityThresholdMillionth", "5"},
        {"StreamJobNOUVolumeCap", "2147483647"},
        {"InterpolationFrameRotVelocityThresholdMillionth", "5"},
        {"WorldStepMax", "90"},
        {"TimestepArbiterHumanoidLinearVelThreshold", "1"},
        {"InterpolationFramePositionThresholdMillionth", "5"},
        {"TimestepArbiterHumanoidTurningVelThreshold", "1"},
        {"MaxTimestepMultiplierContstraint", "2147483647"},
        {"GameNetPVHeaderLinearVelocityZeroCutoffExponent", "-5000"},
        {"CheckPVCachedVelThresholdPercent", "10"},
        {"TimestepArbiterOmegaThou", "1073741823"},
        {"MaxAcceptableUpdateDelay", "1"},
        {"LargeReplicatorSerializeWrite4", "true"},
    }

    for _, data in ipairs(flags) do
        pcall(function()
            setfflag(data[1], data[2])
        end)
    end

    local char = LocalPlayer.Character
    if not char then return end

    local humanoid = char:FindFirstChildWhichIsA("Humanoid")
    if humanoid then
        humanoid:ChangeState(Enum.HumanoidStateType.Dead)
    end

    char:ClearAllChildren()

    local fakeModel = Instance.new("Model", workspace)
    LocalPlayer.Character = fakeModel
    task.wait()
    LocalPlayer.Character = char
    fakeModel:Destroy()
end

local function DesyncV3Start()
    DesyncV3()
end

local SCALE = 0.85
local PANEL_WIDTH, PANEL_HEIGHT = math.floor(160*SCALE), math.floor(229*SCALE)
local PANEL_RADIUS = math.floor(13*SCALE)
local TITLE_HEIGHT = math.floor(20*SCALE)
local BTN_WIDTH = math.floor(0.880*PANEL_WIDTH)
local BTN_HEIGHT_DESYNC = math.floor(42*SCALE)
local BTN_HEIGHT_NORMAL = math.floor(32*SCALE)
local BTN_RADIUS = math.floor(9*SCALE)
local BTN_FONT_SIZE = math.floor(15*SCALE)
local TITLE_FONT_SIZE = math.floor(15*SCALE)
local ICON_SIZE = math.floor(14*SCALE)
local BTN_ICON_PAD = math.floor(6*SCALE)
local BTN_Y0 = math.floor(25*SCALE)
local BTN_SPACING = math.floor(8*SCALE)

local old = playerGui:FindFirstChild("Lisandapanel")
if old then old:Destroy() end

local gui = Instance.new("ScreenGui")
gui.Name = "Lisandapanel"
gui.ResetOnSpawn = false
gui.Parent = playerGui

local main = Instance.new("Frame", gui)
main.Name = "MainPanel"
main.Size = UDim2.new(0, PANEL_WIDTH, 0, PANEL_HEIGHT)
main.Position = UDim2.new(1, -PANEL_WIDTH-10, 0, 10)
main.BackgroundColor3 = Color3.fromRGB(30, 25, 30)
main.BackgroundTransparency = 0.1
main.BorderSizePixel = 0
main.Active = true
Instance.new("UICorner", main).CornerRadius = UDim.new(0, PANEL_RADIUS)

do
    local dragging, dragInput, dragStart, startPos
    main.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = main.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    main.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input == dragInput then
            local delta = input.Position - dragStart
            main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

local title = Instance.new("TextLabel", main)
title.Name = "Title"
title.Size = UDim2.new(1, 0, 0, TITLE_HEIGHT)
title.Position = UDim2.new(0,0,0,-0.6)
title.Text = "ds.gg/A9E8Apby6"
title.Font = Enum.Font.Arcade
title.TextSize = TITLE_FONT_SIZE
title.BackgroundTransparency = 1
title.TextColor3 = Color3.fromRGB(255, 255, 255)

local function createCircleIcon(parent, y, btnHeight, on, isDesync)
    local icon = Instance.new("Frame", parent)
    icon.Size = UDim2.new(0, ICON_SIZE, 0, ICON_SIZE)
    icon.Position = UDim2.new(0, BTN_ICON_PAD, 0, y + math.floor((btnHeight-ICON_SIZE)/2))
    icon.BackgroundTransparency = 1
    local circle = Instance.new("ImageLabel", icon)
    circle.Size = UDim2.new(1, 0, 1, 0)
    circle.Position = UDim2.new(0,0,0,0)
    circle.BackgroundTransparency = 1
    circle.Image = "rbxassetid://10137946418"
    
    if isDesync then
        circle.ImageColor3 = (on and Color3.fromRGB(200, 200, 0)) or Color3.fromRGB(150, 60, 60)
    else
        circle.ImageColor3 = (on and Color3.fromRGB(120, 200, 120)) or Color3.fromRGB(150, 60, 60)
    end
    
    return icon, circle
end

local function makeToggleBtn(parent, label, y, btnHeight, callback, isDesync)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(0, BTN_WIDTH, 0, btnHeight)
    btn.Position = UDim2.new(0, math.floor((PANEL_WIDTH-BTN_WIDTH)/2), 0, y)
    
    if isDesync then
        btn.BackgroundColor3 = Color3.fromRGB(150, 60, 60)
    else
        btn.BackgroundColor3 = Color3.fromRGB(60, 0, 0)
    end
    
    btn.Text = label
    btn.Font = Enum.Font.Arcade
    btn.TextSize = BTN_FONT_SIZE
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, BTN_RADIUS)
    btn.TextXAlignment = Enum.TextXAlignment.Center
    local icon, circle = createCircleIcon(parent, y, btnHeight, false, isDesync)
    icon.ZIndex = btn.ZIndex+1
    btn.ZIndex = btn.ZIndex+2
    btn.LayoutOrder = y
    btn.AutoButtonColor = false
    local state = false
    local function updateVisual()
        if isDesync then
            circle.ImageColor3 = (state and Color3.fromRGB(200, 200, 0)) or Color3.fromRGB(150, 60, 60)
            btn.BackgroundColor3 = state and Color3.fromRGB(200, 200, 0) or Color3.fromRGB(60, 0, 0)
        else
            circle.ImageColor3 = (state and Color3.fromRGB(120, 200, 120)) or Color3.fromRGB(150, 60, 60)
            btn.BackgroundColor3 = state and Color3.fromRGB(40, 160, 40) or Color3.fromRGB(60, 0, 0)
        end
    end
    btn.MouseButton1Click:Connect(function()
        state = not state
        callback(state, btn)
        updateVisual()
    end)
    icon.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            state = not state
            callback(state, btn)
            updateVisual()
        end
    end)
    updateVisual()
    return btn, function(v)
        state = v
        callback(state, btn)
        updateVisual()
    end
end

local btnDesyncBody, setDesyncBodyState = makeToggleBtn(main, "NOSYNC (Beta)", BTN_Y0, BTN_HEIGHT_DESYNC, function(on, btn)
    if on then
        task.spawn(function()
            DesyncV3Start()
        end)
    end
end, true)

local btnInfJump, setInfJumpState = makeToggleBtn(main, "INF JUMP", BTN_Y0 + BTN_HEIGHT_DESYNC + BTN_SPACING, BTN_HEIGHT_NORMAL, function(on, btn)
    switchGravityJump()
end, false)

local btnFlyToBase, setFlyToBaseState = makeToggleBtn(main, "FLY TO BASE", BTN_Y0 + BTN_HEIGHT_DESYNC + BTN_HEIGHT_NORMAL + BTN_SPACING * 2, BTN_HEIGHT_NORMAL, function(on, btn)
    if on then
        startFlyToBase()
    else
        cleanupFly()
    end
end, false)

local btnGoToBest, setGoToBestState = makeToggleBtn(main, "GO TO BEST", BTN_Y0 + BTN_HEIGHT_DESYNC + BTN_HEIGHT_NORMAL * 2 + BTN_SPACING * 3, BTN_HEIGHT_NORMAL, function(on, btn)
end, false)

local btnThirdFloor, setThirdFloorState = makeToggleBtn(main, "STEAL FLOOR", BTN_Y0 + BTN_HEIGHT_DESYNC + BTN_HEIGHT_NORMAL * 3 + BTN_SPACING * 4, BTN_HEIGHT_NORMAL, function(on, btn)
end, false)

LocalPlayer.CharacterAdded:Connect(function()
    setInfJumpState(false)
    setFlyToBaseState(false)
    setGoToBestState(false)
    setThirdFloorState(false)
    if sourceActive then
        switchGravityJump()
    end
end)
