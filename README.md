-- Players.paulotnl3.PlayerGui.ScreenGui.Frame.r6.LocalScript
local function C_e()
    local script = LMG2L["LocalScript_e"];
    local button = script.Parent
    
    local function oof()    
        local player = game.Players.LocalPlayer    
        local oldChar = player.Character    
        if not oldChar or oldChar.Name:find("_R6") then return end 
        
        local oldHum = oldChar:FindFirstChildOfClass("Humanoid")    
        local oldRoot = oldChar:FindFirstChild("HumanoidRootPart")        
        
        -- 1. CRIAR O MODELO R6
        local r6Asset = game:GetObjects("rbxassetid://1561389244")[1]    
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

        -- 3. AJUSTE DE TAMANHO E FÍSICA (PARA NÃO MORRER NO VOID)
        for _, v in ipairs(oldChar:GetDescendants()) do
            if v:IsA("BasePart") then
                v.CanCollide = false
                v.Massless = true
                -- Se o jogo permitir, deixa o R15 minúsculo
                if v.Name ~= "HumanoidRootPart" then
                    v.Size = Vector3.new(0.1, 0.1, 0.1)
                end
            end
        end
        oldHum.PlatformStand = true

        -- 4. SISTEMA DE ANIMAÇÃO MANUAL (MESMA LÓGICA QUE FUNCIONOU)
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

        local animLoop = game:GetService("RunService").RenderStepped:Connect(function()
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
    
        -- 6. SINCRONIZAÇÃO SEGURA (R15 DENTRO DO R6)
        local syncConnection
        syncConnection = game:GetService("RunService").RenderStepped:Connect(function()    
            if c.Parent and oldChar.Parent then    
                -- Agora o R15 fica EXATAMENTE na mesma posição do R6
                -- Mas como tiramos a colisão e o tamanho, ele não dá fling
                oldRoot.CFrame = c.PrimaryPart.CFrame
                oldRoot.Velocity = Vector3.new(0,0,0)
                
                for _, part in ipairs(oldChar:GetDescendants()) do
                    if part:IsA("BasePart") or part:IsA("Decal") then
                        part.LocalTransparencyModifier = 1
                    end
                end
                
                -- Se o R6 realmente morrer (dano de player ou reset)
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
                animLoop:Disconnect()
            end
        end)
    end

    button.MouseButton1Click:Connect(oof)
end;
task.spawn(C_e);
