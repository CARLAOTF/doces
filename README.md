-- Serviços do Roblox
local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")

local LocalPlayer = Players.LocalPlayer

-- 🆔 ID DA SUA GAME PASS CONFIGURADO
local GAMEPASS_ID = 1823236629  

-- Função para verificar se o jogador possui a Game Pass
local function checkGamepass()
    local success, hasPass = pcall(function()
        return MarketplaceService:UserOwnsGamePassAsync(LocalPlayer.UserId, GAMEPASS_ID)
    end)
    return success and hasPass
end

-- Se o jogador NÃO tiver a Game Pass, abre o menu de compra e bloqueia o script
if not checkGamepass() then
    pcall(function()
        MarketplaceService:PromptGamePassPurchase(LocalPlayer, GAMEPASS_ID)
    end)
    warn("[Candy Farm] Acesso negado: Você precisa comprar a Game Pass para usar este script!")
    return -- Para a execução aqui se não tiver a pass
end

-- =========================================================================
--  SCRIPT PRINCIPAL (SÓ RODA SE TIVER A GAME PASS)
-- =========================================================================

local running = false
local lastPosition = nil
local candiesCollected = 0  -- Mantém salvo o histórico acumulado
local isExploreMode = true  -- true = Explorar Mapa | false = Parado

-- ⏱️ CONFIGURAÇÕES
local TERRAIN_TP_INTERVAL = 1.0  -- Intervalo de 1 segundo entre os TPs aleatórios
local TERRAIN_SAFE_HEIGHT = 40   -- Altura segura acima do chão firme
local MIN_DIST_BETWEEN_TPS = 120 -- Distância mínima para teletransportar longe
local CANDY_TP_HEIGHT = 3.5      -- Altura precisa para pegar os doces
local CANDY_WAIT_TIME = 0.5      -- Tempo exato de espera no doce (0.5s)

-- Remote de Coleta
local pickupRemote = nil
pcall(function()
    pickupRemote = ReplicatedStorage.Features.HalloweenCandy.Remotes.Pickup
end)

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
Frame.Size = UDim2.new(0, 230, 0, 160)
Frame.Position = UDim2.new(0.4, 0, 0.3, 0)
Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
Frame.BorderSizePixel = 0
Frame.Active = true
Frame.Draggable = true

-- Título
local Title = Instance.new("TextLabel")
Title.Parent = Frame
Title.Size = UDim2.new(1, -20, 0, 25)
Title.Position = UDim2.new(0, 10, 0, 5)
Title.BackgroundTransparency = 1
Title.Text = "Candy Farm | Carlão"
Title.TextColor3 = Color3.fromRGB(255, 170, 0)
Title.TextSize, Title.Font = 13, Enum.Font.SourceSansBold
Title.TextXAlignment = Enum.TextXAlignment.Left

-- Contador de Doces
local CounterLabel = Instance.new("TextLabel")
CounterLabel.Parent = Frame
CounterLabel.Size = UDim2.new(1, -20, 0, 20)
CounterLabel.Position = UDim2.new(0, 10, 0, 28)
CounterLabel.BackgroundTransparency = 1
CounterLabel.Text = "Doces Coletados: " .. candiesCollected
CounterLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
CounterLabel.TextSize, CounterLabel.Font = 12, Enum.Font.SourceSans
CounterLabel.TextXAlignment = Enum.TextXAlignment.Left

-- Botão de Alternar Modo
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
StartBtn.Size = UDim2.new(0.42, 0, 0, 38)
StartBtn.Position = UDim2.new(0.06, 0, 0.68, 0)
StartBtn.BackgroundColor3 = Color3.fromRGB(50, 180, 50)
StartBtn.Text = "Começar"
StartBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StartBtn.Font, StartBtn.TextSize = Enum.Font.SourceSansBold, 16

-- Botão Parar
local StopBtn = Instance.new("TextButton")
StopBtn.Parent = Frame
StopBtn.Size = UDim2.new(0.42, 0, 0, 38)
StopBtn.Position = UDim2.new(0.52, 0, 0.68, 0)
StopBtn.BackgroundColor3 = Color3.fromRGB(180, 50, 50)
StopBtn.Text = "Parar"
StopBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
StopBtn.Font, StopBtn.TextSize = Enum.Font.SourceSansBold, 16

-- Busca os doces na Workspace
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

-- Posição aleatória e segura no Terrain
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

-- Extrai o ID ou parâmetro correto do doce
local function getCandyIdentifier(candy)
    if not candy then return nil end
    
    if candy:GetAttribute("CandyId") then return candy:GetAttribute("CandyId") end
    if candy:FindFirstChild("CandyId") then return candy.CandyId.Value end
    if candy:FindFirstChild("ID") then return candy.ID.Value end
    
    return candy.Name
end

-- Valida se a resposta do servidor confirma o sucesso da coleta
local function isColectSuccessful(response)
    if type(response) == "table" then
        if response.ok == true then return true end
        if response[1] and type(response[1]) == "table" and response[1].ok == true then return true end
    elseif response == true then
        return true
    end
    return false
end

-- Teleporta para o doce, dispara o Remote correto e valida a coleta
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
        -- Teleporta para o doce
        hrp.CFrame = targetCFrame + Vector3.new(0, CANDY_TP_HEIGHT, 0)
        
        local candyId = getCandyIdentifier(candy)
        local collectedSuccess = false
        
        -- Invoca o Remote no servidor com o ID dinâmico
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
        
        -- Aguarda o tempo necessário no doce
        task.wait(CANDY_WAIT_TIME)
        
        -- Fallback: Se o Remote funcionou ou se o doce sumiu do jogo, confirma a coleta
        if collectedSuccess or not (candy and candy.Parent) then
            candiesCollected = candiesCollected + 1
            CounterLabel.Text = "Doces Coletados: " .. candiesCollected
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
                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                local camera = workspace.CurrentCamera
                
                -- Checkpoint do local exato antes de ir buscar o doce
                local checkpointCFrame = hrp and hrp.CFrame
                local originalCamCFrame = camera and camera.CFrame
                
                -- Se estiver no modo parado, trava a câmera no lugar fixo
                if not isExploreMode and camera and originalCamCFrame then
                    camera.CameraType = Enum.CameraType.Scriptable
                    camera.CFrame = originalCamCFrame
                end
                
                -- Coleta os doces
                for _, candy in ipairs(candies) do
                    if not running then break end
                    if candy and candy.Parent then
                        collectCandy(candy)
                    end
                end
                
                -- Volta exatamente para o checkpoint
                if checkpointCFrame and hrp and running then
                    hrp.CFrame = checkpointCFrame
                    task.wait(0.1)
                end
                
                -- Restaura a câmera normal após retornar ao corpo
                if camera then
                    camera.CameraType = Enum.CameraType.Custom
                end
            else
                -- Sem doces
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

-- Conexões dos Botões
StartBtn.MouseButton1Click:Connect(function()
    startFarming()
end)

StopBtn.MouseButton1Click:Connect(function()
    running = false
    -- Restaura a câmera se o script for parado no meio de uma coleta
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
    ScreenGui:Destroy()
end)

