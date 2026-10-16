-- Serviços do Roblox
local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local running = false
local lastPosition = nil

-- ⏱️ CONFIGURAÇÕES
local TERRAIN_TP_INTERVAL = 1.0  -- Intervalo de 1 segundo entre os TPs aleatórios
local TERRAIN_SAFE_HEIGHT = 40   -- Altura segura acima do chão firme
local MIN_DIST_BETWEEN_TPS = 120 -- Distância mínima para teletransportar longe
local CANDY_TP_HEIGHT = 3.5      -- Altura precisa para pegar os doces

-- Interface Gráfica (GUI)
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "CandyFarm_Carlao"
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
Frame.Size = UDim2.new(0, 220, 0, 110)
Frame.Position = UDim2.new(0.4, 0, 0.3, 0)
Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true

-- Título
local Title = Instance.new("TextLabel")
Title.Parent = Frame
Title.Size = UDim2.new(1, -20, 0, 30)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "Candy Farm | Carlão"
Title.TextColor3 = Color3.fromRGB(255, 170, 0)
Title.TextSize, Title.Font = 13, Enum.Font.SourceSansBold
Title.TextXAlignment = Enum.TextXAlignment.Left

-- Botão Fechar [X]
local CloseBtn = Instance.new("TextButton")
CloseBtn.Parent = Frame
CloseBtn.Size = UDim2.new(0, 25, 0, 25)
CloseBtn.Position = UDim2.new(1, -28, 0, 3)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.Font, CloseBtn.TextSize = Enum.Font.SourceSansBold, 14

-- Botão Começar
local StartBtn = Instance.new("TextButton")
StartBtn.Parent = Frame
StartBtn.Size = UDim2.new(0.42, 0, 0, 40)
StartBtn.Position = UDim2.new(0.06, 0, 0.45, 0)
StartBtn.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
StartBtn.Text = "Começar"
StartBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StartBtn.Font, StartBtn.TextSize = Enum.Font.SourceSansBold, 16

-- Botão Parar
local StopBtn = Instance.new("TextButton")
StopBtn.Parent = Frame
StopBtn.Size = UDim2.new(0.42, 0, 0, 40)
StopBtn.Position = UDim2.new(0.52, 0, 0.45, 0)
StopBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
StopBtn.Text = "Parar"
StopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StopBtn.Font, StopBtn.TextSize = Enum.Font.SourceSansBold, 16

-- Função para buscar os doces na Workspace
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

-- Gera uma posição jogável e segura dentro dos limites do chão firme
local function getValidMapPosition()
    local terrain = workspace:FindFirstChildOfClass("Terrain")
    if not terrain then return nil end
    
    local success, cframe, size = pcall(function()
        return terrain:GetBoundingBox()
    end)
    
    if success and cframe and size then
        -- Tenta até 15 vezes achar um ponto válido COM CHÃO FIRME dentro da área jogável
        for _ = 1, 15 do
            -- Reduz a área de busca (0.35) para ignorar os limites externos/vazio do mapa
            local rx = math.random(-size.X * 0.35, size.X * 0.35)
            local rz = math.random(-size.Z * 0.35, size.Z * 0.35)
            
            local origin = cframe.Position + Vector3.new(rx, size.Y / 2 + 100, rz)
            local raycastParams = RaycastParams.new()
            raycastParams.IgnoreWater = true
            
            local result = workspace:Raycast(origin, Vector3.new(0, -size.Y * 2 - 200, 0), raycastParams)
            
            -- Garante que o raio acertou alguma coisa (chão) e que não é água nem ar
            if result and result.Instance then
                local candidatePos = result.Position + Vector3.new(0, TERRAIN_SAFE_HEIGHT, 0)
                
                -- Checa se é longe da posição anterior para não ficar preso no mesmo lugar
                if not lastPosition or (candidatePos - lastPosition).Magnitude >= MIN_DIST_BETWEEN_TPS then
                    return candidatePos
                end
            end
        end
    end
    
    -- Fallback: Se não achar ponto distante no Terrain, sorteia uma direção a partir da posição atual do jogador
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

-- Teleporta para o doce com precisão
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
        lastPosition = hrp.Position
        
        task.wait(0.1)
        if char:FindFirstChildOfClass("Humanoid") then
            char:FindFirstChildOfClass("Humanoid"):ChangeState(Enum.HumanoidStateType.Freefall)
        end
        
        local startTime = tick()
        while running and candy and candy.Parent do
            if tick() - startTime > 1.2 then
                break
            end
            task.wait(0.1)
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
                -- Se houver doces, coleta todos um por um
                for _, candy in ipairs(candies) do
                    if not running then break end
                    if candy and candy.Parent then
                        collectCandy(candy)
                    end
                end
            else
                -- Se não houver doces, dá TP em uma área interna e segura do terreno
                local safePos = getValidMapPosition()
                local char = LocalPlayer.Character
                
                if safePos and char and char:FindFirstChild("HumanoidRootPart") then
                    char.HumanoidRootPart.CFrame = CFrame.new(safePos)
                    lastPosition = safePos
                    
                    task.wait(0.1)
                    if char:FindFirstChildOfClass("Humanoid") then
                        char:FindFirstChildOfClass("Humanoid"):ChangeState(Enum.HumanoidStateType.Freefall)
                    end
                end
                
                -- Aguarda 1 segundo antes do próximo teleporte
                local timer = 0
                while running and timer < TERRAIN_TP_INTERVAL do
                    if #getCandies() > 0 then
                        break
                    end
                    task.wait(0.1)
                    timer = timer + 0.1
                end
            end
            task.wait(0.1)
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
