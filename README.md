--[[
    PAINEL UNIVERSAL v6
    Fly CORRIGIDO (joystick mobile OK + sem vai-e-vem)
    Sem Auto-Parry / Auto-Block / Auto-Farm
    Mobile-friendly
]]

if _G.UniversalPanelLoaded then
    pcall(function()
        if _G.UniversalPanelGUI then _G.UniversalPanelGUI:Destroy() end
    end)
end
_G.UniversalPanelLoaded = true

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local HttpService = game:GetService("HttpService")
local VirtualUser = game:GetService("VirtualUser")
local VirtualInputManager = game:GetService("VirtualInputManager")

local LocalPlayer = Players.LocalPlayer

local function getGuiParent()
    local ok, result = pcall(function() return CoreGui end)
    if ok and result then return result end
    return LocalPlayer:WaitForChild("PlayerGui")
end

local IS_MOBILE = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

local CONFIG = {
    DefaultSpeed = 16,
    FlySpeedMin = 20,
    FlySpeedMax = 300,
    FlySpeedDefault = 100,
    UI_Color = Color3.fromRGB(12, 12, 14),
    UI_Card = Color3.fromRGB(22, 22, 25),
    UI_Header = Color3.fromRGB(8, 8, 10),
    UI_Border = Color3.fromRGB(38, 38, 42),
    UI_Bar = Color3.fromRGB(30, 30, 34),
    UI_Accent = Color3.fromRGB(220, 220, 230),
    UI_Success = Color3.fromRGB(70, 200, 120),
    UI_Off = Color3.fromRGB(45, 45, 50),
    UI_Danger = Color3.fromRGB(230, 60, 70),
    UI_Text = Color3.fromRGB(240, 240, 245),
    UI_SubText = Color3.fromRGB(150, 150, 160),
    UI_FlyBtn = Color3.fromRGB(200, 200, 210),
    UI_FOV = Color3.fromRGB(255, 255, 255),
    UI_AimBtn = Color3.fromRGB(15, 15, 15),
    UI_AimBtnHold = Color3.fromRGB(60, 60, 60),
}

local State = {
    InfiniteJump = false,
    Noclip = false,
    Fly = false,
    ESP = false,
    FlySpeed = CONFIG.FlySpeedDefault,
    WalkSpeed = CONFIG.DefaultSpeed,
    LockWalkSpeed = false,
    AimEnabled = false,
    ShowFOV = false,
    FOV = 150,
    TeamCheck = false,
    AimBtnOpacity = 0.6,
    AntiAFK = false,
    AutoClicker = false,
    AutoClickerCPS = 15,
}

local Connections = {}
local function addConn(c) table.insert(Connections, c); return c end

local function create(cls, props)
    local i = Instance.new(cls)
    for k, v in pairs(props or {}) do i[k] = v end
    return i
end
local function corner(p, r)
    return create("UICorner", {CornerRadius = r or UDim.new(0, 10), Parent = p})
end
local function stroke(p, c, t)
    return create("UIStroke", {
        Color = c or CONFIG.UI_Border,
        Thickness = t or 1,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
        Parent = p,
    })
end
local function getHumanoid()
    local c = LocalPlayer.Character
    if not c then return nil, nil end
    return c:FindFirstChildOfClass("Humanoid"), c:FindFirstChild("HumanoidRootPart")
end

-- ============================================================
-- SAVE
-- ============================================================
local SAVE_FILE = "UniversalPanelConfig.json"
local function safeSaveToFile(data)
    if writefile and isfile then
        pcall(function() writefile(SAVE_FILE, HttpService:JSONEncode(data)) end)
    end
end
local function safeLoadFromFile()
    if readfile and isfile then
        local ok, content = pcall(function()
            if isfile(SAVE_FILE) then return readfile(SAVE_FILE) end
            return nil
        end)
        if ok and content then
            local ok2, decoded = pcall(function() return HttpService:JSONDecode(content) end)
            if ok2 and decoded then return decoded end
        end
    end
    return nil
end
local savedConfig = safeLoadFromFile() or {}

local function serializeUDim2(u) return {u.X.Scale, u.X.Offset, u.Y.Scale, u.Y.Offset} end
local function deserializeUDim2(t)
    if not t then return nil end
    return UDim2.new(t[1], t[2], t[3], t[4])
end

-- ============================================================
-- UI PRINCIPAL
-- ============================================================
local ScreenGui = create("ScreenGui", {
    Name = "UniversalUI_" .. tostring(math.random(1000, 9999)),
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
})
pcall(function() ScreenGui.Parent = getGuiParent() end)
_G.UniversalPanelGUI = ScreenGui

local DEFAULT_TOGGLE_POS = UDim2.new(0, 16, 0.5, -28)
local ToggleButton = create("TextButton", {
    Name = "ToggleButton",
    Size = UDim2.fromOffset(56, 56),
    Position = deserializeUDim2(savedConfig.togglePos) or DEFAULT_TOGGLE_POS,
    BackgroundColor3 = Color3.fromRGB(20, 20, 24),
    Text = "≡",
    TextSize = 28,
    Font = Enum.Font.GothamBold,
    TextColor3 = CONFIG.UI_Text,
    AutoButtonColor = false,
    Active = true,
    Parent = ScreenGui,
})
corner(ToggleButton, UDim.new(1, 0))
stroke(ToggleButton, Color3.fromRGB(80, 80, 90), 2)

local MainFrame = create("Frame", {
    Name = "MainFrame",
    Size = IS_MOBILE and UDim2.new(0.9, 0, 0.78, 0) or UDim2.fromOffset(420, 540),
    Position = UDim2.new(0.5, 0, 0.5, 0),
    AnchorPoint = Vector2.new(0.5, 0.5),
    BackgroundColor3 = CONFIG.UI_Color,
    BorderSizePixel = 0,
    Visible = false,
    ClipsDescendants = true,
    Active = true,
    Parent = ScreenGui,
})
corner(MainFrame, UDim.new(0, 16))
stroke(MainFrame, Color3.fromRGB(60, 60, 70), 1.5)

local Header = create("Frame", {
    Name = "Header",
    Size = UDim2.new(1, 0, 0, 56),
    BackgroundColor3 = CONFIG.UI_Header,
    BorderSizePixel = 0,
    Active = true,
    Parent = MainFrame,
})
corner(Header, UDim.new(0, 16))
create("Frame", {
    Size = UDim2.new(1, 0, 0, 20),
    Position = UDim2.new(0, 0, 1, -20),
    BackgroundColor3 = CONFIG.UI_Header,
    BorderSizePixel = 0,
    Parent = Header,
})
create("TextLabel", {
    Size = UDim2.new(1, -100, 1, 0),
    Position = UDim2.new(0, 20, 0, 0),
    BackgroundTransparency = 1,
    Text = "⚡ PAINEL UNIVERSAL",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 18,
    Font = Enum.Font.GothamBold,
    TextXAlignment = Enum.TextXAlignment.Left,
    Parent = Header,
})

local CloseButton = create("TextButton", {
    Size = UDim2.fromOffset(40, 40),
    Position = UDim2.new(1, -50, 0.5, -20),
    BackgroundColor3 = CONFIG.UI_Danger,
    Text = "✕",
    TextSize = 18,
    Font = Enum.Font.GothamBold,
    TextColor3 = CONFIG.UI_Text,
    AutoButtonColor = false,
    Parent = Header,
})
corner(CloseButton, UDim.new(0, 8))

local ContentScroll = create("ScrollingFrame", {
    Name = "Content",
    Size = UDim2.new(1, -20, 1, -76),
    Position = UDim2.new(0, 10, 0, 66),
    BackgroundTransparency = 1,
    BorderSizePixel = 0,
    ScrollBarThickness = 4,
    ScrollBarImageColor3 = Color3.fromRGB(120, 120, 130),
    CanvasSize = UDim2.new(0, 0, 0, 0),
    AutomaticCanvasSize = Enum.AutomaticSize.Y,
    Parent = MainFrame,
})
create("UIListLayout", {
    Padding = UDim.new(0, 10),
    SortOrder = Enum.SortOrder.LayoutOrder,
    Parent = ContentScroll,
})

local function createCard(title, subtitle)
    local card = create("Frame", {
        Size = UDim2.new(1, 0, 0, 0),
        AutomaticSize = Enum.AutomaticSize.Y,
        BackgroundColor3 = CONFIG.UI_Card,
        BorderSizePixel = 0,
        Parent = ContentScroll,
    })
    corner(card, UDim.new(0, 12))
    stroke(card, CONFIG.UI_Border, 1)
    create("UIPadding", {
        PaddingTop = UDim.new(0, 12), PaddingBottom = UDim.new(0, 12),
        PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14),
        Parent = card,
    })
    create("UIListLayout", {
        Padding = UDim.new(0, 8),
        SortOrder = Enum.SortOrder.LayoutOrder,
        Parent = card,
    })
    create("TextLabel", {
        Size = UDim2.new(1, 0, 0, 20),
        BackgroundTransparency = 1,
        Text = title,
        TextColor3 = CONFIG.UI_Text,
        TextSize = 16,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Left,
        LayoutOrder = 0,
        Parent = card,
    })
    if subtitle then
        create("TextLabel", {
            Size = UDim2.new(1, 0, 0, 16),
            BackgroundTransparency = 1,
            Text = subtitle,
            TextColor3 = CONFIG.UI_SubText,
            TextSize = 12,
            Font = Enum.Font.Gotham,
            TextXAlignment = Enum.TextXAlignment.Left,
            LayoutOrder = 1,
            Parent = card,
        })
    end
    return card
end

local function createToggle(parent, label, initialValue, callback)
    local row = create("Frame", {
        Size = UDim2.new(1, 0, 0, 44),
        BackgroundTransparency = 1,
        LayoutOrder = 10,
        Parent = parent,
    })
    create("TextLabel", {
        Size = UDim2.new(1, -80, 1, 0),
        BackgroundTransparency = 1,
        Text = label,
        TextColor3 = CONFIG.UI_Text,
        TextSize = 14,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = row,
    })
    local button = create("TextButton", {
        Size = UDim2.fromOffset(64, 32),
        Position = UDim2.new(1, -64, 0.5, -16),
        BackgroundColor3 = initialValue and CONFIG.UI_Success or CONFIG.UI_Off,
        Text = "",
        AutoButtonColor = false,
        Parent = row,
    })
    corner(button, UDim.new(1, 0))
    local knob = create("Frame", {
        Size = UDim2.fromOffset(26, 26),
        Position = initialValue and UDim2.new(1, -30, 0.5, -13) or UDim2.new(0, 4, 0.5, -13),
        BackgroundColor3 = Color3.fromRGB(240, 240, 245),
        BorderSizePixel = 0,
        Parent = button,
    })
    corner(knob, UDim.new(1, 0))
    local isOn = initialValue
    local function refresh()
        local tPos = isOn and UDim2.new(1, -30, 0.5, -13) or UDim2.new(0, 4, 0.5, -13)
        local tCol = isOn and CONFIG.UI_Success or CONFIG.UI_Off
        TweenService:Create(knob, TweenInfo.new(0.2), {Position = tPos}):Play()
        TweenService:Create(button, TweenInfo.new(0.2), {BackgroundColor3 = tCol}):Play()
    end
    button.MouseButton1Click:Connect(function()
        isOn = not isOn
        refresh()
        if callback then callback(isOn) end
    end)
    return {Set = function(v) isOn = v; refresh() end}
end

local function createSlider(parent, label, minV, maxV, default, callback)
    local card = create("Frame", {
        Size = UDim2.new(1, 0, 0, 62),
        BackgroundTransparency = 1,
        LayoutOrder = 20,
        Parent = parent,
    })
    create("TextLabel", {
        Size = UDim2.new(0.6, 0, 0, 20),
        BackgroundTransparency = 1,
        Text = label,
        TextColor3 = CONFIG.UI_Text,
        TextSize = 14,
        Font = Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        Parent = card,
    })
    local valueLabel = create("TextLabel", {
        Size = UDim2.new(0.4, 0, 0, 20),
        Position = UDim2.new(0.6, 0, 0, 0),
        BackgroundTransparency = 1,
        Text = tostring(default),
        TextColor3 = CONFIG.UI_Text,
        TextSize = 14,
        Font = Enum.Font.GothamBold,
        TextXAlignment = Enum.TextXAlignment.Right,
        Parent = card,
    })
    local barBg = create("Frame", {
        Size = UDim2.new(1, 0, 0, 10),
        Position = UDim2.new(0, 0, 0, 34),
        BackgroundColor3 = CONFIG.UI_Bar,
        BorderSizePixel = 0,
        Parent = card,
    })
    corner(barBg, UDim.new(1, 0))
    local barFill = create("Frame", {
        Size = UDim2.new((default - minV) / (maxV - minV), 0, 1, 0),
        BackgroundColor3 = CONFIG.UI_Accent,
        BorderSizePixel = 0,
        Parent = barBg,
    })
    corner(barFill, UDim.new(1, 0))
    local knob = create("Frame", {
        Size = UDim2.fromOffset(20, 20),
        Position = UDim2.new((default - minV) / (maxV - minV), -10, 0.5, -10),
        BackgroundColor3 = Color3.fromRGB(240, 240, 245),
        BorderSizePixel = 0,
        ZIndex = 2,
        Parent = barBg,
    })
    corner(knob, UDim.new(1, 0))
    local dragging, current = false, default
    local function setValue(v, callCb)
        v = math.clamp(v, minV, maxV)
        current = v
        local a = (v - minV) / (maxV - minV)
        barFill.Size = UDim2.new(a, 0, 1, 0)
        knob.Position = UDim2.new(a, -10, 0.5, -10)
        valueLabel.Text = tostring(math.floor(v * 100) / 100)
        if callCb and callback then callback(v) end
    end
    local function updateFromInput(input)
        local x = input.Position.X - barBg.AbsolutePosition.X
        local a = math.clamp(x / math.max(barBg.AbsoluteSize.X, 1), 0, 1)
        setValue(minV + a * (maxV - minV), true)
    end
    barBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            updateFromInput(input)
        end
    end)
    addConn(UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch) then
            updateFromInput(input)
        end
    end))
    addConn(UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end))
    return {Set = function(v) setValue(v, false) end, Get = function() return current end}
end

local function createButton(parent, label, color, callback)
    local btn = create("TextButton", {
        Size = UDim2.new(1, 0, 0, 40),
        BackgroundColor3 = color,
        Text = label,
        TextColor3 = CONFIG.UI_Text,
        TextSize = 14,
        Font = Enum.Font.GothamBold,
        AutoButtonColor = false,
        LayoutOrder = 30,
        Parent = parent,
    })
    corner(btn, UDim.new(0, 8))
    btn.MouseButton1Click:Connect(function() if callback then callback() end end)
    return btn
end

-- ============================================================
-- SPEED LOCK
-- ============================================================
local speedLockConn
local function startSpeedLock()
    if speedLockConn then return end
    speedLockConn = RunService.Heartbeat:Connect(function()
        if not State.LockWalkSpeed then return end
        local hum = getHumanoid()
        if hum and hum.WalkSpeed ~= State.WalkSpeed then
            hum.WalkSpeed = State.WalkSpeed
        end
    end)
end
local function stopSpeedLock()
    if speedLockConn then speedLockConn:Disconnect(); speedLockConn = nil end
end

-- ============================================================
-- FOV CIRCLE
-- ============================================================
local FOVCircle = create("Frame", {
    Name = "FOVCircle",
    Size = UDim2.fromOffset(State.FOV * 2, State.FOV * 2),
    Position = UDim2.new(0.5, 0, 0.5, 0),
    AnchorPoint = Vector2.new(0.5, 0.5),
    BackgroundTransparency = 1,
    Visible = false,
    ZIndex = 999,
    Parent = ScreenGui,
})
corner(FOVCircle, UDim.new(1, 0))
stroke(FOVCircle, CONFIG.UI_FOV, 2)

-- ============================================================
-- CARDS
-- ============================================================
local jumpCard = createCard("🦘 Pulo Infinito", "Pular continuamente no ar")
createToggle(jumpCard, "Ativar Pulo Infinito", false, function(v)
    State.InfiniteJump = v
end)

local speedCard = createCard("🏃 Velocidade", "Ajustar a velocidade")
local speedSlider = createSlider(speedCard, "Velocidade", 8, 200, CONFIG.DefaultSpeed, function(v)
    State.WalkSpeed = v
    local hum = getHumanoid()
    if hum then hum.WalkSpeed = v end
end)
createToggle(speedCard, "🔒 Travar Velocidade", false, function(v)
    State.LockWalkSpeed = v
    if v then startSpeedLock() else stopSpeedLock() end
end)
createButton(speedCard, "↺ Restaurar Padrão", Color3.fromRGB(35, 35, 40), function()
    speedSlider.Set(CONFIG.DefaultSpeed)
    State.WalkSpeed = CONFIG.DefaultSpeed
    local hum = getHumanoid()
    if hum then hum.WalkSpeed = CONFIG.DefaultSpeed end
end)

local espCard = createCard("👁 ESP", "Nome + distância + silhueta azul")
createToggle(espCard, "Ativar ESP", false, function(v)
    State.ESP = v
    if v then enableESP() else disableESP() end
end)

local noclipCard = createCard("🚪 Atravessar Paredes", "Atravessar objetos do mapa")
createToggle(noclipCard, "Ativar Noclip", false, function(v)
    State.Noclip = v
    applyNoclip(v)
end)

-- ============================================================
-- CARD: TELEPORTAR
-- ============================================================
local tpCard = createCard("🌀 Teleportar", "Escolha um jogador e teleporte até ele")

local tpButton = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "Selecionar Jogador ▼",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 10,
    Parent = tpCard,
})
corner(tpButton, UDim.new(0, 8))
stroke(tpButton, CONFIG.UI_Border, 1)

local tpListFrame = create("Frame", {
    Size = UDim2.new(1, 0, 0, 0),
    AutomaticSize = Enum.AutomaticSize.Y,
    BackgroundColor3 = Color3.fromRGB(18, 18, 20),
    Visible = false,
    LayoutOrder = 11,
    Parent = tpCard,
})
corner(tpListFrame, UDim.new(0, 8))
stroke(tpListFrame, CONFIG.UI_Border, 1)
create("UIPadding", {
    PaddingTop = UDim.new(0, 6), PaddingBottom = UDim.new(0, 6),
    PaddingLeft = UDim.new(0, 6), PaddingRight = UDim.new(0, 6),
    Parent = tpListFrame,
})
create("UIListLayout", {
    Padding = UDim.new(0, 4),
    SortOrder = Enum.SortOrder.LayoutOrder,
    Parent = tpListFrame,
})

local selectedPlayer = nil
local function refreshPlayerList()
    for _, child in ipairs(tpListFrame:GetChildren()) do
        if child:IsA("TextButton") then child:Destroy() end
    end
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            local btn = create("TextButton", {
                Size = UDim2.new(1, 0, 0, 32),
                BackgroundColor3 = Color3.fromRGB(30, 30, 34),
                Text = plr.Name,
                TextColor3 = CONFIG.UI_Text,
                TextSize = 13,
                Font = Enum.Font.GothamMedium,
                AutoButtonColor = false,
                Parent = tpListFrame,
            })
            corner(btn, UDim.new(0, 6))
            btn.MouseButton1Click:Connect(function()
                selectedPlayer = plr
                tpButton.Text = "🎯 " .. plr.Name .. " ▼"
                tpListFrame.Visible = false
            end)
        end
    end
end

tpButton.MouseButton1Click:Connect(function()
    tpListFrame.Visible = not tpListFrame.Visible
    if tpListFrame.Visible then refreshPlayerList() end
end)

local tpGoBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(70, 130, 200),
    Text = "🌀 Teleportar até o Jogador",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 12,
    Parent = tpCard,
})
corner(tpGoBtn, UDim.new(0, 8))

tpGoBtn.MouseButton1Click:Connect(function()
    if not selectedPlayer then
        tpGoBtn.Text = "⚠ Selecione um jogador!"
        task.delay(1.5, function() tpGoBtn.Text = "🌀 Teleportar até o Jogador" end)
        return
    end
    local targetChar = selectedPlayer.Character
    if not targetChar then
        tpGoBtn.Text = "⚠ Jogador sem personagem!"
        task.delay(1.5, function() tpGoBtn.Text = "🌀 Teleportar até o Jogador" end)
        return
    end
    local targetHRP = targetChar:FindFirstChild("HumanoidRootPart")
    local myChar = LocalPlayer.Character
    local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if targetHRP and myHRP then
        myHRP.CFrame = targetHRP.CFrame * CFrame.new(0, 0, 3)
        tpGoBtn.Text = "✅ Teleportado!"
        task.delay(1.2, function() tpGoBtn.Text = "🌀 Teleportar até o Jogador" end)
    else
        tpGoBtn.Text = "⚠ Sem HumanoidRootPart!"
        task.delay(1.5, function() tpGoBtn.Text = "🌀 Teleportar até o Jogador" end)
    end
end)

-- ============================================================
-- CARD: FLY
-- ============================================================
local flyCard = createCard("✈ Voo", "Joystick = direção | ▲▼ = subir/descer")
createToggle(flyCard, "Ativar Voo", false, function(v)
    State.Fly = v
    if v then startFly() else stopFly() end
end)
createSlider(flyCard, "Velocidade de Voo", CONFIG.FlySpeedMin, CONFIG.FlySpeedMax, CONFIG.FlySpeedDefault, function(v)
    State.FlySpeed = v
end)

-- ============================================================
-- CARD: UTILIDADES
-- ============================================================
local utilCard = createCard("🛠 Utilidades", "Anti-AFK | Auto-Clicker")

createToggle(utilCard, "🛡 Anti-AFK", false, function(v)
    State.AntiAFK = v
end)

createToggle(utilCard, "🖱 Auto-Clicker (mobile OK)", false, function(v)
    State.AutoClicker = v
    if v then startAutoClicker() else stopAutoClicker() end
end)

createSlider(utilCard, "Auto-Clicker CPS", 1, 50, 15, function(v)
    State.AutoClickerCPS = v
end)

-- ============================================================
-- CARD: AIM
-- ============================================================
local DEFAULT_AIMBTN_POS = UDim2.new(1, -140, 0.5, -30)

local AimButton = create("TextButton", {
    Name = "AimButton",
    Size = UDim2.fromOffset(120, 120),
    Position = deserializeUDim2(savedConfig.aimBtnPos) or DEFAULT_AIMBTN_POS,
    BackgroundColor3 = CONFIG.UI_AimBtn,
    BackgroundTransparency = 1 - (savedConfig.aimBtnOpacity or State.AimBtnOpacity),
    Text = "💢",
    TextSize = 52,
    Font = Enum.Font.GothamBold,
    TextColor3 = Color3.fromRGB(255, 255, 255),
    AutoButtonColor = false,
    Active = true,
    Parent = ScreenGui,
})
corner(AimButton, UDim.new(1, 0))
stroke(AimButton, Color3.fromRGB(255, 255, 255), 2)

if savedConfig.aimBtnVisible ~= nil then
    AimButton.Visible = savedConfig.aimBtnVisible
end

local aimCard = createCard("🎯 Mira Assistida", "Segure o 💢 → mira gruda")

createToggle(aimCard, "Mostrar Botão 💢", true, function(v)
    AimButton.Visible = v
    savedConfig.aimBtnVisible = v
    safeSaveToFile(savedConfig)
end)

createSlider(aimCard, "Opacidade do Botão 💢", 0.1, 1, State.AimBtnOpacity, function(v)
    State.AimBtnOpacity = v
    AimButton.BackgroundTransparency = 1 - v
    savedConfig.aimBtnOpacity = v
    safeSaveToFile(savedConfig)
end)

createToggle(aimCard, "Ativar Aim Assist", false, function(v)
    State.AimEnabled = v
    if not v then
        FOVCircle.Visible = false
    else
        FOVCircle.Visible = State.ShowFOV
    end
end)

createToggle(aimCard, "Mostrar Círculo FOV", false, function(v)
    State.ShowFOV = v
    FOVCircle.Visible = v and State.AimEnabled
end)

createSlider(aimCard, "Tamanho do FOV", 50, 600, 150, function(v)
    State.FOV = v
    FOVCircle.Size = UDim2.fromOffset(v * 2, v * 2)
end)

createToggle(aimCard, "Ignorar Mesmo Time", false, function(v)
    State.TeamCheck = v
end)

local aimMoveStatus = create("TextLabel", {
    Size = UDim2.new(1, 0, 0, 20),
    BackgroundTransparency = 1,
    Text = "Mover 💢: 🔒 Fixo",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 13,
    Font = Enum.Font.GothamBold,
    TextXAlignment = Enum.TextXAlignment.Left,
    LayoutOrder = 12,
    Parent = aimCard,
})
local aimMoveBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "🎯 Ativar Modo Mover o Botão",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 13,
    Parent = aimCard,
})
corner(aimMoveBtn, UDim.new(0, 8))
stroke(aimMoveBtn, CONFIG.UI_Border, 1)
local aimSaveBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "💾 Salvar Posição do Botão",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 14,
    Parent = aimCard,
})
corner(aimSaveBtn, UDim.new(0, 8))
stroke(aimSaveBtn, CONFIG.UI_Border, 1)
local aimResetBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "↺ Redefinir posição do botão",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 15,
    Parent = aimCard,
})
corner(aimResetBtn, UDim.new(0, 8))
stroke(aimResetBtn, CONFIG.UI_Border, 1)

-- ============================================================
-- AIM LOGIC
-- ============================================================
local aiming = false
local currentTarget = nil
local targetHighlight = nil

local function isSameTeam(plr)
    if not State.TeamCheck then return false end
    if not LocalPlayer.Team or not plr.Team then return false end
    return LocalPlayer.Team == plr.Team
end

local function getTargetPart(char)
    return char:FindFirstChild("Head")
        or char:FindFirstChild("UpperTorso")
        or char:FindFirstChild("Torso")
        or char:FindFirstChild("HumanoidRootPart")
end

local function screenDistFromCenter(worldPos, cam)
    local center = cam.ViewportSize / 2
    local sp, onScreen = cam:WorldToViewportPoint(worldPos)
    if not onScreen then return math.huge end
    return (Vector2.new(sp.X, sp.Y) - center).Magnitude
end

local function isValidTarget(t)
    if not t then return false end
    local c = t.char
    if not c or not c.Parent then return false end
    local h = c:FindFirstChildOfClass("Humanoid")
    if not h or h.Health <= 0 then return false end
    local p = t.part
    if not p or not p.Parent then return false end
    return true
end

local function findBestTarget()
    local cam = workspace.CurrentCamera
    if not cam then return nil end
    local best, bestDist = nil, State.FOV
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        if isSameTeam(plr) then continue end
        local char = plr.Character
        if not char then continue end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health <= 0 then continue end
        local part = getTargetPart(char)
        if not part then continue end
        local dist = screenDistFromCenter(part.Position, cam)
        if dist < bestDist then
            best = {player = plr, part = part, char = char, dist = dist}
            bestDist = dist
        end
    end
    return best
end

local function clearTargetHighlight()
    if targetHighlight then
        pcall(function() targetHighlight:Destroy() end)
        targetHighlight = nil
    end
end

local function setTargetHighlight(char)
    clearTargetHighlight()
    if not char or not char.Parent then return end
    local existing = char:FindFirstChild("AIM_TargetHighlight")
    if existing then existing:Destroy() end
    targetHighlight = create("Highlight", {
        Name = "AIM_TargetHighlight",
        Adornee = char,
        FillColor = Color3.fromRGB(255, 40, 40),
        FillTransparency = 0.5,
        OutlineColor = Color3.fromRGB(255, 0, 0),
        OutlineTransparency = 0,
        DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
        Parent = char,
    })
end

local aimBtnDragging = false
local aimBtnDragStart, aimBtnStartPos
local aimBtnMoveMode = false

AimButton.InputBegan:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseButton1
    and input.UserInputType ~= Enum.UserInputType.Touch then return end
    aimBtnDragStart = input.Position
    aimBtnStartPos = AimButton.Position
    if aimBtnMoveMode then
        aimBtnDragging = true
        return
    end
    if State.AimEnabled then aiming = true end
    TweenService:Create(AimButton, TweenInfo.new(0.1), {
        BackgroundColor3 = CONFIG.UI_AimBtnHold
    }):Play()
end)

AimButton.InputChanged:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseMovement
    and input.UserInputType ~= Enum.UserInputType.Touch then return end
    if aimBtnMoveMode and aimBtnDragging then
        local delta = input.Position - aimBtnDragStart
        AimButton.Position = UDim2.new(
            aimBtnStartPos.X.Scale, aimBtnStartPos.X.Offset + delta.X,
            aimBtnStartPos.Y.Scale, aimBtnStartPos.Y.Offset + delta.Y
        )
    end
end)

AimButton.InputEnded:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseButton1
    and input.UserInputType ~= Enum.UserInputType.Touch then return end
    if aimBtnMoveMode then
        aimBtnDragging = false
        return
    end
    aiming = false
    currentTarget = nil
    clearTargetHighlight()
    TweenService:Create(AimButton, TweenInfo.new(0.15), {
        BackgroundColor3 = CONFIG.UI_AimBtn
    }):Play()
end)

RunService.RenderStepped:Connect(function(dt)
    if not State.AimEnabled or not aiming then
        if currentTarget then
            clearTargetHighlight()
            currentTarget = nil
        end
        return
    end
    local cam = workspace.CurrentCamera
    if not cam then return end
    if currentTarget and isValidTarget(currentTarget) then
        local d = screenDistFromCenter(currentTarget.part.Position, cam)
        if d > State.FOV then
            clearTargetHighlight()
            currentTarget = nil
        end
    else
        if currentTarget then clearTargetHighlight() end
        currentTarget = nil
    end
    if not currentTarget then
        currentTarget = findBestTarget()
        if currentTarget then setTargetHighlight(currentTarget.char) end
    end
    if not currentTarget then return end
    local targetPart = currentTarget.part
    if not targetPart or not targetPart.Parent then
        clearTargetHighlight()
        currentTarget = nil
        return
    end
    local camPos = cam.CFrame.Position
    local dir = (targetPart.Position - camPos)
    if dir.Magnitude < 0.1 then return end
    local targetCF = CFrame.new(camPos, camPos + dir.Unit)
    cam.CFrame = targetCF
end)

aimMoveBtn.MouseButton1Click:Connect(function()
    aimBtnMoveMode = not aimBtnMoveMode
    if aimBtnMoveMode then
        aimMoveBtn.Text = "🔒 Fixar Botão 💢"
        aimMoveBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        aimMoveStatus.Text = "Mover 💢: 🎯 Livre"
        stroke(AimButton, Color3.fromRGB(120, 180, 255), 3)
    else
        aimMoveBtn.Text = "🎯 Ativar Modo Mover o Botão"
        aimMoveBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
        aimMoveStatus.Text = "Mover 💢: 🔒 Fixo"
        stroke(AimButton, Color3.fromRGB(255, 255, 255), 2)
    end
end)

aimSaveBtn.MouseButton1Click:Connect(function()
    savedConfig.aimBtnPos = serializeUDim2(AimButton.Position)
    safeSaveToFile(savedConfig)
    aimSaveBtn.Text = "✅ Salvo!"
    aimSaveBtn.BackgroundColor3 = Color3.fromRGB(70, 200, 120)
    aimMoveStatus.Text = "Mover 💢: 🔒 Fixo — Salvo!"
    task.delay(1.2, function()
        aimSaveBtn.Text = "💾 Salvar Posição do Botão"
        aimSaveBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    end)
    aimBtnMoveMode = false
    aimMoveBtn.Text = "🎯 Ativar Modo Mover o Botão"
    aimMoveBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    stroke(AimButton, Color3.fromRGB(255, 255, 255), 2)
end)

aimResetBtn.MouseButton1Click:Connect(function()
    savedConfig.aimBtnPos = nil
    safeSaveToFile(savedConfig)
    AimButton.Position = DEFAULT_AIMBTN_POS
    aimResetBtn.Text = "↺ Redefinido!"
    task.delay(1, function() aimResetBtn.Text = "↺ Redefinir posição do botão" end)
end)

-- ============================================================
-- EDIT FLY CARD
-- ============================================================
local editCard = createCard("✏ Editar Botões de Voo", "Mova, salve e fixe os botões ▲▼")
local editStatus = create("TextLabel", {
    Size = UDim2.new(1, 0, 0, 20),
    BackgroundTransparency = 1,
    Text = "Modo: 🔒 Fixo",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 13,
    Font = Enum.Font.GothamBold,
    TextXAlignment = Enum.TextXAlignment.Left,
    LayoutOrder = 12,
    Parent = editCard,
})
local modeBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "🎯 Ativar Modo Mover",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 13,
    Parent = editCard,
})
corner(modeBtn, UDim.new(0, 8))
stroke(modeBtn, CONFIG.UI_Border, 1)
local saveBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "💾 Salvar Posição Atual",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 14,
    Parent = editCard,
})
corner(saveBtn, UDim.new(0, 8))
stroke(saveBtn, CONFIG.UI_Border, 1)
local resetBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "↺ Redefinir para Padrão",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 15,
    Parent = editCard,
})
corner(resetBtn, UDim.new(0, 8))
stroke(resetBtn, CONFIG.UI_Border, 1)

-- ============================================================
-- MOVE TOGGLE CARD
-- ============================================================
local toggleMoveCard = createCard("🔘 Mover Botão ≡", "Arraste o botão redondo")
local toggleMoveStatus = create("TextLabel", {
    Size = UDim2.new(1, 0, 0, 20),
    BackgroundTransparency = 1,
    Text = "Modo: 🔒 Fixo",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 13,
    Font = Enum.Font.GothamBold,
    TextXAlignment = Enum.TextXAlignment.Left,
    LayoutOrder = 12,
    Parent = toggleMoveCard,
})
local toggleMoveBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "🎯 Ativar Modo Mover o ≡",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 13,
    Parent = toggleMoveCard,
})
corner(toggleMoveBtn, UDim.new(0, 8))
stroke(toggleMoveBtn, CONFIG.UI_Border, 1)
local toggleSaveBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "💾 Salvar Posição do ≡",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 14,
    Parent = toggleMoveCard,
})
corner(toggleSaveBtn, UDim.new(0, 8))
stroke(toggleSaveBtn, CONFIG.UI_Border, 1)
local toggleResetBtn = create("TextButton", {
    Size = UDim2.new(1, 0, 0, 40),
    BackgroundColor3 = Color3.fromRGB(35, 35, 40),
    Text = "↺ Redefinir posição do ≡",
    TextColor3 = CONFIG.UI_Text,
    TextSize = 14,
    Font = Enum.Font.GothamBold,
    AutoButtonColor = false,
    LayoutOrder = 15,
    Parent = toggleMoveCard,
})
corner(toggleResetBtn, UDim.new(0, 8))
stroke(toggleResetBtn, CONFIG.UI_Border, 1)

-- ============================================================
-- OPEN/CLOSE
-- ============================================================
local uiOpen = false
local function setUIOpen(open)
    uiOpen = open
    if open then
        MainFrame.Visible = true
        MainFrame.BackgroundTransparency = 1
        TweenService:Create(MainFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {BackgroundTransparency = 0}):Play()
    else
        local t = TweenService:Create(MainFrame, TweenInfo.new(0.2), {BackgroundTransparency = 1})
        t:Play()
        t.Completed:Connect(function()
            if not uiOpen then MainFrame.Visible = false end
        end)
    end
end

local toggleMoveMode = false
local toggleDragging = false

ToggleButton.InputBegan:Connect(function(input)
    if input.UserInputType ~= Enum.UserInputType.MouseButton1
    and input.UserInputType ~= Enum.UserInputType.Touch then return end
    if not toggleMoveMode then return end
    toggleDragging = true
    TweenService:Create(ToggleButton, TweenInfo.new(0.15), {
        BackgroundColor3 = Color3.fromRGB(60, 60, 90)
    }):Play()
    local connMoved, connEnded
    connMoved = UserInputService.InputChanged:Connect(function(m)
        if not toggleDragging then return end
        if m.UserInputType ~= Enum.UserInputType.MouseMovement
        and m.UserInputType ~= Enum.UserInputType.Touch then return end
        local mousePos = m.Position
        ToggleButton.Position = UDim2.new(0, mousePos.X - 28, 0, mousePos.Y - 28)
    end)
    connEnded = UserInputService.InputEnded:Connect(function(m)
        if m.UserInputType ~= Enum.UserInputType.MouseButton1
        and m.UserInputType ~= Enum.UserInputType.Touch then return end
        toggleDragging = false
        TweenService:Create(ToggleButton, TweenInfo.new(0.15), {
            BackgroundColor3 = Color3.fromRGB(20, 20, 24)
        }):Play()
        if connMoved then connMoved:Disconnect() end
        if connEnded then connEnded:Disconnect() end
    end)
end)

ToggleButton.MouseButton1Click:Connect(function()
    if toggleMoveMode then return end
    setUIOpen(not uiOpen)
end)

toggleMoveBtn.MouseButton1Click:Connect(function()
    toggleMoveMode = not toggleMoveMode
    if toggleMoveMode then
        toggleMoveBtn.Text = "🔒 Fixar botão ≡"
        toggleMoveBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        toggleMoveStatus.Text = "Modo: 🎯 Livre"
        stroke(ToggleButton, Color3.fromRGB(120, 180, 255), 3)
    else
        toggleMoveBtn.Text = "🎯 Ativar Modo Mover o ≡"
        toggleMoveBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
        toggleMoveStatus.Text = "Modo: 🔒 Fixo"
        stroke(ToggleButton, Color3.fromRGB(80, 80, 90), 2)
    end
end)

toggleSaveBtn.MouseButton1Click:Connect(function()
    savedConfig.togglePos = serializeUDim2(ToggleButton.Position)
    safeSaveToFile(savedConfig)
    toggleSaveBtn.Text = "✅ Salvo!"
    toggleSaveBtn.BackgroundColor3 = Color3.fromRGB(70, 200, 120)
    toggleMoveStatus.Text = "Modo: 🔒 Fixo — Posição salva!"
    task.delay(1.2, function()
        toggleSaveBtn.Text = "💾 Salvar Posição do ≡"
        toggleSaveBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    end)
    toggleMoveMode = false
    toggleMoveBtn.Text = "🎯 Ativar Modo Mover o ≡"
    toggleMoveBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    stroke(ToggleButton, Color3.fromRGB(80, 80, 90), 2)
end)

toggleResetBtn.MouseButton1Click:Connect(function()
    savedConfig.togglePos = nil
    safeSaveToFile(savedConfig)
    ToggleButton.Position = DEFAULT_TOGGLE_POS
    toggleResetBtn.Text = "↺ Redefinido!"
    task.delay(1, function() toggleResetBtn.Text = "↺ Redefinir posição do ≡" end)
end)

CloseButton.MouseButton1Click:Connect(function() setUIOpen(false) end)

do
    local dragging, dragStart, startPos
    local function beginDrag(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            local ok, hit = pcall(function()
                return ScreenGui:GetGuiObjectsAtPosition(input.Position.X, input.Position.Y)
            end)
            if ok and hit then
                for _, obj in ipairs(hit) do
                    if obj == CloseButton or obj:IsDescendantOf(CloseButton) then return end
                end
            end
            dragging = true
            dragStart = input.Position
            startPos = MainFrame.Position
            MainFrame.AnchorPoint = Vector2.new(0.5, 0.5)
        end
    end
    local function moveDrag(input)
        if not dragging then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
        and input.UserInputType ~= Enum.UserInputType.Touch then return end
        local d = input.Position - dragStart
        MainFrame.Position = UDim2.new(
            startPos.X.Scale, startPos.X.Offset + d.X,
            startPos.Y.Scale, startPos.Y.Offset + d.Y
        )
    end
    local function endDrag(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end
    Header.InputBegan:Connect(beginDrag)
    addConn(UserInputService.InputChanged:Connect(moveDrag))
    addConn(UserInputService.InputEnded:Connect(endDrag))
end

addConn(UserInputService.JumpRequest:Connect(function()
    if not State.InfiniteJump then return end
    local hum = getHumanoid()
    if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
end))

-- ============================================================
-- NOCLIP
-- ============================================================
local noclipConn
function applyNoclip(enabled)
    if enabled then
        if noclipConn then noclipConn:Disconnect() end
        noclipConn = RunService.Stepped:Connect(function()
            if not State.Noclip then return end
            local c = LocalPlayer.Character
            if not c then return end
            for _, part in ipairs(c:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide then
                    part.CanCollide = false
                end
            end
        end)
    else
        if noclipConn then noclipConn:Disconnect(); noclipConn = nil end
        local c = LocalPlayer.Character
        if c then
            for _, part in ipairs(c:GetDescendants()) do
                if part:IsA("BasePart") then
                    if part.Name == "HumanoidRootPart" then
                        part.CanCollide = false
                    elseif part.Parent and part.Parent:IsA("Accessory") then
                        part.CanCollide = false
                    else
                        part.CanCollide = true
                    end
                end
            end
        end
    end
end

-- ============================================================
-- ESP
-- ============================================================
local ESPFolder
local espConnections = {}
local ESP_BLUE_FILL = Color3.fromRGB(60, 140, 255)
local ESP_BLUE_OUT = Color3.fromRGB(160, 210, 255)

function enableESP()
    if ESPFolder then ESPFolder:Destroy() end
    ESPFolder = create("Folder", {Name = "ESP_Folder", Parent = ScreenGui})
    local function attach(plr, char)
        if plr == LocalPlayer or not char or not State.ESP then return end
        local head = char:FindFirstChild("Head") or char:WaitForChild("Head", 3)
        if not head then return end
        local billboard = create("BillboardGui", {
            Name = "ESP_Billboard",
            Size = UDim2.fromOffset(220, 44),
            StudsOffset = Vector3.new(0, 3.2, 0),
            AlwaysOnTop = true,
            Parent = head,
        })
        local label = create("TextLabel", {
            Size = UDim2.fromScale(1, 1),
            BackgroundTransparency = 1,
            Text = plr.Name,
            TextColor3 = ESP_BLUE_OUT,
            TextStrokeTransparency = 0,
            TextStrokeColor3 = Color3.new(0, 0, 0),
            TextSize = 14,
            Font = Enum.Font.GothamBold,
            Parent = billboard,
        })
        create("Highlight", {
            Name = "ESP_Highlight",
            FillColor = ESP_BLUE_FILL,
            FillTransparency = 0.35,
            OutlineColor = ESP_BLUE_OUT,
            OutlineTransparency = 0,
            DepthMode = Enum.HighlightDepthMode.AlwaysOnTop,
            Parent = char,
        })
        local distConn
        distConn = RunService.RenderStepped:Connect(function()
            if not State.ESP or not char.Parent or not head.Parent then
                if distConn then distConn:Disconnect() end
                return
            end
            local myChar = LocalPlayer.Character
            local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
            if not myHRP then return end
            local dist = (myHRP.Position - head.Position).Magnitude
            label.Text = string.format("%s  [%.0f studs]", plr.Name, dist)
        end)
        table.insert(espConnections, distConn)
    end
    local function setupFor(plr)
        if plr == LocalPlayer then return end
        if plr.Character then attach(plr, plr.Character) end
        table.insert(espConnections, plr.CharacterAdded:Connect(function(char)
            if State.ESP then attach(plr, char) end
        end))
    end
    for _, plr in ipairs(Players:GetPlayers()) do setupFor(plr) end
    table.insert(espConnections, Players.PlayerAdded:Connect(function(plr)
        if State.ESP then setupFor(plr) end
    end))
end

function disableESP()
    for _, c in ipairs(espConnections) do
        if typeof(c) == "RBXScriptConnection" and c.Connected then c:Disconnect() end
    end
    table.clear(espConnections)
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr.Character then
            for _, d in ipairs(plr.Character:GetDescendants()) do
                if d.Name == "ESP_Billboard" or d.Name == "ESP_Highlight" then d:Destroy() end
            end
        end
    end
    if ESPFolder then ESPFolder:Destroy(); ESPFolder = nil end
end

-- ============================================================
-- FLY (v6 - joystick mobile funciona + sem vai-e-vem)
-- ============================================================
local flyConn
local flyKeys = {W=false, A=false, S=false, D=false, Up=false, Down=false}
local mobileFlyUI
local flyBtnUp, flyBtnDown
local flySavedCollision = {}
local moveMode = false

local DEFAULT_UP_POS = UDim2.new(1, -110, 0.5, -90)
local DEFAULT_DOWN_POS = UDim2.new(1, -110, 0.5, 10)

local function applySavedPositions()
    if not flyBtnUp or not flyBtnDown then return end
    flyBtnUp.Position = deserializeUDim2(savedConfig.flyUp) or DEFAULT_UP_POS
    flyBtnDown.Position = deserializeUDim2(savedConfig.flyDown) or DEFAULT_DOWN_POS
end

local function makeDraggable(btn, key)
    local dragging = false
    local dragStart, startPos
    local function startHolding()
        flyKeys[key] = true
        TweenService:Create(btn, TweenInfo.new(0.08), {
            BackgroundColor3 = Color3.fromRGB(150, 150, 160)
        }):Play()
    end
    local function stopHolding()
        flyKeys[key] = false
        TweenService:Create(btn, TweenInfo.new(0.08), {
            BackgroundColor3 = CONFIG.UI_FlyBtn
        }):Play()
    end
    btn.InputBegan:Connect(function(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1
        and input.UserInputType ~= Enum.UserInputType.Touch then return end
        dragging = false
        dragStart = input.Position
        startPos = btn.Position
        if not moveMode then startHolding() end
        local moveConn
        moveConn = UserInputService.InputChanged:Connect(function(m)
            if m.UserInputType ~= Enum.UserInputType.MouseMovement
            and m.UserInputType ~= Enum.UserInputType.Touch then return end
            if not moveMode then return end
            local delta = m.Position - dragStart
            if not dragging and delta.Magnitude > 3 then dragging = true end
            if dragging then
                btn.Position = UDim2.new(
                    startPos.X.Scale, startPos.X.Offset + delta.X,
                    startPos.Y.Scale, startPos.Y.Offset + delta.Y
                )
            end
        end)
        local endConn
        endConn = UserInputService.InputEnded:Connect(function(m)
            if m.UserInputType ~= Enum.UserInputType.MouseButton1
            and m.UserInputType ~= Enum.UserInputType.Touch then return end
            if not moveMode then stopHolding() end
            dragging = false
            if moveConn then moveConn:Disconnect() end
            if endConn then endConn:Disconnect() end
        end)
    end)
end

local function buildMobileFlyUI()
    local container = create("Frame", {
        Name = "MobileFlyControls",
        Size = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Visible = false,
        Parent = ScreenGui,
    })
    flyBtnUp = create("TextButton", {
        Name = "FlyButtonUp",
        Size = UDim2.fromOffset(80, 80),
        BackgroundColor3 = CONFIG.UI_FlyBtn,
        Text = "▲",
        TextColor3 = Color3.fromRGB(20, 20, 20),
        TextSize = 30,
        Font = Enum.Font.GothamBold,
        AutoButtonColor = false,
        Parent = container,
    })
    corner(flyBtnUp, UDim.new(1, 0))
    stroke(flyBtnUp, Color3.fromRGB(120, 120, 130), 2)
    flyBtnDown = create("TextButton", {
        Name = "FlyButtonDown",
        Size = UDim2.fromOffset(80, 80),
        BackgroundColor3 = CONFIG.UI_FlyBtn,
        Text = "▼",
        TextColor3 = Color3.fromRGB(20, 20, 20),
        TextSize = 30,
        Font = Enum.Font.GothamBold,
        AutoButtonColor = false,
        Parent = container,
    })
    corner(flyBtnDown, UDim.new(1, 0))
    stroke(flyBtnDown, Color3.fromRGB(120, 120, 130), 2)
    makeDraggable(flyBtnUp, "Up")
    makeDraggable(flyBtnDown, "Down")
    applySavedPositions()
    return container
end

if IS_MOBILE then mobileFlyUI = buildMobileFlyUI() end

modeBtn.MouseButton1Click:Connect(function()
    moveMode = not moveMode
    if moveMode then
        modeBtn.Text = "🔒 Fixar posição"
        modeBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        editStatus.Text = "Modo: 🎯 Livre"
    else
        modeBtn.Text = "🎯 Ativar Modo Mover"
        modeBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
        editStatus.Text = "Modo: 🔒 Fixo"
    end
end)

saveBtn.MouseButton1Click:Connect(function()
    if not flyBtnUp or not flyBtnDown then
        editStatus.Text = "⚠ Ative o Voo primeiro para editar"
        return
    end
    savedConfig.flyUp = serializeUDim2(flyBtnUp.Position)
    savedConfig.flyDown = serializeUDim2(flyBtnDown.Position)
    safeSaveToFile(savedConfig)
    saveBtn.Text = "✅ Salvo!"
    saveBtn.BackgroundColor3 = Color3.fromRGB(70, 200, 120)
    editStatus.Text = "Modo: 🔒 Fixo — Posição salva!"
    task.delay(1.2, function()
        saveBtn.Text = "💾 Salvar Posição Atual"
        saveBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    end)
    moveMode = false
    modeBtn.Text = "🎯 Ativar Modo Mover"
    modeBtn.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
end)

resetBtn.MouseButton1Click:Connect(function()
    savedConfig.flyUp = nil
    savedConfig.flyDown = nil
    safeSaveToFile(savedConfig)
    if flyBtnUp then flyBtnUp.Position = DEFAULT_UP_POS end
    if flyBtnDown then flyBtnDown.Position = DEFAULT_DOWN_POS end
    resetBtn.Text = "↺ Redefinido!"
    task.delay(1, function() resetBtn.Text = "↺ Redefinir para Padrão" end)
end)

local function setCharacterCollision(char, enabled)
    if not char then return end
    if enabled then
        for part, v in pairs(flySavedCollision) do
            if part and part.Parent then part.CanCollide = v end
        end
        table.clear(flySavedCollision)
    else
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                if flySavedCollision[part] == nil then flySavedCollision[part] = part.CanCollide end
                part.CanCollide = false
            end
        end
    end
end

function stopFly()
    if flyConn then flyConn:Disconnect(); flyConn = nil end
    if mobileFlyUI then mobileFlyUI.Visible = false end
    flyKeys.W, flyKeys.A, flyKeys.S, flyKeys.D, flyKeys.Up, flyKeys.Down = false, false, false, false, false, false
    local char = LocalPlayer.Character
    local hum = getHumanoid()
    if hum then
        hum.PlatformStand = false
        hum.AutoRotate = true
        hum:SetStateEnabled(Enum.HumanoidStateType.Freefall, true)
        hum:SetStateEnabled(Enum.HumanoidStateType.Landed, true)
        hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, true)
        hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, true)
        hum:SetStateEnabled(Enum.HumanoidStateType.PlatformStanding, true)
        hum:SetStateEnabled(Enum.HumanoidStateType.GettingUp, true)
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
    end
    setCharacterCollision(char, true)
end

function startFly()
    local hum, hrp = getHumanoid()
    if not hum or not hrp then return end
    hum.AutoRotate = false
    hum.PlatformStand = true
    hum:SetStateEnabled(Enum.HumanoidStateType.Freefall, false)
    hum:SetStateEnabled(Enum.HumanoidStateType.Landed, false)
    hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
    hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
    hum:SetStateEnabled(Enum.HumanoidStateType.PlatformStanding, false)
    hum:SetStateEnabled(Enum.HumanoidStateType.GettingUp, false)
    hrp.AssemblyLinearVelocity = Vector3.zero
    hrp.AssemblyAngularVelocity = Vector3.zero
    setCharacterCollision(LocalPlayer.Character, false)

    if mobileFlyUI then mobileFlyUI.Visible = true end

    -- Variável para suavizar o MoveDirection (evita tremor/vai-e-vem)
    local smoothMove = Vector3.zero

    flyConn = RunService.RenderStepped:Connect(function(dt)
        if not State.Fly then return end

        local cHum, cHrp = getHumanoid()
        if not cHum or not cHrp or cHum.Health <= 0 then return end

        local cCam = workspace.CurrentCamera
        if not cCam then return end

        local camCF = cCam.CFrame
        local look = camCF.LookVector
        local right = camCF.RightVector

        local flatLook = Vector3.new(look.X, 0, look.Z)
        if flatLook.Magnitude < 0.01 then flatLook = Vector3.new(0, 0, -1) end
        flatLook = flatLook.Unit

        local flatRight = Vector3.new(right.X, 0, right.Z)
        if flatRight.Magnitude < 0.01 then flatRight = Vector3.new(1, 0, 0) end
        flatRight = flatRight.Unit

        -- Direção vinda das teclas (PC)
        local keyDir = Vector3.zero
        if flyKeys.W then keyDir += flatLook end
        if flyKeys.S then keyDir -= flatLook end
        if flyKeys.A then keyDir -= flatRight end
        if flyKeys.D then keyDir += flatRight end

        -- Direção vinda do joystick mobile (MoveDirection)
        local nativeDir = cHum.MoveDirection
        local flatNative = Vector3.new(nativeDir.X, 0, nativeDir.Z)

        local moveDir
        if flatNative.Magnitude > 0.05 then
            -- Joystick mobile está sendo usado
            -- Projeta o MoveDirection no referencial da câmera (frente/trás/lado)
            local normalized = flatNative.Unit
            local dotF = normalized:Dot(flatLook)
            local dotR = normalized:Dot(flatRight)
            moveDir = flatLook * dotF + flatRight * dotR
            if moveDir.Magnitude > 1 then moveDir = moveDir.Unit end
        elseif keyDir.Magnitude > 0 then
            moveDir = keyDir.Unit
        else
            moveDir = Vector3.zero
        end

        -- Suaviza o movimento (evita tremor/vai-e-vem)
        smoothMove = smoothMove:Lerp(moveDir, math.clamp(dt * 10, 0, 1))
        if smoothMove.Magnitude < 0.05 then
            smoothMove = Vector3.zero
        end

        local vertical = 0
        if flyKeys.Up then vertical += 1 end
        if flyKeys.Down then vertical -= 1 end

        -- Zera velocidade ANTES de mover
        cHrp.AssemblyLinearVelocity = Vector3.zero
        cHrp.AssemblyAngularVelocity = Vector3.zero

        -- Aplica movimento
        local delta = smoothMove * State.FlySpeed * dt
        delta += Vector3.new(0, vertical * State.FlySpeed * dt, 0)

        if delta.Magnitude > 0 then
            local newPos = cHrp.Position + delta
            cHrp.CFrame = CFrame.new(newPos) * CFrame.Angles(0, math.atan2(-flatLook.X, -flatLook.Z), 0)
        end
    end)
end

addConn(UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if not State.Fly then return end
    if input.KeyCode == Enum.KeyCode.W then flyKeys.W = true end
    if input.KeyCode == Enum.KeyCode.A then flyKeys.A = true end
    if input.KeyCode == Enum.KeyCode.S then flyKeys.S = true end
    if input.KeyCode == Enum.KeyCode.D then flyKeys.D = true end
    if input.KeyCode == Enum.KeyCode.Space then flyKeys.Up = true end
    if input.KeyCode == Enum.KeyCode.LeftShift then flyKeys.Down = true end
end))
addConn(UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.W then flyKeys.W = false end
    if input.KeyCode == Enum.KeyCode.A then flyKeys.A = false end
    if input.KeyCode == Enum.KeyCode.S then flyKeys.S = false end
    if input.KeyCode == Enum.KeyCode.D then flyKeys.D = false end
    if input.KeyCode == Enum.KeyCode.Space then flyKeys.Up = false end
    if input.KeyCode == Enum.KeyCode.LeftShift then flyKeys.Down = false end
end))

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1)
    if State.Noclip then applyNoclip(true) end
    if State.Fly then
        stopFly()
        task.wait(0.2)
        startFly()
    end
    local hum = getHumanoid()
    if hum then hum.WalkSpeed = State.WalkSpeed end
end)

addConn(UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.K then setUIOpen(not uiOpen) end
end))

-- ============================================================
-- ANTI-AFK
-- ============================================================
LocalPlayer.Idled:Connect(function()
    if not State.AntiAFK then return end
    VirtualUser:CaptureController()
    VirtualUser:ClickButton2(Vector2.new())
end)

-- ============================================================
-- AUTO-CLICKER MOBILE
-- ============================================================
local autoClickerRunning = false
local autoClickerThread = nil

function startAutoClicker()
    if autoClickerRunning then return end
    autoClickerRunning = true
    autoClickerThread = task.spawn(function()
        while State.AutoClicker and autoClickerRunning do
            local cam = workspace.CurrentCamera
            if cam then
                local center = cam.ViewportSize / 2
                pcall(function()
                    VirtualInputManager:SendMouseButtonEvent(
                        math.floor(center.X), math.floor(center.Y),
                        0, true, game, 0
                    )
                end)
                task.wait(0.01)
                pcall(function()
                    VirtualInputManager:SendMouseButtonEvent(
                        math.floor(center.X), math.floor(center.Y),
                        0, false, game, 0
                    )
                end)
            else
                pcall(function()
                    VirtualUser:CaptureController()
                    VirtualUser:Button1Down(Vector2.new())
                    task.wait(0.01)
                    VirtualUser:Button1Up(Vector2.new())
                end)
            end
            task.wait(1 / math.max(State.AutoClickerCPS, 1))
        end
    end)
end

function stopAutoClicker()
    autoClickerRunning = false
    if autoClickerThread then
        pcall(function() task.cancel(autoClickerThread) end)
        autoClickerThread = nil
    end
end

addConn(UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.R then
        State.AutoClicker = not State.AutoClicker
        if State.AutoClicker then startAutoClicker() else stopAutoClicker() end
    end
end))

print("[Painel Universal v6] Carregado! Fly com joystick mobile OK.")
