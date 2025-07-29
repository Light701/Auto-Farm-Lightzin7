local player = game.Players.LocalPlayer
local data = game.ReplicatedStorage:WaitForChild("Datas")[player.UserId]
local events = game.ReplicatedStorage:WaitForChild("Package").Events

-- Gerar ID exclusivo por execução
local function gerarID()
    local random = Random.new()
    return string.format("%08x%04x", random:NextInteger(0, 0xffffffff), random:NextInteger(0, 0xffff))
end

getgenv().SessionID_ = gerarID()
getgenv()["farm_" .. getgenv().SessionID_] = false
local sessionVar = "farm_" .. getgenv().SessionID_
local guiName = "AutoFarmUI_" .. getgenv().SessionID_
local delayOffset = (#getgenv().SessionID_ % 5) * 0.01

-- Lista de ataques com requisitos mínimos
local attacks = {
    {name = "Vôlei de energia", min = 4000, type = "energy"},
    {name = "Godslicer", min = 60000000, type = "melee"},
    {name = "Super Dragon Fist", min = 50000000, type = "melee"},
    {name = "Mach Kick", min = 90000, type = "melee"},
    {name = "Wolf Fang", min = 2000, type = "melee"}
}

-- NPCs por mundo
local npcs = {
    Earth = {
        {name = "Kid Nohag", min = 0},
        {name = "Top X Fighter", min = 112500},
        {name = "Super Vegetable", min = 187500},
        {name = "Chilly", min = 550000},
        {name = "Perfect Atom", min = 875000},
        {name = "Kai-fist Master", min = 1625000},
        {name = "SSJB Wukong", min = 2000000},
        {name = "Broccoli", min = 12500000},
        {name = "SSJG Kakata", min = 37500000}
    },
    Bills = {
        {name = "Vegetable (GoD in-training)", min = 50000000},
        {name = "Wukong (Omen)", min = 75000000},
        {name = "Bills (50%)", min = 150000000},
        {name = "Vis (20%)", min = 250000000},
        {name = "Vegetable (LBSSJ4)", min = 450000000},
        {name = "Wukong (LBSSJ4)", min = 675000000},
        {name = "Vekuta (LBSSJ4)", min = 1050000000},
        {name = "Wukong Rose", min = 1250000000},
        {name = "Vekuta (SSJBUI)", min = 1375000000},
        {name = "Wukong (MUI)", min = 1875000000}
    }
}

-- Interface (GUI)
local gui = Instance.new("ScreenGui", game.CoreGui)
gui.Name = guiName

local frame = Instance.new("Frame", gui)
frame.Size = UDim2.new(0, 200, 0, 50)
frame.Position = UDim2.new(0.5, -100, 0.9, -25)
frame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
frame.Active = true
frame.Draggable = true
Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)

local button = Instance.new("TextButton", frame)
button.Size = UDim2.new(1, 0, 1, 0)
button.Text = "Auto Farm: OFF"
button.Font = Enum.Font.GothamBold
button.TextColor3 = Color3.fromRGB(255, 255, 255)
button.BackgroundColor3 = Color3.fromRGB(60, 0, 120)
button.TextSize = 18
Instance.new("UICorner", button).CornerRadius = UDim.new(0, 6)

button.MouseButton1Click:Connect(function()
    getgenv()[sessionVar] = not getgenv()[sessionVar]
    button.Text = getgenv()[sessionVar] and "Auto Farm: ON" or "Auto Farm: OFF"
end)

-- Detectar o mundo atual
local function getCurrentWorld()
    return workspace:FindFirstChild("PlanetName") and workspace.PlanetName.Value or "Earth"
end

-- Checar se player pode enfrentar o NPC
local function canEnter(min)
    return data.Strength.Value >= min
       and data.Speed.Value >= min
       and data.Defense.Value >= min
       and data.Energy.Value >= min
end

-- Seleciona o NPC mais forte possível
local function getBestNPC()
    local world = getCurrentWorld()
    local npcList = npcs[world]
    for i = #npcList, 1, -1 do
        if canEnter(npcList[i].min) then
            return npcList[i].name
        end
    end
    return nil
end

-- Teleportar e interagir com Vis (Bills Planet)
local function tryTeleportBills()
    local zeni = data.Zeni.Value
    if getCurrentWorld() == "Earth" and data.Strength.Value >= 3e8 and zeni >= 15000 then
        local vis = workspace:FindFirstChild("Others").NPCs:FindFirstChild("Vis")
        if vis and vis:FindFirstChild("HumanoidRootPart") then
            player.Character.HumanoidRootPart.CFrame = vis.HumanoidRootPart.CFrame * CFrame.new(0, 0, -3)
            task.wait(1 + delayOffset)
            events.Qaction:InvokeServer(vis)
            task.wait(1 + delayOffset)
            local ui = player.PlayerGui:FindFirstChild("Dialogue")
            if ui then
                for _, v in ipairs(ui:GetDescendants()) do
                    if v:IsA("TextButton") and (v.Text:lower():find("yes") or v.Text:lower():find("go")) then
                        v:Click()
                        break
                    end
                end
            end
        end
    end
end

-- Aceitar Missão
local function aceitarMissao(npcName)
    local npc = workspace:FindFirstChild("Others").NPCs:FindFirstChild(npcName)
    if npc and npc:FindFirstChild("HumanoidRootPart") then
        player.Character.HumanoidRootPart.CFrame = npc.HumanoidRootPart.CFrame * CFrame.new(0, 0, -3)
        task.wait(0.4 + delayOffset)
        events.Qaction:InvokeServer(npc)
    end
end

-- Atacar Boss
local function atacarBoss(nome)
    for _, boss in ipairs(workspace.Living:GetChildren()) do
        if boss.Name == nome and boss:FindFirstChild("Humanoid") and boss.Humanoid.Health > 0 then
            local hrp = boss:FindFirstChild("HumanoidRootPart")
            if not hrp then return end

            player.Character.HumanoidRootPart.CFrame = hrp.CFrame * CFrame.new(0, 0, 2)
            task.wait(0.2 + delayOffset)

            while boss and boss:FindFirstChild("Humanoid") and boss.Humanoid.Health > 0 and getgenv()[sessionVar] do
                for _, atk in ipairs(attacks) do
                    if atk.type == "melee" and data.Strength.Value >= atk.min then
                        events.damage:FireServer(atk.name, "Blacknwhite27")
                        task.wait(0.13 + delayOffset)
                    elseif atk.type == "energy" and data.Energy.Value >= atk.min then
                        events.voleys:InvokeServer(atk.name, "Blacknwhite27")
                        task.wait(0.13 + delayOffset)
                    end
                end
                task.wait(0.25 + delayOffset)
            end
        end
    end
end

-- Loop principal
task.spawn(function()
    while true do
        task.wait(0.6 + delayOffset)
        if getgenv()[sessionVar] then
            pcall(function()
                tryTeleportBills()
                local quest = data.Quest.Value
                if quest == "" then
                    local npcName = getBestNPC()
                    if npcName then aceitarMissao(npcName) end
                else
                    atacarBoss(quest)
                end
            end)
        end
    end
end)

