local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer
local env = (getgenv and getgenv()) or _G

if not env.AntiAFK_Loaded then
    env.AntiAFK_Loaded = true

    local function antiAfkPulse()
        pcall(function()
            VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Unknown, false, game)
            VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Unknown, false, game)
        end)
        
        pcall(function()
            local position = UserInputService:GetMouseLocation()
            VirtualInputManager:SendMouseMoveEvent(position.X, position.Y, game)
        end)

        local fire = env.firesignal or env.fireSignal
        if type(fire) == "function" then
            pcall(fire, UserInputService.WindowFocused)
        end
    end

    pcall(function()
        LocalPlayer.Idled:Connect(function()
            antiAfkPulse()
            pcall(function()
                local VirtualUser = game:GetService("VirtualUser")
                VirtualUser:CaptureController()
                VirtualUser:ClickButton2(Vector2.new(0, 0))
            end)
        end)
    end)

    task.spawn(function()
        while true do
            task.wait(60)
            antiAfkPulse()
        end
    end)
end

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

local running = false
local candiesCollected = 0
local candyWaitTime = 0.5
local mapTpWaitTime = 1.0
local currentLocationIndex = 1
local activeTimeSeconds = 0

local pickupRemote = nil
pcall(function()
    pickupRemote = ReplicatedStorage.Features.HalloweenCandy.Remotes.Pickup
end)

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CandyFarm_Carlao"
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
Frame.Size = UDim2.new(0, 260, 0, 250)
Frame.Position = UDim2.new(0.5, -130, 0.4, -125)
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
Title.Size = UDim2.new(1, -40, 1, 0)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "🎃 Farm Candy Carlão 🎃"
Title.TextColor3 = Color3.fromRGB(255, 140, 0)
Title.TextSize = 13
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left

local CloseBtn = Instance.new("TextButton")
CloseBtn.Parent = TopBar
CloseBtn.Size = UDim2.new(0, 22, 0, 22)
CloseBtn.Position = UDim2.new(1, -27, 0, 5)
CloseBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
CloseBtn.Text = "×"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.TextSize = 14

local CloseCorner = Instance.new("UICorner")
CloseCorner.CornerRadius = UDim.new(0, 5)
CloseCorner.Parent = CloseBtn

local CounterFrame = Instance.new("Frame")
CounterFrame.Parent = Frame
CounterFrame.Size = UDim2.new(0.9, 0, 0, 28)
CounterFrame.Position = UDim2.new(0.05, 0, 0.16, 0)
CounterFrame.BackgroundColor3 = Color3.fromRGB(35, 20, 50)
CounterFrame.BorderSizePixel = 0

local CounterCorner = Instance.new("UICorner")
CounterCorner.CornerRadius = UDim.new(0, 6)
CounterCorner.Parent = CounterFrame

local CounterText = Instance.new("TextLabel")
CounterText.Parent = CounterFrame
CounterText.Size = UDim2.new(1, 0, 1, 0)
CounterText.BackgroundTransparency = 1
CounterText.Text = "🍬 Doces Coletados: 0"
CounterText.TextColor3 = Color3.fromRGB(255, 200, 50)
CounterText.TextSize = 12
CounterText.Font = Enum.Font.GothamBold

local TimerFrame = Instance.new("Frame")
TimerFrame.Parent = Frame
TimerFrame.Size = UDim2.new(0.9, 0, 0, 28)
TimerFrame.Position = UDim2.new(0.05, 0, 0.29, 0)
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
TimerText.TextSize = 12
TimerText.Font = Enum.Font.GothamBold

local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Parent = Frame
ToggleBtn.Size = UDim2.new(0.9, 0, 0, 32)
ToggleBtn.Position = UDim2.new(0.05, 0, 0.43, 0)
ToggleBtn.BackgroundColor3 = Color3.fromRGB(40, 25, 55)
ToggleBtn.Text = "AUTO FARM: OFF"
ToggleBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
ToggleBtn.Font = Enum.Font.GothamBold
ToggleBtn.TextSize = 12

local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(0, 6)
ToggleCorner.Parent = ToggleBtn

local ToggleStroke = Instance.new("UIStroke")
ToggleStroke.Parent = ToggleBtn
ToggleStroke.Color = Color3.fromRGB(130, 60, 180)
ToggleStroke.Thickness = 1

local function createInputGroup(parent, pos, labelText, defaultVal)
    local container = Instance.new("Frame")
    container.Parent = parent
    container.Size = UDim2.new(0.42, 0, 0, 45)
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
    box.Size = UDim2.new(1, 0, 0, 28)
    box.Position = UDim2.new(0, 0, 0, 17)
    box.BackgroundColor3 = Color3.fromRGB(30, 18, 42)
    box.Text = tostring(defaultVal)
    box.TextColor3 = Color3.fromRGB(255, 140, 0)
    box.Font = Enum.Font.GothamBold
    box.TextSize = 12
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

local CandyDelayBox = createInputGroup(Frame, UDim2.new(0.05, 0, 0.59, 0), "Delay Doce (s)", candyWaitTime)
local MapDelayBox = createInputGroup(Frame, UDim2.new(0.53, 0, 0.59, 0), "Delay Mapa (s)", mapTpWaitTime)

CandyDelayBox.FocusLost:Connect(function()
    local num = tonumber(CandyDelayBox.Text)
    if num and num >= 0 then
        candyWaitTime = num
    else
        CandyDelayBox.Text = tostring(candyWaitTime)
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

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Parent = Frame
StatusLabel.Size = UDim2.new(0.9, 0, 0, 20)
StatusLabel.Position = UDim2.new(0.05, 0, 0.88, 0)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Parado"
StatusLabel.TextColor3 = Color3.fromRGB(160, 140, 180)
StatusLabel.TextSize = 10
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left

local function formatTime(seconds)
    local hrs = math.floor(seconds / 3600)
    local mins = math.floor((seconds % 3600) / 60)
    local secs = seconds % 60
    return string.format("%02d:%02d:%02d", hrs, mins, secs)
end

task.spawn(function()
    while true do
        if running then
            activeTimeSeconds = activeTimeSeconds + 1
            TimerText.Text = "⏱️ Tempo Ativo: " .. formatTime(activeTimeSeconds)
        end
        task.wait(1)
    end
end)

local function getCandies()
    local list = {}
    local targets = {
        workspace:FindFirstChild("HalloweenCandiesLocal"),
        workspace:FindFirstChild("CandysTemplate")
    }
    
    for _, parentObj in ipairs(targets) do
        if parentObj then
            for _, item in ipairs(parentObj:GetDescendants()) do
                if item:IsA("BasePart") or item:IsA("Model") then
                    table.insert(list, item)
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

local function collectCandy(candy)
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    
    local hrp = char.HumanoidRootPart
    local targetCFrame
    
    if candy:IsA("BasePart") then
        targetCFrame = candy.CFrame
    elseif candy:IsA("Model") then
        targetCFrame = candy:GetPivot()
    end
    
    if targetCFrame then
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.CFrame = targetCFrame + Vector3.new(0, 0.2, 0)
        
        local candyId = getCandyIdentifier(candy)
        local collectedSuccess = false
        
        if pickupRemote and candyId then
            pcall(function()
                local res1 = pickupRemote:InvokeServer(candyId)
                if isColectSuccessful(res1) then
                    collectedSuccess = true
                else
                    local res2 = pickupRemote:InvokeServer(candy)
                    if isColectSuccessful(res2) then
                        collectedSuccess = true
                    end
                end
            end)
        end
        
        if not collectedSuccess and candy and candy.Parent then
            hrp.CFrame = targetCFrame
            hrp.AssemblyLinearVelocity = Vector3.zero
            task.wait(0.1)
            if pickupRemote and candyId then
                pcall(function()
                    local resRetry = pickupRemote:InvokeServer(candyId)
                    if isColectSuccessful(resRetry) then
                        collectedSuccess = true
                    end
                end)
            end
        end
        
        task.wait(candyWaitTime)
        
        if collectedSuccess or not (candy and candy.Parent) then
            candiesCollected = candiesCollected + 1
            CounterText.Text = "🍬 Doces Coletados: " .. candiesCollected
        end
    end
end

local function startFarming()
    task.spawn(function()
        while running do
            local candies = getCandies()
            
            if #candies > 0 then
                StatusLabel.Text = "Status: Coletando Doces (" .. #candies .. ")"
                for _, candy in ipairs(candies) do
                    if not running then break end
                    if candy and candy.Parent then
                        collectCandy(candy)
                    end
                end
            else
                local targetPos = locations[currentLocationIndex]
                local char = LocalPlayer.Character
                
                if targetPos and char and char:FindFirstChild("HumanoidRootPart") then
                    StatusLabel.Text = "Status: Procurando em [" .. currentLocationIndex .. "/27]"
                    char.HumanoidRootPart.AssemblyLinearVelocity = Vector3.zero
                    char.HumanoidRootPart.CFrame = CFrame.new(targetPos)
                    
                    currentLocationIndex = currentLocationIndex + 1
                    if currentLocationIndex > #locations then
                        currentLocationIndex = 1
                    end
                end
                
                local timer = 0
                while running and timer < mapTpWaitTime do
                    if #getCandies() > 0 then break end
                    task.wait(0.1)
                    timer = timer + 0.1
                end
            end
            task.wait(0.1)
        end
        StatusLabel.Text = "Status: Parado"
    end)
end

ToggleBtn.MouseButton1Click:Connect(function()
    running = not running
    if running then
        ToggleBtn.Text = "AUTO FARM: ON"
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 80)
        ToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        startFarming()
    else
        ToggleBtn.Text = "AUTO FARM: OFF"
        ToggleBtn.BackgroundColor3 = Color3.fromRGB(40, 25, 55)
        ToggleBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
    end
end)

CloseBtn.MouseButton1Click:Connect(function()
    running = false
    ScreenGui:Destroy()
end)
