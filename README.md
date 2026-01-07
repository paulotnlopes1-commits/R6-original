-- Coloque este script dentro de um TextButton (StarterGui > ScreenGui > Frame > Button)

local button = script.Parent

local function oof()    
    local player = game.Players.LocalPlayer    
    local oldChar = player.Character    
    
    -- Verifica se o personagem existe e se já não é um R6 transformado
    if not oldChar or oldChar.Name:find("_R6") then return end 
    
    local oldHum = oldChar:FindFirstChildOfClass("Humanoid")    
    local oldRoot = oldChar:FindFirstChild("HumanoidRootPart")        
    if not oldRoot or not oldHum then return end

    -- 1. CRIAR O MODELO R6
    -- Usamos pcall para evitar erro caso o ID falhe ao carregar
    local success, r6Asset = pcall(function()
        return game:GetObjects("rbxassetid://1561389244")[1]
    end)
    
    if not success or not r6Asset then 
        warn("Falha ao carregar o modelo R6")
        return 
    end

    r6Asset.Parent = game.Workspace.Terrain
    local c = r6Asset:Clone()    
    c.Parent = game.Workspace    
    c:SetPrimaryPartCFrame(oldRoot.CFrame) 
    r6Asset:Destroy()    
    
    c.Name = player.Name .. "_R6"    
    local newHum = c:WaitForChild("Humanoid")
    
    -- 2. COPIAR SKIN
    local desc = oldHum:GetAppliedDescription()    
    newHum:ApplyDescription(desc)

    -- 3. AJUSTE DE TAMANHO E FÍSICA
    for _, v in ipairs(oldChar:GetDescendants()) do
        if v:IsA("BasePart") then
            v.CanCollide = false
            v.Massless = true
            if v.Name ~= "HumanoidRootPart" then
                v.Size = Vector3.new(0.1, 0.1, 0.1)
            end
        end
    end
    oldHum.PlatformStand = true

    -- 4. SISTEMA DE ANIMAÇÃO MANUAL
    local anims = {
        idle = "rbxassetid://180435571",
        walk = "rbxassetid://180426354",
        jump = "rbxassetid://125750702",
        fall = "rbxassetid://180436148"
    }
    
    local tracks = {}
    for name, id in pairs(anims) do
        local a = Instance.new("Animation")
        a.AnimationId = id
        tracks[name] = newHum:LoadAnimation(a)
    end

    local currentTrack = nil
    local function play(name)
        if currentTrack ~= tracks[name] then
            if currentTrack then currentTrack:Stop() end
            currentTrack = tracks[name]
            currentTrack:Play()
        end
    end

    local animLoop
    animLoop = game:GetService("RunService").RenderStepped:Connect(function()
        if not c.Parent then animLoop:Disconnect() return end
        
        if newHum.MoveDirection.Magnitude > 0 then
            play("walk")
            tracks.walk:AdjustSpeed(newHum.WalkSpeed / 16)
        else
            play("idle")
        end
        
        if newHum.FloorMaterial == Enum.Material.Air then
            if newHum.RootPart.Velocity.Y > 0 then play("jump") else play("fall") end
        end
    end)

    -- 5. MUDAR CONTROLE
    player.Character = c    
    game.Workspace.CurrentCamera.CameraSubject = newHum    

    -- 6. SINCRONIZAÇÃO SEGURA
    local syncConnection
    syncConnection = game:GetService("RunService").RenderStepped:Connect(function()    
        if c.Parent and oldChar.Parent then    
            -- Mantém o R15 invisível e na posição do R6 para o servidor não resetar
            oldRoot.CFrame = c.PrimaryPart.CFrame
            oldRoot.Velocity = Vector3.zero
            
            for _, part in ipairs(oldChar:GetDescendants()) do
                if part:IsA("BasePart") or part:IsA("Decal") then
                    part.LocalTransparencyModifier = 1
                end
            end
            
            -- Se morrer
            if newHum.Health <= 0 or oldHum.Health <= 0 then
                syncConnection:Disconnect()
                animLoop:Disconnect()
                oldHum.Health = 0
                task.wait(0.1)
                player.Character = oldChar
                c:Destroy()
            end
        else
            syncConnection:Disconnect()
            if animLoop then animLoop:Disconnect() end
        end
    end)
end

-- Conecta a função ao clique do botão
button.MouseButton1Click:Connect(oof)
