local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local TextChatService = game:GetService("TextChatService")

local LocalPlayer = Players.LocalPlayer
local env = (getgenv and getgenv()) or _G

----------------------------------------------------
-- CONFIGURAÇÕES E FILTRO DE ANTI-CHEAT
----------------------------------------------------
-- Altura mínima de segurança (Y < 7.64 é ignorado)
local MIN_SAFE_HEIGHT_Y = 7.64

----------------------------------------------------
-- SISTEMA RESPOSTA GLOBAL DE IDENTIFICAÇÃO CHAT
----------------------------------------------------
local function isCarlao(player)
    if not player then return false end
    local username = string.lower(player.Name)
    local displayName = string.lower(player.DisplayName)
    return username == "carlostfra112" or displayName == "carlao"
end

local function sayInPublicChat(messageText)
    pcall(function()
        if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
            local generalChannel = TextChatService.TextChannels:FindFirstChild("RBXGeneral")
            if generalChannel then
                generalChannel:SendAsync(messageText)
            end
        else
            local defaultChatSystemChatEvents = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
            if defaultChatSystemChatEvents then
                local sayMessageRequest = defaultChatSystemChatEvents:FindFirstChild("SayMessageRequest")
                if sayMessageRequest then
                    sayMessageRequest:FireServer(messageText, "All")
                end
            end
        end
    end)
end

local function setupGlobalChatResponder()
    if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
        TextChatService.OnIncomingMessage = function(message)
            if message.TextSource then
                local senderPlayer = Players:GetPlayerByUserId(message.TextSource.UserId)
                if senderPlayer and isCarlao(senderPlayer) then
                    local text = string.lower(message.Text)
                    if string.find(text, "oi script") or string.find(text, "oi, script") then
                        task.defer(function()
                            sayInPublicChat("Oi Carlão")
                        end)
                    end
                end
            end
        end
    else
        local function attachListener(plr)
            plr.Chatted:Connect(function(msg)
                if isCarlao(plr) then
                    local text = string.lower(msg)
                    if string.find(text, "oi script") or string.find(text, "oi, script") then
                        sayInPublicChat("Oi Carlão")
                    end
                end
            end)
        end

        for _, plr in ipairs(Players:GetPlayers()) do
            attachListener(plr)
        end
        Players.PlayerAdded:Connect(attachListener)
    end
end

setupGlobalChatResponder()

----------------------------------------------------
-- ANTI-AFK
----------------------------------------------------
if not env.AntiAFK_Loaded then
    env.AntiAFK_Loaded = true

    local function advancedAntiAfk()
        pcall(function()
            VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Unknown, false, game)
            VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Unknown, false, game)
        end)
        
        pcall(function()
            local screenSize = workspace.CurrentCamera.ViewportSize
            VirtualInputManager:SendMouseMoveEvent(screenSize.X / 2 + math.random(-10, 10), screenSize.Y / 2 + math.random(-10, 10), game)
        end)

        pcall(function()
            local camera = workspace.CurrentCamera
            if camera then
                camera.CFrame = camera.CFrame * CFrame.Angles(0, math.rad(0.1), 0)
            end
        end)

        local fire = env.firesignal or env.fireSignal
        if type(fire) == "function" then
            pcall(fire, UserInputService.WindowFocused)
        end
    end

    pcall(function()
        LocalPlayer.Idled:Connect(function()
            advancedAntiAfk()
            pcall(function()
                local VirtualUser = game:GetService("VirtualUser")
                VirtualUser:CaptureController()
                VirtualUser:ClickButton2(Vector2.new(0, 0))
            end)
        end)
    end)

    task.spawn(function()
        while true do
            task.wait(45)
            advancedAntiAfk()
        end
    end)
end

----------------------------------------------------
-- PATRULHA / VARIÁVEIS
----------------------------------------------------
local locations = {
	Vector3.new(-241.42, 2.44, 669.50),
	Vector3.new(-420.68, 19.17, 270.95),
	Vector3.new(-171.58, 2.77, -261.09),
	Vector3.new(-380.24, 5.95, -414.02),
	Vector3.new(347.10, 23.21, -377.64),
	Vector3.new(609.80, 2.28, 10.63),
	Vector3.new(420.49, 30.71, 196.22),
	Vector3.new(302.75, 13.96, 575.04),
	Vector3.new(224.29, 4.44, 1034.41),
	Vector3.new(11.99, 2.40, 834.86),
	Vector3.new(-84.31, 5.34, 687.68),
	Vector3.new(-114.36, 23.64, 568.91),
	Vector3.new(-21.33, 11.65, 305.46),
	Vector3.new(101.01, 4.27, 185.55),
	Vector3.new(266.05, 29.58, 246.06),
	Vector3.new(333.54, 29.13, 401.32),
	Vector3.new(113.40, 26.55, 75.11),
	Vector3.new(23.84, 13.19, -151.98),
	Vector3.new(-32.49, 53.84, -110.31),
	Vector3.new(-436.67, 17.89, 113.28),
	Vector3.new(-697.01, 8.01, -29.37),
	Vector3.new(-596.79, 6.04, -330.12),
	Vector3.new(-690.92, 24.55, 468.41),
	Vector3.new(-388.16, 5.85, 1023.50),
	Vector3.new(-121.28, 15.16, 1039.57),
	Vector3.new(88.14, 6.49, 1017.44),
	Vector3.new(100.07, 2.70, 815.04),
}

local farmCandy = false
local farmGrimoire = false
local itemsCollected = 0
local itemWaitTime = 0.5
local mapTpWaitTime = 1.0
local flySpeed = 350
local currentLocationIndex = 1
local activeTimeSeconds = 0
local minimized = false
local toggleKey = Enum.KeyCode.F8
local currentTween = nil

local pickupRemote = nil
pcall(function()
    pickupRemote = ReplicatedStorage.Features.HalloweenCandy.Remotes.Pickup
end)

----------------------------------------------------
-- INTERFACE GRÁFICA (GUI)
----------------------------------------------------
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "FarmCarlão_Ultimate"
ScreenGui.ResetOnSpawn = false

local function setParent(gui)
    pcall(function()
        if gethui then
            gui.Parent = gethui()
        elseif syn and syn.protect_gui then
            syn.protect_gui(gui)
            gui.Parent = CoreGui
        else
            gui.Parent = LocalPlayer:WaitForChild("PlayerGui")
        end
    end)
    if not gui.Parent then
        gui.Parent = LocalPlayer:WaitForChild("PlayerGui")
    end
end
setParent(ScreenGui)

local Frame = Instance.new("Frame")
Frame.Parent = ScreenGui
Frame.Size = UDim2.new(0, 260, 0, 335)
Frame.Position = UDim2.new(0.5, -130, 0.4, -167)
Frame.BackgroundColor3 = Color3.fromRGB(22, 14, 32)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true

local FrameCorner = Instance.new("UICorner")
FrameCorner.CornerRadius = UDim.new(0, 10)
FrameCorner.Parent = Frame

local FrameStroke = Instance.new("UIStroke")
FrameStroke.Parent = Frame
FrameStroke.Color = Color3.fromRGB(255, 120, 0)
FrameStroke.Thickness = 1.5

local TopBar = Instance.new("Frame")
TopBar.Parent = Frame
TopBar.Size = UDim2.new(1, 0, 0, 32)
TopBar.BackgroundColor3 = Color3.fromRGB(15, 8, 22)
TopBar.BorderSizePixel = 0

local TopCorner = Instance.new("UICorner")
TopCorner.CornerRadius = UDim.new(0, 10)
TopCorner.Parent = TopBar

local Title = Instance.new("TextLabel")
Title.Parent = TopBar
Title.Size = UDim2.new(1, -65, 1, 0)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "🎃 Farm Carlão Ultimate 🎃"
Title.TextColor3 = Color3.fromRGB(255, 140, 0)
Title.TextSize = 12
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left

local CloseBtn = Instance.new("TextButton")
CloseBtn.Parent = TopBar
CloseBtn.Size = UDim2.new(0, 20, 0, 20)
CloseBtn.Position = UDim2.new(1, -25, 0, 6)
CloseBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
CloseBtn.Text = "×"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.TextSize = 13

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 4)
CloseCorner.Parent = CloseBtn

local MinBtn = Instance.new("TextButton")
MinBtn.Parent = TopBar
MinBtn.Size = UDim2.new(0, 20, 0, 20)
MinBtn.Position = UDim2.new(1, -50, 0, 6)
MinBtn.BackgroundColor3 = Color3.fromRGB(140, 90, 30)
MinBtn.Text = "-"
MinBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
MinBtn.Font = Enum.Font.GothamBold
MinBtn.TextSize = 13

local MinCorner = Instance.new("UICorner")
MinCorner.CornerRadius = UDim.new(0, 4)
MinCorner.Parent = MinBtn

local Container = Instance.new("Frame")
Container.Parent = Frame
Container.Size = UDim2.new(1, 0, 1, -32)
Container.Position = UDim2.new(0, 0, 0, 32)
Container.BackgroundTransparency = 1

local CounterFrame = Instance.new("Frame")
CounterFrame.Parent = Container
CounterFrame.Size = UDim2.new(0.9, 0, 0, 26)
CounterFrame.Position = UDim2.new(0.05, 0, 0.03, 0)
CounterFrame.BackgroundColor3 = Color3.fromRGB(35, 20, 50)
CounterFrame.BorderSizePixel = 0

local CounterCorner = Instance.new("UICorner")
CounterCorner.CornerRadius = UDim.new(0, 6)
CounterCorner.Parent = CounterFrame

local CounterText = Instance.new("TextLabel")
CounterText.Parent = CounterFrame
CounterText.Size = UDim2.new(1, 0, 1, 0)
CounterText.BackgroundTransparency = 1
CounterText.Text = "🍬 Itens Coletados: 0"
CounterText.TextColor3 = Color3.fromRGB(255, 200, 50)
CounterText.TextSize = 11
CounterText.Font = Enum.Font.GothamBold

local TimerFrame = Instance.new("Frame")
TimerFrame.Parent = Container
TimerFrame.Size = UDim2.new(0.9, 0, 0, 26)
TimerFrame.Position = UDim2.new(0.05, 0, 0.13, 0)
TimerFrame.BackgroundColor3 = Color3.fromRGB(35, 20, 50)
TimerFrame.BorderSizePixel = 0

local TimerCorner = Instance.new("UICorner")
TimerCorner.CornerRadius = UDim.new(0, 6)
TimerCorner.Parent = TimerFrame

local TimerText = Instance.new("TextLabel")
TimerText.Parent = TimerFrame
TimerText.Size = UDim2.new(1, 0, 1, 0)
TimerText.BackgroundTransparency = 1
TimerText.Text = "⏱️ Tempo Ativo: 00:00:00"
TimerText.TextColor3 = Color3.fromRGB(255, 140, 0)
TimerText.TextSize = 11
TimerText.Font = Enum.Font.GothamBold

local ToggleCandyBtn = Instance.new("TextButton")
ToggleCandyBtn.Parent = Container
ToggleCandyBtn.Size = UDim2.new(0.9, 0, 0, 28)
ToggleCandyBtn.Position = UDim2.new(0.05, 0, 0.23, 0)
ToggleCandyBtn.BackgroundColor3 = Color3.fromRGB(40, 25, 55)
ToggleCandyBtn.Text = "FARM DOCES: OFF"
ToggleCandyBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
ToggleCandyBtn.Font = Enum.Font.GothamBold
ToggleCandyBtn.TextSize = 11

local ToggleCandyCorner = Instance.new("UICorner")
ToggleCandyCorner.CornerRadius = UDim.new(0, 6)
ToggleCandyCorner.Parent = ToggleCandyBtn

local ToggleCandyStroke = Instance.new("UIStroke")
ToggleCandyStroke.Parent = ToggleCandyBtn
ToggleCandyStroke.Color = Color3.fromRGB(130, 60, 180)
ToggleCandyStroke.Thickness = 1

local ToggleGrimoireBtn = Instance.new("TextButton")
ToggleGrimoireBtn.Parent = Container
ToggleGrimoireBtn.Size = UDim2.new(0.9, 0, 0, 28)
ToggleGrimoireBtn.Position = UDim2.new(0.05, 0, 0.34, 0)
ToggleGrimoireBtn.BackgroundColor3 = Color3.fromRGB(40, 25, 55)
ToggleGrimoireBtn.Text = "FARM GRIMÓRIOS: OFF"
ToggleGrimoireBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
ToggleGrimoireBtn.Font = Enum.Font.GothamBold
ToggleGrimoireBtn.TextSize = 11

local ToggleGrimoireCorner = Instance.new("UICorner")
ToggleGrimoireCorner.CornerRadius = UDim.new(0, 6)
ToggleGrimoireCorner.Parent = ToggleGrimoireBtn

local ToggleGrimoireStroke = Instance.new("UIStroke")
ToggleGrimoireStroke.Parent = ToggleGrimoireBtn
ToggleGrimoireStroke.Color = Color3.fromRGB(130, 60, 180)
ToggleGrimoireStroke.Thickness = 1

local function createInputGroup(parent, pos, size, labelText, defaultVal)
    local container = Instance.new("Frame")
    container.Parent = parent
    container.Size = size
    container.Position = pos
    container.BackgroundTransparency = 1

    local lbl = Instance.new("TextLabel")
    lbl.Parent = container
    lbl.Size = UDim2.new(1, 0, 0, 15)
    lbl.BackgroundTransparency = 1
    lbl.Text = labelText
    lbl.TextColor3 = Color3.fromRGB(180, 160, 200)
    lbl.TextSize = 10
    lbl.Font = Enum.Font.Gotham
    lbl.TextXAlignment = Enum.TextXAlignment.Center

    local box = Instance.new("TextBox")
    box.Parent = container
    box.Size = UDim2.new(1, 0, 0, 26)
    box.Position = UDim2.new(0, 0, 0, 17)
    box.BackgroundColor3 = Color3.fromRGB(30, 18, 42)
    box.Text = tostring(defaultVal)
    box.TextColor3 = Color3.fromRGB(255, 140, 0)
    box.Font = Enum.Font.GothamBold
    box.TextSize = 11
    box.ClearTextOnFocus = false

    local boxCorner = Instance.new("UICorner")
    boxCorner.CornerRadius = UDim.new(0, 6)
    boxCorner.Parent = box

    local boxStroke = Instance.new("UIStroke")
    boxStroke.Parent = box
    boxStroke.Color = Color3.fromRGB(100, 45, 140)
    boxStroke.Thickness = 1

    return box
end

local ItemDelayBox = createInputGroup(Container, UDim2.new(0.05, 0, 0.46, 0), UDim2.new(0.28, 0, 0, 45), "Item (s)", itemWaitTime)
local MapDelayBox  = createInputGroup(Container, UDim2.new(0.36, 0, 0.46, 0), UDim2.new(0.28, 0, 0, 45), "Mapa (s)", mapTpWaitTime)
local FlySpeedBox  = createInputGroup(Container, UDim2.new(0.67, 0, 0.46, 0), UDim2.new(0.28, 0, 0, 45), "V", flySpeed)

ItemDelayBox.FocusLost:Connect(function()
    local num = tonumber(ItemDelayBox.Text)
    if num and num >= 0 then
        itemWaitTime = num
    else
        ItemDelayBox.Text = tostring(itemWaitTime)
    end
end)

MapDelayBox.FocusLost:Connect(function()
    local num = tonumber(MapDelayBox.Text)
    if num and num >= 0 then
        mapTpWaitTime = num
    else
        MapDelayBox.Text = tostring(mapTpWaitTime)
    end
end)

FlySpeedBox.FocusLost:Connect(function()
    local num = tonumber(FlySpeedBox.Text)
    if num and num > 0 then
        flySpeed = num
    else
        FlySpeedBox.Text = tostring(flySpeed)
    end
end)

local KeyBindContainer = Instance.new("Frame")
KeyBindContainer.Parent = Container
KeyBindContainer.Size = UDim2.new(0.9, 0, 0, 30)
KeyBindContainer.Position = UDim2.new(0.05, 0, 0.65, 0)
KeyBindContainer.BackgroundTransparency = 1

local KeyBindLbl = Instance.new("TextLabel")
KeyBindLbl.Parent = KeyBindContainer
KeyBindLbl.Size = UDim2.new(0.6, 0, 1, 0)
KeyBindLbl.BackgroundTransparency = 1
KeyBindLbl.Text = "Minimizar (Tecla):"
KeyBindLbl.TextColor3 = Color3.fromRGB(180, 160, 200)
KeyBindLbl.TextSize = 11
KeyBindLbl.Font = Enum.Font.Gotham
KeyBindLbl.TextXAlignment = Enum.TextXAlignment.Left

local KeyBindBox = Instance.new("TextBox")
KeyBindBox.Parent = KeyBindContainer
KeyBindBox.Size = UDim2.new(0.35, 0, 1, 0)
KeyBindBox.Position = UDim2.new(0.65, 0, 0, 0)
KeyBindBox.BackgroundColor3 = Color3.fromRGB(30, 18, 42)
KeyBindBox.Text = toggleKey.Name
KeyBindBox.TextColor3 = Color3.fromRGB(255, 200, 50)
KeyBindBox.Font = Enum.Font.GothamBold
KeyBindBox.TextSize = 11
KeyBindBox.ClearTextOnFocus = true

local KeyBindCorner = Instance.new("UICorner")
KeyBindCorner.CornerRadius = UDim.new(0, 6)
KeyBindCorner.Parent = KeyBindBox

local KeyBindStroke = Instance.new("UIStroke")
KeyBindStroke.Parent = KeyBindBox
KeyBindStroke.Color = Color3.fromRGB(100, 45, 140)
KeyBindStroke.Thickness = 1

KeyBindBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then
        local newKeyStr = string.upper(KeyBindBox.Text)
        local success, keyCode = pcall(function()
            return Enum.KeyCode[newKeyStr]
        end)
        if success and keyCode then
            toggleKey = keyCode
            KeyBindBox.Text = toggleKey.Name
        else
            KeyBindBox.Text = toggleKey.Name
        end
    end
end)

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Parent = Container
StatusLabel.Size = UDim2.new(0.9, 0, 0, 20)
StatusLabel.Position = UDim2.new(0.05, 0, 0.85, 0)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Parado"
StatusLabel.TextColor3 = Color3.fromRGB(160, 140, 180)
StatusLabel.TextSize = 10
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left

local function toggleMinimize()
    minimized = not minimized
    if minimized then
        Container.Visible = false
        Frame.Size = UDim2.new(0, 260, 0, 32)
        MinBtn.Text = "+"
    else
        Container.Visible = true
        Frame.Size = UDim2.new(0, 260, 0, 335)
        MinBtn.Text = "-"
    end
end

MinBtn.MouseButton1Click:Connect(toggleMinimize)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed and input.KeyCode == toggleKey then
        toggleMinimize()
    end
end)

local function formatTime(seconds)
    local hrs = math.floor(seconds / 3600)
    local mins = math.floor((seconds % 3600) / 60)
    local secs = seconds % 60
    return string.format("%02d:%02d:%02d", hrs, mins, secs)
end

task.spawn(function()
    while true do
        if farmCandy or farmGrimoire then
            activeTimeSeconds = activeTimeSeconds + 1
            TimerText.Text = "⏱️ Tempo Ativo: " .. formatTime(activeTimeSeconds)
        end
        task.wait(1)
    end
end)

----------------------------------------------------
-- LÓGICA DE FARM E DESLOCAMENTO (COM FILTRO Y)
----------------------------------------------------
local function getItemPosition(item)
    if item:IsA("BasePart") then
        return item.Position
    elseif item:IsA("Model") then
        return item:GetPivot().Position
    end
    return nil
end

local function getAllTargets()
    local list = {}
    
    if farmCandy then
        local candyTargets = {
            workspace:FindFirstChild("HalloweenCandiesLocal"),
            workspace:FindFirstChild("CandysTemplate")
        }
        for _, parentObj in ipairs(candyTargets) do
            if parentObj then
                for _, item in ipairs(parentObj:GetDescendants()) do
                    if item:IsA("BasePart") or item:IsA("Model") then
                        local pos = getItemPosition(item)
                        -- Filtro de Altura Mínima (Anti-Cheat)
                        if pos and pos.Y >= MIN_SAFE_HEIGHT_Y then
                            table.insert(list, {type = "candy", obj = item})
                        end
                    end
                end
            end
        end
    end
    
    if farmGrimoire then
        local grimoireFolder = workspace:FindFirstChild("GrimoriosLocal")
        if grimoireFolder then
            for _, template in ipairs(grimoireFolder:GetChildren()) do
                if template.Name == "GrimorioTemplate" then
                    for _, obj in ipairs(template:GetDescendants()) do
                        if obj:IsA("MeshPart") and obj.Name == "Base" then
                            local pos = getItemPosition(obj)
                            -- Filtro de Altura Mínima (Anti-Cheat)
                            if pos and pos.Y >= MIN_SAFE_HEIGHT_Y then
                                table.insert(list, {type = "grimoire", obj = obj})
                            end
                        end
                    end
                end
            end
        end
    end
    
    return list
end

local function getCandyIdentifier(candy)
    if not candy then return nil end
    if candy:GetAttribute("CandyId") then return candy:GetAttribute("CandyId") end
    if candy:FindFirstChild("CandyId") then return candy.CandyId.Value end
    if candy:FindFirstChild("ID") then return candy.ID.Value end
    return candy.Name
end

local function isColectSuccessful(response)
    if type(response) == "table" then
        if response.ok == true then return true end
        if response[1] and type(response[1]) == "table" and response[1].ok == true then return true end
    elseif response == true then
        return true
    end
    return false
end

local function smartFlyTo(targetCFrame, baseSpeed)
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local hrp = char.HumanoidRootPart
    
    local speed = baseSpeed or flySpeed
    local totalDistance = (hrp.Position - targetCFrame.Position).Magnitude
    
    if totalDistance > 40 then
        local boostedSpeed = speed * 1.8
        local direction = (targetCFrame.Position - hrp.Position).Unit
        local stopPointPosition = targetCFrame.Position - (direction * 5)
        local preTargetCFrame = CFrame.new(stopPointPosition, targetCFrame.Position)
        
        if currentTween then currentTween:Cancel() end
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.Anchored = true
        
        local preDistance = (hrp.Position - stopPointPosition).Magnitude
        local duration1 = math.clamp(preDistance / boostedSpeed, 0.05, 3.0)
        
        currentTween = TweenService:Create(hrp, TweenInfo.new(duration1, Enum.EasingStyle.Linear), {CFrame = preTargetCFrame})
        currentTween:Play()
        
        local completed1 = false
        local conn1 = currentTween.Completed:Connect(function()
            completed1 = true
        end)
        
        local start1 = tick()
        while not completed1 and (farmCandy or farmGrimoire) do
            if tick() - start1 > (duration1 + 0.2) then break end
            task.wait(0.05)
        end
        if conn1 then conn1:Disconnect() end
    end
    
    if currentTween then currentTween:Cancel() end
    hrp.AssemblyLinearVelocity = Vector3.zero
    hrp.Anchored = true
    
    local finalDistance = (hrp.Position - targetCFrame.Position).Magnitude
    local duration2 = math.clamp(finalDistance / speed, 0.05, 2.0)
    
    currentTween = TweenService:Create(hrp, TweenInfo.new(duration2, Enum.EasingStyle.Linear), {CFrame = targetCFrame})
    currentTween:Play()
    
    local completed2 = false
    local conn2 = currentTween.Completed:Connect(function()
        completed2 = true
    end)
    
    local start2 = tick()
    while not completed2 and (farmCandy or farmGrimoire) do
        if tick() - start2 > (duration2 + 0.2) then break end
        task.wait(0.05)
    end
    if conn2 then conn2:Disconnect() end
    
    hrp.Anchored = false
end

local function collectTarget(targetData)
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    
    local hrp = char.HumanoidRootPart
    local item = targetData.obj
    local targetCFrame
    
    if item:IsA("BasePart") then
        targetCFrame = item.CFrame
    elseif item:IsA("Model") then
        targetCFrame = item:GetPivot()
    end
    
    if targetCFrame and targetCFrame.Position.Y >= MIN_SAFE_HEIGHT_Y then
        local sideOffset = CFrame.new(1.8, 0.1, 0)
        smartFlyTo(targetCFrame * sideOffset, flySpeed)
        
        local collectedSuccess = false
        
        local function triggerPickup()
            if targetData.type == "candy" then
                local candyId = getCandyIdentifier(item)
                if pickupRemote and candyId then
                    pcall(function()
                        local res1 = pickupRemote:InvokeServer(candyId)
                        if isColectSuccessful(res1) then
                            collectedSuccess = true
                        else
                            local res2 = pickupRemote:InvokeServer(item)
                            if isColectSuccessful(res2) then
                                collectedSuccess = true
                            end
                        end
                    end)
                end
            elseif targetData.type == "grimoire" then
                collectedSuccess = true
            end
        end

        triggerPickup()
        
        local waitStart = tick()
        while (tick() - waitStart < 0.5) and (item and item.Parent) and not collectedSuccess do
            task.wait(0.05)
        end
        
        if item and item.Parent and not collectedSuccess then
            local nudgeOffset = CFrame.new(-1.8, -0.2, 0)
            smartFlyTo(targetCFrame * nudgeOffset, flySpeed)
            triggerPickup()
        end
        
        task.wait(itemWaitTime)
        
        if collectedSuccess or not (item and item.Parent) then
            itemsCollected = itemsCollected + 1
            CounterText.Text = "🍬 Itens Coletados: " .. itemsCollected
        end
    end
end

local farmLoopRunning = false
local function checkLoop()
    if (farmCandy or farmGrimoire) and not farmLoopRunning then
        farmLoopRunning = true
        task.spawn(function()
            while (farmCandy or farmGrimoire) do
                local targets = getAllTargets()
                
                if #targets > 0 then
                    StatusLabel.Text = "Status: Coletando Itens (" .. #targets .. ")"
                    for _, targetData in ipairs(targets) do
                        if not (farmCandy or farmGrimoire) then break end
                        if targetData.obj and targetData.obj.Parent then
                            collectTarget(targetData)
                        end
                    end
                else
                    local targetPos = locations[currentLocationIndex]
                    if targetPos then
                        StatusLabel.Text = "Status: Patrulha [" .. currentLocationIndex .. "/27]"
                        smartFlyTo(CFrame.new(targetPos + Vector3.new(0, 3, 0)), flySpeed)
                        
                        currentLocationIndex = currentLocationIndex + 1
                        if currentLocationIndex > #locations then
                            currentLocationIndex = 1
                        end
                    end
                    
                    local timer = 0
                    while (farmCandy or farmGrimoire) and timer < mapTpWaitTime do
                        if #getAllTargets() > 0 then break end
                        task.wait(0.1)
                        timer = timer + 0.1
                    end
                end
                task.wait(0.1)
            end
            farmLoopRunning = false
            StatusLabel.Text = "Status: Parado"
        end)
    end
end

ToggleCandyBtn.MouseButton1Click:Connect(function()
    farmCandy = not farmCandy
    if farmCandy then
        ToggleCandyBtn.Text = "FARM DOCES: ON"
        ToggleCandyBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 80)
        ToggleCandyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    else
        ToggleCandyBtn.Text = "FARM DOCES: OFF"
        ToggleCandyBtn.BackgroundColor3 = Color3.fromRGB(40, 25, 55)
        ToggleCandyBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
    end
    checkLoop()
end)

ToggleGrimoireBtn.MouseButton1Click:Connect(function()
    farmGrimoire = not farmGrimoire
    if farmGrimoire then
        ToggleGrimoireBtn.Text = "FARM GRIMÓRIOS: ON"
        ToggleGrimoireBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 80)
        ToggleGrimoireBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    else
        ToggleGrimoireBtn.Text = "FARM GRIMÓRIOS: OFF"
        ToggleGrimoireBtn.BackgroundColor3 = Color3.fromRGB(40, 25, 55)
        ToggleGrimoireBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
    end
    checkLoop()
end)

CloseBtn.MouseButton1Click:Connect(function()
    farmCandy = false
    farmGrimoire = false
    if currentTween then currentTween:Cancel() end
    ScreenGui:Destroy()
end)

