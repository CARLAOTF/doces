-- Serviços do Roblox
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local running = false
local candiesCollected = 0

-- Interface Gráfica (GUI)
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CandyFarmGui"
ScreenGui.ResetOnSpawn = false

-- Proteção simples para colocar na CoreGui ou PlayerGui
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
Frame.Size = UDim2.new(0, 220, 0, 180)
Frame.Position = UDim2.new(0.4, 0, 0.3, 0)
Frame.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true -- Permite arrastar o painel

-- Título
local Title = Instance.new("TextLabel")
Title.Parent = Frame
Title.Size = UDim2.new(1, -30, 0, 30)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "Candy Farm"
Title.TextColor3 = Color3.fromRGB(255, 170, 0)
Title.TextSize = 18
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

-- Contador de Doces
local CounterLabel = Instance.new("TextLabel")
CounterLabel.Parent = Frame
CounterLabel.Size = UDim2.new(1, -20, 0, 30)
CounterLabel.Position = UDim2.new(0, 10, 0, 35)
CounterLabel.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
CounterLabel.Text = "Doces Pegos: 0"
CounterLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
CounterLabel.TextSize = 16
CounterLabel.Font = Enum.Font.SourceSans

-- Botão Começar
local StartBtn = Instance.new("TextButton")
StartBtn.Parent = Frame
StartBtn.Size = UDim2.new(0.42, 0, 0, 35)
StartBtn.Position = UDim2.new(0.06, 0, 0.7, 0)
StartBtn.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
StartBtn.Text = "Começar"
StartBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StartBtn.Font = Enum.Font.SourceSansBold
StartBtn.TextSize = 16

-- Botão Terminar / Parar
local StopBtn = Instance.new("TextButton")
StopBtn.Parent = Frame
StopBtn.Size = UDim2.new(0.42, 0, 0, 35)
StopBtn.Position = UDim2.new(0.52, 0, 0.7, 0)
StopBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
StopBtn.Text = "Parar"
StopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StopBtn.Font = Enum.Font.SourceSansBold
StopBtn.TextSize = 16

-- Função para buscar os doces na Workspace
local function getCandies()
    local list = {}
    -- Procura nos alvos especificados
    local targets = {
        workspace:FindFirstChild("HalloweenCandiesLocal"),
        workspace:FindFirstChild("CandysTemplate")
    }
    
    for _, parentObj in ipairs(targets) do
        if parentObj then
            for _, item in ipairs(parentObj:GetDescendants()) do
                -- Garante que o item possui uma parte/posição válida
                if item:IsA("BasePart") or item:IsA("Model") then
                    table.insert(list, item)
                end
            end
        end
    end
    return list
end

-- Função de Teleporte (TP a ~1m/3.5 studs de altura e queda)
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
        -- 1 metro de altura no Roblox equivale a aproximadamente 3.5 a 4 studs de altura no eixo Y
        local offsetCFrame = targetCFrame + Vector3.new(0, 3.5, 0)
        
        -- Teleporta 1m acima
        hrp.CFrame = offsetCFrame
        
        -- Garante a física/queda em cima do doce
        task.wait(0.1)
        if character:FindFirstChildOfClass("Humanoid") then
            character:FindFirstChildOfClass("Humanoid"):ChangeState(Enum.HumanoidStateType.Freefall)
        end
    end
end

-- Loop Principal de Coleta
local function startFarming()
    if running then return end
    running = true
    
    task.spawn(function()
        while running do
            local candies = getCandies()
            if #candies == 0 then
                task.wait(1)
            else
                for _, candy in ipairs(candies) do
                    if not running then break end
                    
                    -- Verifica se o objeto ainda existe no jogo
                    if candy and candy.Parent then
                        tpToCandy(candy)
                        
                        -- Aguarda o tempo do toque/coleta
                        task.wait(0.3)
                        
                        -- Incrementa o contador
                        candiesCollected = candiesCollected + 1
                        CounterLabel.Text = "Doces Pegos: " .. tostring(candiesCollected)
                    end
                end
            end
            task.wait(0.5)
        end
    end)
end

-- Conexão dos Botões
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
