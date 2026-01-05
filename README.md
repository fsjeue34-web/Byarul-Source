

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local StarterGui = game:GetService("StarterGui")
local player = Players.LocalPlayer
wait(1)

-- ========= SAFE FUNCTION WRAPPER =========
local function SafeCall(func, ...)
    local success, err = pcall(func, ...)
    if not success then
        warn("⚠️ Error at:", debug.traceback(), "\nDetails:", err)
    end
    return success
end

-- ========= FILE SYSTEM PROTECTION =========
local hasFileSystem = (writefile ~= nil and readfile ~= nil and isfile ~= nil)

if not hasFileSystem then
    warn("⚠️ File system tidak tersedia. Script akan berjalan tanpa fitur Save/Load.")
    writefile = function() end
    readfile = function() return "" end
    isfile = function() return false end
end

-- ========= OPTIMIZED CONFIGURATION =========
local RECORDING_FPS = 60
local MAX_FRAMES = 30000
local MIN_DISTANCE_THRESHOLD = 0.008
local VELOCITY_SCALE = 1
local VELOCITY_Y_SCALE = 1
local TIMELINE_STEP_SECONDS = 0.04
local JUMP_VELOCITY_THRESHOLD = 10
local STATE_CHANGE_COOLDOWN = 0.08
local TRANSITION_FRAMES = 6
local RESUME_DISTANCE_THRESHOLD = 40
local PLAYBACK_FIXED_TIMESTEP = 1 / 60
local LOOP_TRANSITION_DELAY = 0.08
local AUTO_LOOP_RETRY_DELAY = 0.3
local TIME_BYPASS_THRESHOLD = 0.05
local LAG_DETECTION_THRESHOLD = 0.15
local MAX_LAG_FRAMES_TO_SKIP = 3
local INTERPOLATE_AFTER_LAG = true
local ENABLE_FRAME_SMOOTHING = false
local SMOOTHING_WINDOW = 3
local USE_VELOCITY_PLAYBACK = false
local INTERPOLATION_LOOKAHEAD = 2

-- ========= FIELD MAPPING FOR OBFUSCATION =========
local FIELD_MAPPING = {
    Position = "11",
    LookVector = "88", 
    UpVector = "55",
    Velocity = "22",
    MoveState = "33",
    WalkSpeed = "44",
    Timestamp = "66"
}

local REVERSE_MAPPING = {
    ["11"] = "Position",
    ["88"] = "LookVector",
    ["55"] = "UpVector", 
    ["22"] = "Velocity",
    ["33"] = "MoveState",
    ["44"] = "WalkSpeed",
    ["66"] = "Timestamp"
}

-- ========= PRE-DECLARE UI REFERENCES =========
local ScreenGui, MainFrame, PlaybackControl, RecordingStudio, MiniButton
local PlayBtnControl, LoopBtnControl, JumpBtnControl, RespawnBtnControl
local ShiftLockBtnControl, ResetBtnControl, ShowRuteBtnControl
local StartBtn, SaveBtn, ResumeBtn, PrevBtn, NextBtn
local SpeedBox, FilenameBox, WalkSpeedBox, RecordingsList
local Title, CheckAllBtn

-- ========= VARIABLES =========
local IsRecording = false
local IsPlaying = false
local IsPaused = false
local IsReversing = false
local IsForwarding = false
local IsTimelineMode = false
local CurrentSpeed = 1.0
local CurrentWalkSpeed = 16
local RecordedMovements = {}
local RecordingOrder = {}
local CurrentRecording = {Frames = {}, StartTime = 0, Name = ""}
local AutoRespawn = false
local InfiniteJump = false
local AutoLoop = false
local recordConnection = nil
local playbackConnection = nil
local loopConnection = nil
local jumpConnection = nil
local reverseConnection = nil
local forwardConnection = nil
local lastRecordTime = 0
local lastRecordPos = nil
local checkpointNames = {}
local PathVisualization = {}
local ShowPaths = false
local PathAutoHide = true
local playbackStartTime = 0
local totalPausedDuration = 0
local pauseStartTime = 0
local currentPlaybackFrame = 1
local prePauseHumanoidState = nil
local prePauseWalkSpeed = 16
local prePauseAutoRotate = true
local prePauseJumpPower = 50
local prePausePlatformStand = false
local prePauseSit = false
local lastPlaybackState = nil
local lastStateChangeTime = 0
local IsAutoLoopPlaying = false
local LastKnownWalkSpeed = 16  
local WalkSpeedBeforePlayback = 16
local CurrentLoopIndex = 1
local LoopPauseStartTime = 0
local LoopTotalPausedDuration = 0
local shiftLockConnection = nil
local originalMouseBehavior = nil
local ShiftLockEnabled = false
local isShiftLockActive = false
local StudioIsRecording = false
local StudioCurrentRecording = {Frames = {}, StartTime = 0, Name = ""}
local lastStudioRecordTime = 0
local lastStudioRecordPos = nil
local activeConnections = {}
local CheckedRecordings = {}
local CurrentTimelineFrame = 0
local TimelinePosition = 0
local AutoReset = false
local CurrentPlayingRecording = nil
local PausedAtFrame = 0
local playbackAccumulator = 0
local LastPausePosition = nil
local LastPauseRecording = nil
local LastPauseFrame = 0
local NearestRecordingDistance = math.huge
local LoopRetryAttempts = 0
local MaxLoopRetries = 999
local IsLoopTransitioning = false
local titlePulseConnection = nil
local previousFrameData = nil
local PathHasBeenUsed = {}
local PathsHiddenOnce = false
local ShiftLockVisualIndicator = nil
local ShiftLockCameraOffset = Vector3.new(1.75, 0, 0)
local ShiftLockUpdateConnection = nil
local OriginalCameraOffset = nil
local ShiftLockSavedBeforePlayback = false

-- ========= SOUND EFFECTS =========
local SoundEffects = {
    Click = "rbxassetid://4499400560",
    Toggle = "rbxassetid://7468131335",
    Error = "rbxassetid://7772283448",
    Success = "rbxassetid://2865227271"
}

-- ========= HELPER FUNCTIONS =========

local function AddConnection(connection)
    SafeCall(function()
        if connection then
            table.insert(activeConnections, connection)
        end
    end)
end

local function CleanupConnections()
    SafeCall(function()
        for _, connection in ipairs(activeConnections) do
            if connection then
                pcall(function() connection:Disconnect() end)
            end
        end
        activeConnections = {}
        
        if recordConnection then pcall(function() recordConnection:Disconnect() end) recordConnection = nil end
        if playbackConnection then pcall(function() playbackConnection:Disconnect() end) playbackConnection = nil end
        if loopConnection then pcall(function() task.cancel(loopConnection) end) loopConnection = nil end
        if shiftLockConnection then pcall(function() shiftLockConnection:Disconnect() end) shiftLockConnection = nil end
        if jumpConnection then pcall(function() jumpConnection:Disconnect() end) jumpConnection = nil end
        if reverseConnection then pcall(function() reverseConnection:Disconnect() end) reverseConnection = nil end
        if forwardConnection then pcall(function() forwardConnection:Disconnect() end) forwardConnection = nil end
        if titlePulseConnection then pcall(function() titlePulseConnection:Disconnect() end) titlePulseConnection = nil end
        if ShiftLockUpdateConnection then pcall(function() ShiftLockUpdateConnection:Disconnect() end) ShiftLockUpdateConnection = nil end
    end)
end

local function PlaySound(soundType)
    task.spawn(function()
        SafeCall(function()
            local sound = Instance.new("Sound")
            sound.SoundId = SoundEffects[soundType] or SoundEffects.Click
            sound.Volume = 0.3
            sound.Parent = workspace
            sound:Play()
            game:GetService("Debris"):AddItem(sound, 2)
        end)
    end)
end

local function AnimateButtonClick(button)
    if not button then return end
    PlaySound("Click")
    SafeCall(function()
        local originalColor = button.BackgroundColor3
        local brighterColor = Color3.new(
            math.min(originalColor.R * 1.3, 1),
            math.min(originalColor.G * 1.3, 1), 
            math.min(originalColor.B * 1.3, 1)
        )
        
        TweenService:Create(button, TweenInfo.new(0.1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = brighterColor
        }):Play()
        
        task.wait(0.1)
        
        TweenService:Create(button, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            BackgroundColor3 = originalColor
        }):Play()
    end)
end

local function ResetCharacter()
    SafeCall(function()
        local char = player.Character
        if char then
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if humanoid then
                humanoid.Health = 0
            end
        end
    end)
end

local function WaitForRespawn()
    local startTime = tick()
    local timeout = 10
    repeat
        task.wait(0.05)
        if tick() - startTime > timeout then return false end
    until player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChildOfClass("Humanoid") and player.Character.Humanoid.Health > 0
    task.wait(0.3)
    return true
end

local function IsCharacterReady()
    local char = player.Character
    if not char then return false end
    if not char:FindFirstChild("HumanoidRootPart") then return false end
    if not char:FindFirstChildOfClass("Humanoid") then return false end
    if char.Humanoid.Health <= 0 then return false end
    return true
end

local function CompleteCharacterReset(char)
    if not char or not char:IsDescendantOf(workspace) then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not humanoid or not hrp then return end
    
    task.spawn(function()
        SafeCall(function()
            local currentState = humanoid:GetState()
            
            humanoid.PlatformStand = false
            
            if LastKnownWalkSpeed > 0 then
                humanoid.WalkSpeed = LastKnownWalkSpeed
            elseif WalkSpeedBeforePlayback > 0 then
                humanoid.WalkSpeed = WalkSpeedBeforePlayback
            else
                humanoid.WalkSpeed = CurrentWalkSpeed
            end
            
            humanoid.JumpPower = prePauseJumpPower or 50
            humanoid.Sit = false
            
            if currentState == Enum.HumanoidStateType.Climbing then
                hrp.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
                humanoid.AutoRotate = false
                
            elseif currentState == Enum.HumanoidStateType.Swimming then
                hrp.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
                
            elseif currentState == Enum.HumanoidStateType.Jumping or
                   currentState == Enum.HumanoidStateType.Freefall then
                hrp.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
                
            else
                hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                hrp.AssemblyAngularVelocity = Vector3.new(0, 0, 0)
                humanoid.AutoRotate = true
                humanoid:ChangeState(Enum.HumanoidStateType.Running)
            end
        end)
    end)
end

-- ========= SHIFTLOCK SYSTEM =========

local function CreateShiftLockIndicator()
    SafeCall(function()
        if ShiftLockVisualIndicator then
            ShiftLockVisualIndicator:Destroy()
            ShiftLockVisualIndicator = nil
        end
        
        local ScreenGui = Instance.new("ScreenGui")
        ScreenGui.Name = "ShiftLockIndicator"
        ScreenGui.ResetOnSpawn = false
        ScreenGui.DisplayOrder = 999
        
        local indicator = Instance.new("ImageLabel")
        indicator.Name = "LockIcon"
        indicator.Size = UDim2.fromOffset(32, 32)
        indicator.Position = UDim2.new(0.5, 16, 0.5, 0)
        indicator.AnchorPoint = Vector2.new(0.5, 0.5)
        indicator.BackgroundTransparency = 1
        indicator.Image = "rbxasset://textures/ui/MouseLockedCursor.png"
        indicator.ImageColor3 = Color3.fromRGB(255, 255, 255)
        indicator.ImageTransparency = 0
        indicator.Parent = ScreenGui
        
        ScreenGui.Parent = player:WaitForChild("PlayerGui")
        ShiftLockVisualIndicator = ScreenGui
    end)
end

local function RemoveShiftLockIndicator()
    SafeCall(function()
        if ShiftLockVisualIndicator then
            ShiftLockVisualIndicator:Destroy()
            ShiftLockVisualIndicator = nil
        end
    end)
end

-- ⭐ FIXED: ShiftLock yang PERSISTENT selama playback
local function ApplyVisualShiftLock()
    if not ShiftLockEnabled then return end
    if not player.Character then return end
    
    SafeCall(function()
        local char = player.Character
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local camera = workspace.CurrentCamera
        
        if not humanoid or not hrp or not camera then return end
        
        -- ✅ APPLY SHIFT LOCK bahkan saat playback
        humanoid.AutoRotate = false
        
        local cameraCFrame = camera.CFrame
        local lookVector = cameraCFrame.LookVector
        local horizontalLook = Vector3.new(lookVector.X, 0, lookVector.Z)
        
        if horizontalLook.Magnitude > 0.01 then
            local targetCFrame = CFrame.new(hrp.Position, hrp.Position + horizontalLook)
            hrp.CFrame = targetCFrame
        end
        
        if not OriginalCameraOffset then
            OriginalCameraOffset = humanoid.CameraOffset
        end
        humanoid.CameraOffset = ShiftLockCameraOffset
    end)
end

local function EnableVisibleShiftLock()
    if ShiftLockUpdateConnection then return end
    
    SafeCall(function()
        isShiftLockActive = true
        ShiftLockEnabled = true
        
        CreateShiftLockIndicator()
        
        local char = player.Character
        if char then
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if humanoid and not OriginalCameraOffset then
                OriginalCameraOffset = humanoid.CameraOffset
            end
        end
        
        ShiftLockUpdateConnection = RunService.RenderStepped:Connect(function()
            if ShiftLockEnabled and player.Character then
                ApplyVisualShiftLock()
            end
        end)
        
        AddConnection(ShiftLockUpdateConnection)
        PlaySound("Toggle")
        
        if ShiftLockBtnControl then
            ShiftLockBtnControl.Text = "Shift ON"
            ShiftLockBtnControl.BackgroundColor3 = Color3.fromRGB(40, 180, 80)
        end
    end)
end

local function DisableVisibleShiftLock()
    SafeCall(function()
        if ShiftLockUpdateConnection then
            ShiftLockUpdateConnection:Disconnect()
            ShiftLockUpdateConnection = nil
        end
        
        RemoveShiftLockIndicator()
        
        local char = player.Character
        if char and char:FindFirstChildOfClass("Humanoid") then
            local humanoid = char.Humanoid
            humanoid.AutoRotate = true
            
            if OriginalCameraOffset then
                humanoid.CameraOffset = OriginalCameraOffset
                OriginalCameraOffset = nil
            else
                humanoid.CameraOffset = Vector3.new(0, 0, 0)
            end
        end
        
        isShiftLockActive = false
        ShiftLockEnabled = false
        PlaySound("Toggle")
        
        if ShiftLockBtnControl then
            ShiftLockBtnControl.Text = "Shift OFF"
            ShiftLockBtnControl.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
        end
    end)
end

local function ToggleVisibleShiftLock()
    if ShiftLockEnabled then
        DisableVisibleShiftLock()
    else
        EnableVisibleShiftLock()
    end
end

-- ⭐ REMOVED: SaveShiftLockState & RestoreShiftLockState
-- ShiftLock sekarang PERSISTENT, tidak perlu save/restore

-- ========= INFINITE JUMP =========

local function EnableInfiniteJump()
    if jumpConnection then return end
    jumpConnection = UserInputService.JumpRequest:Connect(function()
        if InfiniteJump and player.Character then
            SafeCall(function()
                local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
                if humanoid then
                    humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
                end
            end)
        end
    end)
    AddConnection(jumpConnection)
end

local function DisableInfiniteJump()
    if jumpConnection then
        SafeCall(function() jumpConnection:Disconnect() end)
        jumpConnection = nil
    end
end

local function ToggleInfiniteJump()
    InfiniteJump = not InfiniteJump
    if InfiniteJump then
        EnableInfiniteJump()
        if JumpBtnControl then
            JumpBtnControl.Text = "Jump ON"
            JumpBtnControl.BackgroundColor3 = Color3.fromRGB(40, 180, 80)
        end
    else
        DisableInfiniteJump()
        if JumpBtnControl then
            JumpBtnControl.Text = "Jump OFF"
            JumpBtnControl.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
        end
    end
end

-- ========= HUMANOID STATE MANAGEMENT =========

local function SaveHumanoidState()
    SafeCall(function()
        local char = player.Character
        if not char then return end
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            prePauseAutoRotate = humanoid.AutoRotate
            prePauseWalkSpeed = humanoid.WalkSpeed
            prePauseJumpPower = humanoid.JumpPower
            prePausePlatformStand = humanoid.PlatformStand
            prePauseSit = humanoid.Sit
            prePauseHumanoidState = humanoid:GetState()
        end
    end)
end

local function RestoreHumanoidState()
    SafeCall(function()
        local char = player.Character
        if not char then return end
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.AutoRotate = prePauseAutoRotate
            humanoid.WalkSpeed = prePauseWalkSpeed
            humanoid.JumpPower = prePauseJumpPower
            humanoid.PlatformStand = prePausePlatformStand
            humanoid.Sit = prePauseSit
        end
    end)
end

local function RestoreFullUserControl()
    SafeCall(function()
        local char = player.Character
        if not char then return end
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        
        if humanoid then
            local currentState = humanoid:GetState()
            
            if ShiftLockEnabled then
                humanoid.AutoRotate = false
            elseif currentState == Enum.HumanoidStateType.Climbing then
                humanoid.AutoRotate = false
            else
                humanoid.AutoRotate = true
            end          
            
            if LastKnownWalkSpeed > 0 then
                humanoid.WalkSpeed = LastKnownWalkSpeed  
            elseif WalkSpeedBeforePlayback > 0 then
                humanoid.WalkSpeed = WalkSpeedBeforePlayback  
            else
                humanoid.WalkSpeed = CurrentWalkSpeed 
            end
            
            humanoid.JumpPower = prePauseJumpPower or 50
            humanoid.PlatformStand = false
            humanoid.Sit = false
            
            if currentState ~= Enum.HumanoidStateType.Climbing and 
               currentState ~= Enum.HumanoidStateType.Swimming and
               currentState ~= Enum.HumanoidStateType.Jumping and
               currentState ~= Enum.HumanoidStateType.Freefall then
                humanoid:ChangeState(Enum.HumanoidStateType.Running)
            end
            
            if ShiftLockEnabled then
                humanoid.CameraOffset = ShiftLockCameraOffset
            else
                if OriginalCameraOffset then
                    humanoid.CameraOffset = OriginalCameraOffset
                else
                    humanoid.CameraOffset = Vector3.new(0, 0, 0)
                end
            end
        end
        
        if hrp then
            local currentState = humanoid and humanoid:GetState()
            
            if currentState == Enum.HumanoidStateType.Running or
               currentState == Enum.HumanoidStateType.RunningNoPhysics or
               currentState == Enum.HumanoidStateType.Landed then
                hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                hrp.AssemblyAngularVelocity = Vector3.new
