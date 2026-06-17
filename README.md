--[[
    Script Desenvolvido por: Peixoto_TZ
    Funções: Fly, Noclip, Hitbox Changer, Aimbot, FOV, ESP
    Framework de UI: Orion Library
--]]

-- // Serviços do Roblox
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- // Inicialização da Orion Library (UI Escura e Minimalista)
local OrionLib = loadstring(game:HttpGet(('https://raw.githubusercontent.com/shlexware/Orion/main/source')))()
local Window = OrionLib:MakeWindow({Name = "Peixoto_TZ Hub", HidePremium = false, SaveConfig = true, ConfigFolder = "PeixotoConfig"})

-- // Variáveis de Estado (Toggles)
local States = {
    Fly = false,
    Noclip = false,
    Hitbox = false,
    Aimbot = false,
    ESP = false,
    FOV = false
}

-- // Configurações de Valores
local HitboxSize = 2
local FOVRadius = 150
local FlySpeed = 50

-- // Elementos do FOV (Drawing API)
local FOVCircle = Drawing.new("Circle")
FOVCircle.Color = Color3.fromRGB(255, 255, 255)
FOVCircle.Thickness = 1
FOVCircle.NumSides = 60
FOVCircle.Radius = FOVRadius
FOVCircle.Filled = false
FOVCircle.Visible = false

-- // Conexões de Loops (para evitar memory leak)
local noclipConnection
local flyConnection
local aimbotConnection
local fovConnection

--- // 1. SISTEMA DE FLY (MODO VOAR)
local function ToggleFly(state)
    States.Fly = state
    if flyConnection then flyConnection:Disconnect() end
    
    if States.Fly then
        local character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
        local root = character:WaitForChild("HumanoidRootPart")
        
        -- Cria uma força para anular a gravidade temporariamente
        local bv = Instance.new("BodyVelocity")
        bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
        bv.Velocity = Vector3.new(0, 0, 0)
        bv.Parent = root
        
        flyConnection = RunService.RenderStepped:Connect(function()
            if not States.Fly or not root or not root.Parent then 
                bv:Destroy()
                flyConnection:Disconnect()
                return 
            end
            
            -- Lógica de movimentação 3D baseada na Câmera
            local camCFrame = Camera.CFrame
            local direction = Vector3.new()
            
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then direction = direction + camCFrame.LookVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then direction = direction - camCFrame.LookVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then direction = direction - camCFrame.RightVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then direction = direction + camCFrame.RightVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then direction = direction + Vector3.new(0, 1, 0) end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then direction = direction - Vector3.new(0, 1, 0) end
            
            bv.Velocity = direction.Magnitude > 0 and direction.Unit * FlySpeed or Vector3.new(0, 0, 0)
        end)
    end
end

--- // 2. SISTEMA DE NOCLIP (ATRAVESSAR PAREDES)
local function ToggleNoclip(state)
    States.Noclip = state
    if noclipConnection then noclipConnection:Disconnect() end
    
    if States.Noclip then
        noclipConnection = RunService.Stepped:Connect(function()
            if not States.Noclip then 
                noclipConnection:Disconnect() 
                return 
            end
            local character = LocalPlayer.Character
            if character then
                for _, part in ipairs(character:GetChildren()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = false
                    end
                end
            end
        end)
    end
end

--- // 3. SISTEMA DE HITBOX (MODIFICADOR)
local function UpdateHitboxes()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local root = player.Character:FindFirstChild("HumanoidRootPart")
            if root then
                if States.Hitbox then
                    root.Size = Vector3.new(HitboxSize, HitboxSize, HitboxSize)
                    root.Transparency = 0.7
                    root.Color = Color3.fromRGB(255, 0, 0)
                    root.CanCollide = false
                else
                    -- Reseta para o padrão
                    root.Size = Vector3.new(2, 2, 1)
                    root.Transparency = 1
                    root.CanCollide = true
                end
            end
        end
    end
end

--- // 4. AUXÍLIOS: AIMBOT, FOV & ESP

-- Função auxiliar para achar o jogador mais próximo dentro do FOV
local function GetClosestPlayer()
    local closestPlayer = nil
    local shortestDistance = math.huge
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChildOfClass("Humanoid").Health > 0 then
            local root = player.Character.HumanoidRootPart
            local pos, onScreen = Camera:WorldToViewportPoint(root.Position)
            
            if onScreen then
                local mousePos = UserInputService:GetMouseLocation()
                local distance = (Vector2.new(pos.X, pos.Y) - mousePos).Magnitude
                
                -- Se o FOV estiver ativo, limita pelo raio do círculo
                if States.FOV and distance > FOVRadius then continue end
                
                if distance < shortestDistance then
                    closestPlayer = player
                    shortestDistance = distance
                end
            end
        end
    end
    return closestPlayer
end

-- Gerenciador do Aimbot
local function ToggleAimbot(state)
    States.Aimbot = state
    if aimbotConnection then aimbotConnection:Disconnect() end
    
    if States.Aimbot then
        aimbotConnection = RunService.RenderStepped:Connect(function()
            if not States.Aimbot then aimbotConnection:Disconnect() return end
            
            local target = GetClosestPlayer()
            if target and target.Character and target.Character:FindFirstChild("HumanoidRootPart") then
                -- Suavização básica (Tween/Lerp) para grudar a câmera no alvo
                local targetPos = target.Character.HumanoidRootPart.Position
                Camera.CFrame = CFrame.new(Camera.CFrame.Position, targetPos)
            end
        end)
    end
end

-- Gerenciador do ESP (Usa Highlights nativos do Roblox)
local function ToggleESP(state)
    States.ESP = state
    
    local function applyESP(character)
        if not character then return end
        local highlight = character:FindFirstChild("ESPHighlight")
        if state then
            if not highlight then
                highlight = Instance.new("Highlight")
                highlight.Name = "ESPHighlight"
                highlight.FillColor = Color3.fromRGB(255, 0, 100)
                highlight.FillTransparency = 0.5
                highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
                highlight.OutlineTransparency = 0
                highlight.Adornee = character
                highlight.Parent = character
            end
        else
            if highlight then highlight:Destroy() end
        end
    end

    -- Aplica nos players atuais e futuros
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then applyESP(p.Character) end
        p.CharacterAdded:Connect(function(char) if States.ESP then applyESP(char) end end)
    end
end

-- Gerenciador do Círculo de FOV
if RunService:IsClient() then
    fovConnection = RunService.RenderStepped:Connect(function()
        if States.FOV then
            FOVCircle.Position = UserInputService:GetMouseLocation()
            FOVCircle.Radius = FOVRadius
            FOVCircle.Visible = true
        else
            FOVCircle.Visible = false
        end
    end)
end


-- // ==========================================
-- // CONSTRUÇÃO DA INTERFACE GRÁFICA (TABS & BOTÕES)
-- // ==========================================

-- Aba Principal (Movimentação)
local TabMovement = Window:NewTab("Movimentação")
local SectionMov = TabMovement:NewSection({"Funções de Locomoção"})

SectionMov:NewToggle("Ativar Modo Voar (Fly)", "Permite voar pelo mapa (W,A,S,D + Espaço/Shift)", function(state)
