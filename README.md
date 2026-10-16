-- Serviços do Roblox
local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local running = false

-- ⏱️ CONFIGURAÇÃO DE DELAY (Mantido conforme o anterior)
local CANDY_TELEPORT_DELAY = 1.0  -- Espera de 1 segundo entre um doce e outro
local WANDER_TELEPORT_DELAY = 2.0 -- Espera de 2 segundos entre os teleportes aleatórios nas barreiras

-- Interface Gráfica (GUI)
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CandyFarmGui_v4"
ScreenGui.ResetOnSpawn = false

if syn and syn.protect_gui then
    syn.protect_gui(ScreenGui)
    ScreenGui.Parent = CoreGui
elseif gethui then
    ScreenGui.Parent = gethui()
else
    ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
end

-- Frame Principal
local Frame = Instance.new("Frame")
Frame.Parent = ScreenGui
Frame.Size = UDim2.new(0, 210, 0, 110)
Frame.Position = UDim2.new(0.4, 0, 0.3, 0)
Frame.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true

-- Título
local Title = Instance.new("TextLabel")
Title.Parent = Frame
Title.Size = UDim2.new(1, -30, 0, 30)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "Candy Farm"
Title.TextColor3 = Color3.fromRGB(255, 170, 0)
Title.TextSize = 16
Title.Font = Enum.Font.SourceSansBold
Title.TextXAlignment = Enum.TextXAlignment.Left

-- Botão Fechar [X]
local CloseBtn = Instance.new("TextButton")
CloseBtn.Parent = Frame
CloseBtn.Size = UDim2.new(0, 25, 0, 25)
CloseBtn.Position = UDim2.new(1, -28, 0, 3)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font = Enum.Font.SourceSansBold
CloseBtn.TextSize = 14

-- Botão Começar
local StartBtn = Instance.new("TextButton")
StartBtn.Parent = Frame
StartBtn.Size = UDim2.new(0.42, 0, 0, 40)
StartBtn.Position = UDim2.new(0.06, 0, 0.45, 0)
StartBtn.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
StartBtn.Text = "Começar"
StartBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StartBtn.Font = Enum.Font.SourceSansBold
StartBtn.TextSize = 16

-- Botão Parar
local StopBtn = Instance.new("TextButton")
StopBtn.Parent = Frame
StopBtn.Size = UDim2.new(0.42, 0, 0, 40)
StopBtn.Position = UDim2.new(0.52, 0, 0.45, 0)
StopBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
StopBtn.Text = "Parar"
StopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 16

-- Função para buscar os doces
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

-- Função para buscar partes nas barreiras para o teleporte aleatório
local function getRandomBarrierPosition()
    local barriersFolder = workspace:FindFirstChild("barriers", true) or workspace:FindFirstChild("Barriers", true)
    
    if barriersFolder then
        local parts = {}
        for _, child in ipairs(barriersFolder:GetDescendants()) do
            if child:IsA("BasePart") then
                table.insert(parts, child)
            end
        end
        
        if #parts > 0 then
            local randomPart = parts[math.random(1, #parts)]
            return randomPart.CFrame + Vector3.new(math.random(-15, 15), 5, math.random(-15, 15))
        end
    end
    
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local pos = char.HumanoidRootPart.Position
        return CFrame.new(pos + Vector3.new(math.random(-50, 50), 10, math.random(-50, 50)))
    end
    
    return nil
end

-- Função de TP em cima do doce
local function tpToCandy(candy)
    local character = LocalPlayer.Character
    if not character or not character:FindFirstChild("HumanoidRootPart") then return end
    
    local hrp = character.HumanoidRootPart
    local targetCFrame
    
    if candy:IsA("BasePart") then
        targetCFrame = candy.CFrame
    elseif candy:IsA("Model") then
        targetCFrame = candy:GetPivot()
    end
    
    if targetCFrame then
        -- Teleporta 1m de altura (aprox 3.5 studs) acima do doce
        hrp.CFrame = targetCFrame + Vector3.new(0, 3.5, 0)
        
        task.wait(0.15)
        if character:FindFirstChildOfClass("Humanoid") then
            character:FindFirstChildOfClass("Humanoid"):ChangeState(Enum.HumanoidStateType.Freefall)
        end
    end
end

-- Loop Principal
local function startFarming()
    if running then return end
    running = true
    
    task.spawn(function()
        while running do
            local candies = getCandies()
            
            if #candies > 0 then
                for _, candy in ipairs(candies) do
                    if not running then break end
                    
                    if candy and candy.Parent then
                        tpToCandy(candy)
                        task.wait(CANDY_TELEPORT_DELAY)
                    end
                end
            else
                local randomPos = getRandomBarrierPosition()
                if randomPos then
                    local character = LocalPlayer.Character
                    if character and character:FindFirstChild("HumanoidRootPart") then
                        character.HumanoidRootPart.CFrame = randomPos
                    end
                end
                task.wait(WANDER_TELEPORT_DELAY)
            end
            task.wait(0.2)
        end
    end)
end

-- Conexões dos Botões
StartBtn.MouseButton1Click:Connect(function()
    startFarming()
end)

StopBtn.MouseButton1Click:Connect(function()
    running = false
end)

CloseBtn.MouseButton1Click:Connect(function()
    running = false
    ScreenGui:Destroy()
end)
