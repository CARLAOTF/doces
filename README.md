local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")

local LocalPlayer = Players.LocalPlayer
local GAMEPASS_ID = 1823236629

-- Função de parentamento universal para prevenir crash em executores de PC
local function getSafeParent()
    local parent = nil
    pcall(function()
        if gethui then
            parent = gethui()
        elseif syn and syn.protect_gui then
            parent = CoreGui
        else
            parent = LocalPlayer:WaitForChild("PlayerGui", 5)
        end
    end)
    return parent or LocalPlayer:FindFirstChildOfClass("PlayerGui") or CoreGui
end

-- =========================================================================
--  ESTRUTURA DA INTERFACE PRINCIPAL DO FARM
-- =========================================================================

local running = false
local lastPosition = nil
local candiesCollected = 0
local isExploreMode = true

local TERRAIN_TP_INTERVAL = 1.0
local TERRAIN_SAFE_HEIGHT = 40
local MIN_DIST_BETWEEN_TPS = 120
local CANDY_TP_HEIGHT = 3.5
local CANDY_WAIT_TIME = 0.5

local pickupRemote = nil
pcall(function()
    pickupRemote = ReplicatedStorage.Features.HalloweenCandy.Remotes.Pickup
end)

local MainGui = Instance.new("ScreenGui")
MainGui.Name = "CandyFarm_Carlao"
MainGui.ResetOnSpawn = false
MainGui.Enabled = false -- Mantém oculto até validar a Game Pass

local Frame = Instance.new("Frame")
Frame.Parent = MainGui
Frame.Size = UDim2.new(0, 230, 0, 160)
Frame.Position = UDim2.new(0.4, 0, 0.3, 0)
Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true

local Title = Instance.new("TextLabel")
Title.Parent = Frame
Title.Size = UDim2.new(1, -20, 0, 25)
Title.Position = UDim2.new(0, 10, 0, 5)
Title.BackgroundTransparency = 1
Title.Text = "Candy Farm | Carlao"
Title.TextColor3 = Color3.fromRGB(255, 170, 0)
Title.TextSize, Title.Font = 13, Enum.Font.SourceSansBold
Title.TextXAlignment = Enum.TextXAlignment.Left

local CounterLabel = Instance.new("TextLabel")
CounterLabel.Parent = Frame
CounterLabel.Size = UDim2.new(1, -20, 0, 20)
CounterLabel.Position = UDim2.new(0, 10, 0, 28)
CounterLabel.BackgroundTransparency = 1
CounterLabel.Text = "Doces Coletados: " .. candiesCollected
CounterLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
CounterLabel.TextSize, CounterLabel.Font = 12, Enum.Font.SourceSans
CounterLabel.TextXAlignment = Enum.TextXAlignment.Left

local ModeBtn = Instance.new("TextButton")
ModeBtn.Parent = Frame
ModeBtn.Size = UDim2.new(1, -20, 0, 28)
ModeBtn.Position = UDim2.new(0, 10, 0, 52)
ModeBtn.BackgroundColor3 = Color3.fromRGB(50, 100, 180)
ModeBtn.Text = "Modo: Explorar Mapa"
ModeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ModeBtn.Font, ModeBtn.TextSize = Enum.Font.SourceSansBold, 12

ModeBtn.MouseButton1Click:Connect(function()
    isExploreMode = not isExploreMode
    if isExploreMode then
        ModeBtn.Text = "Modo: Explorar Mapa"
        ModeBtn.BackgroundColor3 = Color3.fromRGB(50, 100, 180)
    else
        ModeBtn.Text = "Modo: Parado (Esperar Doces)"
        ModeBtn.BackgroundColor3 = Color3.fromRGB(120, 50, 150)
    end
end)

local CloseBtn = Instance.new("TextButton")
CloseBtn.Parent = Frame
CloseBtn.Size = UDim2.new(0, 25, 0, 25)
CloseBtn.Position = UDim2.new(1, -28, 0, 3)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font, CloseBtn.TextSize = Enum.Font.SourceSansBold, 14

local StartBtn = Instance.new("TextButton")
StartBtn.Parent = Frame
StartBtn.Size = UDim2.new(0.42, 0, 0, 38)
StartBtn.Position = UDim2.new(0.06, 0, 0.68, 0)
StartBtn.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
StartBtn.Text = "Comecar"
StartBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StartBtn.Font, StartBtn.TextSize = Enum.Font.SourceSansBold, 16

local StopBtn = Instance.new("TextButton")
StopBtn.Parent = Frame
StopBtn.Size = UDim2.new(0.42, 0, 0, 38)
StopBtn.Position = UDim2.new(0.52, 0, 0.68, 0)
StopBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
StopBtn.Text = "Parar"
StopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StopBtn.Font, StopBtn.TextSize = Enum.Font.SourceSansBold, 16

MainGui.Parent = getSafeParent()

-- =========================================================================
--  TELA DE BLOQUEIO DA GAME PASS
-- =========================================================================

local LockGui = Instance.new("ScreenGui")
LockGui.Name = "CandyFarm_LockScreen"
LockGui.ResetOnSpawn = false
LockGui.Enabled = false

local LockFrame = Instance.new("Frame")
LockFrame.Parent = LockGui
LockFrame.Size = UDim2.new(0, 260, 0, 140)
LockFrame.Position = UDim2.new(0.5, -130, 0.4, -70)
LockFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
LockFrame.BorderSizePixel = 0
LockFrame.Active = true
LockFrame.Draggable = true

local LockTitle = Instance.new("TextLabel")
LockTitle.Parent = LockFrame
LockTitle.Size = UDim2.new(1, -20, 0, 30)
LockTitle.Position = UDim2.new(0, 10, 0, 10)
LockTitle.BackgroundTransparency = 1
LockTitle.Text = "Candy Farm | Carlao"
LockTitle.TextColor3 = Color3.fromRGB(255, 170, 0)
LockTitle.TextSize, LockTitle.Font = 14, Enum.Font.SourceSansBold

local LockNotice = Instance.new("TextLabel")
LockNotice.Parent = LockFrame
LockNotice.Size = UDim2.new(1, -20, 0, 20)
LockNotice.Position = UDim2.new(0, 10, 0, 38)
LockNotice.BackgroundTransparency = 1
LockNotice.Text = "Verificando Game Pass..."
LockNotice.TextColor3 = Color3.fromRGB(200, 200, 200)
LockNotice.TextSize, LockNotice.Font = 11, Enum.Font.SourceSans

local BuyBtn = Instance.new("TextButton")
BuyBtn.Parent = LockFrame
BuyBtn.Size = UDim2.new(0.9, 0, 0, 40)
BuyBtn.Position = UDim2.new(0.05, 0, 0.6, 0)
BuyBtn.BackgroundColor3 = Color3.fromRGB(0, 162, 232)
BuyBtn.Text = "Comprar Game Pass para desbloquear"
BuyBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
BuyBtn.TextWrapped = true
BuyBtn.Font, BuyBtn.TextSize = Enum.Font.SourceSansBold, 12

local CloseLockBtn = Instance.new("TextButton")
CloseLockBtn.Parent = LockFrame
CloseLockBtn.Size = UDim2.new(0, 25, 0, 25)
CloseLockBtn.Position = UDim2.new(1, -28, 0, 3)
CloseLockBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseLockBtn.Text = "X"
CloseLockBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseLockBtn.Font, CloseLockBtn.TextSize = Enum.Font.SourceSansBold, 14

LockGui.Parent = getSafeParent()

CloseLockBtn.MouseButton1Click:Connect(function()
    LockGui:Destroy()
end)

BuyBtn.MouseButton1Click:Connect(function()
    pcall(function()
        MarketplaceService:PromptGamePassPurchase(LocalPlayer, GAMEPASS_ID)
    end)
end)

-- Verificação em segundo plano sem travar o script
task.spawn(function()
    local hasPass = false
    local success = pcall(function()
        hasPass = MarketplaceService:UserOwnsGamePassAsync(LocalPlayer.UserId, GAMEPASS_ID)
    end)

    if success and hasPass then
        MainGui.Enabled = true
        LockGui:Destroy()
    else
        LockNotice.Text = "Acesso bloqueado: Game Pass necessaria!"
        LockGui.Enabled = true
    end
end)

-- =========================================================================
--  LÓGICA DE COLETA E MODO PARADO COM CÂMERA
-- =========================================================================

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

local function getValidMapPosition()
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if not terrain then return nil end
    
    local success, cframe, size = pcall(function()
        return terrain:GetBoundingBox()
    end)
    
    if success and cframe and size then
        for _ = 1, 15 do
            local rx = math.random(-size.X * 0.35, size.X * 0.35)
            local rz = math.random(-size.Z * 0.35, size.Z * 0.35)
            
            local origin = cframe.Position + Vector3.new(rx, size.Y / 2 + 100, rz)
            local raycastParams = RaycastParams.new()
            raycastParams.IgnoreWater = true
            
            local result = workspace:Raycast(origin, Vector3.new(0, -size.Y * 2 - 200, 0), raycastParams)
            
            if result and result.Instance then
                local candidatePos = result.Position + Vector3.new(0, TERRAIN_SAFE_HEIGHT, 0)
                
                if not lastPosition or (candidatePos - lastPosition).Magnitude >= MIN_DIST_BETWEEN_TPS then
                    return candidatePos
                end
            end
        end
    end
    
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local angle = math.rad(math.random(0, 360))
        local dist = math.random(80, 150)
        local offset = Vector3.new(math.cos(angle) * dist, 100, math.sin(angle) * dist)
        local origin = char.HumanoidRootPart.Position + offset
        
        local raycastParams = RaycastParams.new()
        raycastParams.IgnoreWater = true
        local result = workspace:Raycast(origin, Vector3.new(0, -300, 0), raycastParams)
        if result then
            return result.Position + Vector3.new(0, TERRAIN_SAFE_HEIGHT, 0)
        end
    end
    
    return nil
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
        hrp.CFrame = targetCFrame + Vector3.new(0, CANDY_TP_HEIGHT, 0)
        
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
        
        task.wait(CANDY_WAIT_TIME)
        
        if collectedSuccess or not (candy and candy.Parent) then
            candiesCollected = candiesCollected + 1
            CounterLabel.Text = "Doces Coletados: " .. candiesCollected
        end
    end
end

local function startFarming()
    if running then return end
    running = true
    
    task.spawn(function()
        while running do
            local candies = getCandies()
            
            if #candies > 0 then
                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                local camera = workspace.CurrentCamera
                
                local checkpointCFrame = hrp and hrp.CFrame
                local originalCamCFrame = camera and camera.CFrame
                
                if not isExploreMode and camera and originalCamCFrame then
                    camera.CameraType = Enum.CameraType.Scriptable
                    camera.CFrame = originalCamCFrame
                end
                
                for _, candy in ipairs(candies) do
                    if not running then break end
                    if candy and candy.Parent then
                        collectCandy(candy)
                    end
                end
                
                if checkpointCFrame and hrp and running then
                    hrp.CFrame = checkpointCFrame
                    task.wait(0.1)
                end
                
                if camera then
                    camera.CameraType = Enum.CameraType.Custom
                end
            else
                if isExploreMode then
                    local safePos = getValidMapPosition()
                    local char = LocalPlayer.Character
                    
                    if safePos and char and char:FindFirstChild("HumanoidRootPart") then
                        char.HumanoidRootPart.CFrame = CFrame.new(safePos)
                        lastPosition = safePos
                    end
                    
                    local timer = 0
                    while running and timer < TERRAIN_TP_INTERVAL do
                        if #getCandies() > 0 then
                            break
                        end
                        task.wait(0.1)
                        timer = timer + 0.1
                    end
                else
                    task.wait(0.5)
                end
            end
            task.wait(0.1)
        end
    end)
end

StartBtn.MouseButton1Click:Connect(function()
    startFarming()
end)

StopBtn.MouseButton1Click:Connect(function()
    running = false
    local camera = workspace.CurrentCamera
    if camera then
        camera.CameraType = Enum.CameraType.Custom
    end
end)

CloseBtn.MouseButton1Click:Connect(function()
    running = false
    local camera = workspace.CurrentCamera
    if camera then
        camera.CameraType = Enum.CameraType.Custom
    end
    MainGui:Destroy()
end)
