local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

local ANTIFLASH_CONFIG = {
    Enabled = true,
    FlashNames = {
        -- Common flash effect names
        "Flash", "Flashbang", "FlashEffect", "FlashScreen", "WhiteScreen",
        "Blind", "BlindEffect", "BlindScreen", "Blinded", "Flashed",
        "Stun", "StunEffect", "StunScreen", "Stunned",
        -- Pattern matches (lower case)
        "flash", "blind", "white", "bright", "stun",
        -- Common container names
        "ScreenEffects", "EffectsFolder", "StatusEffects"
    },
    Properties = {
        MinBrightness = 0.5, -- More sensitive brightness detection
        MaxTransparency = 0.1, -- More aggressive transparency detection
        RemovalDelay = 0, -- Instant removal
        ForcedTransparency = 1, -- Completely transparent
        DetectionRadius = 0.3 -- How close to white a color needs to be
    },
    Locations = {
        "PlayerGui",
        "CoreGui",
        "Camera",
        "LocalPlayer",
        "Lighting"
    },
    ProtectedGUIs = {
        -- Default Roblox GUIs
        "PlayerList", "Chat", "BubbleChat", "RobloxGui",
        "PlayerListMaster", "LoadingGui", "PurchasePrompt",
        "MenuUI", "TopBarApp", "CoreGui", "InGameMenu",
        -- Common game UIs
        "Menu", "Settings", "Inventory", "Shop", "StoreGui",
        "GameMenu", "PauseMenu", "ScoreBoard", "LeaderBoard",
        "MainGui", "CoreGui", "HUD", "MainHUD", "GameHUD",
        -- Protected patterns
        "Menu", "Score", "Leader", "Store", "Settings",
        "Pause", "Options", "HUD", "UI", "Interface",
        -- Game-specific UI
        "Weapon", "Gun", "Tool", "Arms", "Viewmodel",
        "Ammo", "Health", "Stamina", "Energy",
        "Counter", "Timer", "Round", "Score",
        "Killfeed", "Hitmarker", "Crosshair",
        "Status", "State", "Indicator",
        -- Common containers
        "GUI", "Interface", "Display", "Screen",
        "Main", "Base", "Root", "Container"
    },
    UI = {
        Size = UDim2.new(0, 200, 0, 40),
        Position = UDim2.new(0.5, -100, 0, 20),
        Color = {
            Background = Color3.fromRGB(25, 25, 25),
            Text = Color3.fromRGB(255, 255, 255),
            Enabled = Color3.fromRGB(0, 255, 0),
            Disabled = Color3.fromRGB(255, 0, 0)
        },
        Transparency = 0.2
    },
    Settings = {
        CustomHotkey = Enum.KeyCode.RightAlt,
        NotificationsEnabled = true,
        AutoStartEnabled = true,
        PreventScreenshake = true,
        BlockBlindnessEffects = true
    },
    Stats = {
        FlashesBlocked = 0,
        TimeActive = 0,
        LastFlash = 0
    },
    Presets = {
        Default = {
            MinBrightness = 0.8,
            MaxTransparency = 0.2
        },
        Aggressive = {
            MinBrightness = 0.6,
            MaxTransparency = 0.4
        },
        Minimal = {
            MinBrightness = 0.9,
            MaxTransparency = 0.1
        }
    },
    Notification = {
        Duration = 2,
        Position = UDim2.new(0.5, 0, 0, 80),
        BackgroundColor = Color3.fromRGB(30, 30, 30),
        TextColor = Color3.fromRGB(255, 255, 255)
    }
}

-- Add performance monitoring
ANTIFLASH_CONFIG.Performance = {
    LastCheckTime = 0,
    AverageCheckTime = 0,
    CheckCount = 0,
    MaxCheckTime = 0,
    InstanceCache = {},
    ThrottleTime = 0.05, -- Minimum time between checks
    CacheTimeout = 5     -- Cache timeout in seconds
}

-- Add whitelist for important game mechanics
ANTIFLASH_CONFIG.GameMechanics = {
    AllowedEffects = {
        "DamageIndicator",
        "HitEffect",
        "HealthEffect",
        "RegenerationEffect",
        "PowerupEffect",
        "GameUI",
        "Score",
        "Weapon", "Gun", "Tool", "Arms",
        "Viewmodel", "Bullet", "Projectile",
        "Ammo", "Magazine", "Reload",
        "Scope", "Sight", "Aim",
        "Fire", "Shoot", "Recoil"
    },
    PreserveProperties = {
        "Health",
        "Stamina",
        "Energy",
        "Power"
    },
    MinimumTransparency = 0.7, -- Less aggressive transparency
    MaximumDuration = 1.5 -- Maximum time to modify effects
}

-- Optimize pattern matching
local flashPatterns = {}
for _, pattern in ipairs(ANTIFLASH_CONFIG.FlashNames) do
    flashPatterns[pattern:lower()] = true
end

local protectedPatterns = {}
for _, pattern in ipairs(ANTIFLASH_CONFIG.ProtectedGUIs) do
    protectedPatterns[pattern:lower()] = true
end

-- Optimized protected GUI check
local function isProtectedGui(instance)
    local cacheKey = tostring(instance)
    local cached = ANTIFLASH_CONFIG.Performance.InstanceCache[cacheKey]
    if cached and (tick() - cached.time) < ANTIFLASH_CONFIG.Performance.CacheTimeout then
        return cached.protected
    end

    local current = instance
    local protection = false
    
    while current and not protection do
        local nameLower = current.Name:lower()
        protection = protectedPatterns[nameLower] or current:IsA("CoreGui")
        current = current.Parent
    end
    
    ANTIFLASH_CONFIG.Performance.InstanceCache[cacheKey] = {
        time = tick(),
        protected = protection
    }
    
    return protection
end

-- Add new flash handling configuration
ANTIFLASH_CONFIG.FlashHandling = {
    MaxBrightness = 0.3, -- Lower threshold for brightness
    MinTransparency = 0.95, -- Higher forced transparency
    CheckInterval = 0.01, -- Check more frequently
    BlockRadius = 1000, -- Large blocking radius
    RemoveDelay = 0 -- Instant removal
}

-- Modified flash detection to be more aggressive
local function isFlashEffect(instance)
    -- First check protected elements
    if instance:IsA("Tool") or instance:IsA("Model") or instance.Name:lower():find("gun") then
        return false
    end
    
    -- Check if it's UI
    if instance.Name:lower():find("hud") or instance.Name:lower():find("health") then
        return false
    end

    -- Aggressive flash detection
    if instance:IsA("GuiObject") then
        local color = instance.BackgroundColor3
        -- Detect ANY bright screen effect
        if (color.R + color.G + color.B)/3 > 0.6 and 
           instance.BackgroundTransparency < 0.5 and
           not instance.Name:lower():find("hud") then
            return true
        end
    end
    
    -- Check for flash effects
    if instance:IsA("BlurEffect") or 
       instance:IsA("ColorCorrectionEffect") or
       instance:IsA("SunRaysEffect") then
        return true
    end
    
    return instance.Name:lower():find("flash") or 
           instance.Name:lower():find("blind") or
           instance.Name:lower():find("stun")
end

local function ShowNotification(text, duration)
    duration = duration or ANTIFLASH_CONFIG.Notification.Duration
    
    local notification = Instance.new("Frame")
    notification.Size = UDim2.new(0, 200, 0, 30)
    notification.Position = ANTIFLASH_CONFIG.Notification.Position
    notification.AnchorPoint = Vector2.new(0.5, 0)
    notification.BackgroundColor3 = ANTIFLASH_CONFIG.Notification.BackgroundColor
    notification.BackgroundTransparency = 0.2
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = notification
    
    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = text
    textLabel.TextColor3 = ANTIFLASH_CONFIG.Notification.TextColor
    textLabel.Font = Enum.Font.GothamBold
    textLabel.TextSize = 14
    textLabel.Parent = notification
    
    notification.Parent = toggleUI
    
    game:GetService("Debris"):AddItem(notification, duration)
end

local function UpdateStats(flashBlocked)
    if flashBlocked then
        ANTIFLASH_CONFIG.Stats.FlashesBlocked += 1
        ANTIFLASH_CONFIG.Stats.LastFlash = tick()
    end
end

local function isGameMechanic(instance)
    local name = instance.Name:lower()
    for _, allowed in ipairs(ANTIFLASH_CONFIG.GameMechanics.AllowedEffects) do
        if name:find(allowed:lower()) then
            return true
        end
    end
    return false
end

-- Modified flash handling for immediate effect
local function handleFlash(instance)
    if not ANTIFLASH_CONFIG.Enabled then return end
    
    pcall(function()
        if isFlashEffect(instance) then
            -- Handle GUI elements aggressively
            if instance:IsA("GuiObject") then
                instance.BackgroundTransparency = 1
                instance.BackgroundColor3 = Color3.new(0, 0, 0)
                if instance:IsA("ImageLabel") then
                    instance.ImageTransparency = 1
                end
            end
            
            -- Disable effects immediately
            if instance:IsA("BlurEffect") or 
               instance:IsA("ColorCorrectionEffect") or 
               instance:IsA("SunRaysEffect") then
                instance.Enabled = false
            end
            
            -- Try to remove the entire flash effect hierarchy
            pcall(function()
                local parent = instance.Parent
                if parent and 
                   (parent.Name:lower():find("flash") or 
                    parent.Name:lower():find("effect")) then
                    parent:Destroy()
                end
            end)
            
            -- Remove the instance itself
            game:GetService("Debris"):AddItem(instance, 0)
            
            UpdateStats(true)
        end
    end)
end

-- Modify the container check function
local function checkContainer(container)
    if not container then return end
    
    -- Don't check protected containers
    if container.Name:lower():find("weapon") or
       container.Name:lower():find("gun") or
       container.Name:lower():find("hud") or
       container.Name:lower():find("health") then
        return
    end
    
    pcall(function()
        -- Only check immediate children
        for _, item in ipairs(container:GetChildren()) do
            if item.Parent == container then
                handleFlash(item)
            end
        end
    end)
end

-- Modify main loop to check more frequently
local function startAntiFlash()
    spawn(function()
        local lastCleanup = tick()
        
        while true do
            if ANTIFLASH_CONFIG.Enabled then
                -- Clean cache periodically
                if tick() - lastCleanup > 30 then
                    ANTIFLASH_CONFIG.Performance.InstanceCache = {}
                    lastCleanup = tick()
                end
                
                pcall(function()
                    checkContainer(LocalPlayer.PlayerGui)
                    checkContainer(workspace.CurrentCamera)
                    checkContainer(game:GetService("Lighting"))
                    
                    -- Throttle the loop based on performance metrics
                    local waitTime = math.max(0.1, ANTIFLASH_CONFIG.Performance.AverageCheckTime * 2)
                    wait(waitTime)
                end)
            else
                wait(0.5) -- Longer wait when disabled
            end
        end
    end)
end

-- Declare toggle function first
local toggleAntiFlash

-- Create UI with proper toggle reference
local function CreateToggleUI()
    local ToggleGui = Instance.new("ScreenGui")
    ToggleGui.Name = "AntiFlashToggle"
    
    -- Try to parent to CoreGui
    pcall(function()
        if syn and syn.protect_gui then
            syn.protect_gui(ToggleGui)
            ToggleGui.Parent = game:GetService("CoreGui")
        else
            ToggleGui.Parent = game:GetService("CoreGui")
        end
    end)
    
    local Frame = Instance.new("Frame")
    Frame.Size = ANTIFLASH_CONFIG.UI.Size
    Frame.Position = ANTIFLASH_CONFIG.UI.Position
    Frame.BackgroundColor3 = ANTIFLASH_CONFIG.UI.Color.Background
    Frame.BackgroundTransparency = ANTIFLASH_CONFIG.UI.Transparency
    Frame.BorderSizePixel = 0
    Frame.Parent = ToggleGui
    
    -- Add rounded corners
    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(0, 6)
    Corner.Parent = Frame
    
    -- Add status text
    local StatusText = Instance.new("TextLabel")
    StatusText.Name = "Status"
    StatusText.Size = UDim2.new(1, 0, 1, 0)
    StatusText.BackgroundTransparency = 1
    StatusText.Text = "Anti-Flash: ON"
    StatusText.TextColor3 = ANTIFLASH_CONFIG.UI.Color.Enabled
    StatusText.TextSize = 16
    StatusText.Font = Enum.Font.GothamBold
    StatusText.Parent = Frame
    
    -- Make UI draggable
    local dragging
    local dragInput
    local dragStart
    local startPos
    
    Frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            startPos = Frame.Position
        end
    end)
    
    Frame.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    
    game:GetService("UserInputService").InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement and dragging then
            local delta = input.Position - dragStart
            Frame.Position = UDim2.new(
                startPos.X.Scale,
                startPos.X.Offset + delta.X,
                startPos.Y.Scale,
                startPos.Y.Offset + delta.Y
            )
        end
    end)
    
    -- Define toggle function
    toggleAntiFlash = function()
        ANTIFLASH_CONFIG.Enabled = not ANTIFLASH_CONFIG.Enabled
        StatusText.Text = "Anti-Flash: " .. (ANTIFLASH_CONFIG.Enabled and "ON" or "OFF")
        StatusText.TextColor3 = ANTIFLASH_CONFIG.Enabled and 
            ANTIFLASH_CONFIG.UI.Color.Enabled or 
            ANTIFLASH_CONFIG.UI.Color.Disabled
    end
    
    -- Add click to toggle
    Frame.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton2 then
            toggleAntiFlash()
        end
    end)
    
    return ToggleGui, toggleAntiFlash
end

local function CreateSettingsMenu()
    local settingsUI = Instance.new("Frame")
    -- ... settings UI creation code ...
    
    -- Add preset buttons
    for name, preset in pairs(ANTIFLASH_CONFIG.Presets) do
        local button = Instance.new("TextButton")
        button.Text = name
        button.MouseButton1Click:Connect(function()
            ANTIFLASH_CONFIG.Properties.MinBrightness = preset.MinBrightness
            ANTIFLASH_CONFIG.Properties.MaxTransparency = preset.MaxTransparency
            ShowNotification("Applied " .. name .. " preset")
        end)
    end
    
    -- Add hotkey changer
    local hotkeyButton = Instance.new("TextButton")
    hotkeyButton.Text = "Change Hotkey"
    hotkeyButton.MouseButton1Click:Connect(function()
        hotkeyButton.Text = "Press any key..."
        local connection
        connection = game:GetService("UserInputService").InputBegan:Connect(function(input)
            if input.KeyCode ~= Enum.KeyCode.Unknown then
                ANTIFLASH_CONFIG.Settings.CustomHotkey = input.KeyCode
                hotkeyButton.Text = "Hotkey: " .. input.KeyCode.Name
                connection:Disconnect()
                ShowNotification("Hotkey changed to " .. input.KeyCode.Name)
            end
        end)
    end)
    
    return settingsUI
end

-- Initialize UI and get toggle function
local toggleUI, toggleFunc = CreateToggleUI()

-- Keybind to toggle using the correct function
game:GetService("UserInputService").InputBegan:Connect(function(input)
    if input.KeyCode == ANTIFLASH_CONFIG.Settings.CustomHotkey then -- Change key as needed
        toggleFunc()
    end
end)

-- Start the anti-flash system
pcall(function()
    startAntiFlash()
    print("Anti-Flash initialized successfully (Loop Mode)")
end)

-- Add performance monitoring to the return table
local api = {
    toggle = toggleFunc,
    setTransparency = function(value) 
        ANTIFLASH_CONFIG.Transparency = math.clamp(value, 0, 1)
    end,
    toggleUI = toggleUI,
    getStats = function()
        return ANTIFLASH_CONFIG.Stats
    end,
    setHotkey = function(keycode)
        ANTIFLASH_CONFIG.Settings.CustomHotkey = keycode
    end,
    showSettings = function()
        CreateSettingsMenu()
    end,
    applyPreset = function(presetName)
        local preset = ANTIFLASH_CONFIG.Presets[presetName]
        if preset then
            ANTIFLASH_CONFIG.Properties.MinBrightness = preset.MinBrightness
            ANTIFLASH_CONFIG.Properties.MaxTransparency = preset.MaxTransparency
            return true
        end
        return false
    end,
    getPerformanceStats = function()
        return {
            averageCheckTime = ANTIFLASH_CONFIG.Performance.AverageCheckTime,
            maxCheckTime = ANTIFLASH_CONFIG.Performance.MaxCheckTime,
            checkCount = ANTIFLASH_CONFIG.Performance.CheckCount,
            cacheSize = #ANTIFLASH_CONFIG.Performance.InstanceCache
        }
    end,
    clearCache = function()
        ANTIFLASH_CONFIG.Performance.InstanceCache = {}
        ANTIFLASH_CONFIG.Performance.LastCheckTime = 0
    end
}

return api
