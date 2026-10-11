local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local HttpService = game:GetService("HttpService")
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

local targetParent = (gethui and gethui()) or game:GetService("CoreGui") or PlayerGui

if targetParent:FindFirstChild("jayceexe") then
    targetParent.jayceexe:Destroy()
end

local oldAtm = Lighting:FindFirstChild("pwdAtmosphere")
if oldAtm then oldAtm:Destroy() end

local ACCENT       = Color3.fromRGB(187, 204, 27)
local ACCENT_DARK  = Color3.fromRGB(140, 155, 15)
local ACCENT_LIGHT = Color3.fromRGB(210, 225, 60)
local TEXT_DARK    = Color3.fromRGB(25, 30, 0)
local WHITE        = Color3.fromRGB(255, 255, 255)

local Theme = {
    AccentColor      = ACCENT_DARK,
    DimColor         = ACCENT_DARK,
    HighlightColor   = WHITE,
    BgColor          = ACCENT,
    SectionBg        = ACCENT_LIGHT,
    TrayColor        = ACCENT_DARK,
    TextColor        = TEXT_DARK,
    DimTextColor     = Color3.fromRGB(60, 70, 5),
    BorderColor      = WHITE,
    DarkBg           = ACCENT_LIGHT,
    HeaderBg         = ACCENT_DARK,
    ButtonBg         = ACCENT_LIGHT,
}

local cfg = {
    silentAim = true,
    silentAimKey = Enum.KeyCode.V,
    wallCheck = true,
    hitChance = 100,
    fov = 150,
    maxDist = 1000,
    hitPart = "Head",
    closestPart = false,
    teamCheck = false,
    predX = 0,
    predY = 0,
    bypassRevolver = false,
    camlock = false,
    camlockKey = Enum.KeyCode.C,
    camlockSmooth = 6,
    camlockPart = "Head",
    fly = false,
    flyKey = Enum.KeyCode.F,
    flySpeed = 120,
    speed = false,
    speedKey = Enum.KeyCode.X,
    speedValue = 50,
    jumpValue = 50,
    autoSprint = false,
    autoSprintSpeed = 32,
    atmDensity = 0.3,
    atmHaze = 0,
    atmDecay = 0.1,
    brightness = Lighting.Brightness,
    clockTime = Lighting.ClockTime,
    fogEnabled = false,
    fogR = 145,
    fogG = 210,
    fogB = 240,
    fogStart = 0,
    fogEnd = 500,
    antiFall = false,
    antiAim = false,
    aimAcc0 = false,
    headless = false,
    avatarOn = false,
    avatarTarget = "",
    whitelistActive = true,
    whitelist = {},
    uiToggleKey = Enum.KeyCode.RightControl,
    uiVisible = true,
}

local atmos = Instance.new("Atmosphere")
atmos.Name = "pwdAtmosphere"
atmos.Parent = Lighting

local cachedPart = nil
local lockedPlayer = nil
local flyConn, bodyVel, bodyGyro, antiFallConn, antiAimConn = nil, nil, nil, nil, nil

local bodyParts = {
    "Head","UpperTorso","HumanoidRootPart","LowerTorso",
    "LeftUpperArm","RightUpperArm","LeftLowerArm","RightLowerArm",
    "LeftHand","RightHand","LeftUpperLeg","RightUpperLeg",
    "LeftLowerLeg","RightLowerLeg","LeftFoot","RightFoot"
}

local function isHoldingRevolver()
    if not cfg.bypassRevolver then return false end
    local char = LocalPlayer.Character
    if not char then return false end
    local tool = char:FindFirstChildOfClass("Tool")
    if tool then
        local n = string.lower(tool.Name)
        if string.find(n, "revolver") or string.find(n, "rev") then return true end
    end
    return false
end

local function isAlive(p)
    local h = p and p.Character and p.Character:FindFirstChildOfClass("Humanoid")
    return h and h.Health > 0
end

local function isWL(p)
    if not cfg.whitelistActive then return false end
    return cfg.whitelist[p.Name] == true
end

local function sameTeam(p)
    return LocalPlayer.Team and p.Team and LocalPlayer.Team == p.Team
end

local function wallBetween(pos)
    if not cfg.wallCheck then return false end
    local ro = Camera.CFrame.Position
    local rd = pos - ro
    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {LocalPlayer.Character}
    params.FilterType = Enum.RaycastFilterType.Exclude
    local hit = workspace:Raycast(ro, rd, params)
    if not hit then return false end
    for _, p in pairs(Players:GetPlayers()) do
        if p.Character and hit.Instance:IsDescendantOf(p.Character) then return false end
    end
    return true
end

local function safeW2S(pos)
    local sp, on = Vector3.new(), false
    pcall(function() sp, on = Camera:WorldToViewportPoint(pos) end)
    return sp, on
end

local function getClosestBodyPart(char)
    local closestPart, shortestDist = nil, math.huge
    local m = UIS:GetMouseLocation()
    for _, pName in ipairs(bodyParts) do
        local part = char:FindFirstChild(pName)
        if part then
            local sp, on = safeW2S(part.Position)
            local d = on and (Vector2.new(sp.X, sp.Y) - m).Magnitude or math.huge
            if d < shortestDist then
                shortestDist = d
                closestPart = part
            end
        end
    end
    return closestPart or char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
end

local function getTargetPart(char)
    if not char then return nil end
    if cfg.closestPart then return getClosestBodyPart(char) end
    return char:FindFirstChild(cfg.hitPart) or char:FindFirstChild("Head") or char:FindFirstChild("HumanoidRootPart")
end

local function getClosestPlayerToCursor()
    if isHoldingRevolver() then return nil end
    local closestPlayer, shortestDist = nil, cfg.fov
    local m = UIS:GetMouseLocation()
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and isAlive(p) and not isWL(p) then
            if cfg.teamCheck and sameTeam(p) then continue end
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                local d3 = (hrp.Position - Camera.CFrame.Position).Magnitude
                if d3 <= cfg.maxDist then
                    local sp, on = safeW2S(hrp.Position)
                    if on then
                        local d2 = (Vector2.new(sp.X, sp.Y) - m).Magnitude
                        if d2 < shortestDist then
                            local part = getTargetPart(p.Character)
                            if part and not wallBetween(part.Position) then
                                shortestDist = d2
                                closestPlayer = part
                            end
                        end
                    end
                end
            end
        end
    end
    return closestPlayer
end

local _grm = getrawmetatable(game)
local _oldIndex = _grm.__index
setreadonly(_grm, false)
_grm.__index = function(self, key)
    if not checkcaller() and self == Mouse and cfg.silentAim and not isHoldingRevolver() then
        if (key == "Hit" or key == "Target" or key == "UnitRay") and cachedPart then
            if math.random(1, 100) <= cfg.hitChance then
                local origin = Camera.CFrame.Position
                local hitPos = cachedPart.Position + Vector3.new(
                    cachedPart.AssemblyLinearVelocity.X * cfg.predX,
                    cachedPart.AssemblyLinearVelocity.Y * cfg.predY,
                    cachedPart.AssemblyLinearVelocity.Z * cfg.predX
                )
                if key == "UnitRay" then
                    return Ray.new(origin, (hitPos - origin).Unit)
                elseif key == "Hit" then
                    return CFrame.new(hitPos)
                elseif key == "Target" then
                    return cachedPart
                end
            end
        end
    end
    return _oldIndex(self, key)
end
setreadonly(_grm, true)

task.spawn(function()
    while task.wait(1) do
        if cfg.fogEnabled then
            local fc = Color3.fromRGB(cfg.fogR, cfg.fogG, cfg.fogB)
            for _, child in pairs(Lighting:GetChildren()) do
                if child:IsA("Atmosphere") then
                    pcall(function()
                        child.Color = fc
                        child.Density = cfg.atmDensity
                        child.Offset = 0
                        child.Glare = 0
                    end)
                end
            end
        end
    end
end)

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "jayceexe"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = targetParent

local WindowOuterBorder = Instance.new("Frame")
WindowOuterBorder.Name = "WindowOuterBorder"
WindowOuterBorder.Size = UDim2.new(0, 620, 0, 420)
WindowOuterBorder.Position = UDim2.new(0.5, -310, 0.5, -210)
WindowOuterBorder.BackgroundColor3 = Theme.TrayColor
WindowOuterBorder.BorderSizePixel = 0
WindowOuterBorder.Active = true
WindowOuterBorder.ClipsDescendants = true
WindowOuterBorder.Parent = ScreenGui

local OuterCorner = Instance.new("UICorner")
OuterCorner.CornerRadius = UDim.new(0, 10)
OuterCorner.Parent = WindowOuterBorder

local InnerSpacer = Instance.new("Frame")
InnerSpacer.Name = "InnerSpacer"
InnerSpacer.Size = UDim2.new(1, -2, 1, -2)
InnerSpacer.Position = UDim2.new(0, 1, 0, 1)
InnerSpacer.BackgroundColor3 = Theme.DarkBg
InnerSpacer.BorderSizePixel = 0
InnerSpacer.ClipsDescendants = true
InnerSpacer.Parent = WindowOuterBorder

local InnerCorner = Instance.new("UICorner")
InnerCorner.CornerRadius = UDim.new(0, 9)
InnerCorner.Parent = InnerSpacer

local MainBody = Instance.new("Frame")
MainBody.Name = "MainBody"
MainBody.Size = UDim2.new(1, -2, 1, -2)
MainBody.Position = UDim2.new(0, 1, 0, 1)
MainBody.BackgroundColor3 = Theme.BgColor
MainBody.BorderColor3 = Theme.BorderColor
MainBody.BorderSizePixel = 1
MainBody.ClipsDescendants = true
MainBody.Parent = InnerSpacer

local BodyCorner = Instance.new("UICorner")
BodyCorner.CornerRadius = UDim.new(0, 8)
BodyCorner.Parent = MainBody

local HeaderBar = Instance.new("Frame")
HeaderBar.Name = "HeaderBar"
HeaderBar.Size = UDim2.new(1, 0, 0, 32)
HeaderBar.BackgroundColor3 = Theme.HeaderBg
HeaderBar.BorderSizePixel = 0
HeaderBar.ZIndex = 5
HeaderBar.Parent = MainBody

local HeaderCorner = Instance.new("UICorner")
HeaderCorner.CornerRadius = UDim.new(0, 8)
HeaderCorner.Parent = HeaderBar

local KeroppiIcon = Instance.new("ImageLabel")
KeroppiIcon.Size = UDim2.new(0, 22, 0, 22)
KeroppiIcon.Position = UDim2.new(0, 8, 0.5, -11)
KeroppiIcon.BackgroundTransparency = 1
KeroppiIcon.Image = "rbxassetid://119330498492613"
KeroppiIcon.ScaleType = Enum.ScaleType.Fit
KeroppiIcon.ZIndex = 10
KeroppiIcon.Parent = HeaderBar

local TitleText = Instance.new("TextLabel")
TitleText.Name = "TitleText"
TitleText.Size = UDim2.new(0.5, 0, 1, 0)
TitleText.Position = UDim2.new(0, 34, 0, 0)
TitleText.BackgroundTransparency = 1
TitleText.Text = "jayce.exe"
TitleText.TextColor3 = WHITE
TitleText.TextSize = 13
TitleText.Font = Enum.Font.RobotoMono
TitleText.TextXAlignment = Enum.TextXAlignment.Left
TitleText.ZIndex = 6
TitleText.Parent = HeaderBar

local DateText = Instance.new("TextLabel")
DateText.Name = "DateText"
DateText.Size = UDim2.new(0.5, -15, 1, 0)
DateText.Position = UDim2.new(0.5, 0, 0, 0)
DateText.BackgroundTransparency = 1
DateText.Text = ""
DateText.TextColor3 = WHITE
DateText.TextSize = 11
DateText.Font = Enum.Font.RobotoMono
DateText.TextXAlignment = Enum.TextXAlignment.Right
DateText.ZIndex = 6
DateText.Parent = HeaderBar

local function updateClock()
    DateText.Text = os.date("%A, %d %B, %y")
end
updateClock()
task.spawn(function()
    while task.wait(60) do updateClock() end
end)

local AccentLine = Instance.new("Frame")
AccentLine.Name = "AccentLine"
AccentLine.Size = UDim2.new(1, 0, 0, 2)
AccentLine.Position = UDim2.new(0, 0, 1, 0)
AccentLine.BackgroundColor3 = WHITE
AccentLine.BorderSizePixel = 0
AccentLine.ZIndex = 10
AccentLine.Parent = HeaderBar

local dragging, dragInput, dragStart, startPos
local function updateDrag(input)
    local delta = input.Position - dragStart
    WindowOuterBorder.Position = UDim2.new(
        startPos.Width.Scale, startPos.Width.Offset + delta.X,
        startPos.Height.Scale, startPos.Height.Offset + delta.Y
    )
end

HeaderBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = WindowOuterBorder.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
HeaderBar.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)
UIS.InputChanged:Connect(function(input)
    if input == dragInput and dragging then updateDrag(input) end
end)

local LayoutContent = Instance.new("Frame")
LayoutContent.Name = "LayoutContent"
LayoutContent.Size = UDim2.new(1, 0, 1, -32)
LayoutContent.Position = UDim2.new(0, 0, 0, 32)
LayoutContent.BackgroundTransparency = 1
LayoutContent.ClipsDescendants = true
LayoutContent.ZIndex = 2
LayoutContent.Parent = MainBody

local Sidebar = Instance.new("Frame")
Sidebar.Name = "Sidebar"
Sidebar.Size = UDim2.new(0, 125, 1, 0)
Sidebar.BackgroundColor3 = Theme.SectionBg
Sidebar.BorderColor3 = Theme.BorderColor
Sidebar.BorderSizePixel = 0
Sidebar.ZIndex = 3
Sidebar.Parent = LayoutContent

local SidebarDivider = Instance.new("Frame")
SidebarDivider.Size = UDim2.new(0, 1, 1, 0)
SidebarDivider.Position = UDim2.new(1, -1, 0, 0)
SidebarDivider.BackgroundColor3 = WHITE
SidebarDivider.BorderSizePixel = 0
SidebarDivider.ZIndex = 4
SidebarDivider.Parent = Sidebar

local SidebarTitle = Instance.new("TextLabel")
SidebarTitle.Size = UDim2.new(1, -12, 0, 18)
SidebarTitle.Position = UDim2.new(0, 8, 0, 6)
SidebarTitle.BackgroundTransparency = 1
SidebarTitle.Text = "// tabs"
SidebarTitle.TextColor3 = TEXT_DARK
SidebarTitle.Font = Enum.Font.RobotoMono
SidebarTitle.TextSize = 11
SidebarTitle.TextXAlignment = Enum.TextXAlignment.Left
SidebarTitle.ZIndex = 5
SidebarTitle.Parent = Sidebar

local SidebarButtons = Instance.new("Frame")
SidebarButtons.Size = UDim2.new(1, -8, 1, -30)
SidebarButtons.Position = UDim2.new(0, 4, 0, 26)
SidebarButtons.BackgroundTransparency = 1
SidebarButtons.ZIndex = 4
SidebarButtons.Parent = Sidebar

local SidebarList = Instance.new("UIListLayout")
SidebarList.Padding = UDim.new(0, 3)
SidebarList.SortOrder = Enum.SortOrder.LayoutOrder
SidebarList.Parent = SidebarButtons

local Content = Instance.new("Frame")
Content.Name = "Content"
Content.Size = UDim2.new(1, -125, 1, 0)
Content.Position = UDim2.new(0, 125, 0, 0)
Content.BackgroundColor3 = Theme.BgColor
Content.BorderSizePixel = 0
Content.ZIndex = 3
Content.Parent = LayoutContent

local ContentPadding = Instance.new("UIPadding")
ContentPadding.PaddingTop = UDim.new(0, 12)
ContentPadding.PaddingLeft = UDim.new(0, 14)
ContentPadding.PaddingRight = UDim.new(0, 14)
ContentPadding.PaddingBottom = UDim.new(0, 12)
ContentPadding.Parent = Content

local pages = {}
local pageButtons = {}
local activePage = nil

local function createPage(name)
    local f = Instance.new("ScrollingFrame")
    f.Size = UDim2.new(1, 0, 1, 0)
    f.BackgroundTransparency = 1
    f.BorderSizePixel = 0
    f.ScrollBarThickness = 2
    f.ScrollBarImageColor3 = WHITE
    f.CanvasSize = UDim2.new(0, 0, 0, 0)
    f.AutomaticCanvasSize = Enum.AutomaticSize.Y
    f.Visible = false
    f.ZIndex = 3
    f.Parent = Content

    local l = Instance.new("UIListLayout")
    l.Padding = UDim.new(0, 5)
    l.SortOrder = Enum.SortOrder.LayoutOrder
    l.Parent = f

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 28)
    btn.BackgroundColor3 = Theme.ButtonBg
    btn.BorderColor3 = WHITE
    btn.BorderSizePixel = 1
    btn.Text = "  " .. name
    btn.TextColor3 = TEXT_DARK
    btn.Font = Enum.Font.RobotoMono
    btn.TextSize = 12
    btn.TextXAlignment = Enum.TextXAlignment.Left
    btn.AutoButtonColor = false
    btn.ZIndex = 4
    btn.Parent = SidebarButtons

    local bCorner = Instance.new("UICorner")
    bCorner.CornerRadius = UDim.new(0, 4)
    bCorner.Parent = btn

    btn.MouseButton1Click:Connect(function()
        for _, pg in pairs(pages) do pg.Frame.Visible = false end
        for _, b in pairs(pageButtons) do
            b.BackgroundColor3 = Theme.ButtonBg
            b.TextColor3 = TEXT_DARK
            b.BorderColor3 = WHITE
        end
        f.Visible = true
        btn.BackgroundColor3 = WHITE
        btn.TextColor3 = TEXT_DARK
        btn.BorderColor3 = WHITE
        activePage = name
    end)

    pages[name] = {Frame = f, Button = btn}
    table.insert(pageButtons, btn)

    if not activePage then
        f.Visible = true
        btn.BackgroundColor3 = WHITE
        btn.TextColor3 = TEXT_DARK
        btn.BorderColor3 = WHITE
        activePage = name
    end
    return f
end

local combatPage     = createPage("combat")
local camlockPage    = createPage("camlock")
local visualsPage    = createPage("visuals")
local whitelistPage  = createPage("whitelist")
local avatarPage     = createPage("avatar")
local miscPage       = createPage("misc")
local protectionPage = createPage("protection")
local settingsPage   = createPage("settings")

local function addLabel(parent, text)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 0, 16)
    l.BackgroundTransparency = 1
    l.Text = text
    l.TextColor3 = TEXT_DARK
    l.Font = Enum.Font.RobotoMono
    l.TextSize = 11
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = parent
end

local function addToggle(parent, text, default, callback)
    local Row = Instance.new("Frame")
    Row.Size = UDim2.new(1, 0, 0, 20)
    Row.BackgroundTransparency = 1
    Row.Parent = parent

    local Box = Instance.new("TextButton")
    Box.Size = UDim2.new(0, 12, 0, 12)
    Box.Position = UDim2.new(0, 0, 0.5, -6)
    Box.BackgroundColor3 = default and WHITE or Theme.ButtonBg
    Box.BorderColor3 = WHITE
    Box.BorderSizePixel = 1
    Box.Text = ""
    Box.AutoButtonColor = false
    Box.Parent = Row

    local Lbl = Instance.new("TextLabel")
    Lbl.Size = UDim2.new(1, -22, 1, 0)
    Lbl.Position = UDim2.new(0, 22, 0, 0)
    Lbl.BackgroundTransparency = 1
    Lbl.Text = text
    Lbl.TextColor3 = TEXT_DARK
    Lbl.TextSize = 11
    Lbl.Font = Enum.Font.RobotoMono
    Lbl.TextXAlignment = Enum.TextXAlignment.Left
    Lbl.Parent = Row

    local state = default
    local function set(newState)
        state = newState
        Box.BackgroundColor3 = state and WHITE or Theme.ButtonBg
        Box.BorderColor3 = WHITE
        Lbl.TextColor3 = TEXT_DARK
    end

    Box.MouseButton1Click:Connect(function()
        set(not state)
        callback(state)
    end)

    return set
end

local function addKeybind(parent, text, defaultKey, callback)
    local Row = Instance.new("Frame")
    Row.Size = UDim2.new(1, 0, 0, 22)
    Row.BackgroundTransparency = 1
    Row.Parent = parent

    local Lbl = Instance.new("TextLabel")
    Lbl.Size = UDim2.new(0, 120, 1, 0)
    Lbl.BackgroundTransparency = 1
    Lbl.Text = text
    Lbl.TextColor3 = TEXT_DARK
    Lbl.TextSize = 11
    Lbl.Font = Enum.Font.RobotoMono
    Lbl.TextXAlignment = Enum.TextXAlignment.Left
    Lbl.Parent = Row

    local BBtn = Instance.new("TextButton")
    BBtn.Size = UDim2.new(0, 100, 1, 0)
    BBtn.Position = UDim2.new(1, -100, 0, 0)
    BBtn.BackgroundColor3 = Theme.ButtonBg
    BBtn.BorderColor3 = WHITE
    BBtn.BorderSizePixel = 1
    BBtn.Text = "[" .. defaultKey.Name .. "]"
    BBtn.TextColor3 = TEXT_DARK
    BBtn.Font = Enum.Font.RobotoMono
    BBtn.TextSize = 11
    BBtn.AutoButtonColor = false
    BBtn.Parent = Row

    local bC = Instance.new("UICorner"); bC.CornerRadius = UDim.new(0, 4); bC.Parent = BBtn

    local listening = false
    BBtn.MouseButton1Click:Connect(function()
        if listening then return end
        listening = true
        BBtn.Text = "[...]"
        BBtn.TextColor3 = ACCENT_DARK
        local conn
        conn = UIS.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Keyboard then
                BBtn.Text = "[" .. input.KeyCode.Name .. "]"
                BBtn.TextColor3 = TEXT_DARK
                listening = false
                callback(input.KeyCode)
                conn:Disconnect()
            end
        end)
    end)
end

local function addSlider(parent, text, min, max, default, callback)
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, 0, 0, 30)
    container.BackgroundTransparency = 1
    container.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.7, 0, 0, 14)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = TEXT_DARK
    lbl.Font = Enum.Font.RobotoMono
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = container

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(0.3, 0, 0, 14)
    valLbl.Position = UDim2.new(0.7, 0, 0, 0)
    valLbl.BackgroundTransparency = 1
    valLbl.Text = tostring(default)
    valLbl.TextColor3 = TEXT_DARK
    valLbl.Font = Enum.Font.RobotoMono
    valLbl.TextSize = 11
    valLbl.TextXAlignment = Enum.TextXAlignment.Right
    valLbl.Parent = container

    local bg = Instance.new("Frame")
    bg.Size = UDim2.new(1, 0, 0, 6)
    bg.Position = UDim2.new(0, 0, 0, 20)
    bg.BackgroundColor3 = Theme.ButtonBg
    bg.BorderColor3 = WHITE
    bg.BorderSizePixel = 1
    bg.Parent = container

    local bgC = Instance.new("UICorner"); bgC.CornerRadius = UDim.new(1,0); bgC.Parent = bg

    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(math.clamp((default - min) / (max - min), 0, 1), 0, 1, 0)
    fill.BackgroundColor3 = WHITE
    fill.BorderSizePixel = 0
    fill.Parent = bg

    local fillC = Instance.new("UICorner"); fillC.CornerRadius = UDim.new(1,0); fillC.Parent = fill

    local drag = false
    local function update(input)
        local pos = math.clamp((input.Position.X - bg.AbsolutePosition.X) / bg.AbsoluteSize.X, 0, 1)
        local val = math.floor(min + (max - min) * pos)
        fill.Size = UDim2.new(pos, 0, 1, 0)
        valLbl.Text = tostring(val)
        callback(val)
    end
    bg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            drag = true
            update(input)
        end
    end)
    UIS.InputChanged:Connect(function(input)
        if drag and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            update(input)
        end
    end)
    UIS.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            drag = false
        end
    end)
end

local function addDropdown(parent, text, list, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 22)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.5, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = TEXT_DARK
    lbl.Font = Enum.Font.RobotoMono
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 130, 1, 0)
    btn.Position = UDim2.new(1, -130, 0, 0)
    btn.BackgroundColor3 = Theme.ButtonBg
    btn.BorderColor3 = WHITE
    btn.BorderSizePixel = 1
    btn.Text = default
    btn.TextColor3 = TEXT_DARK
    btn.Font = Enum.Font.RobotoMono
    btn.TextSize = 11
    btn.AutoButtonColor = false
    btn.Parent = row

    local bC = Instance.new("UICorner"); bC.CornerRadius = UDim.new(0, 4); bC.Parent = btn

    local idx = 1
    for i, v in ipairs(list) do if v == default then idx = i end end
    btn.MouseButton1Click:Connect(function()
        idx = idx + 1
        if idx > #list then idx = 1 end
        btn.Text = list[idx]
        callback(list[idx])
    end)
end

local function addButton(parent, text, callback)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 24)
    btn.BackgroundColor3 = Theme.ButtonBg
    btn.BorderColor3 = WHITE
    btn.BorderSizePixel = 1
    btn.Text = text
    btn.TextColor3 = TEXT_DARK
    btn.Font = Enum.Font.RobotoMono
    btn.TextSize = 11
    btn.AutoButtonColor = false
    btn.Parent = parent

    local bC = Instance.new("UICorner"); bC.CornerRadius = UDim.new(0, 4); bC.Parent = btn

    btn.MouseButton1Click:Connect(callback)
end

local function addTextInput(parent, text, default, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 22)
    row.BackgroundTransparency = 1
    row.Parent = parent

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.4, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = text
    lbl.TextColor3 = TEXT_DARK
    lbl.Font = Enum.Font.RobotoMono
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = row

    local box = Instance.new("TextBox")
    box.Size = UDim2.new(0, 180, 1, 0)
    box.Position = UDim2.new(1, -180, 0, 0)
    box.BackgroundColor3 = Theme.ButtonBg
    box.BorderColor3 = WHITE
    box.BorderSizePixel = 1
    box.Text = default or ""
    box.PlaceholderText = "..."
    box.TextColor3 = TEXT_DARK
    box.PlaceholderColor3 = TEXT_DARK
    box.Font = Enum.Font.RobotoMono
    box.TextSize = 11
    box.ClearTextOnFocus = false
    box.Parent = row

    local bC = Instance.new("UICorner"); bC.CornerRadius = UDim.new(0, 4); bC.Parent = box

    box:GetPropertyChangedSignal("Text"):Connect(function()
        callback(box.Text)
    end)
end

addLabel(combatPage, "// silent aim")
local combatSilentToggle = addToggle(combatPage, "enable", cfg.silentAim, function(v) cfg.silentAim = v end)
addKeybind(combatPage, "toggle key", cfg.silentAimKey, function(k) cfg.silentAimKey = k end)
addToggle(combatPage, "wall check", cfg.wallCheck, function(v) cfg.wallCheck = v end)
addToggle(combatPage, "team check", cfg.teamCheck, function(v) cfg.teamCheck = v end)
addToggle(combatPage, "bypass revolver", cfg.bypassRevolver, function(v) cfg.bypassRevolver = v end)
addToggle(combatPage, "closest part", cfg.closestPart, function(v) cfg.closestPart = v end)
addSlider(combatPage, "fov", 10, 500, cfg.fov, function(v) cfg.fov = v end)
addSlider(combatPage, "hit chance", 0, 100, cfg.hitChance, function(v) cfg.hitChance = v end)
addSlider(combatPage, "max dist", 100, 3000, cfg.maxDist, function(v) cfg.maxDist = v end)
addDropdown(combatPage, "hitpart", bodyParts, cfg.hitPart, function(v) cfg.hitPart = v end)

addLabel(camlockPage, "// camlock")
local camlockT = addToggle(camlockPage, "enable", cfg.camlock, function(v)
    cfg.camlock = v
    if not v then lockedPlayer = nil end
end)
addKeybind(camlockPage, "toggle key", cfg.camlockKey, function(k) cfg.camlockKey = k end)
addSlider(camlockPage, "smoothness", 1, 20, cfg.camlockSmooth, function(v) cfg.camlockSmooth = v end)
addDropdown(camlockPage, "hitpart", {"Head","HumanoidRootPart","UpperTorso"}, cfg.camlockPart, function(v) cfg.camlockPart = v end)

addLabel(visualsPage, "// atmosphere")
addSlider(visualsPage, "density", 0, 1, cfg.atmDensity, function(v) cfg.atmDensity = v end)
addSlider(visualsPage, "haze", 0, 10, cfg.atmHaze, function(v) cfg.atmHaze = v end)
addSlider(visualsPage, "decay", 0, 2, cfg.atmDecay, function(v) cfg.atmDecay = v end)
addLabel(visualsPage, "// lighting")
addSlider(visualsPage, "brightness", 0, 10, cfg.brightness, function(v) cfg.brightness = v end)
addSlider(visualsPage, "clocktime", 0, 24, cfg.clockTime, function(v) cfg.clockTime = v end)

addLabel(visualsPage, "// fog color")
addToggle(visualsPage, "enable fog", cfg.fogEnabled, function(v) cfg.fogEnabled = v end)

local fogPreviewRow = Instance.new("Frame")
fogPreviewRow.Size = UDim2.new(1, 0, 0, 22)
fogPreviewRow.BackgroundTransparency = 1
fogPreviewRow.Parent = visualsPage

local fogPreviewLbl = Instance.new("TextLabel")
fogPreviewLbl.Size = UDim2.new(0.5, 0, 1, 0)
fogPreviewLbl.BackgroundTransparency = 1
fogPreviewLbl.Text = "preview"
fogPreviewLbl.TextColor3 = TEXT_DARK
fogPreviewLbl.Font = Enum.Font.RobotoMono
fogPreviewLbl.TextSize = 11
fogPreviewLbl.TextXAlignment = Enum.TextXAlignment.Left
fogPreviewLbl.Parent = fogPreviewRow

local fogPreview = Instance.new("Frame")
fogPreview.Size = UDim2.new(0, 60, 0, 16)
fogPreview.Position = UDim2.new(1, -60, 0.5, -8)
fogPreview.BackgroundColor3 = Color3.fromRGB(cfg.fogR, cfg.fogG, cfg.fogB)
fogPreview.BorderColor3 = WHITE
fogPreview.BorderSizePixel = 1
fogPreview.Parent = fogPreviewRow

local fogPreviewCorner = Instance.new("UICorner")
fogPreviewCorner.CornerRadius = UDim.new(0, 4)
fogPreviewCorner.Parent = fogPreview

local function updateFogPreview()
    fogPreview.BackgroundColor3 = Color3.fromRGB(cfg.fogR, cfg.fogG, cfg.fogB)
end

local function setFogColor(r, g, b)
    cfg.fogR = r
    cfg.fogG = g
    cfg.fogB = b
    updateFogPreview()
end

addLabel(visualsPage, "// color presets")

local presetRow = Instance.new("Frame")
presetRow.Size = UDim2.new(1, 0, 0, 22)
presetRow.BackgroundTransparency = 1
presetRow.Parent = visualsPage

local presetLayout = Instance.new("UIListLayout")
presetLayout.FillDirection = Enum.FillDirection.Horizontal
presetLayout.Padding = UDim.new(0, 4)
presetLayout.SortOrder = Enum.SortOrder.LayoutOrder
presetLayout.Parent = presetRow

local presets = {
    {name="pink",   r=255, g=175, b=205},
    {name="blue",   r=145, g=210, b=240},
    {name="purple", r=180, g=140, b=255},
    {name="red",    r=255, g=90,  b=90},
    {name="green",  r=120, g=230, b=140},
    {name="cyan",   r=90,  g=230, b=230},
    {name="black",  r=15,  g=15,  b=20},
    {name="white",  r=255, g=255, b=255},
}

for _, preset in ipairs(presets) do
    local pBtn = Instance.new("TextButton")
    pBtn.Size = UDim2.new(0, 30, 1, 0)
    pBtn.BackgroundColor3 = Color3.fromRGB(preset.r, preset.g, preset.b)
    pBtn.BorderColor3 = WHITE
    pBtn.BorderSizePixel = 1
    pBtn.Text = ""
    pBtn.AutoButtonColor = false
    pBtn.Parent = presetRow
    local pC = Instance.new("UICorner"); pC.CornerRadius = UDim.new(0,4); pC.Parent = pBtn
    pBtn.MouseButton1Click:Connect(function()
        setFogColor(preset.r, preset.g, preset.b)
    end)
end

addLabel(visualsPage, "// rgb custom")
addSlider(visualsPage, "red",   0, 255, cfg.fogR, function(v) cfg.fogR = v; updateFogPreview() end)
addSlider(visualsPage, "green", 0, 255, cfg.fogG, function(v) cfg.fogG = v; updateFogPreview() end)
addSlider(visualsPage, "blue",  0, 255, cfg.fogB, function(v) cfg.fogB = v; updateFogPreview() end)

addLabel(visualsPage, "// fog range")
addSlider(visualsPage, "fog start", 0, 500, cfg.fogStart, function(v) cfg.fogStart = v end)
addSlider(visualsPage, "fog end",   50, 2000, cfg.fogEnd, function(v) cfg.fogEnd = v end)

addLabel(whitelistPage, "// whitelist peoples activate")
local wlActiveToggle = addToggle(whitelistPage, "whitelist active", cfg.whitelistActive, function(v) cfg.whitelistActive = v end)

local wlCountLbl = Instance.new("TextLabel")
wlCountLbl.Size = UDim2.new(1, 0, 0, 16)
wlCountLbl.BackgroundTransparency = 1
wlCountLbl.Text = "whitelisted: 0"
wlCountLbl.TextColor3 = TEXT_DARK
wlCountLbl.Font = Enum.Font.RobotoMono
wlCountLbl.TextSize = 11
wlCountLbl.TextXAlignment = Enum.TextXAlignment.Left
wlCountLbl.Parent = whitelistPage

local wlBtnRow = Instance.new("Frame")
wlBtnRow.Size = UDim2.new(1, 0, 0, 22)
wlBtnRow.BackgroundTransparency = 1
wlBtnRow.Parent = whitelistPage

local wlRefreshBtn = Instance.new("TextButton")
wlRefreshBtn.Size = UDim2.new(0, 80, 1, 0)
wlRefreshBtn.BackgroundColor3 = Theme.ButtonBg
wlRefreshBtn.BorderColor3 = WHITE
wlRefreshBtn.BorderSizePixel = 1
wlRefreshBtn.Text = "refresh"
wlRefreshBtn.TextColor3 = TEXT_DARK
wlRefreshBtn.Font = Enum.Font.RobotoMono
wlRefreshBtn.TextSize = 10
wlRefreshBtn.AutoButtonColor = false
wlRefreshBtn.Parent = wlBtnRow
local rc1 = Instance.new("UICorner"); rc1.CornerRadius = UDim.new(0,4); rc1.Parent = wlRefreshBtn

local wlClearBtn = Instance.new("TextButton")
wlClearBtn.Size = UDim2.new(0, 80, 1, 0)
wlClearBtn.Position = UDim2.new(0, 84, 0, 0)
wlClearBtn.BackgroundColor3 = Theme.ButtonBg
wlClearBtn.BorderColor3 = WHITE
wlClearBtn.BorderSizePixel = 1
wlClearBtn.Text = "clear all"
wlClearBtn.TextColor3 = TEXT_DARK
wlClearBtn.Font = Enum.Font.RobotoMono
wlClearBtn.TextSize = 10
wlClearBtn.AutoButtonColor = false
wlClearBtn.Parent = wlBtnRow
local rc2 = Instance.new("UICorner"); rc2.CornerRadius = UDim.new(0,4); rc2.Parent = wlClearBtn

local wlHolder = Instance.new("Frame")
wlHolder.Size = UDim2.new(1, 0, 0, 200)
wlHolder.BackgroundTransparency = 1
wlHolder.Parent = whitelistPage

local wlScroll = Instance.new("ScrollingFrame")
wlScroll.Size = UDim2.new(1, 0, 1, 0)
wlScroll.BackgroundColor3 = Theme.ButtonBg
wlScroll.BorderColor3 = WHITE
wlScroll.BorderSizePixel = 1
wlScroll.ScrollBarThickness = 3
wlScroll.ScrollBarImageColor3 = WHITE
wlScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
wlScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
wlScroll.Parent = wlHolder
local rc3 = Instance.new("UICorner"); rc3.CornerRadius = UDim.new(0,4); rc3.Parent = wlScroll

local wlList = Instance.new("UIListLayout")
wlList.Padding = UDim.new(0, 2)
wlList.Parent = wlScroll

local wlBtns = {}

local function refreshWL()
    for _, b in pairs(wlBtns) do b:Destroy() end
    wlBtns = {}
    local count = 0
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then
            local b = Instance.new("TextButton")
            b.Size = UDim2.new(1, -6, 0, 22)
            b.Position = UDim2.new(0, 3, 0, 0)
            b.BackgroundColor3 = cfg.whitelist[p.Name] and WHITE or Theme.ButtonBg
            b.BorderSizePixel = 0
            b.Text = "  " .. p.Name .. (cfg.whitelist[p.Name] and "  [wl]" or "")
            b.TextColor3 = TEXT_DARK
            b.Font = Enum.Font.RobotoMono
            b.TextSize = 10
            b.TextXAlignment = Enum.TextXAlignment.Left
            b.AutoButtonColor = false
            b.Parent = wlScroll
            local bc = Instance.new("UICorner"); bc.CornerRadius = UDim.new(0,3); bc.Parent = b
            b.MouseButton1Click:Connect(function()
                cfg.whitelist[p.Name] = not cfg.whitelist[p.Name]
                refreshWL()
            end)
            table.insert(wlBtns, b)
            if cfg.whitelist[p.Name] then count = count + 1 end
        end
    end
    wlCountLbl.Text = "whitelisted: " .. count
end

wlRefreshBtn.MouseButton1Click:Connect(refreshWL)
wlClearBtn.MouseButton1Click:Connect(function()
    cfg.whitelist = {}
    refreshWL()
end)

refreshWL()
Players.PlayerAdded:Connect(function() task.wait(0.5) refreshWL() end)
Players.PlayerRemoving:Connect(function() task.wait(0.5) refreshWL() end)

addLabel(avatarPage, "// jayce.char")
local avatarToggle = addToggle(avatarPage, "enable avatar", cfg.avatarOn, function(v) cfg.avatarOn = v end)
addTextInput(avatarPage, "user id / username", cfg.avatarTarget, function(v) cfg.avatarTarget = v end)
addLabel(avatarPage, "// headless")
local headlessToggle = addToggle(avatarPage, "headless mode", cfg.headless, function(v) cfg.headless = v end)

local function hideHead(char)
    local h = char:WaitForChild("Head", 5)
    if not h then return end
    h.Transparency = 1
    h.CanCollide = false
    local f = h:FindFirstChild("face") or h:FindFirstChildOfClass("Decal")
    if f then f.Transparency = 1 end
end

LocalPlayer.CharacterAdded:Connect(function(char)
    if cfg.headless then
        task.wait(0.3)
        hideHead(char)
    end
end)

task.spawn(function()
    while task.wait(0.5) do
        if cfg.headless and LocalPlayer.Character then
            local h = LocalPlayer.Character:FindFirstChild("Head")
            if h and h.Transparency ~= 1 then
                h.Transparency = 1
                h.CanCollide = false
                local f = h:FindFirstChild("face") or h:FindFirstChildOfClass("Decal")
                if f then f.Transparency = 1 end
            end
        end
    end
end)

addLabel(miscPage, "// misc")
addToggle(miscPage, "anti-fall", cfg.antiFall, function(v)
    cfg.antiFall = v
    if v then
        if antiFallConn then antiFallConn:Disconnect() end
        antiFallConn = RunService.Heartbeat:Connect(function()
            if not cfg.antiFall then return end
            local char = LocalPlayer.Character
            if char then
                local hrp = char:FindFirstChild("HumanoidRootPart")
                if hrp and hrp.Velocity.Y < -100 then
                    hrp.Velocity = Vector3.new(hrp.Velocity.X, 0, hrp.Velocity.Z)
                end
            end
        end)
    else
        if antiFallConn then antiFallConn:Disconnect(); antiFallConn = nil end
    end
end)

addToggle(miscPage, "auto sprint", cfg.autoSprint, function(v) cfg.autoSprint = v end)
addSlider(miscPage, "auto sprint speed", 16, 60, cfg.autoSprintSpeed, function(v) cfg.autoSprintSpeed = v end)

local speedToggle = addToggle(miscPage, "speed", cfg.speed, function(v) cfg.speed = v end)
addKeybind(miscPage, "speed key", cfg.speedKey, function(k) cfg.speedKey = k end)
addSlider(miscPage, "speed value", 1, 300, cfg.speedValue, function(v) cfg.speedValue = v end)
addSlider(miscPage, "jump power", 1, 300, cfg.jumpValue, function(v) cfg.jumpValue = v end)

local function toggleFly(v)
    cfg.fly = v
    local char = LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if v then
        if hrp then
            if bodyVel then bodyVel:Destroy() end
            if bodyGyro then bodyGyro:Destroy() end
            bodyVel = Instance.new("BodyVelocity")
            bodyVel.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bodyVel.Velocity = Vector3.new(0,0,0)
            bodyVel.Parent = hrp
            bodyGyro = Instance.new("BodyGyro")
            bodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
            bodyGyro.CFrame = hrp.CFrame
            bodyGyro.Parent = hrp
        end
        if hum then hum.PlatformStand = true end
        if flyConn then flyConn:Disconnect() end
        flyConn = RunService.RenderStepped:Connect(function()
            local c = LocalPlayer.Character
            local h = c and c:FindFirstChild("HumanoidRootPart")
            if not cfg.fly or not h or not bodyVel or not bodyGyro then return end
            local camCF = Camera.CFrame
            local moveDir = Vector3.new(0,0,0)
            if UIS:IsKeyDown(Enum.KeyCode.W) then moveDir = moveDir + camCF.LookVector end
            if UIS:IsKeyDown(Enum.KeyCode.S) then moveDir = moveDir - camCF.LookVector end
            if UIS:IsKeyDown(Enum.KeyCode.A) then moveDir = moveDir - camCF.RightVector end
            if UIS:IsKeyDown(Enum.KeyCode.D) then moveDir = moveDir + camCF.RightVector end
            if UIS:IsKeyDown(Enum.KeyCode.Space) then moveDir = moveDir + Vector3.new(0,1,0) end
            if UIS:IsKeyDown(Enum.KeyCode.LeftShift) then moveDir = moveDir - Vector3.new(0,1,0) end
            if moveDir.Magnitude > 0 then
                bodyVel.Velocity = moveDir.Unit * cfg.flySpeed
            else
                bodyVel.Velocity = Vector3.new(0,0,0)
            end
            bodyGyro.CFrame = camCF
        end)
    else
        if flyConn then flyConn:Disconnect(); flyConn = nil end
        if bodyVel then bodyVel:Destroy(); bodyVel = nil end
        if bodyGyro then bodyGyro:Destroy(); bodyGyro = nil end
        if hum then hum.PlatformStand = false end
    end
end

addToggle(miscPage, "fly", cfg.fly, toggleFly)
addKeybind(miscPage, "fly key", cfg.flyKey, function(k) cfg.flyKey = k end)
addSlider(miscPage, "fly speed", 1, 500, cfg.flySpeed, function(v) cfg.flySpeed = v end)

addLabel(protectionPage, "// protection")
addToggle(protectionPage, "anti aim viewer", cfg.antiAim, function(v)
    cfg.antiAim = v
    if v then
        if antiAimConn then antiAimConn:Disconnect() end
        antiAimConn = RunService.Heartbeat:Connect(function()
            if not cfg.antiAim then return end
            local char = LocalPlayer.Character
            if char then
                local hrp = char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    hrp.CFrame = CFrame.new(hrp.Position) * CFrame.Angles(0, math.rad(math.random(-180, 180)), 0)
                end
            end
        end)
    else
        if antiAimConn then antiAimConn:Disconnect(); antiAimConn = nil end
    end
end)
addToggle(protectionPage, "0 aim accuracy", cfg.aimAcc0, function(v) cfg.aimAcc0 = v end)

addLabel(settingsPage, "// settings")
addKeybind(settingsPage, "ui toggle key", cfg.uiToggleKey, function(k) cfg.uiToggleKey = k end)

local function saveCfg()
    local data = {}
    for k, v in pairs(cfg) do
        if type(v) == "table" then
            data[k] = v
        elseif typeof(v) == "Color3" then
            data[k] = {v.R, v.G, v.B}
        elseif typeof(v) == "EnumItem" then
            data[k] = v.Name
        else
            data[k] = v
        end
    end
    if writefile then
        pcall(function()
            writefile("jayce_config.json", HttpService:JSONEncode(data))
        end)
    end
end

local function loadCfg()
    if not (isfile and readfile) then return end
    if not isfile("jayce_config.json") then return end
    local ok, decoded = pcall(function()
        return HttpService:JSONDecode(readfile("jayce_config.json"))
    end)
    if not ok or type(decoded) ~= "table" then return end
    for k, v in pairs(decoded) do
        if type(v) == "table" and #v == 3 then
            cfg[k] = Color3.new(v[1], v[2], v[3])
        elseif type(v) == "string" and cfg[k] and typeof(cfg[k]) == "EnumItem" then
            pcall(function() cfg[k] = Enum.KeyCode[v] end)
        else
            cfg[k] = v
        end
    end
end

loadCfg()

addButton(settingsPage, "save config", saveCfg)
addButton(settingsPage, "reset config", function()
    if delfile and isfile and isfile("jayce_config.json") then
        delfile("jayce_config.json")
    end
end)

UIS.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    if input.KeyCode == cfg.uiToggleKey then
        cfg.uiVisible = not cfg.uiVisible
        WindowOuterBorder.Visible = cfg.uiVisible
    elseif input.KeyCode == cfg.silentAimKey then
        cfg.silentAim = not cfg.silentAim
        if combatSilentToggle then combatSilentToggle(cfg.silentAim) end
    elseif input.KeyCode == cfg.camlockKey then
        cfg.camlock = not cfg.camlock
        if camlockT then camlockT(cfg.camlock) end
        if not cfg.camlock then lockedPlayer = nil end
    elseif input.KeyCode == cfg.flyKey then
        toggleFly(not cfg.fly)
    elseif input.KeyCode == cfg.speedKey then
        cfg.speed = not cfg.speed
        if speedToggle then speedToggle(cfg.speed) end
    end
end)

RunService.RenderStepped:Connect(function()
    if cfg.silentAim then
        cachedPart = getClosestPlayerToCursor()
    else
        cachedPart = nil
    end

    local char = LocalPlayer.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            if cfg.speed then
                hum.WalkSpeed = cfg.speedValue
                hum.JumpPower = cfg.jumpValue
            elseif cfg.autoSprint then
                if hum.MoveDirection.Magnitude > 0 then
                    hum.WalkSpeed = cfg.autoSprintSpeed
                else
                    hum.WalkSpeed = 16
                end
            end
        end
    end

    if cfg.camlock then
        if not lockedPlayer or not (lockedPlayer.Character and lockedPlayer.Character:FindFirstChildOfClass("Humanoid") and lockedPlayer.Character.Humanoid.Health > 0) then
            local closestDist = math.huge
            local bestPlayer = nil
            local m = UIS:GetMouseLocation()
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= LocalPlayer and player.Character and not isWL(player) then
                    local hum = player.Character:FindFirstChildOfClass("Humanoid")
                    if hum and hum.Health > 0 then
                        local pPart = player.Character:FindFirstChild(cfg.camlockPart) or player.Character:FindFirstChild("HumanoidRootPart")
                        if pPart then
                            local sp, on = Camera:WorldToViewportPoint(pPart.Position)
                            if on then
                                local d = (Vector2.new(sp.X, sp.Y) - m).Magnitude
                                if d < closestDist then
                                    closestDist = d
                                    bestPlayer = player
                                end
                            end
                        end
                    end
                end
            end
            lockedPlayer = bestPlayer
        end
        if lockedPlayer and lockedPlayer.Character then
            local targetPart = lockedPlayer.Character:FindFirstChild(cfg.camlockPart) or lockedPlayer.Character:FindFirstChild("HumanoidRootPart")
            if targetPart then
                local currentCF = Camera.CFrame
                local targetCF = CFrame.new(currentCF.Position, targetPart.Position)
                local smooth = math.clamp(cfg.camlockSmooth, 1, 20)
                Camera.CFrame = currentCF:Lerp(targetCF, 1 / smooth)
            end
        end
    else
        lockedPlayer = nil
    end

    pcall(function()
        atmos.Density = cfg.atmDensity
        atmos.Haze = cfg.atmHaze
        atmos.Decay = cfg.atmDecay
        Lighting.Brightness = cfg.brightness
        Lighting.ClockTime = cfg.clockTime
        if cfg.fogEnabled then
            local fc = Color3.fromRGB(cfg.fogR, cfg.fogG, cfg.fogB)
            Lighting.FogColor = fc
            Lighting.FogStart = cfg.fogStart
            Lighting.FogEnd = cfg.fogEnd
            atmos.Color = fc
        end
    end)
end)
