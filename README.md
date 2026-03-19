-- [[ GTL AFK SYSTEM: PLATFORM + SMART RESTORE 500 + LOOP SCREEN (60s Black / 5s Normal) ]] --
local Player = game:GetService("Players").LocalPlayer
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local Lighting = game:GetService("Lighting")

-- [ 1. CONFIGURATION ]
local PLATFORM_NAME = "GTL_STABLE_PLATFORM"
local STORAGE_NAME = "GTL_RESTORE_STORAGE"
local isAFKActive = false
local RANGE = 500 

local storage = game:GetService("ReplicatedStorage"):FindFirstChild(STORAGE_NAME) or Instance.new("Folder")
storage.Name = STORAGE_NAME
storage.Parent = game:GetService("ReplicatedStorage")

local savedData = {} 

local originalFog = {
    End = Lighting.FogEnd,
    Start = Lighting.FogStart,
    FarClip = Lighting.FarClipDistance
}

local function Notify(title, text)
    StarterGui:SetCore("SendNotification", {Title = title, Text = text, Duration = 3})
end

-- [ 2. PLATFORM SYSTEM ]
local function ManagePlatform(active)
    local existing = workspace:FindFirstChild(PLATFORM_NAME)
    if existing then existing:Destroy() end
    
    if active then
        local Char = Player.Character
        local Root = Char and Char:FindFirstChild("HumanoidRootPart")
        if Root then
            local Part = Instance.new("Part")
            Part.Name = PLATFORM_NAME
            Part.Size = Vector3.new(50, 1, 50)
            Part.Color = Color3.fromRGB(255, 255, 0)
            Part.Material = Enum.Material.Neon
            Part.CanCollide = true
            Part.Anchored = true
            Part.Parent = workspace
            Part.CFrame = Root.CFrame * CFrame.new(0, -3.5, 0)
        end
    end
end

-- [ 3. SMART HIDE & RESTORE SYSTEM ]
local function HandleEnvironment(active)
    local Char = Player.Character
    local Root = Char and Char:FindFirstChild("HumanoidRootPart")
    if not Root then return end

    if active then
        local currentPos = Root.Position
        for _, v in pairs(workspace:GetDescendants()) do
            if v:IsA("BasePart") and not v:IsDescendantOf(Char) and v.Name ~= PLATFORM_NAME then
                local model = v:FindFirstAncestorOfClass("Model")
                if not (model and model:FindFirstChildOfClass("Humanoid")) then
                    local dist = (v.Position - currentPos).Magnitude
                    if dist <= RANGE then
                        savedData[v] = v.Parent
                        v.Parent = storage 
                    end
                end
            end
        end

        task.delay(3, function()
            if isAFKActive then
                Lighting.FogEnd = 50 
                Lighting.FogStart = 10
                Lighting.FarClipDistance = 50
            end
        end)
    else
        for part, originalParent in pairs(savedData) do
            if part then
                part.Parent = originalParent
            end
        end
        table.clear(savedData)

        Lighting.FogEnd = originalFog.End
        Lighting.FogStart = originalFog.Start
        Lighting.FarClipDistance = originalFog.FarClip
    end
end

-- [ 4. GUI SYSTEM & LOOP LOGIC ]
local function BuildGUI()
    if Player.PlayerGui:FindFirstChild("GTL_AFK_GUI") then
        Player.PlayerGui["GTL_AFK_GUI"]:Destroy()
    end

    local sg = Instance.new("ScreenGui", Player:WaitForChild("PlayerGui"))
    sg.Name = "GTL_AFK_GUI"
    sg.ResetOnSpawn = false

    local BlackFrame = Instance.new("Frame", sg)
    BlackFrame.Size = UDim2.new(1, 0, 1, 0)
    BlackFrame.BackgroundColor3 = Color3.new(0, 0, 0)
    BlackFrame.BorderSizePixel = 0
    BlackFrame.Visible = false -- เริ่มต้นด้วยปิดไว้ก่อน
    BlackFrame.ZIndex = 999

    local AFKText = Instance.new("TextLabel", BlackFrame)
    AFKText.Size = UDim2.new(1, 0, 1, 0)
    AFKText.BackgroundTransparency = 1
    AFKText.TextColor3 = Color3.new(1, 1, 1)
    AFKText.Font = Enum.Font.GothamBold
    AFKText.TextSize = 30
    AFKText.Text = "AFK MODE ACTIVE\n(Screen will cycle 60s/5s)"

    local Btn = Instance.new("TextButton", sg)
    Btn.Size = UDim2.new(0, 160, 0, 50)
    Btn.Position = UDim2.new(0, 20, 0.5, -25)
    Btn.BackgroundColor3 = Color3.new(0.8, 0, 0)
    Btn.Text = "START AFK"
    Btn.Font = Enum.Font.GothamBold
    Btn.TextColor3 = Color3.new(1, 1, 1)
    Btn.ZIndex = 1000

    -- ระบบจัดการ Loop หน้าจอ
    task.spawn(function()
        while true do
            if isAFKActive then
                -- ช่วงที่ 1: ดำสนิท 1 นาที (60 วินาที)
                BlackFrame.Visible = true
                BlackFrame.BackgroundTransparency = 0
                AFKText.Visible = true
                
                local timer = 60
                while timer > 0 and isAFKActive do
                    AFKText.Text = "AFK MODE ACTIVE\nReturning to normal in: "..timer.."s"
                    task.wait(1)
                    timer = timer - 1
                end
                
                -- ช่วงที่ 2: จอปปกติ 5 วินาที
                if isAFKActive then
                    BlackFrame.BackgroundTransparency = 1 -- ทำให้โปร่งใสจนมองเห็นปกติ
                    AFKText.Visible = false
                    task.wait(5)
                end
            else
                BlackFrame.Visible = false
                task.wait(1)
            end
        end
    end)

    Btn.MouseButton1Click:Connect(function()
        isAFKActive = not isAFKActive
        Btn.Text = isAFKActive and "STOP AFK" or "START AFK"
        Btn.BackgroundColor3 = isAFKActive and Color3.new(0, 0.6, 0) or Color3.new(0.8, 0, 0)
        
        ManagePlatform(isAFKActive)
        HandleEnvironment(isAFKActive)
    end)
end

-- [ 5. AUTO-RUN & ANTI-AFK ]
task.spawn(function()
    while true do
        if not Player.PlayerGui:FindFirstChild("GTL_AFK_GUI") then BuildGUI() end
        task.wait(5)
    end
end)

Player.Idled:Connect(function()
    game:GetService("VirtualUser"):Button2Down(Vector2.new(0,0),workspace.CurrentCamera.CFrame)
    task.wait(1)
    game:GetService("VirtualUser"):Button2Up(Vector2.new(0,0),workspace.CurrentCamera.CFrame)
end)

Notify("GTL AFK LOOP", "ดำสนิท 1 นาที / ปกติ 5 วิ พร้อมใช้งาน!")
