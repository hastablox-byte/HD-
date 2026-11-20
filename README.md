--========================================================
-- HD ADMIN v2.2.0
-- by Voltex Studios
-- Interface inspired by the supplied reference image
-- Client-side utilities for your own character only.
--========================================================

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local LP = Players.LocalPlayer

-- Remove an older copy
pcall(function()
    local old = (gethui and gethui()) or game:GetService("CoreGui")
    local found = old:FindFirstChild("HD_ADMIN")
    if found then found:Destroy() end
end)

local GUI = Instance.new("ScreenGui")
GUI.Name = "HD_ADMIN"
GUI.ResetOnSpawn = false
GUI.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local parentOK = pcall(function()
    GUI.Parent = (gethui and gethui()) or game:GetService("CoreGui")
end)
if not parentOK or not GUI.Parent then
    GUI.Parent = LP:WaitForChild("PlayerGui")
end

local Settings = {
    Theme = "Dark",
    BackgroundTransparency = 0,
    Rounded = true,
    Scale = 1,
    Accent = Color3.fromRGB(245,245,245),
    Background = Color3.fromRGB(10,10,11),
    Panel = Color3.fromRGB(24,24,26),
    Tile = Color3.fromRGB(37,37,40),
    Text = Color3.fromRGB(248,248,248),
    Muted = Color3.fromRGB(155,155,160)
}

local function create(class, parent, props)
    local obj = Instance.new(class)
    for k,v in pairs(props or {}) do
        obj[k] = v
    end
    obj.Parent = parent
    return obj
end

local function corner(obj, radius)
    return create("UICorner", obj, {
        CornerRadius = UDim.new(0, Settings.Rounded and radius or 2)
    })
end

local function outline(obj, color)
    return create("UIStroke", obj, {
        Color = color or Color3.fromRGB(58,58,62),
        Thickness = 1,
        Transparency = 0.12
    })
end

local function tween(obj, props, duration)
    TweenService:Create(obj, TweenInfo.new(duration or 0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), props):Play()
end

local function label(parent, text, size, pos, color, bold)
    return create("TextLabel", parent, {
        BackgroundTransparency = 1,
        Text = text,
        TextColor3 = color or Settings.Text,
        TextSize = size,
        Position = pos,
        Font = bold and Enum.Font.GothamBold or Enum.Font.GothamMedium,
        TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center
    })
end

local logs = {}
local notification
local function log(message)
    table.insert(logs, 1, os.date("%H:%M:%S") .. "  •  " .. tostring(message))
    while #logs > 50 do table.remove(logs) end
end

local function notify(message)
    log(message)
    if notification then notification:Destroy() end
    notification = create("Frame", GUI, {
        Size = UDim2.fromOffset(245, 52),
        Position = UDim2.new(1,-260,1,-68),
        BackgroundColor3 = Settings.Panel,
        BackgroundTransparency = Settings.BackgroundTransparency,
        BorderSizePixel = 0,
        ZIndex = 20
    })
    corner(notification, 9)
    outline(notification)
    label(notification, "HD ADMIN", 11, UDim2.fromOffset(12,5), Settings.Muted, true).Size = UDim2.new(1,-20,0,16)
    local msg = label(notification, tostring(message), 12, UDim2.fromOffset(12,23), Settings.Text, false)
    msg.Size = UDim2.new(1,-20,0,22)
    msg.TextTruncate = Enum.TextTruncate.AtEnd
    task.delay(2.8, function()
        if notification then notification:Destroy(); notification = nil end
    end)
end

-- Character helpers
local function character() return LP.Character end
local function humanoid()
    local c = character()
    return c and c:FindFirstChildOfClass("Humanoid")
end
local function root()
    local c = character()
    return c and c:FindFirstChild("HumanoidRootPart")
end
local function head()
    local c = character()
    return c and c:FindFirstChild("Head")
end

local original = {}
local function saveCharacter()
    table.clear(original)
    local c = character()
    if not c then return end
    for _,obj in ipairs(c:GetDescendants()) do
        if obj:IsA("BasePart") then
            original[obj] = {
                Size = obj.Size,
                Material = obj.Material,
                Color = obj.Color,
                Transparency = obj.Transparency,
                LocalTransparencyModifier = obj.LocalTransparencyModifier,
                CanCollide = obj.CanCollide
            }
        end
    end
end
if character() then saveCharacter() end
LP.CharacterAdded:Connect(function()
    task.wait(1)
    saveCharacter()
end)

local states = {Fly=false, NoClip=false, InfiniteJump=false}
local connections = {}
local flyObjects = {}

local function stopConnection(name)
    if connections[name] then
        connections[name]:Disconnect()
        connections[name] = nil
    end
end

local function fly(speed)
    states.Fly = not states.Fly
    stopConnection("Fly")
    for _,obj in ipairs(flyObjects) do pcall(function() obj:Destroy() end) end
    table.clear(flyObjects)
    if not states.Fly then notify("Fly desativado"); return end

    local r = root()
    if not r then states.Fly=false; notify("Personagem não encontrado"); return end

    local velocity = Instance.new("BodyVelocity")
    velocity.MaxForce = Vector3.new(1e6,1e6,1e6)
    velocity.Velocity = Vector3.zero
    velocity.Parent = r
    local gyro = Instance.new("BodyGyro")
    gyro.MaxTorque = Vector3.new(1e6,1e6,1e6)
    gyro.P = 10000
    gyro.CFrame = workspace.CurrentCamera.CFrame
    gyro.Parent = r
    flyObjects = {velocity,gyro}

    connections.Fly = RunService.RenderStepped:Connect(function()
        if not states.Fly or not r.Parent then return end
        local camera = workspace.CurrentCamera
        local direction = Vector3.zero
        if UIS:IsKeyDown(Enum.KeyCode.W) then direction += camera.CFrame.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.S) then direction -= camera.CFrame.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.A) then direction -= camera.CFrame.RightVector end
        if UIS:IsKeyDown(Enum.KeyCode.D) then direction += camera.CFrame.RightVector end
        if UIS:IsKeyDown(Enum.KeyCode.Space) then direction += Vector3.yAxis end
        if UIS:IsKeyDown(Enum.KeyCode.LeftControl) then direction -= Vector3.yAxis end
        velocity.Velocity = direction * (speed or 60)
        gyro.CFrame = camera.CFrame
    end)
    notify("Fly ativado — WASD / Espaço / Ctrl")
end

local function noclip()
    states.NoClip = not states.NoClip
    stopConnection("NoClip")
    if states.NoClip then
        connections.NoClip = RunService.Stepped:Connect(function()
            local c = character()
            if c then
                for _,obj in ipairs(c:GetDescendants()) do
                    if obj:IsA("BasePart") then obj.CanCollide = false end
                end
            end
        end)
        notify("NoClip ativado")
    else
        for obj,data in pairs(original) do
            if obj.Parent then obj.CanCollide = data.CanCollide end
        end
        notify("NoClip desativado")
    end
end

local function infiniteJump()
    states.InfiniteJump = not states.InfiniteJump
    stopConnection("InfiniteJump")
    if states.InfiniteJump then
        connections.InfiniteJump = UIS.JumpRequest:Connect(function()
            local h = humanoid()
            if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
        end)
        notify("Infinite Jump ativado")
    else
        notify("Infinite Jump desativado")
    end
end

local function applyMaterial(material)
    local c = character()
    if not c then return end
    for _,obj in ipairs(c:GetDescendants()) do
        if obj:IsA("BasePart") then obj.Material = material end
    end
    notify(material.Name .. " aplicado")
end

local function headSize(value)
    local h = head()
    if not h then return end
    local data = original[h]
    h.Size = (data and data.Size or h.Size) * math.clamp(value,0.5,5)
    notify("HeadSize: " .. tostring(value))
end

local function bodySize(value)
    local h = humanoid()
    if not h then return end
    value = math.clamp(value,0.5,3)
    for _,name in ipairs({"BodyWidthScale","BodyHeightScale","BodyDepthScale","HeadScale"}) do
        local scale = h:FindFirstChild(name)
        if scale and scale:IsA("NumberValue") then scale.Value = value end
    end
    notify("Size: " .. tostring(value))
end

local function setTitle(text)
    local h = head()
    if not h then return end
    local old = h:FindFirstChild("HDAdminTitle")
    if old then old:Destroy() end
    local billboard = create("BillboardGui", h, {
        Name = "HDAdminTitle",
        Size = UDim2.fromOffset(220,45),
        StudsOffset = Vector3.new(0,2.7,0),
        AlwaysOnTop = true
    })
    create("TextLabel", billboard, {
        Size = UDim2.fromScale(1,1),
        BackgroundTransparency = 1,
        Text = tostring(text),
        TextColor3 = Settings.Accent,
        TextStrokeTransparency = 0,
        Font = Enum.Font.GothamBold,
        TextSize = 18
    })
    notify("Título local aplicado")
end

local function setSpeed(value)
    local h = humanoid()
    if h then h.WalkSpeed = math.clamp(value,0,200); notify("Speed: "..value) end
end
local function setJump(value)
    local h = humanoid()
    if h then h.UseJumpPower=true; h.JumpPower=math.clamp(value,0,200); notify("Jump: "..value) end
end
local function heal()
    local h = humanoid()
    if h then h.Health=h.MaxHealth; notify("Vida restaurada") end
end
local function god()
    local h = humanoid()
    if h then h.MaxHealth=100000; h.Health=h.MaxHealth; notify("Vida local alterada") end
end
local function invisible()
    local c = character()
    if not c then return end
    for _,obj in ipairs(c:GetDescendants()) do
        if obj:IsA("BasePart") then obj.LocalTransparencyModifier=1 end
    end
    notify("Invisibilidade local ativada")
end

local function restore()
    states.Fly=false; states.NoClip=false; states.InfiniteJump=false
    for name in pairs(connections) do stopConnection(name) end
    for _,obj in ipairs(flyObjects) do pcall(function() obj:Destroy() end) end
    table.clear(flyObjects)
    local h = humanoid()
    if h then
        h.WalkSpeed=16
        h.UseJumpPower=true
        h.JumpPower=50
        h.MaxHealth=100
        h.Health=100
    end
    for obj,data in pairs(original) do
        if obj and obj.Parent then
            obj.Size=data.Size
            obj.Material=data.Material
            obj.Color=data.Color
            obj.Transparency=data.Transparency
            obj.LocalTransparencyModifier=data.LocalTransparencyModifier
            obj.CanCollide=data.CanCollide
        end
    end
    local hd=head()
    if hd then
        local titleObj=hd:FindFirstChild("HDAdminTitle")
        if titleObj then titleObj:Destroy() end
    end
    notify("Configurações originais restauradas")
end

local commands = {
    fly=function() fly(60) end,
    fly2=function() fly(100) end,
    noclip=noclip,
    infinitejump=infiniteJump,
    neon=function() applyMaterial(Enum.Material.Neon) end,
    glass=function() applyMaterial(Enum.Material.Glass) end,
    heal=heal,
    god=god,
    invisible=invisible,
    restore=restore,
    reset=function() if character() then character():BreakJoints() end end,
    speed=function(a) setSpeed(tonumber(a[3]) or 50) end,
    jump=function(a) setJump(tonumber(a[3]) or 100) end,
    headsize=function(a) headSize(tonumber(a[3]) or 2) end,
    size=function(a) bodySize(tonumber(a[3]) or 2) end,
    title=function(a) setTitle(table.concat(a," ",3) ~= "" and table.concat(a," ",3) or "ADMIN") end
}

local function executeChat(message)
    if message:sub(1,1) ~= ";" then return end
    local args = string.split(message," ")
    local command = (args[1] or ""):sub(2):lower()
    local target = (args[2] or "me"):lower()
    if target ~= "me" and target ~= LP.Name:lower() then
        notify("Por segurança, esta versão só altera seu personagem")
        return
    end
    local fn = commands[command]
    if fn then
        local ok,err = pcall(function() fn(args) end)
        if not ok then notify("Erro: "..tostring(err)) end
    else
        notify("Comando desconhecido: "..command)
    end
end
LP.Chatted:Connect(executeChat)

-- Main window: proportions and layout follow the reference image
local main = create("Frame", GUI, {
    Name="MainWindow",
    Size=UDim2.fromOffset(320,480),
    Position=UDim2.new(0.5,-160,0.5,-240),
    BackgroundColor3=Settings.Background,
    BackgroundTransparency=Settings.BackgroundTransparency,
    BorderSizePixel=0,
    Visible=false,
    ClipsDescendants=true
})
corner(main,14)
outline(main,Color3.fromRGB(65,65,70))

local top = create("Frame", main, {
    Name="TopBar",
    Size=UDim2.new(1,0,0,64),
    BackgroundColor3=Color3.fromRGB(0,0,0),
    BorderSizePixel=0,
    Active=true
})
corner(top,13)

local version = label(top,"v2.1.0",9,UDim2.fromOffset(9,5),Settings.Muted,true)
version.Rotation=-45
version.Size=UDim2.fromOffset(42,14)

local logo = create("TextLabel",top,{
    Size=UDim2.fromOffset(24,24),
    Position=UDim2.new(0.5,-66,0,14),
    BackgroundColor3=Color3.fromRGB(248,248,248),
    Text="HD",
    TextColor3=Color3.fromRGB(8,8,8),
    TextSize=11,
    Font=Enum.Font.GothamBlack
})
corner(logo,5)
local menuTitle = label(top,"Menu",17,UDim2.new(0.5,-36,0,12),Settings.Text,true)
menuTitle.Size=UDim2.fromOffset(75,28)

local minimize = create("TextButton",top,{
    Size=UDim2.fromOffset(28,34),
    Position=UDim2.new(1,-76,0,13),
    BackgroundTransparency=1,
    Text="−",
    TextColor3=Settings.Text,
    TextSize=25,
    Font=Enum.Font.Gotham
})
local close = create("TextButton",top,{
    Size=UDim2.fromOffset(34,34),
    Position=UDim2.new(1,-40,0,13),
    BackgroundTransparency=1,
    Text="×",
    TextColor3=Settings.Text,
    TextSize=27,
    Font=Enum.Font.Gotham
})

local content = create("Frame",main,{
    Name="Content",
    Size=UDim2.new(1,0,1,-64),
    Position=UDim2.fromOffset(0,64),
    BackgroundTransparency=1
})

-- Drag with mouse or touch; header buttons remain usable
local dragging=false
local dragStart,startPosition,dragInput
top.InputBegan:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then
        local p=input.Position
        local x0=main.AbsolutePosition.X
        local right=main.AbsolutePosition.X+main.AbsoluteSize.X
        if p.X < right-82 then
            dragging=true
            dragStart=p
            startPosition=main.Position
            input.Changed:Connect(function()
                if input.UserInputState==Enum.UserInputState.End then dragging=false end
            end)
        end
    end
end)
top.InputChanged:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseMovement or input.UserInputType==Enum.UserInputType.Touch then
        dragInput=input
    end
end)
UIS.InputChanged:Connect(function(input)
    if dragging and input==dragInput then
        local delta=input.Position-dragStart
        main.Position=UDim2.new(startPosition.X.Scale,startPosition.X.Offset+delta.X,startPosition.Y.Scale,startPosition.Y.Offset+delta.Y)
    end
end)

local pages={}
local function newPage(name)
    local page=create("ScrollingFrame",content,{
        Name=name,
        Size=UDim2.new(1,-20,1,-18),
        Position=UDim2.fromOffset(10,9),
        BackgroundTransparency=1,
        BorderSizePixel=0,
        ScrollBarThickness=3,
        ScrollBarImageColor3=Settings.Muted,
        CanvasSize=UDim2.new(0,0,0,0),
        Visible=false
    })
    local list=create("UIListLayout",page,{
        Padding=UDim.new(0,6),
        SortOrder=Enum.SortOrder.LayoutOrder
    })
    list:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        page.CanvasSize=UDim2.fromOffset(0,list.AbsoluteContentSize.Y+12)
    end)
    pages[name]=page
    return page
end

local home=newPage("Home")
local commandsPage=newPage("Commands")
local analyticsPage=newPage("Analytics")
local rolesPage=newPage("Roles")
local moderationPage=newPage("Moderation")
local tasksPage=newPage("Tasks")
local logsPage=newPage("Logs")
local settingsPage=newPage("Settings")
local infoPage=newPage("Info")

local function showPage(name)
    if not pages[name] then return end
    for n,p in pairs(pages) do p.Visible=(n==name) end
    menuTitle.Text=(name=="Home" and "Menu" or name)
end

local function makeButton(parent,text,height,order)
    local b=create("TextButton",parent,{
        Size=UDim2.new(1,-4,0,height or 38),
        BackgroundColor3=Settings.Tile,
        BackgroundTransparency=Settings.BackgroundTransparency,
        Text=text,
        TextColor3=Settings.Text,
        TextSize=13,
        Font=Enum.Font.GothamMedium,
        AutoButtonColor=false,
        LayoutOrder=order or 1
    })
    corner(b,9); outline(b)
    b.MouseEnter:Connect(function() tween(b,{BackgroundColor3=Color3.fromRGB(55,55,60)},0.1) end)
    b.MouseLeave:Connect(function() tween(b,{BackgroundColor3=Settings.Tile},0.1) end)
    return b
end

local function section(parent,text,order)
    local t=label(parent,text,12,UDim2.new(),Settings.Muted,true)
    t.Size=UDim2.new(1,-4,0,24)
    t.LayoutOrder=order or 1
    return t
end

local function action(parent,text,callback,order)
    local b=makeButton(parent,text,38,order)
    b.MouseButton1Click:Connect(callback)
    return b
end

-- HOME: exact two-column card arrangement like the reference
local tileHolder=create("Frame",home,{
    Size=UDim2.new(1,0,0,410),
    BackgroundTransparency=1,
    LayoutOrder=1
})
local grid=create("UIGridLayout",tileHolder,{
    CellSize=UDim2.new(0.5,-4,0,98),
    CellPadding=UDim2.fromOffset(7,6),
    SortOrder=Enum.SortOrder.LayoutOrder
})
local tileData={
    {"▣","Commands","Commands"},
    {"▥","Analytics","Analytics"},
    {"⬟","Roles","Roles"},
    {"⚑","Moderation","Moderation"},
    {"▤","Tasks","Tasks"},
    {"◩","Logs","Logs"},
    {"⚙","Settings","Settings"},
    {"●","Info","Info"}
}
for i,data in ipairs(tileData) do
    local tile=create("TextButton",tileHolder,{
        BackgroundColor3=Settings.Tile,
        BackgroundTransparency=Settings.BackgroundTransparency,
        Text="",
        AutoButtonColor=false,
        LayoutOrder=i
    })
    corner(tile,10); outline(tile)
    local ico=label(tile,data[1],29,UDim2.new(0,0,0,13),Settings.Text,true)
    ico.Size=UDim2.new(1,0,0,38)
    ico.TextXAlignment=Enum.TextXAlignment.Center
    local txt=label(tile,data[2],14,UDim2.new(0,0,0,58),Settings.Text,false)
    txt.Size=UDim2.new(1,0,0,24)
    txt.TextXAlignment=Enum.TextXAlignment.Center
    tile.MouseEnter:Connect(function() tween(tile,{BackgroundColor3=Color3.fromRGB(54,54,59)},0.1) end)
    tile.MouseLeave:Connect(function() tween(tile,{BackgroundColor3=Settings.Tile},0.1) end)
    tile.MouseButton1Click:Connect(function() showPage(data[3]) end)
end

-- All secondary pages have a back button
for name,page in pairs(pages) do
    if name~="Home" then
        local back=makeButton(page,"‹   Back to menu",34,-10)
        back.MouseButton1Click:Connect(function() showPage("Home") end)
    end
end

-- COMMANDS
section(commandsPage,"PLAYER COMMANDS",1)
local search=create("TextBox",commandsPage,{
    Size=UDim2.new(1,-4,0,36),
    BackgroundColor3=Settings.Panel,
    BackgroundTransparency=Settings.BackgroundTransparency,
    TextColor3=Settings.Text,
    PlaceholderColor3=Settings.Muted,
    PlaceholderText="Search commands...",
    Text="",
    ClearTextOnFocus=false,
    Font=Enum.Font.Gotham,
    TextSize=12,
    LayoutOrder=2
})
corner(search,8); outline(search)

local commandNames={"Fly","Fly2","NoClip","InfiniteJump","Neon","Glass","HeadSize","Size","Title","Speed","Jump","Heal","God","Invisible","Restore","Reset"}
local commandButtons={}
local function runCommand(name)
    local defaults={speed="50",jump="100",headsize="2",size="2",title="ADMIN"}
    local fn=commands[name:lower()]
    if fn then
        local ok,err=pcall(function() fn({";"..name:lower(),"me",defaults[name:lower()] or "2"}) end)
        if not ok then notify("Erro: "..tostring(err)) end
    end
end
local function renderCommands(filter)
    for _,b in ipairs(commandButtons) do b:Destroy() end
    table.clear(commandButtons)
    for _,name in ipairs(commandNames) do
        if name:lower():find((filter or ""):lower(),1,true) then
            local b=makeButton(commandsPage,"◈     "..name.."                         Run",35,3+#commandButtons)
            b.TextXAlignment=Enum.TextXAlignment.Left
            b.MouseButton1Click:Connect(function() runCommand(name) end)
            table.insert(commandButtons,b)
        end
    end
end
search:GetPropertyChangedSignal("Text"):Connect(function() renderCommands(search.Text) end)
renderCommands("")

-- ANALYTICS
section(analyticsPage,"SESSION ANALYTICS",1)
local analyticsText=label(analyticsPage,"Press refresh to view session details.",12,UDim2.new(),Settings.Text,false)
analyticsText.Size=UDim2.new(1,-4,0,150)
analyticsText.TextYAlignment=Enum.TextYAlignment.Top
analyticsText.TextWrapped=true
analyticsText.LayoutOrder=2
action(analyticsPage,"↻   Refresh statistics",function()
    local h=humanoid()
    analyticsText.Text="Player: "..LP.Name..
        "\nPlayers in server: "..#Players:GetPlayers()..
        "\nCharacter: "..(character() and "Loaded" or "Not loaded")..
        "\nHealth: "..(h and math.floor(h.Health).."/"..math.floor(h.MaxHealth) or "N/A")..
        "\nWalkSpeed: "..(h and tostring(h.WalkSpeed) or "N/A")..
        "\nJumpPower: "..(h and tostring(h.JumpPower) or "N/A")
end,3)

-- ROLES: informational, no fake server permissions
section(rolesPage,"ROLE INFORMATION",1)
local roleText=label(rolesPage,"Player: "..LP.Name.."\n\nThis menu does not grant server ranks or admin permissions. Real roles are controlled by the game server.",12,UDim2.new(),Settings.Text)
roleText.Size=UDim2.new(1,-4,0,130)
roleText.TextWrapped=true
roleText.TextYAlignment=Enum.TextYAlignment.Top
roleText.LayoutOrder=2
action(rolesPage,"Show my username",function() notify(LP.Name) end,3)

-- MODERATION: local-only controls
section(moderationPage,"LOCAL CHARACTER",1)
local moderationText=label(moderationPage,"These controls only affect your own client-side character. They cannot kick, ban, mute, or control other players.",12,UDim2.new(),Settings.Text)
moderationText.Size=UDim2.new(1,-4,0,75)
moderationText.TextWrapped=true
moderationText.TextYAlignment=Enum.TextYAlignment.Top
moderationText.LayoutOrder=2
action(moderationPage,"Restore my character",restore,3)
action(moderationPage,"Reset my character",function() if character() then character():BreakJoints() end end,4)

-- TASKS
section(tasksPage,"QUICK ACTIONS",1)
action(tasksPage,"Heal character",heal,2)
action(tasksPage,"Restore character",restore,3)
action(tasksPage,"Toggle Fly",function() fly(60) end,4)
action(tasksPage,"Toggle NoClip",noclip,5)
action(tasksPage,"Toggle Infinite Jump",infiniteJump,6)
action(tasksPage,"Apply Neon",function() applyMaterial(Enum.Material.Neon) end,7)
action(tasksPage,"Apply Glass",function() applyMaterial(Enum.Material.Glass) end,8)

-- LOGS
section(logsPage,"RECENT ACTIONS",1)
local logsText=label(logsPage,"No actions recorded.",11,UDim2.new(),Settings.Text)
logsText.Size=UDim2.new(1,-4,0,250)
logsText.TextYAlignment=Enum.TextYAlignment.Top
logsText.TextWrapped=true
logsText.LayoutOrder=2
action(logsPage,"Refresh logs",function() logsText.Text=#logs>0 and table.concat(logs,"\n\n") or "No actions recorded." end,3)
action(logsPage,"Clear logs",function() table.clear(logs); logsText.Text="Logs cleared." end,4)

-- SETTINGS / DECORATION
section(settingsPage,"THEMES",1)
local themeText=label(settingsPage,"Current theme: Dark",11,UDim2.new(),Settings.Muted)
themeText.Size=UDim2.new(1,-4,0,22)
themeText.LayoutOrder=2

local open -- assigned below; keep the floating opener black across themes
local function applyTheme(name)
    local themes={
        Dark={Color3.fromRGB(10,10,11),Color3.fromRGB(24,24,26),Color3.fromRGB(37,37,40),Color3.fromRGB(245,245,245)},
        Midnight={Color3.fromRGB(7,13,24),Color3.fromRGB(17,27,44),Color3.fromRGB(28,43,65),Color3.fromRGB(105,170,255)},
        Purple={Color3.fromRGB(17,10,24),Color3.fromRGB(34,20,46),Color3.fromRGB(53,32,69),Color3.fromRGB(195,135,255)},
        Red={Color3.fromRGB(22,9,11),Color3.fromRGB(43,19,22),Color3.fromRGB(65,29,33),Color3.fromRGB(255,105,115)},
        Green={Color3.fromRGB(8,18,14),Color3.fromRGB(18,37,28),Color3.fromRGB(30,57,43),Color3.fromRGB(100,220,155)}
    }
    local t=themes[name]
    if not t then return end
    Settings.Theme=name
    Settings.Background,Settings.Panel,Settings.Tile,Settings.Accent=t[1],t[2],t[3],t[4]
    main.BackgroundColor3=Settings.Background
    for _,obj in ipairs(GUI:GetDescendants()) do
        if obj:IsA("TextButton") and obj~=close and obj~=minimize and obj~=open then
            obj.BackgroundColor3=Settings.Tile
        elseif obj:IsA("TextBox") then
            obj.BackgroundColor3=Settings.Panel
        end
        if obj:IsA("TextLabel") and obj.Text=="HD" then
            obj.TextColor3=Color3.fromRGB(8,8,8)
        end
    end
    themeText.Text="Current theme: "..name
    notify(name.." theme applied")
end

action(settingsPage,"●   Dark",function() applyTheme("Dark") end,3)
action(settingsPage,"●   Midnight Blue",function() applyTheme("Midnight") end,4)
action(settingsPage,"●   Purple",function() applyTheme("Purple") end,5)
action(settingsPage,"●   Red",function() applyTheme("Red") end,6)
action(settingsPage,"●   Green",function() applyTheme("Green") end,7)
section(settingsPage,"WINDOW OPTIONS",8)
action(settingsPage,"◐   Toggle transparency",function()
    Settings.BackgroundTransparency=(Settings.BackgroundTransparency==0 and 0.12 or 0)
    main.BackgroundTransparency=Settings.BackgroundTransparency
    for _,obj in ipairs(GUI:GetDescendants()) do
        if (obj:IsA("TextButton") or obj:IsA("TextBox")) and obj.BackgroundTransparency<1 then
            obj.BackgroundTransparency=Settings.BackgroundTransparency
        end
    end
    notify("Transparency updated")
end,9)
action(settingsPage,"▢   Toggle rounded corners",function()
    Settings.Rounded=not Settings.Rounded
    for _,obj in ipairs(GUI:GetDescendants()) do
        local c=obj:FindFirstChildOfClass("UICorner")
        if c then c.CornerRadius=UDim.new(0,Settings.Rounded and 9 or 2) end
    end
    notify("Rounded corners updated")
end,10)
action(settingsPage,"＋   Increase UI size",function()
    Settings.Scale=math.clamp(Settings.Scale+0.05,0.8,1.2)
    main.Size=UDim2.fromOffset(320*Settings.Scale,480*Settings.Scale)
    notify("UI scale: "..math.floor(Settings.Scale*100).."%")
end,11)
action(settingsPage,"－   Decrease UI size",function()
    Settings.Scale=math.clamp(Settings.Scale-0.05,0.8,1.2)
    main.Size=UDim2.fromOffset(320*Settings.Scale,480*Settings.Scale)
    notify("UI scale: "..math.floor(Settings.Scale*100).."%")
end,12)
action(settingsPage,"↔   Center window",function()
    main.Position=UDim2.new(0.5,-main.Size.X.Offset/2,0.5,-main.Size.Y.Offset/2)
    notify("Window centered")
end,13)
action(settingsPage,"↺   Reset appearance",function()
    Settings.BackgroundTransparency=0
    Settings.Rounded=true
    Settings.Scale=1
    applyTheme("Dark")
    main.Size=UDim2.fromOffset(320,480)
    main.BackgroundTransparency=0
    notify("Appearance reset")
end,14)

-- INFO
section(infoPage,"HD ADMIN • VOLTEX STUDIOS",1)
local about=label(infoPage,"Version 2.2.0\n\nInterface based on the supplied HD Admin menu reference.\n\nRightControl: show / hide\nDrag the top bar: move window\n− : minimize\n× : hide window\n\nChat commands begin with ; and target 'me'.",12,UDim2.new(),Settings.Text)
about.Size=UDim2.new(1,-4,0,210)
about.TextWrapped=true
about.TextYAlignment=Enum.TextYAlignment.Top
about.LayoutOrder=2

-- MINIMIZE / CLOSE
local minimized=false
local fullSize=UDim2.fromOffset(320,480)
minimize.MouseButton1Click:Connect(function()
    minimized=not minimized
    content.Visible=not minimized
    main.Size=minimized and UDim2.fromOffset(320,64) or fullSize
    minimize.Text=minimized and "+" or "−"
end)
close.MouseButton1Click:Connect(function() main.Visible=false end)

-- Black circular floating opener
open=create("TextButton",GUI,{
    Name="OpenButton",
    Size=UDim2.fromOffset(54,54),
    Position=UDim2.new(0,18,0.5,-27),
    BackgroundColor3=Color3.fromRGB(0,0,0),
    Text="HD",
    TextColor3=Color3.fromRGB(255,255,255),
    TextSize=16,
    Font=Enum.Font.GothamBlack,
    AutoButtonColor=false
})
create("UICorner",open,{CornerRadius=UDim.new(1,0)})
outline(open,Color3.fromRGB(90,90,95))
open.MouseEnter:Connect(function() tween(open,{BackgroundColor3=Color3.fromRGB(35,35,38)},0.12) end)
open.MouseLeave:Connect(function() tween(open,{BackgroundColor3=Color3.fromRGB(0,0,0)},0.12) end)

local function showWindow()
    main.Visible=not main.Visible
    if main.Visible and minimized then
        minimized=false
        content.Visible=true
        main.Size=fullSize
        minimize.Text="−"
    end
end
open.MouseButton1Click:Connect(showWindow)

UIS.InputBegan:Connect(function(input,processed)
    if not processed and input.KeyCode==Enum.KeyCode.RightControl then
        showWindow()
    end
end)

showPage("Home")
notify("HD Admin v2.2.0 carregado")
print("HD Admin v2.2.0 by Voltex Studios loaded")
