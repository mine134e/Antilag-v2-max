# Antilag-v2-max
Credit by @PepsiMannumero1
--[[
Antilag Total: remove efeitos visuais (partÃ­culas, luzes, smoke, sparkles, trails, fire etc)
em tudo: mapa, personagens, NPCs, acessÃ³rios, ferramentas e objetos de missÃ£o.
SÃ³ NÃƒO remove: GUIs/ScreenGuis/SurfaceGui/BillboardGui/Decals/SelectionBox/Handles/BoxHandleAdornment
NotificaÃ§Ã£o sÃ³ na primeira execuÃ§Ã£o. 100% LocalScript seguro (StarterPlayerScripts)
]]

local Players = game:GetService("Players")
local Lighting = game:GetService("Lighting")
local Workspace = game:GetService("Workspace")

local FPS_TRIGGER = 65
local COOLDOWN = 5

local running = false
local lastRun = 0
local notificouAntilag = false

local function isVisualProtected(obj)
    -- SÃ³ protege Gui, Decal, adornos de interface (nÃ£o efeitos de partes)
    if obj:IsA("BillboardGui") or obj:IsA("SurfaceGui") or obj:IsA("ScreenGui")
    or obj:IsA("SelectionBox") or obj:IsA("Handles") or obj:IsA("BoxHandleAdornment")
    or obj:IsA("Decal") or obj:IsA("UIComponent") or obj:IsA("Highlight") then
        return true
    end
    return false
end

local function processarLotes(lista, func, lote)
    lote = lote or 800
    for i = 1, #lista, lote do
        for j = i, math.min(i + lote - 1, #lista) do
            func(lista[j])
        end
        task.wait(0.05)
    end
end

local function try(fn)
    local ok, _ = pcall(fn)
    return ok
end

local function mostrarNotificacao(texto, tempo)
    tempo = tempo or 3
    local player = Players.LocalPlayer
    if not player then return end
    local playerGui = player:FindFirstChildOfClass("PlayerGui")
    if not playerGui then return end
    local existente = playerGui:FindFirstChild("AntilagNotificacao")
    if existente then existente:Destroy() end
    local gui = Instance.new("ScreenGui")
    gui.Name = "AntilagNotificacao"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 270, 0, 54)
    frame.Position = UDim2.new(1, -280, 1, -120)
    frame.AnchorPoint = Vector2.new(0, 0)
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    frame.BackgroundTransparency = 0.18
    frame.BorderSizePixel = 0
    frame.Parent = gui
    local textoLabel = Instance.new("TextLabel")
    textoLabel.Size = UDim2.new(1, -20, 1, -20)
    textoLabel.Position = UDim2.new(0, 10, 0, 10)
    textoLabel.BackgroundTransparency = 1
    textoLabel.Text = texto
    textoLabel.TextColor3 = Color3.new(1,1,1)
    textoLabel.TextScaled = true
    textoLabel.Font = Enum.Font.GothamBold
    textoLabel.TextStrokeTransparency = 0.5
    textoLabel.Parent = frame
    gui.Parent = playerGui
    task.spawn(function()
        wait(tempo)
        for i = 0, 1, 0.08 do
            frame.BackgroundTransparency = 0.18 + i * 0.82
            textoLabel.TextTransparency = i
            textoLabel.TextStrokeTransparency = 0.5 + (i * 0.5)
            task.wait(0.021)
        end
        gui:Destroy()
    end)
end

local function simplificarTudo()
    if running then return end
    running = true

    local objetos = Workspace:GetDescendants()
    processarLotes(objetos, function(obj)
        if isVisualProtected(obj) then return end

        if obj:IsA("BasePart") and obj.Material ~= Enum.Material.SmoothPlastic then
            try(function() obj.Material = Enum.Material.SmoothPlastic end)
        end

        if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Smoke")
        or obj:IsA("Sparkles") or obj:IsA("Fire") then
            if obj.Enabled ~= false then
                try(function() obj.Enabled = false end)
            end
        end
        if obj:IsA("Explosion") and obj.Visible ~= false then
            try(function() obj.Visible = false end)
        end
        if (obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight")) then
            if obj.Enabled ~= false then
                try(function() obj.Enabled = false end)
            end
            if obj.Brightness and obj.Brightness ~= 0 then
                try(function() obj.Brightness = 0 end)
            end
        end
    end, 800)

    objetos = nil

    for _, v in ipairs(Lighting:GetChildren()) do
        if v:IsA("Sky") or v:IsA("Atmosphere") or v:IsA("Clouds")
        or v:IsA("BloomEffect") or v:IsA("SunRaysEffect")
        or v:IsA("ColorCorrectionEffect") or v:IsA("DepthOfFieldEffect")
        or v:IsA("BlurEffect") then
            try(function() v:Destroy() end)
        end
    end

    try(function()
        Lighting.FogEnd = 1e10
        Lighting.FogStart = 1e10
        Lighting.Brightness = 1
        Lighting.GlobalShadows = false
        if Lighting:FindFirstChild("EnvironmentDiffuseScale") then
            Lighting.EnvironmentDiffuseScale = 0
        end
        if Lighting:FindFirstChild("EnvironmentSpecularScale") then
            Lighting.EnvironmentSpecularScale = 0
        end
    end)

    local Terrain = Workspace:FindFirstChildOfClass("Terrain")
    if Terrain then
        try(function()
            Terrain.WaterReflectance = 0
            Terrain.WaterTransparency = 1
            Terrain.WaterWaveSize = 0
            Terrain.WaterWaveSpeed = 0
            Terrain.Decoration = false
        end)
    end

    if not notificouAntilag then
        notificouAntilag = true
        mostrarNotificacao("Antilag ativado! (Efeitos removidos de tudo)", 3)
    end

    running = false
end

task.spawn(function()
    while true do
        if not running then
            local fps = 60
            pcall(function()
                if Workspace.GetRealPhysicsFPS then
                    fps = Workspace:GetRealPhysicsFPS()
                end
            end)
            if fps <= FPS_TRIGGER and (tick() - lastRun > COOLDOWN) then
                simplificarTudo()
                lastRun = tick()
            end
        end
        task.wait(0.5)
    end
end)
